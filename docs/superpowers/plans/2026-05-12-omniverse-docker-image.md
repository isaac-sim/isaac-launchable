# omniverse-docker-image Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** isaac-launchable 레포의 4개 Dockerfile(vscode, usd-composer, nginx, viewer)을 신규 omniverse-docker-image 레포로 분리하고, git tag prefix 기반 GitHub Actions CI/CD로 `docker.io/xiilab/<image>:<version>`에 OCI manifest annotation을 포함해 빌드/푸시한다.

**Architecture:** tag push 이벤트 → `build.yml`이 tag prefix(`vscode/`, `usd-composer/`, `nginx/`, `viewer/`)로 1개 job만 활성화 → reusable workflow `_build-image.yml` 호출 → annotation 라인을 `scripts/parse_annotations.sh`가 검증·플래그화 → `docker buildx build --annotation manifest:... --output type=registry,oci-mediatypes=true` 로 Docker Hub에 푸시. 보조 이미지(nginx/viewer)는 annotation 미부착. USD Composer는 NGC 베이스를 그대로 통과시키는 단순 `FROM` Dockerfile.

**Tech Stack:** GitHub Actions, Docker buildx, NGC (nvcr.io) base images, Docker Hub registry, Bash 5+ with `jq`. 로컬 검증은 `hadolint`(Dockerfile lint), `actionlint`(workflow lint), bash native test runner.

**Working directory convention:**
- 소스 레포(이번 작업의 reference): `/Users/xiilab/git/isaac-launchable`
- 새 레포(작업 대상): `/Users/xiilab/git/omniverse-docker-image`
- 모든 git/edit 명령은 새 레포에서 실행 (각 task에 명시)

**Pre-flight assumption:**
- GitHub에 `xiilab/omniverse-docker-image` 빈 레포가 이미 생성되어 있다 (사용자 사전 작업). 없을 경우 Task 1에서 `gh repo create` 한다.

---

## File Structure (new repo)

```
/Users/xiilab/git/omniverse-docker-image/
├── .github/workflows/build.yml            # tag push 진입점
├── .github/workflows/_build-image.yml     # reusable: login → buildx → push
├── scripts/parse_annotations.sh           # annotation 라인 검증/플래그화
├── tests/test_parse_annotations.sh        # parse_annotations.sh 단위 테스트
├── vscode/Dockerfile                       # isaac-launchable-vscode (ISAAC_SIM)
├── vscode/entrypoint.sh
├── vscode/README.md
├── vscode/settings.json
├── vscode/extensions/omni.clipboard.service/  # isaac-launchable의 extensions/ 이관
├── usd-composer/Dockerfile                 # FROM-only pass-through
├── nginx/Dockerfile                        # openresty 기반
├── nginx/entrypoint.sh
├── nginx/nginx.conf
├── viewer/Dockerfile                       # NGC ubuntu + ov-web-sample
├── viewer/entrypoint.sh
├── viewer/clipboard-bridge.ts
├── README.md
└── .gitignore
```

---

## Task 1: Bootstrap omniverse-docker-image repo

**Files:**
- Create: `/Users/xiilab/git/omniverse-docker-image/README.md`
- Create: `/Users/xiilab/git/omniverse-docker-image/.gitignore`

- [ ] **Step 1: Clone or initialize repo**

```bash
cd ~/git
if gh repo view xiilab/omniverse-docker-image >/dev/null 2>&1; then
  git clone https://github.com/xiilab/omniverse-docker-image.git
  cd omniverse-docker-image
else
  echo "Repo not found on GitHub. Creating..."
  gh repo create xiilab/omniverse-docker-image --public --description "Omniverse/Isaac Sim Docker images (Isaac Lab VSCode, USD Composer, nginx, viewer)" --confirm
  git clone https://github.com/xiilab/omniverse-docker-image.git
  cd omniverse-docker-image
fi
pwd  # expect: /Users/xiilab/git/omniverse-docker-image
```

- [ ] **Step 2: Add README.md**

Create `/Users/xiilab/git/omniverse-docker-image/README.md`:

```markdown
# omniverse-docker-image

NVIDIA Omniverse / Isaac Sim 워크로드용 Docker 이미지 빌드 레포.

## 이미지 4종

| 이미지 | omniverse_type | 베이스 | 비고 |
|---|---|---|---|
| `docker.io/xiilab/isaac-launchable-vscode` | `ISAAC_SIM` | `nvcr.io/nvidia/isaac-lab:3.0.0-beta1` | Isaac Sim 6 + Isaac Lab + code-server |
| `docker.io/xiilab/usd-composer-sample` | `USD_COMPOSER` | `nvcr.io/nvidia/omniverse/usd-composer-sample:109.0.4` | pass-through |
| `docker.io/xiilab/isaac-launchable-nginx` | _(없음)_ | `openresty/openresty:1.21.4.1-0-alpine` | livestream 프록시 사이드카 |
| `docker.io/xiilab/isaac-launchable-viewer` | _(없음)_ | `nvcr.io/nvidia/base/ubuntu:22.04_20240212` | Kit 스트리밍 viewer 사이드카 |

## 빌드 트리거

git tag prefix로 1개 이미지만 빌드된다:

```bash
git tag vscode/6.0.0-fix5364
git tag usd-composer/109.0.4
git tag nginx/1.0.0
git tag viewer/1.0.0
git push origin --tags
```

## 필요한 GitHub Secrets

| Secret | 용도 |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub push (`xiilab`) |
| `DOCKERHUB_TOKEN` | Docker Hub PAT (Read/Write) |
| `NGC_API_KEY` | `nvcr.io` 베이스 이미지 pull |

자격증명은 정기 회전 권장.

## Annotation 추출 규칙

각 이미지의 OCI manifest annotation은 `astrago.builtin.*` 키로 부착된다. 다음은 annotation에 **포함하지 않는다**:

1. `valueFrom` (secretKeyRef / configMapKeyRef / fieldRef) → 런타임 결정
2. 노드 IP가 박힌 값 (예: `publicIp=10.61.3.74`)
3. 클러스터 내부 DNS 의존 (`.svc.cluster.local`)
4. command/args 중 deployment에서 override 안 하는 것 → 이미지 기본값 사용
```

