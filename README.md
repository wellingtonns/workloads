# Workloads

Repositorio central de manifests Kubernetes com Kustomize.

Este repositorio guarda a base compartilhada e os overlays por aplicacao. Ele e atualizado pela pipeline dos projetos de aplicacao, publicada em `wellingtonns/templates-pipeline`.

## Visao geral

O fluxo completo funciona assim:

1. Um repositorio de aplicacao, como `wellingtonns/app-clinica`, roda a workflow `App Clinica Pipeline`.
2. Essa workflow chama `wellingtonns/templates-pipeline/.github/workflows/app-pipeline.yml@main`.
3. A pipeline valida o tipo da stack (`frontend` ou `backend`).
4. A pipeline faz o build da imagem Docker com Nixpacks.
5. A imagem e publicada no registry configurado, por exemplo `ghcr.io/wellingtonns/app-clinica:<tag>`.
6. A pipeline garante que exista uma estrutura Kustomize para a aplicacao neste repositorio.
7. Quando a atualizacao de imagem esta habilitada, a pipeline altera o `newTag` do overlay usando `update-kustomize-image.yml`.
8. O deploy pode ser feito por GitOps, como Argo CD ou Flux, observando este repositorio.

## Estrutura

```text
workloads/
  base/
    kustomization.yaml
    rollout.yaml
    hpa.yaml
    ingress.yaml
    service.yaml
    service-canary.yaml
    serviceaccount.yaml
    registry-secret.yaml
    components/
      app-common/
        kustomization.yaml
  workload-template/
    base/
      kustomization.yaml
    overlays/
      dev/
        kustomization.yaml
        configmap.yaml
        externalsecret.yaml
        patches/
      prd/
        kustomization.yaml
        configmap.yaml
        externalsecret.yaml
        patches/
  app/
    app-clinica/
      base/
        kustomization.yaml
      overlays/
        dev/
          kustomization.yaml
          configmap.yaml
          externalsecret.yaml
          patches/
        prd/
          kustomization.yaml
          configmap.yaml
          externalsecret.yaml
          patches/
```

## Base compartilhada

A pasta `base/` contem os recursos Kubernetes comuns a todas as aplicacoes:

- `Rollout` do Argo Rollouts para estrategia canary.
- `HorizontalPodAutoscaler` para escala automatica.
- `Ingress` com NGINX, ExternalDNS e cert-manager.
- `Service` estavel e `Service` canary.
- `ServiceAccount`.
- `registry-secret.yaml`, usado como `imagePullSecrets`.
- Componente `base/components/app-common`, que renomeia recursos e ajusta campos usando replacements do Kustomize.

A base usa nomes genericos como `app`, `app-envs`, `app-secrets` e `app-canary`. O componente `app-common` troca esses nomes pelo nome real da aplicacao usando o `ConfigMap` `service-name` criado no `base` de cada workload.

## Template de workload

A pasta `workload-template/` e o modelo usado pela pipeline para criar uma nova aplicacao automaticamente.

Quando `ensure_kustomize: true`, o workflow `validate-kustomize.yml` clona este repositorio e verifica o caminho configurado em `kustomize_path`, por exemplo:

```yaml
kustomize_path: app/app-clinica
```

Se a pasta ainda nao existir, a pipeline copia `workload-template/` para `app/app-clinica` e troca `{{APP_NAME}}` pelo valor de `app_name`.

Branch da aplicacao define o overlay:

- `dev` usa `overlays/dev`.
- `main` ou `master` usa `overlays/prd`.

## Estrutura de uma aplicacao

Cada aplicacao tem um `base` proprio e overlays por ambiente.

Exemplo:

```text
app/app-clinica/
  base/
    kustomization.yaml
  overlays/
    dev/
      kustomization.yaml
      configmap.yaml
      externalsecret.yaml
      patches/
        rollout.yaml
        hpa.yaml
        ingress.yaml
    prd/
      kustomization.yaml
      configmap.yaml
      externalsecret.yaml
      patches/
        rollout.yaml
        hpa.yaml
        ingress.yaml
```

