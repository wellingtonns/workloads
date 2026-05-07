# Kustomize Share

Repositorio compartilhado de manifests Kustomize para deploy Kubernetes.

## Estrutura

```text
kustomize-share/
  base/
    deployment.yaml
    service.yaml
    ingress.yaml
    kustomization.yaml
  app/
    app-clinica/
      kustomization.yaml
      namespace.yaml
      deployment-patch.yaml
      ingress-patch.yaml
    example-frontend/
    example-backend/
```

## Uso

Cada aplicacao deve ter um overlay em `app/<nome-da-aplicacao>` reutilizando a base:

```yaml
resources:
  - namespace.yaml
  - ../../base
```

A tag da imagem deve ser atualizada pela pipeline usando `newTag`:

```yaml
images:
  - name: app-image
    newName: ghcr.io/wellingtonns/app-clinica
    newTag: latest
```

O deploy pode ser feito por GitOps, como Argo CD ou Flux, apontando para o overlay da aplicacao, ou diretamente pela pipeline com:

```bash
kubectl apply -k app/app-clinica
```