- [ ] **Step 3: Add .gitignore**

Create `/Users/xiilab/git/omniverse-docker-image/.gitignore`:

```
# OS
.DS_Store
Thumbs.db

# Editor
.vscode/
.idea/
*.swp
*~

# Build
node_modules/
__pycache__/
*.pyc
```

- [ ] **Step 4: Commit and push**

```bash
cd /Users/xiilab/git/omniverse-docker-image
git add README.md .gitignore
git commit -m "chore: initial bootstrap (README + .gitignore)"
git push -u origin main
```

Expected: push succeeds, GitHub repo가 README를 보여줌.

---

## Task 2: Implement parse_annotations.sh (TDD)

**Files:**
- Create: `/Users/xiilab/git/omniverse-docker-image/scripts/parse_annotations.sh`
- Create: `/Users/xiilab/git/omniverse-docker-image/tests/test_parse_annotations.sh`

이 스크립트는 multiline annotation 입력을 받아 (1) 라인 포맷 검증, (2) JSON 값 lint, (3) 필수 omniverse_type 확인, (4) `--annotation "manifest:..."` 플래그를 stdout에 출력한다. CI에서 호출되며 로컬에서도 단위 테스트 가능.

- [ ] **Step 1: Write the failing test**

Create `/Users/xiilab/git/omniverse-docker-image/tests/test_parse_annotations.sh`:

```bash
#!/usr/bin/env bash
# tests/test_parse_annotations.sh
# Plain bash test runner for scripts/parse_annotations.sh.
# Exit 0 on all pass, non-zero on any fail.
set -u

SCRIPT="$(cd "$(dirname "$0")/.." && pwd)/scripts/parse_annotations.sh"

PASS=0; FAIL=0
fail() { echo "FAIL: $1" >&2; FAIL=$((FAIL+1)); }
pass() { echo "PASS: $1"; PASS=$((PASS+1)); }

# Test 1: valid vscode input produces 2+ flag lines and exits 0
out=$(printf '%s\n' \
  'astrago.builtin.omniverse_type=ISAAC_SIM' \
  'astrago.builtin.display_name=isaac-launchable-vscode' \
  | "$SCRIPT" vscode 2>&1)
rc=$?
if [ "$rc" -eq 0 ] && echo "$out" | grep -q -- '--annotation' && echo "$out" | grep -q 'manifest:astrago.builtin.omniverse_type=ISAAC_SIM'; then
  pass "vscode with omniverse_type produces flags"
else
  fail "vscode valid input: rc=$rc out=$out"
fi

# Test 2: vscode missing omniverse_type fails
out=$(printf 'astrago.builtin.display_name=foo\n' | "$SCRIPT" vscode 2>&1)
rc=$?
if [ "$rc" -ne 0 ] && echo "$out" | grep -q 'omniverse_type'; then
  pass "vscode missing omniverse_type fails"
else
  fail "vscode missing omniverse_type: rc=$rc out=$out"
fi

# Test 3: usd-composer missing omniverse_type fails
out=$(printf 'astrago.builtin.display_name=foo\n' | "$SCRIPT" usd-composer 2>&1)
rc=$?
if [ "$rc" -ne 0 ] && echo "$out" | grep -q 'omniverse_type'; then
  pass "usd-composer missing omniverse_type fails"
else
  fail "usd-composer missing omniverse_type: rc=$rc out=$out"
fi

# Test 4: nginx missing omniverse_type succeeds (not required)
out=$(printf '' | "$SCRIPT" nginx 2>&1)
rc=$?
if [ "$rc" -eq 0 ]; then
  pass "nginx empty input succeeds (no annotations required)"
else
  fail "nginx empty input: rc=$rc out=$out"
fi

# Test 5: invalid line format fails
out=$(printf 'this is not key=value valid format\n' | "$SCRIPT" nginx 2>&1)
rc=$?
if [ "$rc" -ne 0 ] && echo "$out" | grep -q 'invalid'; then
  pass "invalid line format fails"
else
  fail "invalid line format: rc=$rc out=$out"
fi

# Test 6: invalid JSON in ports fails
out=$(printf 'astrago.builtin.omniverse_type=ISAAC_SIM\nastrago.builtin.ports=[bad json\n' | "$SCRIPT" vscode 2>&1)
rc=$?
if [ "$rc" -ne 0 ] && echo "$out" | grep -qi 'json'; then
  pass "invalid JSON in ports fails"
else
  fail "invalid JSON ports: rc=$rc out=$out"
fi

# Test 7: invalid omniverse_type value fails
out=$(printf 'astrago.builtin.omniverse_type=BANANA\n' | "$SCRIPT" vscode 2>&1)
rc=$?
if [ "$rc" -ne 0 ] && echo "$out" | grep -qi 'BANANA\|ISAAC_SIM\|USD_COMPOSER'; then
  pass "invalid omniverse_type value fails"
else
  fail "invalid omniverse_type value: rc=$rc out=$out"
fi

# Test 8: whitespace-only lines are skipped
out=$(printf 'astrago.builtin.omniverse_type=ISAAC_SIM\n   \n\nastrago.builtin.display_name=foo\n' | "$SCRIPT" vscode 2>&1)
rc=$?
if [ "$rc" -eq 0 ] && [ "$(echo "$out" | grep -c -- '--annotation')" = "2" ]; then
  pass "whitespace-only lines are skipped"
else
  fail "whitespace skip: rc=$rc out=$out"
fi

echo
echo "RESULTS: PASS=$PASS FAIL=$FAIL"
[ "$FAIL" -eq 0 ]
```