O `base` da aplicacao referencia a base compartilhada:

```yaml
resources:
  - ../../../base
components:
  - ../../../base/components/app-common
```

Ele tambem cria o `ConfigMap` `service-name`:

```yaml
configMapGenerator:
  - name: service-name
    literals:
      - SERVICE_NAME=app-clinica
      - APP_PORT=8080
```

Esse `SERVICE_NAME` e usado para renomear Rollout, Services, HPA, Ingress, ServiceAccount e referencias internas.

## Overlays dev e prd

Cada overlay define:

- `namespace` da aplicacao.
- Recursos especificos do ambiente, como `configmap.yaml` e `externalsecret.yaml`.
- Imagem e tag que serao usadas pelo Kubernetes.
- Patches para recursos, limites, HPA e Ingress.

Exemplo simplificado:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: app-clinica
resources:
  - ../../base
  - externalsecret.yaml
  - configmap.yaml
images:
  - name: app
    newName: ghcr.io/wellingtonns/app-clinica
    newTag: 448416e56d15
patches:
  - target:
      kind: Rollout
      name: app-clinica
    path: patches/rollout.yaml
```

## Imagens

A imagem final usada pelo cluster vem do bloco `images` do overlay:

```yaml
images:
  - name: app
    newName: ghcr.io/wellingtonns/app-clinica
    newTag: 448416e56d15
```

No exemplo acima, a imagem completa e:

```text
ghcr.io/wellingtonns/app-clinica:448416e56d15
```

Se a aplicacao usar `update-kustomize-image.yml`, o input `image_alias` precisa bater com o campo `images[].name`. No template atual deste repositorio, o alias e:

```yaml
image_alias: app
```

## Secrets e configuracoes

Cada overlay pode ter:

- `configmap.yaml`: variaveis nao sensiveis.
- `externalsecret.yaml`: integracao com External Secrets Operator.

Exemplo:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-clinica-secrets
spec:
  secretStoreRef:
    name: gcp-secret-store
    kind: ClusterSecretStore
  dataFrom:
    - extract:
        key: app-clinica
```

O Rollout base consome:

```yaml
envFrom:
  - configMapRef:
      name: app-envs
      optional: true
  - secretRef:
      name: app-secrets
      optional: true
```

Depois dos replacements, esses nomes passam a seguir o nome da aplicacao, por exemplo `app-clinica-envs` e `app-clinica-secrets`.

## Como a pipeline escreve neste repositorio

O repositorio da aplicacao deve chamar o template principal assim:

```yaml
jobs:
  pipeline:
    uses: wellingtonns/templates-pipeline/.github/workflows/app-pipeline.yml@main
    with:
      app_name: app-clinica
      stack_type: frontend
      context: .
      image_name: ghcr.io/wellingtonns/app-clinica
      kustomize_repository: wellingtonns/workloads
      kustomize_checkout_path: workloads
      kustomize_path: app/app-clinica
      image_alias: app
      ensure_kustomize: true
      update_kustomize: true
      deploy: false
    secrets: inherit
```

Para escrever neste repositorio a partir de outro repositorio, configure no repositorio da aplicacao um secret chamado:

```text
KUSTOMIZE_REPO_TOKEN
```

Esse token precisa ter permissao `Contents: Read and write` neste repositorio.

## Validacao local

Para renderizar um overlay localmente:

```bash
kustomize build app/app-clinica/overlays/dev
```

Para aplicar diretamente em um cluster:

```bash
kubectl apply -k app/app-clinica/overlays/dev
```

Em producao, o caminho recomendado e GitOps, por exemplo Argo CD observando:

```text
app/app-clinica/overlays/dev
app/app-clinica/overlays/prd
```

## Cuidados

- Nao commite secrets em texto puro.
- Use `ExternalSecret` para credenciais.
- Garanta que `image_alias` no workflow seja igual ao `name` no bloco `images`.
- Garanta que o cluster tenha permissao para puxar imagens do registry usado.
- Antes de alterar a base compartilhada, valide pelo menos um overlay `dev` e um overlay `prd`.
