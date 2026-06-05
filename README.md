# sapari-helm-manifest

Homelab GitOps 소스. **2-tier App of Apps** 구조로 ArgoCD가 이 repo를 읽어 클러스터
전체를 선언적으로 배포·동기화한다. 사람이 하는 건 **최초 1회 루트 App 적용 + Vault unseal**
뿐이고, 이후엔 **git push = 배포**다.

- Repo: `https://gitlab.com/sagopanda/sapari-helm-manifest.git`
- ArgoCD 설치 네임스페이스: `cicd`  (모든 `Application` 오브젝트가 여기 산다)

```
root-app  (kubectl apply 1회)
  └─ apps/*.yaml         ← 카테고리 "부모" App만 (recurse:false)
       ├─ platform        (wave 0)  namespaces · rbac · network-policies · storage-classes
       ├─ operators       (wave 1)  longhorn · external-secrets · vault · cloudnative-pg · strimzi · eck · minio-operator
       ├─ secrets         (wave 2)  cluster-secret-store · ExternalSecret 모음
       ├─ data            (wave 3)  minio · postgresql · mongodb · redis · kafka · elasticsearch
       ├─ observability   (wave 3)  kube-prometheus-stack · loki · alloy · tempo · kibana
       └─ networking      (wave 4)  gateway · httproutes
            └─ 각 부모 App → apps/<category>/*.yaml (leaf App) → 실제 워크로드
```

## 레이아웃

| 경로 | 내용 |
|---|---|
| `bootstrap/root-app.yaml` | App of Apps 루트. `apps/*.yaml`(카테고리 부모)만 읽음(recurse:false). |
| `bootstrap/argocd-values.yaml` | ArgoCD 자체 Helm values. 부트스트랩 시 수동 설치용(repo 추적 X). |
| `apps/<category>.yaml` | 카테고리 **부모** App = ArgoCD 콘솔의 그룹 노드. `apps/<category>/`를 recurse. |
| `apps/<category>/<name>.yaml` | **leaf** App (포인터). Helm 차트 / git 매니페스트를 가리킴. sync-wave 없음. |
| `components/<category>/<name>/` | 실제 배포물 — Helm `values.yaml` · CR · 매니페스트. leaf App이 참조. |

- **무엇을(`apps/`) ↔ 어떻게 생겼나(`components/`)** 를 분리.
- leaf App에는 `sync-wave` 매직넘버를 **넣지 않는다.** 순서는 카테고리 **부모 App의 wave**로만.
- 분류/그룹핑은 **폴더 + `labels.category`** 로.

## Helm leaf 패턴 (차트 + git values)

차트는 Helm repo에서, values는 이 repo에서 가져오는 멀티소스. `ref: values` 가 repo
커밋을 추적하므로 **values 한 줄만 고쳐 push해도** 자동 재렌더 → sync 된다.

```yaml
sources:
  - repoURL: https://grafana.github.io/helm-charts
    chart: loki
    targetRevision: 7.0.0
    helm:
      valueFiles:
        - $values/components/observability/loki/values.yaml
  - repoURL: https://gitlab.com/sagopanda/sapari-helm-manifest.git
    targetRevision: main
    ref: values
```

## 차트 버전 (참고)

| 컴포넌트 | Helm repo | chart | version | ns |
|---|---|---|---|---|
| longhorn | charts.longhorn.io | longhorn | 1.11.2 | longhorn-system |
| external-secrets | charts.external-secrets.io | external-secrets | 2.5.0 (installCRDs=true) | external-secrets |
| vault | helm.releases.hashicorp.com | vault | 0.32.0 | security |
| cloudnative-pg | cloudnative-pg.github.io/charts | cloudnative-pg | 0.28.2 | cnpg-system |
| strimzi | strimzi.io/charts | strimzi-kafka-operator | 1.0.0 | storage |
| eck | helm.elastic.co | eck-operator | 3.4.0 | elastic-system |
| minio-operator | operator.min.io | operator | 7.1.1 | storage |
| minio (tenant) | operator.min.io | tenant | 7.1.1 | storage |
| mongodb | charts.bitnami.com/bitnami | mongodb | 19.0.3 | storage |
| redis | charts.bitnami.com/bitnami | redis | 25.5.3 | storage |
| kube-prometheus-stack | prometheus-community.github.io/helm-charts | kube-prometheus-stack | 86.0.1 | monitoring |
| loki | grafana.github.io/helm-charts | loki | 7.0.0 | monitoring |
| alloy | grafana.github.io/helm-charts | alloy | 1.8.1 | monitoring |
| tempo | grafana.github.io/helm-charts | tempo | 1.24.4 | monitoring |

