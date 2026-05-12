# omniverse-docker-image — Dockerfile 분리 및 태그 기반 CI/CD

**날짜**: 2026-05-12
**대상 레포**: https://github.com/xiilab/omniverse-docker-image.git (신규)
**소스 레포**: isaac-launchable (이번 작업 디렉터리)

## 배경

isaac-launchable 레포에 흩어져 있는 4종 Dockerfile을 **omniverse-docker-image** 라는 신규 레포로 분리하고, git tag 푸시 시 해당 이미지를 빌드/푸시하는 CI/CD를 GitHub Actions에 구축한다. 각 이미지는 빌드 시 OCI manifest annotation을 부착해 astrago builtin 카탈로그가 컨테이너 스펙(omniverse_type, ports, env 등)을 자동 수집할 수 있게 한다.

## 결정 사항 요약

- **레지스트리**: `docker.io/xiilab/<image>`
- **CI 플랫폼**: GitHub Actions, ubuntu-latest 러너
- **태그 스킴**: `<image>/<version>` prefix (예: `vscode/6.0.0-fix5364`)
- **베이스 이미지**: 모두 NGC (`nvcr.io/nvidia/...`)
- **annotation**: `docker buildx build --annotation "manifest:astrago.builtin.*"` 로 부착, `--output type=registry,oci-mediatypes=true` 강제
- **사이드카(nginx/viewer)**: annotation 없이 빌드만

## 빌드 대상

| 이미지 | omniverse_type | 베이스 | 비고 |
|---|---|---|---|
| `isaac-launchable-vscode` | `ISAAC_SIM` | `nvcr.io/nvidia/isaac-lab:3.0.0-beta1` | Isaac Sim 6 + Isaac Lab + code-server + IsaacLab #5364 fix |
| `usd-composer-sample` | `USD_COMPOSER` | `nvcr.io/nvidia/omniverse/usd-composer-sample:109.0.4` | pass-through (FROM only) |
| `isaac-launchable-nginx` | (없음) | `openresty/openresty:1.21.4.1-0-alpine` | livestream 프록시, 사이드카 |
| `isaac-launchable-viewer` | (없음) | `nvcr.io/nvidia/base/ubuntu:22.04_20240212` | Kit 스트리밍 viewer, 사이드카 |

## 레포 구조

```
omniverse-docker-image/
├── .github/
│   └── workflows/
│       ├── build.yml            # tag push 트리거, 이미지 식별, reusable 호출
│       └── _build-image.yml     # reusable: login → buildx → annotate → push
├── vscode/
│   ├── Dockerfile
│   ├── entrypoint.sh
│   ├── README.md                # workspace 안내
│   ├── settings.json            # VSCode 설정
│   └── extensions/
│       └── omni.clipboard.service/   # isaac-launchable에서 이관
├── usd-composer/
│   └── Dockerfile               # FROM-only + 헤더 주석
├── nginx/
│   ├── Dockerfile
│   ├── entrypoint.sh
│   └── nginx.conf
├── viewer/
│   ├── Dockerfile
│   ├── entrypoint.sh
│   └── clipboard-bridge.ts
├── README.md
└── .gitignore
```

**원칙**:
- 이미지별 디렉터리 = build context (`docker buildx build -f vscode/Dockerfile vscode/`)
- isaac-launchable 레포의 기존 Dockerfile은 이번 작업에서 건드리지 않음 (스코프 외, 별도 PR로 deprecate)

## CI/CD Workflow

### `build.yml` (진입점)