Make executable:

```bash
cd /Users/xiilab/git/omniverse-docker-image
mkdir -p scripts tests
# create the test file (use your editor or heredoc per above)
chmod +x tests/test_parse_annotations.sh
```

- [ ] **Step 2: Run the test — expect failure**

```bash
cd /Users/xiilab/git/omniverse-docker-image
./tests/test_parse_annotations.sh
```

Expected: 모든 테스트 FAIL (scripts/parse_annotations.sh 부재).

- [ ] **Step 3: Implement parse_annotations.sh**

Create `/Users/xiilab/git/omniverse-docker-image/scripts/parse_annotations.sh`:

```bash
#!/usr/bin/env bash
# parse_annotations.sh
# stdin:   multiline annotations (one per line, "key=value" format)
# arg $1:  image_kind (vscode | usd-composer | nginx | viewer)
# stdout:  one shell-escaped flag per line, suitable for `xargs -d '\n' docker buildx build ...`
# exit 0:  all lines valid, output written
# exit 1:  validation failure (message on stderr)
#
# Rules enforced:
#   - each non-blank line must match ^[a-zA-Z0-9._-]+=.+$
#   - astrago.builtin.omniverse_type, if present, must be ISAAC_SIM or USD_COMPOSER
#   - astrago.builtin.ports and astrago.builtin.env values must be valid JSON
#   - for image_kind ∈ {vscode, usd-composer}, astrago.builtin.omniverse_type is required
set -euo pipefail

image_kind="${1:?usage: parse_annotations.sh <image_kind>}"

requires_omniverse_type() {
  case "$1" in
    vscode|usd-composer) return 0 ;;
    *) return 1 ;;
  esac
}

has_omniverse_type=0

while IFS= read -r line || [ -n "${line:-}" ]; do
  # strip leading/trailing whitespace
  trimmed="${line#"${line%%[![:space:]]*}"}"
  trimmed="${trimmed%"${trimmed##*[![:space:]]}"}"
  [ -z "$trimmed" ] && continue

  if [[ ! "$trimmed" =~ ^[a-zA-Z0-9._-]+=.+$ ]]; then
    echo "ERROR: annotation line invalid (expected key=value): $trimmed" >&2
    exit 1
  fi

  key="${trimmed%%=*}"
  value="${trimmed#*=}"

  if [ "$key" = "astrago.builtin.omniverse_type" ]; then
    has_omniverse_type=1
    case "$value" in
      ISAAC_SIM|USD_COMPOSER) ;;
      *)
        echo "ERROR: omniverse_type must be ISAAC_SIM or USD_COMPOSER, got: $value" >&2
        exit 1
        ;;
    esac
  fi

  if [ "$key" = "astrago.builtin.ports" ] || [ "$key" = "astrago.builtin.env" ]; then
    if ! printf '%s' "$value" | jq -e . >/dev/null 2>&1; then
      echo "ERROR: $key value is not valid JSON: $value" >&2
      exit 1
    fi
  fi

  # emit two lines: the --annotation flag, and its argument
  printf -- '--annotation\n'
  printf 'manifest:%s\n' "$trimmed"
done

if requires_omniverse_type "$image_kind" && [ "$has_omniverse_type" -eq 0 ]; then
  echo "ERROR: image '$image_kind' requires astrago.builtin.omniverse_type annotation" >&2
  exit 1
fi
```

Make executable:

```bash
chmod +x /Users/xiilab/git/omniverse-docker-image/scripts/parse_annotations.sh
```

- [ ] **Step 4: Run the test — expect pass**

```bash
cd /Users/xiilab/git/omniverse-docker-image
./tests/test_parse_annotations.sh
```

Expected: `RESULTS: PASS=8 FAIL=0`, exit 0.

- [ ] **Step 5: Commit**

```bash
cd /Users/xiilab/git/omniverse-docker-image
git add scripts/parse_annotations.sh tests/test_parse_annotations.sh
git commit -m "feat: add parse_annotations.sh annotation validator with tests"
git push
```

---

## Task 3: Write _build-image.yml reusable workflow

**Files:**
- Create: `/Users/xiilab/git/omniverse-docker-image/.github/workflows/_build-image.yml`

reusable workflow — checkout → version 파싱 → annotation 검증 → 디스크 정리 → NGC/DockerHub 로그인 → buildx 빌드/푸시 → post-push inspect.

- [ ] **Step 1: Create the workflow file**

Create `/Users/xiilab/git/omniverse-docker-image/.github/workflows/_build-image.yml`:

