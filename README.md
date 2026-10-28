# fileshare-gitops

Manifesty Kubernetes dla fileshare-app. Klaster synchronizuje się z tym 
repozytorium przez ArgoCD, więc zmiany wdraża się commitem, a nie `kubectl apply`.

- `apps/` – aplikacje ArgoCD. Czyta je root app założony przez fileshare-infra.
- `k8s/` – manifesty aplikacji (Kustomize), wdrażane do namespace `fileshare`.

Trzeba ustawić ID konta AWS w `k8s/backend-sa.yaml` i `k8s/config.yaml` 
(`aws sts get-caller-identity --query Account --output text`) oraz adres
repozytorium w `apps/fileshare.yaml`. Nazwy roli i bucketu są stałe, więc `tofu destroy`
i ponowny `apply` niczego tu nie zmieniają.

Podgląd tego, co trafi do klastra:

```sh
kubectl kustomize k8s
```