```yaml
name: build
on:
  push:
    tags:
      - 'vscode/**'
      - 'usd-composer/**'
      - 'nginx/**'
      - 'viewer/**'

jobs:
  vscode:
    if: startsWith(github.ref, 'refs/tags/vscode/')
    uses: ./.github/workflows/_build-image.yml
    secrets: inherit
    with:
      image_name: isaac-launchable-vscode
      image_version: ${{ github.ref_name }}
      context: vscode
      dockerfile: vscode/Dockerfile
      annotations: |
        astrago.builtin.omniverse_type=ISAAC_SIM
        astrago.builtin.display_name=isaac-launchable-vscode
        astrago.builtin.command=code-server --cert=false --auth=none --bind-addr=0.0.0.0:8080
        astrago.builtin.ports=[{"name":"vscode","port":8080},{"name":"webrtc-media","port":30998,"protocol":"UDP"},{"name":"webrtc-signal","port":49100,"protocol":"TCP"}]
        astrago.builtin.env=[{"name":"ACCEPT_EULA","value":"Y"},{"name":"OMNI_KIT_ALLOW_ROOT","value":"1"},{"name":"ISAACSIM_STREAM_PORT","value":"30998"},{"name":"ISAACSIM_SIGNAL_PORT","value":"49100"},{"name":"DEFAULT_WORKSPACE","value":"/config/workspace"}]
        astrago.builtin.description=Isaac Sim 6 + Isaac Lab with VSCode

  usd-composer:
    if: startsWith(github.ref, 'refs/tags/usd-composer/')
    uses: ./.github/workflows/_build-image.yml
    secrets: inherit
    with:
      image_name: usd-composer-sample
      image_version: ${{ github.ref_name }}
      context: usd-composer
      dockerfile: usd-composer/Dockerfile
      annotations: |
        astrago.builtin.omniverse_type=USD_COMPOSER
        astrago.builtin.display_name=usd-composer-sample
        astrago.builtin.ports=[{"name":"webrtc-media","port":47999,"protocol":"UDP"},{"name":"webrtc-signal","port":49100,"protocol":"TCP"}]
        astrago.builtin.env=[{"name":"ACCEPT_EULA","value":"Y"},{"name":"NVIDIA_VISIBLE_DEVICES","value":"all"},{"name":"NVIDIA_DRIVER_CAPABILITIES","value":"all"}]
        astrago.builtin.description=Omniverse USD Composer 109.0.4 (Kit livestream)

  nginx:
    if: startsWith(github.ref, 'refs/tags/nginx/')
    uses: ./.github/workflows/_build-image.yml
    secrets: inherit
    with:
      image_name: isaac-launchable-nginx
      image_version: ${{ github.ref_name }}
      context: nginx
      dockerfile: nginx/Dockerfile

  viewer:
    if: startsWith(github.ref, 'refs/tags/viewer/')
    uses: ./.github/workflows/_build-image.yml
    secrets: inherit
    with:
      image_name: isaac-launchable-viewer
      image_version: ${{ github.ref_name }}
      context: viewer
      dockerfile: viewer/Dockerfile
```

### `_build-image.yml` (reusable)

단계 (개념):

1. **checkout** (`actions/checkout@v4`)
2. **버전 파싱**: `IMAGE_TAG=${IMAGE_VERSION#*/}` (prefix 제거). 결과 검증: 빈 문자열이면 fail.
3. **annotation 라인 검증**:
   - 각 줄을 `^[a-zA-Z0-9._-]+=.+$` 로 정규식 매칭
   - `omniverse_type` 라인 필수성: vscode / usd-composer (이미지명으로 판별) 에는 반드시 존재해야 함
   - `astrago.builtin.ports` / `astrago.builtin.env` 의 값을 `jq -e .` 으로 JSON lint
4. **디스크 확보**: `df -h` 출력 → `/usr/share/dotnet`, `/opt/ghc`, `/usr/local/lib/android` 등 제거 (~10GB 확보)
5. **NGC login**:
   ```sh
   echo "$NGC_API_KEY" | docker login nvcr.io -u '$oauthtoken' --password-stdin
   ```
6. **Docker Hub login** (`docker/login-action@v3`, `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN`)
7. **QEMU + buildx setup** (`docker/setup-qemu-action@v3`, `docker/setup-buildx-action@v3`)
8. **annotation flag 합성**: 입력 multiline → 각 줄에 `--annotation "manifest:<line>"` 부착
9. **빌드 + push**:
   ```sh
   docker buildx build \
     --platform linux/amd64 \
     --tag docker.io/xiilab/${IMAGE_NAME}:${IMAGE_TAG} \
     ${ANNOTATION_FLAGS} \
     --output type=registry,oci-mediatypes=true \
     -f ${DOCKERFILE} \
     ${CONTEXT}
   ```
