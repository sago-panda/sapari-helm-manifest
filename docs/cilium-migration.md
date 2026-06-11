# Calico → Cilium 전체 전환 러너북

홈랩 네트워크 스택을 **Calico 기반 → Cilium 단일 스택**으로 순차 전환한다.
CNI뿐 아니라 kube-proxy / LoadBalancer(MetalLB) / Gateway(Envoy Gateway)까지 전부 Cilium으로 흡수.

> 이 문서의 클러스터 명령(`helm`, `kubectl apply` 등)은 **운영자가 직접 실행**한다.
> repo 매니페스트 변경은 phase 3·4에서만 발생하며, **해당 phase 컷오버 시점에 맞춰 push**한다
> (ArgoCD `gateway` App이 `selfHeal:true`라 미리 push하면 즉시 적용되어 순서가 깨짐).

---

## 0. 현재 상태 / 목표 / 결정사항

### 현재 스택 (실측)
| 영역 | 현재 |
|---|---|
| 노드 | 2대 — `kube-master` 192.168.123.105 (control-plane 단일), `kube-worker` 192.168.123.103 (재구축으로 .107→.103). 같은 `/24` L2. 커널 6.12, containerd 2.2.x, k8s 1.35 |
| CNI | Calico (구식 manifest 설치: `kube-system/calico-node` DS, `calico-config` CM, BIRD 백엔드, **IPIP Always** 오버레이, `calico-ipam`) |
| Pod CIDR | `172.16.0.0/16` (calico-ipam, `node.spec.podCIDR` 비어있음) |
| Service CIDR | `10.96.0.0/12` |
| kube-proxy | 있음 (`kube-system/kube-proxy` DS, iptables) |
| LoadBalancer | MetalLB L2, pool `192.168.123.200-210` (`homelab-pool` / `homelab-l2`) |
| Gateway | Envoy Gateway — GatewayClass `eg`, Gateway `public-gateway`(.201) / `admin-gateway`(.202), HTTPRoute 7개 |
| NetworkPolicy | 네이티브 `networking.k8s.io/v1`만 (Calico CRD 0개) |

### 결정사항
- **컷오버: per-node 롤링** (Cilium 공식 마이그레이션, 거의 무중단)
- **데이터패스: native routing** (오버레이 없음, `autoDirectNodeRoutes`, 같은 L2이므로 최적)
- 새 Pod CIDR `10.244.0.0/16` (현 172.16/16·service 10.96/12와 비충돌, 롤링 마이그레이션은 임시 별도 CIDR 필수)

### 절대 불변 조건
- **`.201`/`.202` LoadBalancer IP는 전 과정에서 유지.** `.105` 호스트의 Cloudflare Tunnel이 이 두 IP로 라우팅하므로 바뀌면 외부 접속(`*.ascode.click`) 끊김.
- Longhorn PV(데이터)는 보존. Pod는 재스케줄되지만 볼륨은 살아있음.
- NetworkPolicy 6개는 네이티브 → Cilium이 그대로 시행. **수정 불필요.**

### 버전 주의
- Cilium은 최신 stable로 고정하고, 아래 Helm 값/플래그 이름은 **선택한 버전 문서와 대조**할 것 (마이너 버전 간 키가 바뀜).
  (2026-06 기준 stable = **1.19.4** — 기준 values 는 `bootstrap/cilium-values.yaml`, phase 별 갱신 값 표시됨)
- Gateway API CRD가 현재 **bundle v1.4.1** (Envoy Gateway가 설치). Cilium이 기대하는 Gateway API 버전과 다를 수 있으므로 **phase 4 진입 전 Cilium ↔ Gateway API 지원 매트릭스 확인**(이 전환의 유일한 주요 호환성 리스크).

---

## Phase 0 — 사전 준비 (변경 없음)