```yaml
name: _build-image
on:
  workflow_call:
    inputs:
      image_name:
        description: "Docker image name (without registry/namespace), e.g. isaac-launchable-vscode"
        required: true
        type: string
      image_version:
        description: "Full git ref_name, e.g. vscode/6.0.0-fix5364 (prefix stripped internally)"
        required: true
        type: string
      context:
        description: "Build context directory (relative to repo root)"
        required: true
        type: string
      dockerfile:
        description: "Path to Dockerfile (relative to repo root)"
        required: true
        type: string
      annotations:
        description: "Multiline 'key=value' annotation lines, one per line. Optional (sidecar images omit this)."
        required: false
        type: string
        default: ""

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Parse version from tag
        id: ver
        run: |
          set -euo pipefail
          # image_version comes in as "vscode/6.0.0-fix5364"; strip the first segment
          full="${{ inputs.image_version }}"
          tag="${full#*/}"
          if [ -z "$tag" ] || [ "$tag" = "$full" ]; then
            echo "ERROR: image_version='$full' missing '/' separator" >&2
            exit 1
          fi
          echo "tag=$tag" >> "$GITHUB_OUTPUT"
          echo "Resolved image tag: $tag"

      - name: Determine image kind from context
        id: kind
        run: |
          set -euo pipefail
          # context is "vscode" | "usd-composer" | "nginx" | "viewer"
          ctx="${{ inputs.context }}"
          case "$ctx" in
            vscode|usd-composer|nginx|viewer) echo "kind=$ctx" >> "$GITHUB_OUTPUT" ;;
            *) echo "ERROR: unknown context '$ctx'" >&2; exit 1 ;;
          esac

      - name: Validate and parse annotations
        id: ann
        run: |
          set -euo pipefail
          # Empty inputs.annotations => no flags, no validation (sidecar)
          annotations_input=$(cat <<'EOF'
          ${{ inputs.annotations }}
          EOF
          )

          if [ -z "${annotations_input//[[:space:]]/}" ]; then
            echo "No annotations provided (sidecar image)."
            : > /tmp/annotation_flags.txt
          else
            printf '%s\n' "$annotations_input" \
              | ./scripts/parse_annotations.sh "${{ steps.kind.outputs.kind }}" \
              > /tmp/annotation_flags.txt
            echo "Parsed annotation flags:"
            cat /tmp/annotation_flags.txt
          fi

      - name: Free disk space on runner
        run: |
          echo "Before cleanup:"; df -h /
          sudo rm -rf /usr/share/dotnet /usr/local/lib/android /opt/ghc /opt/hostedtoolcache/CodeQL
          sudo docker image prune --all --force || true
          echo "After cleanup:"; df -h /

      - name: Login to NGC (nvcr.io)
        run: |
          set -euo pipefail
          echo "$NGC_API_KEY" | docker login nvcr.io -u '$oauthtoken' --password-stdin
        env:
          NGC_API_KEY: ${{ secrets.NGC_API_KEY }}

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Set up QEMU
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push with annotations
        run: |
          set -euo pipefail
          IMAGE="docker.io/xiilab/${{ inputs.image_name }}:${{ steps.ver.outputs.tag }}"
          echo "Building $IMAGE"

          # Read annotation flags (each --annotation is two consecutive lines)
          mapfile -t ann_lines < /tmp/annotation_flags.txt

          docker buildx build \
            --platform linux/amd64 \
            --tag "$IMAGE" \
            "${ann_lines[@]}" \
            --output type=registry,oci-mediatypes=true \
            -f "${{ inputs.dockerfile }}" \
            "${{ inputs.context }}"

      - name: Verify pushed manifest annotations
        run: |
          set -euo pipefail
          IMAGE="docker.io/xiilab/${{ inputs.image_name }}:${{ steps.ver.outputs.tag }}"
          echo "Inspecting $IMAGE"
          docker buildx imagetools inspect "$IMAGE" --format '{{json .Manifest.Annotations}}' | tee /tmp/pushed_annotations.json

          # If annotations were provided, assert astrago.builtin.* keys present in output
          if [ -s /tmp/annotation_flags.txt ]; then
            if ! grep -q 'astrago.builtin' /tmp/pushed_annotations.json; then
              echo "ERROR: expected astrago.builtin.* annotations not found in pushed manifest" >&2
              exit 1
            fi
            echo "Annotation verification OK."
          else
            echo "Sidecar image: skipping annotation presence check."
          fi
```

- [ ] **Step 2: Lint the workflow YAML (locally if actionlint installed)**

```bash
cd /Users/xiilab/git/omniverse-docker-image
# Install actionlint if missing: brew install actionlint
which actionlint && actionlint .github/workflows/_build-image.yml || echo "actionlint not installed, skipping local lint"
```

Expected: 0 errors (or skipped).

- [ ] **Step 3: Commit**

```bash
cd /Users/xiilab/git/omniverse-docker-image
git add .github/workflows/_build-image.yml
git commit -m "feat: add _build-image reusable workflow (NGC + DockerHub + buildx annotations)"
git push
```

---

## Task 4: Write build.yml entrypoint workflow

**Files:**
- Create: `/Users/xiilab/git/omniverse-docker-image/.github/workflows/build.yml`

- [ ] **Step 1: Create the entrypoint workflow**

Create `/Users/xiilab/git/omniverse-docker-image/.github/workflows/build.yml`:

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

- [ ] **Step 2: Lint the workflow**

```bash
cd /Users/xiilab/git/omniverse-docker-image
which actionlint && actionlint .github/workflows/build.yml || echo "skip"
```

Expected: 0 errors.

- [ ] **Step 3: Commit**

```bash
cd /Users/xiilab/git/omniverse-docker-image
git add .github/workflows/build.yml
git commit -m "feat: add build entrypoint workflow with per-image tag prefix routing"
git push
```

---

## Task 5: Port nginx image

**Files:**
- Create: `/Users/xiilab/git/omniverse-docker-image/nginx/Dockerfile`
- Create: `/Users/xiilab/git/omniverse-docker-image/nginx/entrypoint.sh`
- Create: `/Users/xiilab/git/omniverse-docker-image/nginx/nginx.conf`

소스: `/Users/xiilab/git/isaac-launchable/isaac-lab/nginx/` (4012 bytes 버전).

- [ ] **Step 1: Copy nginx files**