10. **post-push 검증**:
    ```sh
    docker buildx imagetools inspect docker.io/xiilab/${IMAGE_NAME}:${IMAGE_TAG} \
      --format '{{json .Manifest.Annotations}}'
    ```
    로그에 실제 부착된 annotation 출력. annotation 입력값이 있던 경우 출력에 `astrago.builtin.*` 키가 보이는지 grep 검증.

## Annotation source-of-truth

| Annotation | vscode 출처 | usd-composer 출처 |
|---|---|---|
| `omniverse_type` | 수동 분류 | 수동 분류 |
| `display_name` | 이미지명 | 이미지명 |
| `command` | `entrypoint.sh` 내 code-server invocation | _(생략)_ deployment override 안 함 |
| `ports` | `k8s/isaac-sim/deployment-0.yaml` `containers[name=vscode].ports[]` | `k8s/usd-composer/deployment.yaml` `containers[name=kit-streaming].ports[]` |
| `env` | 동일 deployment의 정적 ENV만 | 동일 deployment의 정적 ENV만 |
| `description` | 자유 텍스트 | 자유 텍스트 |

## Annotation 필터링 규칙

ENV 중 다음은 annotation에 포함하지 않는다:

1. `valueFrom` (secretKeyRef / configMapKeyRef / fieldRef) → 런타임 결정
2. 노드 IP가 박힌 값 (예: `NVDA_KIT_ARGS=...publicIp=10.61.3.74...`)
3. 클러스터 내부 DNS 의존 (`.svc.cluster.local` 포함)
4. `command/args` 중 deployment에서 override 안 하는 것 → 이미지 기본 CMD 사용

## Dockerfile 이관 매핑

### vscode/Dockerfile

소스: `isaac-launchable/isaac-lab/vscode/Dockerfile.isaacsim6`

변경:
- 베이스: `10.61.3.124:30002/library/isaac-lab:3.0.0-beta1` → `nvcr.io/nvidia/isaac-lab:3.0.0-beta1`
- build context: 레포 루트 → `vscode/` 디렉터리
  - `COPY isaac-lab/vscode/README.md ...` → `COPY README.md ...`
  - `COPY isaac-lab/vscode/settings.json ...` → `COPY settings.json ...`
  - `COPY extensions/omni.clipboard.service ...` → `COPY extensions/omni.clipboard.service ...` (디렉터리 안으로 이관)
- 나머지 그대로: code-server 4.96.4 설치, packaging fix, IsaacLab #5364 patch (8개 play.py/train.py), HAMi LD_PRELOAD 등록, OV cache 디렉터리

### usd-composer/Dockerfile

```dockerfile
# USD Composer 109.0.4 - annotation overlay (pass-through).
# 빌드 단계 없음. buildx --annotation 으로 manifest annotation만 부착.
# 컨테이너 스펙 출처: isaac-launchable/k8s/usd-composer/deployment.yaml 의 kit-streaming 컨테이너.
# 제외 항목(annotation 미포함):
#   - NVDA_KIT_ARGS: publicIp=10.61.3.74 노드 IP 의존
#   - AUTO_ENABLE_DRIVER_SHADER_CACHE_WRAPPER: 네임스페이스별 memcached URL
#   - command/args: deployment 에서 override 하지 않으므로 이미지 기본값 사용
#   - lifecycle.postStart: 런타임 동작 (annotation으로 표현 불가)
FROM nvcr.io/nvidia/omniverse/usd-composer-sample:109.0.4
```

### nginx/Dockerfile

소스: `isaac-launchable/isaac-lab/nginx/Dockerfile` (isaac-lab 버전이 더 최신)
함께 이관: `entrypoint.sh`, `nginx.conf` (4012 bytes 버전)

### viewer/Dockerfile

소스: `isaac-launchable/web-viewer-sample/Dockerfile`
함께 이관: `entrypoint.sh`, `clipboard-bridge.ts`

