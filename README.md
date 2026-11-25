# Instalação: OpenShift AI 3.5 + Run:ai + RHCL + MaaS nos nós RTX

Passo a passo direto para instalar a plataforma de IA nos **2 nós Supermicro com 10× NVIDIA RTX 4500 Ada cada** e publicar o **Qwen3.5-35B-A3B** pelo MaaS.

| Entra | Não entra (outra equipe) |
| --- | --- |
| GPU Operator (conferir), User Workload Monitoring, cert-manager, LWS | Nó H200, OpenShift Virtualization, vGPU e VMs |
| NVIDIA Run:ai 3.0 (control plane + cluster, mesmo OpenShift) | FindFace |
| Red Hat Connectivity Link (RHCL) | Modelos que rodam nas VMs (só a publicação no MaaS, opcional, Passo 14) |
| OpenShift AI 3.5 (KServe/llm-d) + MaaS | |
| Qwen3.5 em 4 réplicas TP4 (16 RTX) | |

**Versões:** OpenShift 4.20 a 4.22 · OpenShift AI 3.5 · Run:ai 3.0.x (Support-Entitled) · NVIDIA GPU Operator 26.7.1+ · RHCL 1.4.x · PostgreSQL 16

**Material de referência:** [arquitetura completa com o nó H200 virtualizado](6.%20arquitetura-h200-virtualizado.md) · [documento anterior, com explicações e riscos](5.%20arquitetura-anterior-h200-em-pods.md) · [operação (Parte 3)](5.%20arquitetura-anterior-h200-em-pods.md#parte-3--operação)

## Índice

- [Pré-requisitos](#pré-requisitos)
- [1. Verificações iniciais](#1-verificações-iniciais)
- [2. GPU Operator](#2-gpu-operator)
- [3. User Workload Monitoring](#3-user-workload-monitoring)
- [4. cert-manager e LeaderWorkerSet](#4-cert-manager-e-leaderworkerset)
- [5. NVIDIA Run:ai](#5-nvidia-runai)
- [6. Red Hat Connectivity Link](#6-red-hat-connectivity-link)
- [7. OpenShift AI](#7-openshift-ai)
- [8. Gateways, Routes e MaaS](#8-gateways-routes-e-maas)
- [9. Nós RTX, namespace e cotas](#9-nós-rtx-namespace-e-cotas)
- [10. Qwen3.5 com 1 réplica](#10-qwen35-com-1-réplica)
- [11. Teste de carga e 4 réplicas](#11-teste-de-carga-e-4-réplicas)
- [12. Publicar no MaaS](#12-publicar-no-maas)
- [13. Aplicações: API keys e teste](#13-aplicações-api-keys-e-teste)
- [14. (Opcional) Modelos das VMs no MaaS](#14-opcional-modelos-das-vms-no-maas)
- [Verificação final](#verificação-final)
- [Problemas comuns](#problemas-comuns)

## Pré-requisitos

- [ ] `cluster-admin`, `oc` e **Helm 3.14+**.
- [ ] Storage class padrão com provisionamento dinâmico.
- [ ] **PostgreSQL 16** acessível pelo cluster, com dois bancos: um para o MaaS e um para o Run:ai.
- [ ] **F5** já balanceando `*.apps` para os routers do OpenShift (sem MetalLB).
- [ ] **Três nós sem GPU** para os serviços do Run:ai (mínimo somado: \~20 vCPU e \~32 GB).
- [ ] **NGC Personal Key** (`nvapi-...`) para as imagens do Run:ai Support-Entitled.
- [ ] Operadores pelo console em **Ecosystem → Software Catalog**. No OpenShift 4.21+, desabilite o catálogo OLMv1 (Technology Preview) antes, senão aparece a página de *ClusterExtensions*.

| Passo | O que faz |
| --- | --- |
| 1 a 4 | Verificações e pré-requisitos do cluster |
| 5 | Run:ai |
| 6 a 8 | RHCL, OpenShift AI, gateways e MaaS |
| 9 a 11 | Nós RTX, cotas e o Qwen3.5 |
| 12 a 14 | MaaS: modelos, acessos e aplicações |

## 1. Verificações iniciais

```bash
oc login --token=<token> --server=<api_url>
oc get clusterversion                      # 4.20 a 4.22
oc get storageclass                        # existe uma (default)
oc get nodes -L nvidia.com/gpu.product     # anote os 2 nós RTX e 3 nós sem GPU para o Run:ai
```

## 2. GPU Operator

O nó H200 está em modo VM, configurado pela equipe de virtualização, **e a `ClusterPolicy` é única no cluster**. Não recrie nem edite os campos de `sandboxWorkloads`, `vgpuManager` e `vgpuDeviceManager`.

```bash
oc get csv -n nvidia-gpu-operator | grep -i gpu-operator      # versão 26.7.1 ou superior
oc get clusterpolicy                                          # gpu-cluster-policy
oc get nodes -L nvidia.com/gpu.product -L nvidia.com/gpu.count
# os 2 nós RTX: NVIDIA-RTX-4500-Ada-Generation, 10 GPUs cada
```

- **GPU Operator já instalado** (caso esperado): confira se a versão é 26.7.1 ou superior. Se for menor, combine o upgrade com a equipe de virtualização.
- **GPU Operator não instalado:** instale o **Node Feature Discovery** (com a `NodeFeatureDiscovery` padrão) e o **NVIDIA GPU Operator** no canal `v26.7`. Antes de criar a `ClusterPolicy`, alinhe com a equipe de virtualização, porque o nó H200 precisa das opções de VM.

Label de versão exigido pelo Run:ai (repita a cada upgrade do GPU Operator):

```bash
GPUOP=$(oc get csv -n nvidia-gpu-operator -o jsonpath='{.items[?(@.spec.displayName=="NVIDIA GPU Operator")].spec.version}')
oc label clusterpolicy gpu-cluster-policy app.kubernetes.io/version=v${GPUOP} --overwrite
```

## 3. User Workload Monitoring

Se o ConfigMap já existir, só acrescente a chave `enableUserWorkload: true`.

```bash
oc apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF

oc get pods -n openshift-user-workload-monitoring     # prometheus-user-workload-* Running
```

## 4. cert-manager e LeaderWorkerSet

1. Instale o **cert-manager Operator for Red Hat OpenShift**.
2. Instale o **Red Hat build of Leader Worker Set** (namespace `openshift-lws-operator`).
3. Crie a instância (sem ela o CRD não é criado):

```bash
oc apply -f - <<EOF
apiVersion: operator.openshift.io/v1
kind: LeaderWorkerSetOperator
metadata:
  name: cluster
spec:
  managementState: Managed
  logLevel: Normal
  operatorLogLevel: Normal
EOF

oc get crd leaderworkersets.leaderworkerset.x-k8s.io     # existe
```

## 5. NVIDIA Run:ai

### 5.1 Preparação

```bash
oc label node <no-sistema-1> <no-sistema-2> <no-sistema-3> node-role.kubernetes.io/runai-system=true

oc new-project runai-backend
oc new-project runai
oc label ns runai pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/audit=privileged pod-security.kubernetes.io/warn=privileged --overwrite

oc create secret docker-registry runai-reg-creds -n runai-backend \
  --docker-server=https://nvcr.io --docker-username='$oauthtoken' \
  --docker-password=<NGC_PERSONAL_KEY> --docker-email=<EMAIL>
```

Banco do Run:ai no PostgreSQL 16:

```sql
CREATE ROLE runai WITH LOGIN PASSWORD '<SENHA_RUNAI_DB>';
CREATE DATABASE runai OWNER runai;
GRANT ALL PRIVILEGES ON DATABASE runai TO runai;
```

**Só se o certificado do router (`*.apps`) for de CA interna:** crie o bundle com a CA do router **e** a CA do PostgreSQL, e acrescente `--set global.customCA.enabled=true` nos dois `helm upgrade` (5.2 e 5.3).

```bash
oc -n runai-backend create secret generic runai-ca-cert --from-file=runai-ca.pem=<ca_bundle.pem>
oc -n runai create secret generic runai-ca-cert --from-file=runai-ca.pem=<ca_bundle.pem>
oc -n openshift-monitoring create secret generic runai-ca-cert --from-file=runai-ca.pem=<ca_bundle.pem>
oc label secret runai-ca-cert -n runai run.ai/cluster-wide=true run.ai/name=runai-ca-cert --overwrite
```

### 5.2 Control plane

`runai-db.yaml`:

```yaml
postgresql:
  enabled: false
global:
  postgresql:
    auth:
      host: <PG_HOST>
      port: 5432
      database: runai
      username: runai
      password: <SENHA_RUNAI_DB>
      postgresPassword: <SENHA_RUNAI_DB>   # obrigatório com banco externo; repita a senha do usuário runai
      dbSslMode: require
```

```bash
helm repo add runai https://helm.ngc.nvidia.com/nvidia/runai-oss --force-update
helm search repo runai/control-plane --versions | head      # use a última 3.0.x

DOMAIN=$(oc get ingresses.config cluster -o jsonpath='{.spec.domain}')

helm upgrade -i runai-backend -n runai-backend runai/control-plane \
  --version "<VERSAO_3.0.x>" \
  --set global.domain=runai.${DOMAIN} \
  --set global.config.kubernetesDistribution=openshift \
  --set tenantsManager.config.adminUsername=<EMAIL_ADMIN> \
  --set tenantsManager.config.adminPassword="<SENHA_ADMIN>" \
  --set supportEntitled=true \
  --set global.imagePullSecrets[0].name=runai-reg-creds \
  --set telemetryService.enabled=true \
  -f runai-db.yaml

oc get pods -n runai-backend        # todos Running (~5 min); nenhum pod de postgres
```

- Senha do admin: 8+ caracteres, com maiúscula, minúscula, número e símbolo.
- **Cluster de teste sem licença:** não crie o `runai-reg-creds` e tire as três linhas `supportEntitled`, `imagePullSecrets` e `telemetryService`.

### 5.3 Cluster

1. Abra `https://runai.<dominio-apps>`, entre com o admin e troque a senha.
2. No assistente, dê um nome ao cluster (ex.: `ocp-ia`) e escolha **Same as the control plane**.
3. Rode o comando gerado (o token vale 2 horas), acrescentando as duas últimas linhas:

```bash
helm upgrade -i runai-cluster runai/runai-cluster -n runai \
  --set controlPlane.url=<do-assistente> \
  --set controlPlane.bootstrapToken=<do-assistente> \
  --set cluster.uid=<do-assistente> \
  --set cluster.url=<do-assistente> \
  --version="<VERSAO_3.0.x>" \
  --set clusterConfig.global.ingress.ingressClass=haproxy \
  --set global.fipsMode=off \
  --set clusterConfig.global.nvidiaDcgmExporter.namespace=nvidia-gpu-operator \
  --set clusterConfig.lws.namespace=openshift-lws-operator

oc get pods -n runai                  # Running
oc get runaiconfig runai -n runai     # AVAILABLE=True
```

O assistente deve mostrar **Cluster connected**. Se mostrar *Missing prerequisites*, quase sempre é o GPU Operator abaixo de 26.7.1 ou sem o label de versão (Passo 2).

### 5.4 Configuração geral

Na UI, em **General settings**, ligue **Limit projects from exceeding department quota**.

## 6. Red Hat Connectivity Link

1. Instale o **Red Hat Connectivity Link**, canal 1.4.x, em *All namespaces*.
2. Crie a GatewayClass, o Kuadrant, o TLS do Authorino e o timeout de autenticação:

```bash
oc apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: openshift-default
spec:
  controllerName: openshift.io/gateway-controller/v1
EOF
oc get gatewayclass openshift-default        # ACCEPTED=True

oc create namespace kuadrant-system
oc apply -f - <<EOF
apiVersion: kuadrant.io/v1beta1
kind: Kuadrant
metadata:
  name: kuadrant
  namespace: kuadrant-system
EOF
oc wait Kuadrant -n kuadrant-system kuadrant --for=condition=Ready --timeout=10m

# TLS no Authorino
oc annotate svc/authorino-authorino-authorization -n kuadrant-system \
  service.beta.openshift.io/serving-cert-secret-name=authorino-server-cert --overwrite
oc get secret authorino-server-cert -n kuadrant-system          # aguarde existir

oc apply -f - <<EOF
apiVersion: operator.authorino.kuadrant.io/v1beta1
kind: Authorino
metadata:
  name: authorino
  namespace: kuadrant-system
spec:
  replicas: 1
  clusterWide: true
  listener:
    tls:
      enabled: true
      certSecretRef:
        name: authorino-server-cert
  oidcServer:
    tls:
      enabled: false
EOF
oc -n kuadrant-system set env deployment/authorino \
  SSL_CERT_FILE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt \
  REQUESTS_CA_BUNDLE=/etc/ssl/certs/openshift-service-ca/service-ca-bundle.crt

# Timeout de 2 s na autenticação (known issue RHOAIENG-71638: HTTP 500/503 sob carga)
oc patch subscription rhcl-operator -n openshift-operators --type json \
  -p '[{"op":"add","path":"/spec/config","value":{"env":[{"name":"AUTH_SERVICE_TIMEOUT","value":"2s"}]}}]'
```

- Se o operador foi instalado antes da GatewayClass: `oc delete pod -n openshift-operators -l app=kuadrant`.
- Se a Subscription já tiver `spec.config`, use `"path":"/spec/config/env"`.

## 7. OpenShift AI

1. Instale o **Red Hat OpenShift AI**, canal `stable-3.5`. Não crie o DataScienceCluster pelo formulário.
2. Crie o DataScienceCluster. O Kueue fica `Removed` porque quem faz cota e agendamento é o Run:ai:

```bash
oc apply -f - <<EOF
apiVersion: datasciencecluster.opendatahub.io/v2
kind: DataScienceCluster
metadata:
  name: default-dsc
spec:
  components:
    dashboard:        { managementState: Managed }
    kserve:           { managementState: Managed }
    kueue:            { managementState: Removed }
    aigateway:
      managementState: Removed          # Managed no Passo 8.5
      modelsAsAService: { managementState: Removed }
    ogx:              { managementState: Removed }
    aipipelines:      { managementState: Removed }
    feastoperator:    { managementState: Removed }
    mlflowoperator:   { managementState: Removed }
    modelregistry:    { managementState: Removed }
    ray:              { managementState: Removed }
    trainer:          { managementState: Removed }
    trainingoperator: { managementState: Removed }
    trustyai:         { managementState: Removed }
    workbenches:      { managementState: Removed }
EOF
oc get datasciencecluster default-dsc -w      # Phase: Ready

oc delete configmap migration-gpu-status -n redhat-ods-applications --ignore-not-found
oc rollout restart deployment/rhods-dashboard -n redhat-ods-applications
oc patch odhdashboardconfig odh-dashboard-config -n redhat-ods-applications --type merge \
  -p '{"spec":{"dashboardConfig":{"disableKueue":true}}}'
```

## 8. Gateways, Routes e MaaS

### 8.1 Certificado e domínio

```bash
DOMAIN=$(oc get ingresses.config cluster -o jsonpath='{.spec.domain}')
CERT_SECRET=$(oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.spec.defaultCertificate.name}')
CERT_SECRET=${CERT_SECRET:-router-certs-default}
echo $DOMAIN $CERT_SECRET
```

### 8.2 Banco do MaaS

```bash
oc create namespace redhat-ai-gateway-infra
oc label ns redhat-ai-gateway-infra maas-gateway-access=true
oc create secret generic maas-db-config -n redhat-ai-gateway-infra \
  --from-literal=DB_CONNECTION_URL='postgresql://<usuario>:<senha>@<PG_HOST>:5432/<banco_maas>?sslmode=require'
```

### 8.3 Gateway do MaaS

```bash
oc apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: maas-default-gateway
  namespace: openshift-ingress
  annotations:
    opendatahub.io/managed: "false"
    security.opendatahub.io/authorino-tls-bootstrap: "true"
    networking.istio.io/service-type: ClusterIP
spec:
  gatewayClassName: openshift-default
  listeners:
  - name: https
    hostname: "maas.${DOMAIN}"
    port: 443
    protocol: HTTPS
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            maas-gateway-access: "true"
    tls:
      mode: Terminate
      certificateRefs:
      - name: ${CERT_SECRET}
EOF
oc get gateway maas-default-gateway -n openshift-ingress      # PROGRAMMED=True
```

Todo acesso aos modelos passa pelo MaaS, por isso o gateway `openshift-ai-inference` do llm-d não é criado.

### 8.4 Route pelo F5

```bash
oc get route -n openshift-ingress -l gateway.networking.k8s.io/gateway-name=maas-default-gateway   # se já existir, pule
oc get svc -n openshift-ingress -l gateway.networking.k8s.io/gateway-name=maas-default-gateway \
  -o custom-columns=NOME:.metadata.name,PORTAS:.spec.ports[*].name
```

```bash
oc apply -f - <<EOF
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: maas-default-gateway
  namespace: openshift-ingress
  annotations:
    haproxy.router.openshift.io/timeout-tunnel: 1h
spec:
  host: maas.${DOMAIN}
  to: { kind: Service, name: <service-do-gateway>, weight: 100 }
  port: { targetPort: <porta-443-do-service> }
  tls: { termination: passthrough }
  wildcardPolicy: None
EOF

curl -sk -o /dev/null -w '%{http_code}\n' https://maas.${DOMAIN}/v1/models   # 404 agora; 401 depois do 8.5
```

### 8.5 Ativar o MaaS

```bash
oc patch datasciencecluster default-dsc --type merge -p '{"spec":{"components":{
  "aigateway":{"managementState":"Managed","modelsAsAService":{"managementState":"Managed"}},
  "ogx":{"managementState":"Managed"}}}}'

oc patch odhdashboardconfig odh-dashboard-config -n redhat-ods-applications --type merge \
  -p '{"spec":{"dashboardConfig":{"modelAsService":true,"genAiStudio":true}}}'

oc get pods -n redhat-ai-gateway-infra                                                   # maas-api Running
oc get maastenantconfigs.maas.opendatahub.io default-tenant -n models-as-a-service       # READY=True
oc get envoyfilter kuadrant-maas-default-gateway -n openshift-ingress -o json | grep -o 'timeout[^,]*'   # 2s
```

## 9. Nós RTX, namespace e cotas

### 9.1 Taint e namespace

```bash
oc adm taint nodes <no-rtx-1> <no-rtx-2> nvidia.com/gpu=rtx4500ada:NoSchedule --overwrite
oc get pods -n nvidia-gpu-operator -o wide | grep -E '<no-rtx-1>|<no-rtx-2>'   # continuam Running

oc apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: llm-qwen35
  labels: { tipo: gpu-workload, maas-gateway-access: "true" }
---
apiVersion: v1
kind: ResourceQuota
metadata: { name: gpu-quota, namespace: llm-qwen35 }
spec: { hard: { requests.nvidia.com/gpu: "16" } }
EOF
```

### 9.2 Run:ai: node pool, departamento e projeto

Crie nesta ordem pela UI.

| Onde | O que criar |
| --- | --- |
| **Resources → Node pools** | `rtx4500-ada`: label `nvidia.com/gpu.product` = `NVIDIA-RTX-4500-Ada-Generation`; placement Bin-pack / Bin-pack. Confira: 2 nós, 20 GPUs |
| **Organization → Departments** | `plataforma-ia`: `rtx4500-ada` com cota 20 e max 20; `default` com 0; over-quota desligado |
| **Organization → Projects** | `qwen35` no departamento `plataforma-ia`, namespace existente `llm-qwen35`. Node pools **na ordem `rtx4500-ada`, `default`** (o `default` é para o EPP do llm-d, que não usa GPU). Cota 16 e Max GPU allocation 16 em `rtx4500-ada`; 0 no `default`; over-quota desligado |

```bash
oc get ns llm-qwen35 -L runai/queue      # runai/queue=qwen35 (o Run:ai aplica sozinho)
```

## 10. Qwen3.5 com 1 réplica

O modelo vem do **ModelCar validado pela Red Hat**: imagem OCI com os pesos, que cada nó baixa uma vez só.

```bash
oc apply -f - <<EOF
apiVersion: serving.kserve.io/v1alpha2
kind: LLMInferenceService
metadata:
  name: qwen35
  namespace: llm-qwen35
  labels:
    runai/queue: qwen35
    priorityClassName: very-high
    kai.scheduler/preemptibility: non-preemptible
spec:
  replicas: 1
  model:
    uri: oci://registry.redhat.io/rhai/modelcar-qwen3-5-35b-a3b-fp8-dynamic:3.0
    name: Qwen/Qwen3.5-35B-A3B-FP8            # nome que os clientes usam no campo "model"
  router:
    scheduler: {}
    route: {}
    gateway:
      refs:
      - name: maas-default-gateway
        namespace: openshift-ingress
  template:
    nodeSelector:
      nvidia.com/gpu.product: NVIDIA-RTX-4500-Ada-Generation
    tolerations:
    - { key: nvidia.com/gpu, operator: Equal, value: rtx4500ada, effect: NoSchedule }
    topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels: { app.kubernetes.io/name: qwen35, kserve.io/component: workload }
    containers:
    - name: main
      env:
      - name: VLLM_ADDITIONAL_ARGS
        value: "--tensor-parallel-size 4 --gpu-memory-utilization 0.90 --enable-prefix-caching --kv-cache-dtype fp8"
      resources:
        requests: { nvidia.com/gpu: "4", cpu: "16", memory: 96Gi }
        limits:   { nvidia.com/gpu: "4", cpu: "32", memory: 128Gi }
EOF
```

Validação:

```bash
oc get llmisvc qwen35 -n llm-qwen35                 # READY=True
oc get pods -n llm-qwen35 -o custom-columns=POD:.metadata.name,SCHED:.spec.schedulerName,NODE:.spec.nodeName
# vLLM num nó RTX e EPP (router-scheduler) num nó sem GPU, os dois com runai-scheduler
oc get podgroups -n llm-qwen35 -o custom-columns=NAME:.metadata.name,MIN:.spec.minMember,PREEMPT:.spec.preemptibility,PRIO:.spec.priorityClassName
# minMember 1, non-preemptible, very-high
```

Na UI do Run:ai, o projeto `qwen35` deve mostrar 4 de 16 GPUs.

- Não use `schedulerName` nem `security.opendatahub.io/enable-auth: 'false'`.
- O ModelCar foi validado pela Red Hat em H200. Nas RTX 4500 Ada, confira no log do vLLM que o modelo carregou e quanto sobrou de KV cache (`oc logs -n llm-qwen35 <pod> -c main | grep -i 'kv cache'`).

## 11. Teste de carga e 4 réplicas

1. Meça com 1 réplica os **tokens/min sustentados dentro do SLO** (TTFT e tempo entre tokens no p95). Compare TP4 com TEP4 (acrescente `--enable-expert-parallel`) e fique com o mais rápido.
2. Capacidade do modelo = tokens/min por réplica × 4. Esse número define os limites do Passo 12.
3. Suba para 4 réplicas (2 por nó; sobram 2 RTX livres em cada):

```bash
oc patch llminferenceservice qwen35 -n llm-qwen35 --type merge -p '{"spec":{"replicas":4}}'
oc get pods -n llm-qwen35 -o wide
oc describe resourcequota gpu-quota -n llm-qwen35       # Used 16 / Hard 16
```

## 12. Publicar no MaaS

O nome do `MaaSModelRef` tem de ser **igual** ao do `LLMInferenceService` (nomes diferentes dão 403).

Os limites abaixo são exemplos (capacidade de 4M tokens/min, um usuário técnico por ambiente). Regra: prd com até \~90% da capacidade e hml com até \~10%. **O limite vale por usuário:** com N usuários no ambiente, divida por N.

```bash
oc apply -f - <<EOF
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSModelRef
metadata: { name: qwen35, namespace: llm-qwen35 }
spec: { modelRef: { kind: LLMInferenceService, name: qwen35 } }
---
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSSubscription
metadata: { name: apps-prd, namespace: models-as-a-service }
spec:
  priority: 20
  owner: { groups: [ { name: grp-apps-prd } ], users: [] }
  modelRefs:
  - name: qwen35
    namespace: llm-qwen35
    tokenRateLimits: [ { limit: 3600000, window: 1m } ]
---
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSSubscription
metadata: { name: apps-hml, namespace: models-as-a-service }
spec:
  priority: 10
  owner: { groups: [ { name: grp-apps-hml } ], users: [] }
  modelRefs:
  - name: qwen35
    namespace: llm-qwen35
    tokenRateLimits: [ { limit: 400000, window: 1m } ]
---
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSAuthPolicy
metadata: { name: apps-prd, namespace: models-as-a-service }
spec:
  subjects: { groups: [ { name: grp-apps-prd } ] }
  modelRefs: [ { name: qwen35, namespace: llm-qwen35 } ]
---
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSAuthPolicy
metadata: { name: apps-hml, namespace: models-as-a-service }
spec:
  subjects: { groups: [ { name: grp-apps-hml } ] }
  modelRefs: [ { name: qwen35, namespace: llm-qwen35 } ]
EOF

oc get maasmodelref -n llm-qwen35        # Ready (fica Pending até existirem subscription e policy)
```

- `priority` tem de ser diferente em cada subscription.
- Subscription e policy andam juntas: só a policy dá 429; só a subscription dá 403.
- Os grupos `grp-apps-prd` e `grp-apps-hml` são grupos do OpenShift ou do OIDC.

## 13. Aplicações: API keys e teste

Cada aplicação (`br-<app>-prd`, `br-<app>-hml`) usa uma **ServiceAccount** como usuário técnico:

```bash
NS=br-minhaapp-prd; SUB=apps-prd
oc create sa svc-app -n $NS
# Acrescente "- name: system:serviceaccounts:$NS" em owner.groups da MaaSSubscription $SUB
# e em subjects.groups da MaaSAuthPolicy $SUB (Passo 12)

TOKEN=$(oc create token svc-app -n $NS --duration=10m)
curl -sk -X POST https://maas.${DOMAIN}/maas-api/v1/api-keys \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d "{\"name\":\"$NS\",\"subscription\":\"$SUB\",\"expiresIn\":\"720h\"}"
# resposta: {"key":"sk-oai-...", ...}

oc create secret generic maas-api-key -n $NS --from-literal=API_KEY=<sk-oai-...>
```

Teste:

```bash
KEY=<sk-oai-...>
curl -sk https://maas.${DOMAIN}/v1/models -H "Authorization: Bearer $KEY"
curl -sk https://maas.${DOMAIN}/v1/chat/completions -H "Authorization: Bearer $KEY" -H 'Content-Type: application/json' \
  -d '{"model":"Qwen/Qwen3.5-35B-A3B-FP8","messages":[{"role":"user","content":"oi"}],"max_tokens":20}'
```

Esperado: 401 sem key; 200 com key; 429 quando a key do hml passa do limite, sem afetar as outras keys.

Prazo máximo das keys (opcional):

```bash
oc patch maastenantconfigs.maas.opendatahub.io default-tenant -n models-as-a-service --type merge \
  -p '{"spec":{"apiKeys":{"maxExpirationDays":90}}}'
```

Recomendado: node selector nos projetos `br-*`, para nenhum pod de aplicação cair nos nós de GPU:

```bash
oc label node <nos-de-aplicacao> ai.empresa/tipo-no=app
oc annotate namespace br-minhaapp-prd openshift.io/node-selector=ai.empresa/tipo-no=app --overwrite
```

## 14. (Opcional) Modelos das VMs no MaaS

Para o MaaS controlar acesso e tokens também dos modelos que a outra equipe roda nas VMs. Recurso **Technology Preview** no OpenShift AI 3.5 (`ExternalModel`); só chat completions (`/v1/chat/completions`) tem limite de tokens.

**Peça à equipe das VMs**, por modelo:
- o FQDN e a porta do vLLM (ex.: `flashnext.llms-vm.svc.cluster.local:8000`);
- a chave do vLLM (`--api-key`);
- o `--served-model-name`;
- que a NetworkPolicy deles libere a entrada a partir de `openshift-ingress`.

```bash
oc create namespace modelos-vm
oc label ns modelos-vm maas-gateway-access=true opendatahub.io/dashboard=true

oc create secret generic vllm-flashnext-key -n modelos-vm --from-literal=api-key=<chave-do-vllm>
oc label secret vllm-flashnext-key -n modelos-vm inference.llm-d.ai/ipp-managed=true

oc apply -f - <<EOF
apiVersion: inference.opendatahub.io/v1alpha1
kind: ExternalProvider
metadata: { name: vm-flashnext, namespace: modelos-vm }
spec:
  provider: openai
  endpoint: flashnext.llms-vm.svc.cluster.local      # só o FQDN, sem http:// e sem porta
  auth: { type: apikey, secretRef: { name: vllm-flashnext-key } }
---
apiVersion: inference.opendatahub.io/v1alpha1
kind: ExternalModel
metadata:
  name: flashnext                                    # nome que os clientes usam em "model"
  namespace: modelos-vm
  annotations:
    maas.opendatahub.io/tls: "false"
    maas.opendatahub.io/port: "8000"
spec:
  externalProviderRefs:
  - ref: { name: vm-flashnext }
    targetModel: flashnext                           # --served-model-name na VM
    apiFormat: openai-chat
    path: /v1/chat/completions
    weight: 100
---
apiVersion: maas.opendatahub.io/v1alpha1
kind: MaaSModelRef
metadata: { name: flashnext, namespace: modelos-vm }
spec: { modelRef: { kind: ExternalModel, name: flashnext } }
EOF

oc get externalprovider,externalmodel -n modelos-vm        # Ready
```

Depois, acrescente `{ name: flashnext, namespace: modelos-vm }` (com `tokenRateLimits` na subscription) nas subscriptions e policies do Passo 12, e teste com `"model":"flashnext"`. Repita para cada modelo da VM. A transcrição (`/v1/audio/*`) não passa pelo MaaS 3.5 (o gateway devolve 400); a alternativa está em [6. arquitetura-h200-virtualizado.md, Passo D.6](6.%20arquitetura-h200-virtualizado.md#d6-opcional-transcrição-pelo-rhcl).

## Verificação final

- [ ] Operadores `Succeeded`; `default-dsc` `Ready`; Kueue `Removed`.
- [ ] GPU Operator 26.7.1+ com o label `app.kubernetes.io/version`; CRD `LeaderWorkerSet` presente.
- [ ] Run:ai: cluster **Connected**; node pool `rtx4500-ada` com 2 nós e 20 GPUs; departamento 20; projeto `qwen35` 16/16, com `default` em segundo.
- [ ] `llm-qwen35`: 4 réplicas `Running` (2 por nó RTX), EPP num nó sem GPU, todos com `runai-scheduler`.
- [ ] MaaS: tenant `Ready`, `MaaSModelRef` `Ready`, timeout do Authorino em 2 s.
- [ ] `https://maas.<dominio>` pelo F5: 401 sem key, 200 com key, 429 no hml acima do limite.
- [ ] Nenhum pod de aplicação `br-*` nos nós RTX.

## Problemas comuns

| Sintoma | Causa | Solução |
| --- | --- | --- |
| Run:ai *Missing prerequisites* | GPU Operator < 26.7.1 ou sem o label de versão | Passo 2 |
| Pods do control plane presos em `Init` ("Waiting for <PG_HOST>") | PostgreSQL inacessível ou CA do PostgreSQL fora do `runai-ca-cert` | 5.1 |
| `secret-generator` falha com `postgres-password` | Falta `postgresPassword` no `runai-db.yaml` | 5.2 |
| EPP (`router-scheduler`) `Pending` | Projeto sem o pool `default` em segundo | 9.2; depois `oc delete podgroup` e o pod do EPP |
| Pod `Pending` com `OverLimit` | Cota do projeto esgotada ou pool errado | 9.2 |
| Pod com `default-scheduler` | Namespace sem `runai/queue` | Refazer a associação do projeto (9.2) e recriar o pod |
| MaaS 403 | `MaaSModelRef` com nome diferente do modelo, ou sem `MaaSAuthPolicy` | Passo 12 |
| MaaS 429 sem passar do limite | Policy sem subscription | Passo 12 |
| MaaS 500/503 sob carga | Timeout do Authorino em 200 ms | Passo 6 (`AUTH_SERVICE_TIMEOUT=2s`) |
| 503 "Application is not available" | Route errada (do router, não do gateway) | 8.4 |
| Tenant do MaaS `Degraded` | UWM, `maas-db-config` ou Authorino | `oc logs -n redhat-ods-applications deploy/maas-controller --tail=50` |
