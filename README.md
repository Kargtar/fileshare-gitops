# fileshare-gitops

Manifesty Kubernetes dla fileshare-app. Klaster synchronizuje się z tym 
repozytorium przez ArgoCD, więc zmiany wdraża się commitem, a nie `kubectl apply`.

- `apps/` – aplikacje ArgoCD. Czyta je root app założony przez fileshare-infra.
- `k8s/` – manifesty aplikacji (Kustomize), wdrażane do namespace `fileshare`.

Hasło do Valkey nie jest w repozytorium. Po każdym `tofu apply`, kiedy ArgoCD założy już
namespace, trzeba utworzyć Secret ręcznie:

```sh
kubectl -n fileshare create secret generic valkey --from-literal=password=$(openssl rand -hex 16)
```

Tagi obrazów w `k8s/kustomization.yaml` podbija CI z fileshare-app po każdym pushu na `main`.

Podgląd tego, co trafi do klastra:

```sh
kubectl kustomize k8s
```