```bash
SRC=/Users/xiilab/git/isaac-launchable/isaac-lab/nginx
DST=/Users/xiilab/git/omniverse-docker-image/nginx
mkdir -p "$DST"
cp "$SRC/Dockerfile" "$DST/Dockerfile"
cp "$SRC/entrypoint.sh" "$DST/entrypoint.sh"
cp "$SRC/nginx.conf" "$DST/nginx.conf"
chmod +x "$DST/entrypoint.sh"
```

- [ ] **Step 2: Verify nginx Dockerfile builds locally (dry-run via cacheonly)**

```bash
cd /Users/xiilab/git/omniverse-docker-image
docker buildx build --output type=cacheonly -f nginx/Dockerfile nginx/
```

Expected: build succeeds without push. If you don't want to actually pull the openresty base locally, skip this step — CI will catch errors.

- [ ] **Step 3: Hadolint nginx Dockerfile (optional)**

```bash
which hadolint && hadolint nginx/Dockerfile || echo "hadolint not installed, skipping"
```

Expected: 0 errors or only style warnings.

- [ ] **Step 4: Commit**

```bash
cd /Users/xiilab/git/omniverse-docker-image
git add nginx/
git commit -m "feat(nginx): port livestream proxy image from isaac-launchable"
git push
```

---

## Task 6: Port viewer image

**Files:**
- Create: `/Users/xiilab/git/omniverse-docker-image/viewer/Dockerfile`
- Create: `/Users/xiilab/git/omniverse-docker-image/viewer/entrypoint.sh`
- Create: `/Users/xiilab/git/omniverse-docker-image/viewer/clipboard-bridge.ts`

소스: `/Users/xiilab/git/isaac-launchable/web-viewer-sample/`.

- [ ] **Step 1: Copy viewer files**

```bash
SRC=/Users/xiilab/git/isaac-launchable/web-viewer-sample
DST=/Users/xiilab/git/omniverse-docker-image/viewer
mkdir -p "$DST"
cp "$SRC/Dockerfile" "$DST/Dockerfile"
cp "$SRC/entrypoint.sh" "$DST/entrypoint.sh"
cp "$SRC/clipboard-bridge.ts" "$DST/clipboard-bridge.ts"
chmod +x "$DST/entrypoint.sh"
```

- [ ] **Step 2: Verify Dockerfile paths still resolve**

The viewer Dockerfile uses `COPY clipboard-bridge.ts src/clipboard-bridge.ts` and `COPY entrypoint.sh /entrypoint.sh` — these resolve against the build context. Since context will be `viewer/` and both files exist there, no path edits needed. Run:

```bash
cd /Users/xiilab/git/omniverse-docker-image
grep -E '^COPY ' viewer/Dockerfile
```

Expected output: `COPY clipboard-bridge.ts src/clipboard-bridge.ts` and `COPY entrypoint.sh /entrypoint.sh`. Both reference files inside `viewer/`. ✅

- [ ] **Step 3: Commit**

```bash
cd /Users/xiilab/git/omniverse-docker-image
git add viewer/
git commit -m "feat(viewer): port Kit web-viewer image from isaac-launchable"
git push
```

---

## Task 7: Create usd-composer pass-through Dockerfile

**Files:**
- Create: `/Users/xiilab/git/omniverse-docker-image/usd-composer/Dockerfile`

- [ ] **Step 1: Create the pass-through Dockerfile**

Create `/Users/xiilab/git/omniverse-docker-image/usd-composer/Dockerfile`:

```dockerfile
# USD Composer 109.0.4 - annotation overlay (pass-through).
# 빌드 단계 없음. buildx --annotation 으로 manifest annotation만 부착해 새 매니페스트를 푸시한다.
#
# 컨테이너 스펙 출처: isaac-launchable/k8s/usd-composer/deployment.yaml 의 kit-streaming 컨테이너.
#
# annotation에서 의도적으로 제외한 항목:
#   - NVDA_KIT_ARGS: publicIp=10.61.3.74 노드 IP가 박혀있어 다른 노드/클러스터에서 무효
#   - AUTO_ENABLE_DRIVER_SHADER_CACHE_WRAPPER: 네임스페이스 의존 memcached URL (hsscdns://...svc.cluster.local)
#   - command/args: deployment에서 override하지 않으므로 이미지 기본 CMD 사용
#   - lifecycle.postStart: 런타임 동작 (annotation으로 표현 불가)
FROM nvcr.io/nvidia/omniverse/usd-composer-sample:109.0.4
```

- [ ] **Step 2: Commit**

```bash
cd /Users/xiilab/git/omniverse-docker-image
mkdir -p usd-composer
# (paste the Dockerfile content above into usd-composer/Dockerfile)
git add usd-composer/
git commit -m "feat(usd-composer): add pass-through Dockerfile for annotation overlay"
git push
```

---

## Task 8: Port vscode image

**Files:**
- Create: `/Users/xiilab/git/omniverse-docker-image/vscode/Dockerfile`
- Create: `/Users/xiilab/git/omniverse-docker-image/vscode/entrypoint.sh`
- Create: `/Users/xiilab/git/omniverse-docker-image/vscode/README.md`
- Create: `/Users/xiilab/git/omniverse-docker-image/vscode/settings.json`
- Create: `/Users/xiilab/git/omniverse-docker-image/vscode/extensions/omni.clipboard.service/` (디렉터리 트리)

소스: `isaac-launchable/isaac-lab/vscode/Dockerfile.isaacsim6` + `isaac-launchable/extensions/`.

- [ ] **Step 1: Copy supporting files (entrypoint, README, settings)**