```bash
# etcd / 클러스터 백업
sudo cp -r /etc/kubernetes/pki /root/pki-backup-$(date +%F)
# (etcd 스냅샷)
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  snapshot save /root/etcd-snap-$(date +%F).db

# 현재 네트워킹 상태 스냅샷(롤백 대조용)
kubectl get pods -A -o wide > /root/pre-cilium-pods.txt
kubectl get svc -A -o wide > /root/pre-cilium-svc.txt
kubectl get gateway,httproute -A -o yaml > /root/pre-cilium-gateway.yaml

# 도구 설치
# - cilium-cli, helm
# Longhorn: 중요 볼륨 백업/스냅샷 1회 권장
```

체크: 모든 Pod Running, Longhorn 볼륨 healthy, `.201/.202` 외부 접속 정상.

---

## Phase 1 — CNI: Calico → Cilium (per-node 롤링)

> 핵심 아이디어: Cilium을 **CNI conf를 전역에 쓰지 않는 모드**로 먼저 설치(기존 Calico와 공존) →
> 노드를 1대씩 drain → 해당 노드만 Cilium으로 전환 → 전부 끝나면 Calico 제거 + Cilium을 기본 CNI로 승격.
> 정확한 절차는 Cilium 공식 "Migrating to Cilium (per-node)" 문서를 선택 버전 기준으로 따른다. 아래는 이 클러스터 값에 맞춘 골격.

### 1-1. Cilium 설치 (공존 모드, 새 CIDR)
기준 values = **`bootstrap/cilium-values.yaml`** (아래는 핵심 값 발췌):
```yaml
# 새 Pod CIDR + native routing
ipam:
  mode: cluster-pool
  operator:
    clusterPoolIPv4PodCIDRList: ["10.244.0.0/16"]
    clusterPoolIPv4MaskSize: 24
routingMode: native
autoDirectNodeRoutes: true
ipv4NativeRoutingCIDR: "10.244.0.0/16"
enableIPv4Masquerade: true

# 마이그레이션 동안 Cilium이 CNI를 강제 점유하지 않도록 (공존)
cni:
  customConf: true     # CNI conf를 Cilium이 직접 안 씀(노드별로 켤 때까지)
  uninstall: false
operator:
  unmanagedPodWatcher:
    restart: false      # Calico 관리 Pod를 죽이지 않음

# kube-proxy는 이 단계에선 유지 (phase 2에서 대체)
kubeProxyReplacement: false

# 관측
hubble:
  enabled: true
  relay: { enabled: true }
```
```bash
helm repo add cilium https://helm.cilium.io && helm repo update
helm install cilium cilium/cilium -n kube-system -f cilium-values.yaml
cilium status --wait
```

### 1-2. 노드별 전환 (master 먼저? worker 먼저?)
> **worker(.103) 먼저** 전환해 검증한 뒤 master(.105)를 전환한다 (control-plane 리스크 최소화).
> master 전환 시 잠깐 API/스케줄링이 흔들릴 수 있으므로 점검 시간대에.
> ★ helm install(1-1)만으로는 아무것도 전환되지 않는다(공존 모드) — 아래가 실제 전환 단계.

먼저 "라벨 붙은 노드만 Cilium CNI 활성화" 스위치(1회 생성):
```yaml
# cilium-node-migration.yaml — 공식 per-node migration 가이드 양식 (선택 버전 문서와 대조)
apiVersion: cilium.io/v2
kind: CiliumNodeConfig
metadata:
  name: cilium-default
  namespace: kube-system
spec:
  nodeSelector:
    matchLabels:
      io.cilium.migration/cilium-default: "true"
  defaults:
    write-cni-conf-when-ready: /host/etc/cni/net.d/05-cilium.conflist
    custom-cni-conf: "false"
    cni-chaining-mode: "none"
    cni-exclusive: "true"
```
```bash
kubectl apply -f cilium-node-migration.yaml
```

