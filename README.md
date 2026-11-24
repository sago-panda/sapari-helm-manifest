# sapari-helm-manifest

sapari 애플리케이션의 배포 매니페스트입니다.
클러스터 플랫폼(Argo CD, Vault, 모니터링 등)은 GitHub [`UHeeJoon/homelab`](https://github.com/UHeeJoon/homelab)에 있습니다.

## 구조

```
components/<앱>/    앱 하나 = 네임스페이스 하나 (frontend → frontend)
```

Argo CD Application은 이 저장소에 두지 않습니다. homelab 쪽 ApplicationSet이 `components/` 아래 폴더마다 `sapari-<앱>` App을 만듭니다.

## 배포 흐름

```
앱 저장소 CI가 이미지 digest를 main에 bump → 이 저장소 CI가 cosign 서명 검증 → 통과하면 release로 승격 → Argo CD가 반영
```

Argo CD는 `release` 브랜치만 읽습니다.

## 새 앱 추가

1. homelab에 네임스페이스를 만들고, AppProject `sapari`의 destinations에 추가합니다.
2. 이 저장소에 `components/<앱>/`을 추가합니다.