```bash
SRC_LAB=/Users/xiilab/git/isaac-launchable/isaac-lab/vscode
SRC_EXT=/Users/xiilab/git/isaac-launchable/extensions
DST=/Users/xiilab/git/omniverse-docker-image/vscode

mkdir -p "$DST/extensions"
cp "$SRC_LAB/entrypoint.sh" "$DST/entrypoint.sh"
cp "$SRC_LAB/README.md" "$DST/README.md"
cp "$SRC_LAB/settings.json" "$DST/settings.json"
cp -r "$SRC_EXT/omni.clipboard.service" "$DST/extensions/omni.clipboard.service"
chmod +x "$DST/entrypoint.sh"

ls -la "$DST"
ls "$DST/extensions/omni.clipboard.service"  # expect: config  omni
```

- [ ] **Step 2: Write the Dockerfile with path rewrites**

Create `/Users/xiilab/git/omniverse-docker-image/vscode/Dockerfile`:

```dockerfile
# Isaac Launchable VSCode - Isaac Sim 6 + Isaac Lab 3 + code-server
# Build context: this directory (vscode/)
# Build target:  docker.io/xiilab/isaac-launchable-vscode:<tag>
FROM nvcr.io/nvidia/isaac-lab:3.0.0-beta1

USER root

RUN apt update && \
    apt install -y \
        build-essential \
        cmake \
        curl \
        git

RUN curl -fsSL https://code-server.dev/install.sh | sh -s -- --version=4.96.4

RUN apt update && \
    apt install -y \
        unzip \
        dnsutils \
        iproute2 \
        jq

ENV WORKSPACE_DIR=/workspace

COPY README.md ${WORKSPACE_DIR}/README.md
COPY settings.json ${WORKSPACE_DIR}/.vscode/settings.json

RUN code-server \
    --install-extension ms-python.python \
    --install-extension ms-toolsai.jupyter \
    --force ${WORKSPACE_DIR}

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# OV Cache Directories
RUN mkdir -p /home/isaac-sim/.cache/ov
RUN mkdir -p /home/isaac-sim/.local/share/ov

# Fix: packaging module missing in omni.isaac.core_archive/pip_prebundle
# symlink in extscache/omni.services.pip_archive points to this path but the directory is absent
RUN cp -r /isaac-sim/kit/python/lib/python3.12/site-packages/packaging \
    /isaac-sim/exts/omni.isaac.core_archive/pip_prebundle/packaging

# Clipboard sharing extension (browser <-> Kit via HTTP)
COPY extensions/omni.clipboard.service /isaac-sim/user-exts/omni.clipboard.service

# IsaacLab #5364 fix: --livestream 2 produces no inbound-rtp video track because
# `args_cli.visualizer` defaults to None, sending launch_simulation down a
# non-Kit render path that never binds to the NVST capture target. The play.py
# scripts call parse_known_args() without setting a default — patch all 4
# RL-framework play.py + train.py to inject `visualizer=["kit"]` immediately
# before parse_known_args. Idempotent: skipped if FIX_5364 marker already
# present (e.g. when base image is rebaked).
# https://github.com/isaac-sim/IsaacLab/issues/5364#issuecomment-4321530532
RUN for f in /workspace/isaaclab/scripts/reinforcement_learning/rsl_rl/play.py \
             /workspace/isaaclab/scripts/reinforcement_learning/rsl_rl/train.py \
             /workspace/isaaclab/scripts/reinforcement_learning/sb3/play.py \
             /workspace/isaaclab/scripts/reinforcement_learning/sb3/train.py \
             /workspace/isaaclab/scripts/reinforcement_learning/rl_games/play.py \
             /workspace/isaaclab/scripts/reinforcement_learning/rl_games/train.py \
             /workspace/isaaclab/scripts/reinforcement_learning/skrl/play.py \
             /workspace/isaaclab/scripts/reinforcement_learning/skrl/train.py; do \
        if [ -f "$f" ] && ! grep -q "FIX_5364" "$f"; then \
            sed -i '/^args_cli, hydra_args = parser.parse_known_args()$/i parser.set_defaults(visualizer=["kit"])  # FIX_5364' "$f"; \
            echo "patched: $f"; \
        fi; \
    done

# HAMi vGPU enforcement via system-wide preload.
# HAMi device-plugin mounts libvgpu.so at /usr/local/vgpu/libvgpu.so when the pod requests
# nvidia.com/gpumem or nvidia.com/gpucores. Registering it in /etc/ld.so.preload survives
# scripts (e.g. isaaclab.sh) that overwrite LD_PRELOAD, so every child process still gets
# the CUDA/Vulkan hooks. ld.so silently skips the entry when the file is absent, so images
# run unchanged on nodes without HAMi.
RUN echo "/usr/local/vgpu/libvgpu.so" > /etc/ld.so.preload

ENTRYPOINT ["/bin/bash"]

CMD ["/entrypoint.sh"]
```

- [ ] **Step 3: Sanity-check paths**

```bash
cd /Users/xiilab/git/omniverse-docker-image
ls vscode/Dockerfile vscode/entrypoint.sh vscode/README.md vscode/settings.json
ls vscode/extensions/omni.clipboard.service
```

Expected: all 4 files exist; extensions dir contains `config` and `omni`.

- [ ] **Step 4: Hadolint (optional)**

```bash
which hadolint && hadolint vscode/Dockerfile || echo "skip"
```

Expected: 0 errors or only style warnings (e.g. DL3008 for unpinned apt versions — acceptable).

- [ ] **Step 5: Commit**

```bash
cd /Users/xiilab/git/omniverse-docker-image
git add vscode/
git commit -m "feat(vscode): port Isaac Sim 6 + Isaac Lab + code-server image with #5364 fix"
git push
```

---

## Task 9: Pre-rollout — verify NGC base image and set GitHub Secrets