## Secrets

| Secret | 용도 | 비고 |
|---|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub push | `xiilab` |
| `DOCKERHUB_TOKEN` | Docker Hub push | PAT 권장 (Read/Write scope) |
| `NGC_API_KEY` | `nvcr.io` base 이미지 pull | NGC Personal Key, 사용자=`$oauthtoken` |

**보안 정책**:
- 워크플로우 로그 토큰 마스킹 (`::add-mask::`)
- `gh secret set X < secret.txt` 패턴 권장 (shell history 회피)
- **본 작업 도중 채팅에 노출된 자격증명 2건은 작업 완료 즉시 회전 필요**

## 에러 처리

| 케이스 | 동작 |
|---|---|
| 잘못된 tag prefix | `on.push.tags` 미매칭 → 워크플로우 미실행 |
| annotation 포맷 위반 | 검증 step에서 fail (`exit 1`) |
| 필수 `omniverse_type` 누락 (vscode/usd-composer) | 검증 step에서 fail |
| 베이스 이미지 pull 실패 | buildx step fail → NGC_API_KEY 확인 안내 |
| Docker Hub push 실패 | login-action step fail (빌드 전) |
| ubuntu-latest 디스크 full | cleanup step + `df -h`. 그래도 부족 시 self-hosted 권고 |
| 동일 tag 재푸시 | Docker Hub에서 새 manifest digest로 overwrite. 동일 이미지면 layer push 0개로 빠름 |

## 테스트 / 검증

**자동 (CI)**:
1. annotation 라인 정규식 + JSON lint
2. 필수 `omniverse_type` 존재
3. post-push `imagetools inspect` 로 실제 annotation 출력

**수동 (rollout 시 체크리스트, README에 명시)**:
- [ ] `docker pull docker.io/xiilab/<image>:<tag>` 성공
- [ ] `docker buildx imagetools inspect ...` 출력에 `astrago.builtin.omniverse_type` 보임 (vscode/usd-composer만)
- [ ] k8s deployment image 라인 docker.io로 변경 후 pod 정상 기동
- [ ] astrago builtin 카탈로그 등록 확인

## Rollout 순서

1. omniverse-docker-image 레포에 코드 push (workflows + Dockerfile 4종)
2. `nginx/1.0.0` 태그 push → 가장 단순한 빌드로 워크플로우 검증
3. `viewer/1.0.0` 태그 push → NGC 인증 경로 검증
4. `usd-composer/109.0.4` 태그 push → annotation 부착 검증
5. `vscode/6.0.0-fix5364` 태그 push → 가장 무거운 빌드 (디스크 한계 검증)
6. *(스코프 외, 추후)* isaac-launchable 레포의 k8s 매니페스트 image 라인 docker.io 갱신

## 미해결 항목 & 위험

| 항목 | 위험도 | 대응 |
|---|---|---|
| `nvcr.io/nvidia/isaac-lab:3.0.0-beta1` 실재 여부 | 높음 | 구현 phase 첫 단계에 `docker pull` 검증 → 부재 시 사용자 재질의 (Mirror to Docker Hub fallback) |
| ubuntu-latest 디스크 한계 (vscode 이미지) | 중간 | cleanup step + 모니터링. 실패 시 self-hosted 전환 |
| 채팅 노출 자격증명 (Docker Hub PW + NGC key) | 높음 | 작업 완료 즉시 회전, README에 회전 안내 명시 |
| `xiilab/usd-composer-sample` 이미지명의 NVIDIA 브랜드 재배포 | 낮음 | NGC EULA 재배포 허용. README에 출처 명시 |

## 스코프 외 (Out of Scope)

- isaac-launchable 레포의 기존 Dockerfile 삭제 / k8s 매니페스트 갱신
- isaac-lab 베이스 이미지 자체(`3.0.0-beta1`) 빌드 — NGC public 사용 가정
- 다중 아키텍처 빌드 (arm64 등) — NVIDIA 스택은 amd64 only
- 로컬 dry-run 도구(Makefile/justfile) — 1차에서 제외, 추후 추가 가능
