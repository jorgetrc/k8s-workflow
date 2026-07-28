# k8s-workflow

Manifestos Kubernetes (Kustomize) da aplicação [image_collor](https://github.com/jorgetrc/image_collor).

## Fluxo

1. Um push em `image_collor` dispara a pipeline (`.github/workflows/docker-publish.yml`), que builda a imagem, gera uma nova tag semver e faz o push para o Docker Hub.
2. A mesma pipeline clona este repositório e atualiza `base/kustomization.yml` com a nova tag da imagem.
3. O ArgoCD, instalado no cluster e monitorando o diretório `base/` deste repositório, detecta a mudança e sincroniza automaticamente (`selfHeal` + `prune`) o Deployment/Service no namespace `image-collor`.

## Estrutura

- `base/` — manifestos "vivos", atualizados automaticamente pela pipeline do `image_collor`. É o diretório rastreado pelo ArgoCD.
- `overlays/hml/` — overlay de homologação, hoje fixo em uma tag antiga; não recebe updates automáticos da pipeline.
- `argocd/application.yml` — definição do `Application` do ArgoCD que aponta para `base/`.

## Como replicar o laboratório do zero

Pré-requisitos: `kind`, `kubectl`, `docker`.

```bash
# 1. Subir o cluster kind (usa a config em ~/kind/kind-cluster.yml, ajuste o caminho se necessário)
kind create cluster --config kind-cluster.yml

# 2. Instalar o ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available --timeout=180s deployment/argocd-server -n argocd
kubectl wait --for=condition=available --timeout=180s deployment/argocd-repo-server -n argocd

# 3. Bootstrap da aplicação (cria o namespace image-collor e sincroniza tudo automaticamente)
kubectl apply -f argocd/application.yml

# 4. (Opcional) Acessar a UI do ArgoCD
kubectl port-forward -n argocd svc/argocd-server 8080:443
# usuário: admin / senha:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d

# 5. (Opcional) Acessar a aplicação diretamente
kubectl port-forward -n image-collor svc/image-collor-service 8081:80
```

Após o passo 3, o ArgoCD assume o gerenciamento contínuo: qualquer alteração feita via pipeline em `base/kustomization.yml` é sincronizada automaticamente, sem necessidade de nenhum `kubectl apply` manual adicional.