각 노드에 대해 (worker → 검증 → master 순):
```bash
NODE=kube-worker

# ① 비우기
kubectl cordon $NODE
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data

# ② 이 노드를 Cilium 으로 지정 → 노드의 cilium agent 재시작 (CNI conf 기록됨)
kubectl label node $NODE --overwrite io.cilium.migration/cilium-default=true
kubectl -n kube-system delete pod -l k8s-app=cilium --field-selector spec.nodeName=$NODE
kubectl -n kube-system wait --for=condition=Ready pod -l k8s-app=cilium \
  --field-selector spec.nodeName=$NODE --timeout=3m

# ③ 노드 재부팅 (공식 가이드 권장 — Calico veth/iptables 잔재 청소 + 그 노드 Pod 전부 재생성)
#    (해당 노드에 ssh 로) sudo reboot

# ④ 복귀 + 검증
kubectl uncordon $NODE
kubectl get pods -A -o wide --field-selector spec.nodeName=$NODE   # 새 Pod 가 10.244.x IP 인지
# 노드 간 Pod-to-Pod / Pod-to-Service / DNS 확인:
kubectl run mig-test --rm -it --image=busybox:1.36 --restart=Never \
  --overrides='{"spec":{"nodeName":"'$NODE'"}}' -- nslookup kubernetes.default
```
worker 검증 OK → master(.105) 동일 절차. (master drain 시 hostNetwork 인 control-plane
static pod 들은 영향 없음 — drain 이 건드리는 건 일반 워크로드뿐)

### 1-3. Calico 제거 + Cilium 기본 CNI 승격
모든 노드가 Cilium일 때:
```bash
# Calico 매니페스트 제거 (설치했던 calico.yaml 기준)
kubectl delete -f <calico.yaml>     # calico-node DS, calico-config, RBAC, CRD 등
# 각 노드 잔재 정리: /etc/cni/net.d/ 의 calico conf, calico iptables/route, cali* 인터페이스

# Cilium을 단독 CNI로 승격
helm upgrade cilium cilium/cilium -n kube-system -f cilium-values.yaml \
  --set cni.customConf=false --set cni.exclusive=true
kubectl -n kube-system rollout restart ds/cilium
```

### Phase 1 검증
- `cilium status` 정상, `cilium connectivity test` 통과
- 전 Pod가 `10.244.x` IP, 전 네임스페이스 Running
- **NetworkPolicy 시행 확인**: `default-deny` + 네임스페이스별 netpol이 Cilium에서 동작 (예: 막혀야 할 통신이 막히는지 1건 확인)
- Longhorn 볼륨 healthy, 데이터 정상
- Gateway(.201/.202) 통해 외부 접속 정상 (이 시점엔 아직 Envoy Gateway + MetalLB)

**롤백**: 노드별 전환 중 실패 시 해당 노드를 다시 Calico로 되돌리고(역순) Cilium uninstall. Calico 제거 전까지는 비교적 안전.

---

## Phase 2 — kube-proxy 제거 → Cilium eBPF 대체 (변경 없음, repo 무관)

```bash
helm upgrade cilium cilium/cilium -n kube-system -f cilium-values.yaml \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=192.168.123.105 \
  --set k8sServicePort=6443
kubectl -n kube-system rollout restart ds/cilium
cilium status   # KubeProxyReplacement: True 확인

# Cilium이 서비스 처리 확인 후 kube-proxy 제거
kubectl -n kube-system delete ds kube-proxy
kubectl -n kube-system delete cm kube-proxy
# 각 노드: kube-proxy가 남긴 iptables 규칙 정리 (iptables-save | grep KUBE 후 flush, 또는 노드 재부팅)
```

### Phase 2 검증
- `kubeProxyReplacement: True`, `cilium status` 정상
- ClusterIP/NodePort/LoadBalancer 서비스 모두 도달 (특히 .201/.202 Gateway)
- DNS(CoreDNS) 정상

**cilium-values.yaml 에 위 3개 값을 영구 반영**(이후 upgrade의 기준 파일).

---

## Phase 3 — MetalLB → Cilium LB-IPAM + L2 Announcement

> `.201/.202` 유지가 핵심. MetalLB와 Cilium LB-IPAM이 동시에 같은 풀을 다투지 않도록,
> **Cilium 풀을 먼저 만들고 → Gateway/Service의 IP 고정 방식을 전환 → MetalLB 제거** 순서.