매니페스트형(CR) leaf: postgresql(CNPG), kafka(Strimzi 4.1.2), elasticsearch/kibana(ECK 8.17.0),
gateway/httproutes(Gateway API), namespaces/rbac/network-policies/storage-classes/secrets.

## 부트스트랩 (콜드스타트 순서)

전제: 노드, k8s, CNI, **envoy-gateway + MetalLB**(192.168.123.200)는 이미 떠 있어야 한다
(이들은 GitOps 범위 밖).

1. **ArgoCD 설치** (chicken-and-egg — root App을 돌리려면 ArgoCD가 먼저 있어야 함)
   ```bash
   helm repo add argo https://argoproj.github.io/argo-helm && helm repo update argo
   helm install argocd argo/argo-cd -n cicd --create-namespace \
     --version 9.5.16 -f bootstrap/argocd-values.yaml
   ```
2. **루트 App 적용** (최초 1회)
   ```bash
   kubectl apply -f bootstrap/root-app.yaml
   ```
3. **Vault unseal (수동)** — 콜드스타트 시 Vault는 sealed 상태. unseal 전까지는
   `secrets`(wave 2) 가 Healthy가 안 되어 그 위 wave(data/observability/networking)가
   대기한다. unseal 하면 ExternalSecret이 채워지며 자동으로 진행된다.
   ```bash
   kubectl -n security exec -it vault-0 -- vault operator unseal   # (×3, 키 조각)
   ```
4. 이후엔 `git push` 만 → ArgoCD가 wave 순서로 자동 반영.

> **GitLab private repo 자격**: `secrets`의 `gitlab-repo` ExternalSecret이
> `argocd.argoproj.io/secret-type=repository` 라벨 시크릿을 만든다. Vault
> `secret/cicd/gitlab` 에 `username` + `token`(Deploy Token, scope read_repository) 필요.
> public repo면 불필요.

## 설계 노트 — 원본(gitops-structure.html)에서 바로잡은 점

1. **repoURL** 을 전부 실제 repo(`sagopanda/sapari-helm-manifest.git`)로 치환.
   (원본의 `<group>/homelab.git`, gitlab-repo ExternalSecret의 `url` 포함)
2. **순서 보강** — 부모 App에 `sync-wave`(0~4)를 부여해 의존 순서
   (platform→operators→secrets→data·observability→networking)를 강제. leaf는 그대로 깨끗.
   원본의 "엄격히 원하면 부모 App에만 순서" 옵션을 채택.
3. **`cluster-secret-store` 를 `platform`이 아닌 `secrets`로 이동** — ClusterSecretStore는
   ESO CRD(operators)와 Vault가 떠 있어야 동작하므로 platform(wave 0)에 두면 wave 역전.
   secrets(wave 2)에 배치해 의존 순서를 맞춤.
4. **`namespaces` 를 GitOps 화** — 원본의 `kubectl create ns` 수동 절차를 매니페스트로.
   netpol이 참조하는 `cnpg-system` · `elastic-system` 도 포함(없으면 wave 충돌/CreateNamespace
   소유권 분쟁). k8s 1.21+ 자동 라벨 `kubernetes.io/metadata.name` 덕에 수동 라벨링은 제거.
5. **`default-deny` 를 ns별 매니페스트로 펼침** — 원본은 `for ns` 루프 + namespace 없는
   템플릿이라 GitOps에서 대상이 모호. storage/monitoring/security/cicd/external-secrets
   각각에 명시적으로 적용.
6. **`storage-classes` 를 platform에 추가** — Longhorn StorageClass(longhorn-prod/single)는
   클러스터 토대라 platform으로. (원본 platform 목록에 누락)
7. **values 경로 정합** — 모든 Helm leaf의 `$values/...` 를 실제 `components/<cat>/<name>/values.yaml`
   위치와 일치시킴(불일치 시 ComparisonError).
8. **`ServerSideApply=true`** 를 모든 App에 추가 — kube-prometheus-stack 등 대형 CRD의
   client-side apply 어노테이션 한도 초과 방지.
