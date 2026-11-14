# fileshare-gitops

Manifesty Kubernetes dla fileshare-app. Klaster synchronizuje się z tym repozytorium przez
ArgoCD, więc zmiany wdraża się commitem, a nie `kubectl apply`.

## Zawartość

| Plik | Co tworzy |
|---|---|
| `apps/fileshare.yaml` | Aplikację ArgoCD `fileshare`, która wdraża `k8s/` do namespace `fileshare` |
| `k8s/kustomization.yaml` | Listę manifestów, namespace i tagi obrazów |
| `k8s/namespace.yaml` | Namespace `fileshare` |
| `k8s/backend-sa.yaml` | ServiceAccount `backend` z rolą IAM (IRSA) do bucketu |
| `k8s/config.yaml` | ConfigMap backendu: bucket, region, adres Valkey, limit rozmiaru pliku |
| `k8s/backend.yaml` | Deployment, Service i HPA backendu (2–5 podów, CPU 70%) |
| `k8s/frontend.yaml` | Deployment (2 pody) i Service frontendu |
| `k8s/valkey.yaml` | Deployment (1 pod) i Service Valkey z hasłem z Secretu |
| `k8s/ingress.yaml` | Ingress: `/api` do backendu, `/` do frontendu |

## Jak to działa

Root app z fileshare-infra czyta katalog `apps/` i zakłada aplikację `fileshare`. Ta pilnuje,
żeby zawartość namespace `fileshare` odpowiadała `k8s/` na gałęzi `main`. Dzięki `prune` usuwa
zasoby skasowane z repozytorium, a dzięki `selfHeal` cofa ręczne zmiany w klastrze.

Nowa wersja aplikacji trafia na klaster tak:

1. Push do fileshare-app uruchamia CI, które buduje obrazy z tagiem = SHA commita.
2. CI zmienia `newTag` w `k8s/kustomization.yaml` (`kustomize edit set image`) i robi commit
   tutaj. Deploymenty mają w `image` tylko nazwę (`backend`, `frontend`), a Kustomize podmienia
   ją na pełny obraz z tagiem.
3. ArgoCD widzi nowy commit (sprawdza co ok. 3 minuty) i robi rolling update.

Ingress z `ingressClassName: alb` obsługuje AWS Load Balancer Controller i tworzy z niego
publiczny ALB. `target-type: ip` znaczy, że ALB wysyła ruch prosto do podów, a nie przez węzły.
Health check ALB dla backendu idzie na `/healthz` (adnotacja w Service), bo `/` w backendzie
zwraca 404.

Backend ma dwie sondy. Liveness na `/healthz` restartuje zawieszony proces. Readiness na
`/ready` zdejmuje pod z ruchu, gdy nie ma połączenia z Valkey. HPA korzysta z metrics-server
zainstalowanego przez fileshare-infra i liczy wykorzystanie CPU względem `requests`, dlatego
każdy kontener ma ustawione `requests` i limit pamięci.

## Decyzje projektowe

- **Valkey bez trwałego wolumenu.** Metadane i tak wygasają, a dysk wymagałby sterownika EBS CSI
  i StatefulSetu. Ceną jest to, że restart poda Valkey unieważnia wszystkie aktywne linki.
- **Backend bez `replicas`.** Liczbą podów steruje HPA. Gdyby `replicas` było w manifeście,
  ArgoCD (`selfHeal`) cofałby każdą zmianę zrobioną przez HPA.
- **Hasło do Valkey poza repozytorium.** Repozytorium jest publiczne, więc Secret tworzy się ręcznie.
- **HTTP zamiast HTTPS.** Certyfikat z ACM wymaga własnej domeny. Pliki i tak idą do S3 po HTTPS,
  ale linki z API przechodzą po HTTP. Na produkcji: domena, certyfikat w ACM i adnotacje
  `listen-ports` oraz `ssl-redirect` w Ingressie.

## Po każdym `tofu apply`

Kiedy ArgoCD założy namespace, trzeba utworzyć Secret z hasłem do Valkey. Do tego czasu pody
backendu i Valkey czekają w stanie `CreateContainerConfigError`.

```sh
kubectl -n fileshare create secret generic valkey --from-literal=password=$(openssl rand -hex 16)
```

## Sprawdzenie

```sh
kubectl kustomize k8s                        # podgląd tego, co trafi do klastra
kubectl -n argocd get applications           # root i fileshare: Synced, Healthy
kubectl -n fileshare get pods,hpa,ingress    # adres ALB w kolumnie ADDRESS
```

Przed `tofu destroy` trzeba najpierw usunąć root app, żeby LB controller zdążył skasować ALB.
Dokładna procedura jest w README fileshare-infra.