> ⚠️ 이 클러스터 특이사항 둘 (phase 2 완료 시점 실측):
> 1. **노드 NIC 은 `wlp1s0`(WiFi)** — L2 정책 `interfaces` 는 `^wlp.*` 여야 함 (`^eth/^en` 은 매칭 0 → 전면 단절).
> 2. **Gateway svc 는 `externalTrafficPolicy: Local`** + envoy fleet 전원 worker 거주. Cilium L2 리더 선출은
>    (MetalLB 와 달리) 로컬 백엔드 유무를 안 보므로 master 가 리더가 되면 VIP 블랙홀 —
>    `nodeSelector` 로 광고 노드를 worker 로 한정한다.

### 3-1. Cilium LB-IPAM 기능 활성화 (cilium-values.yaml 에 반영 후)
```bash
# l2announcements.enabled=true, externalIPs.enabled=true 를 파일에 반영한 상태에서
helm upgrade cilium cilium/cilium -n kube-system -f cilium-values.yaml
kubectl -n kube-system rollout restart ds/cilium
# (kube-proxy replacement가 켜진 상태여야 L2 announcement 동작 — phase 2 전제)
```

### 3-2. IP 풀 + L2 정책 (📝 GitOps — `components/networking/lb-ipam/lb-ipam.yaml`)
Cilium 정책 CRD 는 GitOps 관리 원칙에 따라 leaf App(`apps/networking/lb-ipam.yaml`)으로 적용.
풀은 `cilium.io/v2`, L2 정책은 `v2alpha1`(1.19 기준 v2 미지원). 내용 요지:
- `CiliumLoadBalancerIPPool homelab-pool`: `.200~.210` (MetalLB 풀과 동일)
- `CiliumL2AnnouncementPolicy homelab-l2`: `interfaces: ["^wlp.*"]`, `nodeSelector: kube-worker` (위 특이사항 반영)

### 3-3. IP 고정 병행 → 확인 → MetalLB 제거
- `gateway.yaml` 의 `infrastructure.annotations` 에 `io.cilium/lb-ipam-ips` 를 **MetalLB 어노테이션과 병행 추가** (📝 repo).
  둘 다 같은 IP 를 가리키고 광고 노드도 동일(worker)해서 전환기 충돌 없음 — 단 status 를 두 컨트롤러가
  다투므로 겹침 시간은 짧게, 한 자리에서 끝낸다.
- sync 후 `.201/.202` 가 Cilium 풀에서도 동일 할당되는지 확인하고 MetalLB 제거:
```bash
kubectl get ciliumloadbalancerippool homelab-pool   # IPs Available/Used 확인
# 설치가 raw 매니페스트(v0.14.9, 2026-03-15)였으므로 같은 파일로 제거
# (ns·CRD 까지 삭제되며 내부의 IPAddressPool/L2Advertisement 도 함께 정리됨)
kubectl delete -f https://raw.githubusercontent.com/metallb/metallb/v0.14.9/config/manifests/metallb-native.yaml
```
- 외부 접속 정상 확인 후 `metallb.universe.tf/loadBalancerIPs` 어노테이션 제거 (📝 repo 후속 정리).

### 📝 repo 변경 (이 시점에 push)
- `bootstrap/cilium-values.yaml`: `l2announcements`·`externalIPs` enable
- `components/networking/lb-ipam/` + `apps/networking/lb-ipam.yaml` 신규
- `components/networking/gateway/gateway.yaml`: `io.cilium/lb-ipam-ips` 병행 추가 (MetalLB 제거 후 metallb 어노테이션 삭제)

### Phase 3 검증
- `.201/.202`가 Cilium 풀에서 동일하게 할당, 외부 접속 정상
- `kubectl get ciliumloadbalancerippool`, L2 announcement 로그 정상
- MetalLB 네임스페이스/CRD 잔재 없음

---

## Phase 4 — Envoy Gateway → Cilium Gateway API

> HTTPRoute 7개는 모두 Gateway를 **이름**(`public-gateway`/`admin-gateway`)으로 참조하므로,
> Gateway 이름을 유지하면 **HTTPRoute 수정 불필요**. 바꾸는 건 GatewayClass와 Gateway 스펙뿐.