이 작업은 코드 변경 없음. 환경 준비 단계.

- [ ] **Step 1: Verify `nvcr.io/nvidia/isaac-lab:3.0.0-beta1` exists**

이 스펙의 핵심 가정. 부재 시 fallback 필요.

```bash
# Use NGC API key from GitHub Secrets (or paste locally one-time)
echo "$NGC_API_KEY" | docker login nvcr.io -u '$oauthtoken' --password-stdin
docker pull nvcr.io/nvidia/isaac-lab:3.0.0-beta1
```

Expected: pull succeeds. If `manifest unknown` 또는 `not found`, **중단하고 사용자에게 질의**:
- "3.0.0-beta1이 NGC public에 없습니다. 다음 중 어떻게 처리할까요?
  - (A) Mirror to Docker Hub: 사설 harbor의 isaac-lab:3.0.0-beta1을 `docker.io/xiilab/isaac-lab-base:3.0.0-beta1` 로 한 번 push, vscode/Dockerfile의 FROM을 이 경로로 변경
  - (B) NGC public의 다른 태그(`2.x` 등) 사용 가능 여부 확인 → vscode Dockerfile 베이스 교체
  - (C) 사용자가 NGC private에 publish 후 알려주기"

- [ ] **Step 2: Set GitHub Secrets (read from stdin, no shell history)**

```bash
cd /Users/xiilab/git/omniverse-docker-image
gh secret set DOCKERHUB_USERNAME --body "xiilab"

# For sensitive secrets, prefer stdin so the value never appears in shell history:
gh secret set DOCKERHUB_TOKEN   # paste PAT when prompted (or pipe < token.txt)
gh secret set NGC_API_KEY       # paste NGC personal key when prompted
```

Expected: `gh secret list` shows three secrets.

```bash
gh secret list
```

Expected output (mtimes will vary):
```
DOCKERHUB_TOKEN     Updated 2026-05-12
DOCKERHUB_USERNAME  Updated 2026-05-12
NGC_API_KEY         Updated 2026-05-12
```

- [ ] **Step 3: Sanity-check DOCKERHUB_USERNAME**

```bash
docker login -u xiilab  # paste PAT when prompted, verify login works
docker logout
```

Expected: "Login Succeeded".

---

## Task 10: Rollout phase 1 — nginx (smoke test)

가장 단순한 이미지로 전체 워크플로우를 검증한다. annotation 없음, 베이스도 public.

- [ ] **Step 1: Push the nginx tag**

```bash
cd /Users/xiilab/git/omniverse-docker-image
git tag nginx/1.0.0
git push origin nginx/1.0.0
```

- [ ] **Step 2: Watch the workflow run**

```bash
gh run watch
# or
gh run list --workflow build.yml
```

Expected: only the `nginx` job runs (others skipped via `if`). Run completes green.

- [ ] **Step 3: Verify the image was pushed**

```bash
docker pull docker.io/xiilab/isaac-launchable-nginx:1.0.0
docker buildx imagetools inspect docker.io/xiilab/isaac-launchable-nginx:1.0.0 --format '{{json .Manifest.Annotations}}'
```

Expected: pull succeeds; annotations output is either `null`/`{}` or contains only buildx defaults (`org.opencontainers.image.created`, etc.). **No `astrago.builtin.*` keys** (sidecar policy).

- [ ] **Step 4: If anything failed**

- workflow failed on validation: read `gh run view --log` for the failed step
- push failed on auth: re-check `DOCKERHUB_TOKEN` PAT scope (needs Read/Write on `xiilab` namespace)
- workflow not triggered at all: `gh run list` empty → re-check tag prefix matches `on.push.tags` glob

Do not proceed to Phase 2 until nginx is green.

---

## Task 11: Rollout phase 2 — viewer (NGC pull verification)

NGC 인증 경로를 검증한다. annotation 없음.

- [ ] **Step 1: Push the viewer tag**

```bash
cd /Users/xiilab/git/omniverse-docker-image
git tag viewer/1.0.0
git push origin viewer/1.0.0
```

- [ ] **Step 2: Watch run**

```bash
gh run watch
```

Expected: `viewer` job only, completes green. Logs show successful `nvcr.io` login.

- [ ] **Step 3: Verify pushed image**

```bash
docker pull docker.io/xiilab/isaac-launchable-viewer:1.0.0
docker buildx imagetools inspect docker.io/xiilab/isaac-launchable-viewer:1.0.0 --format '{{json .Manifest.Annotations}}'
```

Expected: pull succeeds; no `astrago.builtin.*` annotations (sidecar).

Do not proceed to Phase 3 until viewer is green.

---

## Task 12: Rollout phase 3 — usd-composer (annotation verification)

annotation 부착 경로를 검증한다. 가장 단순한 annotation 케이스 (USD_COMPOSER, ports, env, description).

- [ ] **Step 1: Push the usd-composer tag**

```bash
cd /Users/xiilab/git/omniverse-docker-image
git tag usd-composer/109.0.4
git push origin usd-composer/109.0.4
```

- [ ] **Step 2: Watch run**

```bash
gh run watch
```

Expected: `usd-composer` job only. Run logs include the "Parsed annotation flags:" step showing 5 `--annotation` lines. Post-push verification step prints annotations.

- [ ] **Step 3: Verify pushed annotations**

```bash
docker buildx imagetools inspect docker.io/xiilab/usd-composer-sample:109.0.4 \
  --format '{{json .Manifest.Annotations}}' | jq '.'
```

Expected output contains these keys:
- `astrago.builtin.omniverse_type` = `USD_COMPOSER`
- `astrago.builtin.display_name` = `usd-composer-sample`
- `astrago.builtin.ports` = JSON array (2 ports)
- `astrago.builtin.env` = JSON array (3 envs)
- `astrago.builtin.description` = `Omniverse USD Composer 109.0.4 (Kit livestream)`