### 4-0. 호환성 사전 점검 (필수)
- 현재 Gateway API CRD = **bundle v1.4.1**. 선택한 Cilium 버전이 지원하는 Gateway API 버전과 대조.
  - Cilium은 보통 자체 Gateway API CRD를 설치/관리. 기존 v1.4.1과 충돌하면, Cilium 지원 버전에 맞추거나(다운/업그레이드) Cilium 문서의 CRD 가이드 따름.
- Cilium Gateway API는 **kube-proxy replacement(phase 2 완료)** 가 전제 — 이미 충족.

### 4-1. Cilium Gateway API 활성화
```bash
helm upgrade cilium cilium/cilium -n kube-system -f cilium-values.yaml \
  --set gatewayAPI.enabled=true
kubectl -n kube-system rollout restart deploy/cilium-operator
kubectl get gatewayclass cilium    # 생성 확인
```

### 4-2. Gateway 매니페스트 전환 (📝 repo 변경 — `components/networking/gateway/gateway.yaml`)
변경 요지:
- `GatewayClass`: `eg`(Envoy) → **삭제**, Cilium이 만드는 `cilium` 클래스 사용
- 두 Gateway의 `gatewayClassName: eg` → `cilium`
- `infrastructure.parametersRef`(EnvoyProxy 참조) **제거**, `EnvoyProxy` 오브젝트 **삭제**
- LB IP 고정: MetalLB 어노테이션 → **Cilium LB-IPAM 어노테이션**(`io.cilium/lb-ipam-ips`)으로 교체.
  `.201`/`.202` 그대로 유지.
- HA(public replicas 2): Envoy의 `EnvoyProxy` 대신 **`CiliumGatewayClassConfig`** 로 표현(필요 시).
  단순화를 원하면 우선 기본(replicas 1)로 전환 후, HA는 후속으로.

> ⚠️ `gateway` App은 `prune:true, selfHeal:true`. push 즉시 ArgoCD가 적용 → Envoy Gateway가 만들던 .201/.202 LB 서비스가 사라지고 Cilium이 새로 생성. **점검 시간대에**, 그리고 Cilium Gateway API가 먼저 활성화(4-1)된 뒤 push.

### 4-3. Envoy Gateway 제거
```bash
# 새 Cilium Gateway가 .201/.202로 정상 서비스하는지 확인 후
helm uninstall eg -n envoy-gateway-system    # (설치 방식에 맞게)
kubectl delete ns envoy-gateway-system
# envoyproxies / *.envoyproxy.io CRD 정리
```

### Phase 4 검증
- `kubectl get gateway -A` → 둘 다 `PROGRAMMED=True`, 주소 `.201/.202`
- 7개 HTTPRoute의 호스트(`sapari/argocd/grafana/kibana/vault/minio/storage.ascode.click`) 외부 접속 전부 정상
- Cloudflare Tunnel(.105) 경유 정상
- `category: networking` App들 ArgoCD Synced/Healthy

---

## 완료 후 상태
| 영역 | 전환 후 |
|---|---|
| CNI | Cilium (native routing, cluster-pool `10.244.0.0/16`) |
| 서비스 | Cilium kube-proxy replacement (eBPF), kube-proxy 제거 |
| LoadBalancer | Cilium LB-IPAM + L2 announcement (`192.168.123.200-210`) |
| Gateway | Cilium Gateway API (GatewayClass `cilium`), Envoy/MetalLB 제거 |
| 관측 | Hubble (+relay) |

## repo 변경 요약
- **phase 1·2·3**: 변경 없음 (전부 부트스트랩/cilium-values.yaml 영역)
- **phase 4**: `components/networking/gateway/gateway.yaml` — GatewayClass·gatewayClassName·EnvoyProxy 제거, LB IP 어노테이션 교체
- HTTPRoute(`components/networking/routes/*`), NetworkPolicy(`components/platform/network-policies/*`): **변경 없음**

## 후속(선택)
- Hubble UI, Cilium NetworkPolicy(CRD)로 L7 정책, BGP, Cilium Ingress 대신 Gateway 유지 등은 전환 안정화 후 별도로.