- [ ] **Step 4: Negative test — re-push same tag**

```bash
# Force re-push: delete and recreate the tag
cd /Users/xiilab/git/omniverse-docker-image
git push origin :refs/tags/usd-composer/109.0.4   # delete remote tag
git tag -d usd-composer/109.0.4                    # delete local tag
git tag usd-composer/109.0.4
git push origin usd-composer/109.0.4
```

Expected: workflow re-runs and succeeds. Manifest digest may change (annotation values identical → same digest), but workflow itself is green.

Do not proceed to Phase 4 until usd-composer is green.

---

## Task 13: Rollout phase 4 — vscode (heaviest build, disk limit test)

가장 크고 시간이 오래 걸리는 빌드. ubuntu-latest 디스크 한계가 드러나는 곳.

- [ ] **Step 1: Push the vscode tag**

```bash
cd /Users/xiilab/git/omniverse-docker-image
git tag vscode/6.0.0-fix5364
git push origin vscode/6.0.0-fix5364
```

- [ ] **Step 2: Watch run (expect 20-40 min)**

```bash
gh run watch
```

Watch for two things in logs:
1. `Free disk space on runner` step: after-cleanup `df -h /` should show ≥ 30GB available
2. `Build and push` step: pulls `nvcr.io/nvidia/isaac-lab:3.0.0-beta1` and runs RUN steps

If disk full mid-build:
- Look at the last `df -h` output before failure
- If <5GB free, the additional cleanup step needs tuning. Add a step before build:
  ```yaml
  - name: Extra disk cleanup
    run: |
      sudo rm -rf /opt/hostedtoolcache/*
      sudo rm -rf /usr/share/swift
      df -h /
  ```
- If still insufficient, switch to self-hosted runner (out of scope for this plan — open follow-up task).

- [ ] **Step 3: Verify pushed annotations**

```bash
docker buildx imagetools inspect docker.io/xiilab/isaac-launchable-vscode:6.0.0-fix5364 \
  --format '{{json .Manifest.Annotations}}' | jq '.'
```

Expected: all 6 keys present:
- `astrago.builtin.omniverse_type` = `ISAAC_SIM`
- `astrago.builtin.display_name` = `isaac-launchable-vscode`
- `astrago.builtin.command` = `code-server --cert=false --auth=none --bind-addr=0.0.0.0:8080`
- `astrago.builtin.ports` = JSON (3 ports: vscode 8080, webrtc-media 30998/UDP, webrtc-signal 49100/TCP)
- `astrago.builtin.env` = JSON (5 envs: ACCEPT_EULA, OMNI_KIT_ALLOW_ROOT, ISAACSIM_STREAM_PORT, ISAACSIM_SIGNAL_PORT, DEFAULT_WORKSPACE)
- `astrago.builtin.description` = `Isaac Sim 6 + Isaac Lab with VSCode`

- [ ] **Step 4: Smoke-test the image (optional, requires GPU host)**

On a GPU node (or skip):
```bash
docker pull docker.io/xiilab/isaac-launchable-vscode:6.0.0-fix5364
docker run --rm --gpus all docker.io/xiilab/isaac-launchable-vscode:6.0.0-fix5364 \
  /bin/bash -c 'code-server --version'
```

Expected: prints `4.96.4`.

- [ ] **Step 5: Final rollout commit (post-flight notes)**

```bash
cd /Users/xiilab/git/omniverse-docker-image
# Optionally: amend README with first-rollout date stamp
git log --oneline -10  # confirm history
```

Final state: 4 tags exist remote (`nginx/1.0.0`, `viewer/1.0.0`, `usd-composer/109.0.4`, `vscode/6.0.0-fix5364`), 4 images pushed to `docker.io/xiilab/`, 2 with annotations.

---

## Post-rollout reminders

1. **Rotate the credentials** that were shared in the brainstorming chat (Docker Hub password → revoke and use PAT-only; NGC API key → regenerate at NGC console).
2. **Update isaac-launchable k8s manifests** in a separate PR: replace `10.61.3.124:30002/library/...` image refs with `docker.io/xiilab/...`. Out of scope for this plan.
3. **Confirm astrago builtin catalog ingestion**: ask astrago team to verify the two annotated images are now visible in their catalog.

---

## Self-Review Notes

**Spec coverage check:**
- ✅ Repo structure (Spec §레포 구조) → Task 1
- ✅ build.yml (Spec §CI/CD Workflow > build.yml) → Task 4
- ✅ _build-image.yml (Spec §CI/CD Workflow > _build-image.yml) → Task 3
- ✅ Annotation validation (Spec §Annotation 데이터 플로우 & 검증) → Task 2 (parse_annotations.sh)
- ✅ Dockerfile 4종 (Spec §Dockerfile 이관 매핑) → Tasks 5-8
- ✅ Secrets (Spec §Secrets) → Task 9
- ✅ Rollout 순서 (Spec §Rollout 순서) → Tasks 10-13
- ✅ NGC base image risk (Spec §미해결 항목 > nvcr.io/nvidia/isaac-lab:3.0.0-beta1) → Task 9 Step 1

**Type consistency check:** `parse_annotations.sh`의 인터페이스(stdin multiline → stdout `--annotation` + `manifest:...` 두 줄씩, exit code)와 `_build-image.yml`의 `mapfile -t ann_lines < /tmp/annotation_flags.txt` + `"${ann_lines[@]}"` 사용이 일관됨. ✅

**Placeholder check:** "TBD", "TODO", "implement later" 없음. ✅
