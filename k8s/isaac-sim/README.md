# Isaac Sim 5.1 — ROS2 bridge + WebRTC streaming on a non-hostNetwork pod

이 디렉토리는 Isaac Sim **5.1.0**(vscode launchable) 워크스페이스를 Kubernetes(비-hostNetwork pod,
Calico CNI)에 배포하는 매니페스트와, 그 위에서 **ROS2 bridge**와 **WebRTC 3D 스트리밍**을 함께
동작시키기 위한 설정을 담는다. NVIDIA 공식 isaac-launchable은 `network_mode: host`(docker-compose)를
전제로 하므로, k8s pod(비-hostNetwork) 환경에서는 아래 설정이 추가로 필요하다.

이 문서는 시행착오 끝에 확정된 "왜"를 기록한다. 플래그 하나라도 빼면 다시 깨진다.

---

## TL;DR — 띄우는 법

code-server 터미널에서:

```bash
isaacsim-ros2          # Isaac Sim 5.1 + ROS2 bridge + WebRTC streaming 한 번에 기동
```

그 다음 브라우저에서 web view(스트리밍)를 새로고침하면 3D 화면이 뜬다.
ROS2 확인은 **별도 터미널**에서(자동으로 ros2 CLI가 activate됨):

```bash
ros2 topic list                 # 씬에 ROS2 OmniGraph + Play 후 /clock 등 노출
ros2 topic echo /clock
```

`isaacsim-ros2` 런처는 vscode 이미지(`omniverse-docker-image/vscode/isaacsim-ros2.sh`)에 baked되어
있으며, 아래 모든 플래그를 자동으로 적용한다.

---

## 구조

- **entrypoint = code-server** 뿐이다. Isaac Sim은 자동 기동되지 않고 사용자가 터미널에서 띄운다.
  (한 번에 하나만 — 여러 번 띄우면 좀비 kit 프로세스가 스트리밍 포트를 두고 충돌한다.)
- 컨테이너 3개: `web-viewer`(vite, 5173/TCP), `nginx`(80/TCP), `ov-...`(Isaac Sim/code-server).
- Isaac Sim 컨테이너에 **hostPort 30998/UDP**가 선언되어 있다(아래 참고). 이게 WebRTC 미디어의 핵심.
- TURN(coturn)은 `coturn` 네임스페이스의 hostNetwork DaemonSet(노드마다, `node-ip:3478`).

---

## WebRTC 스트리밍 — 왜 이렇게 해야 하나

### 핵심 사실
- Isaac Sim 5.1은 webrtc 확장 **7.0.0**(Kit 107)을 쓴다. 이 버전은 **서버 relay candidate를 못
  만든다**(TURN 미지원). 그래서 **host candidate(`node-ip:streamPort`)** 하나에 의존한다.
- webrtc 7.0.0은 candidate IP를 **노드 IP로 강제**한다(STUN reflexive). `publicEndpointAddress`,
  `ISAACSIM_HOST`, `HOST_IP`를 podIP로 줘도 candidate IP는 노드 IP가 된다.
- pod는 hostNetwork가 아니므로 실제 미디어 UDP는 **podIP**에서 listen한다.

### 그래서 필요한 두 가지
1. **hostPort 30998/UDP** — `node-ip:30998 → pod:30998` 포워딩(kubelet DNAT, PREROUTING).
   이게 있어야 candidate(`node-ip:30998`)가 실제 서버(pod:30998)에 도달한다.
   → deployment 매니페스트의 Isaac Sim 컨테이너 `ports`에 이미 선언되어 있어야 한다:
   ```yaml
   ports:
     - { containerPort: 30998, hostPort: 30998, protocol: UDP, name: webrtc-media }
     - { containerPort: 49100, protocol: TCP, name: webrtc-signal }
   ```
2. **브라우저 직접 연결(relay 금지)** — 브라우저가 candidate(`node-ip:30998`)로 **직접** 연결해야
   hostPort의 PREROUTING DNAT를 탄다. **절대 `iceTransportPolicy=relay`를 강제하지 말 것.**
   relay로 보내면 coturn(같은 노드 hostNetwork)이 `node-ip:30998`으로 보내는데, 이는 노드의
   **OUTPUT 경로**라 hostPort PREROUTING DNAT를 우회 → pod에 안 닿음 →
   `StreamerNoNominatedCandidatePairs`.
   → web-viewer는 RTCPeerConnection monkey-patch(relay 강제)를 **넣지 않아야** 한다.

### 서버 측 플래그 (isaacsim-ros2가 자동 적용)
```
--/app/livestream/publicEndpointAddress=<podIP>   # ICE candidate gathering 트리거 (없으면 UDP 소켓 0개)
--/app/livestream/publicEndpointPort=30998        # = streamPort
--/app/livestream/fixedHostPort=30998             # 미디어 UDP를 hostPort와 동일 포트에 고정
```
`publicEndpointAddress`가 없으면 NVST가 미디어 UDP 소켓을 아예 안 열어 candidate가 생성되지 않는다
(값 자체는 candidate IP에 반영되지 않지만 — gathering을 켜는 트리거로 반드시 필요).

---

## ROS2 bridge — 왜 이렇게 해야 하나

`isaacsim-ros2`가 `--enable isaacsim.ros2.bridge`와 함께 다음을 자동 처리한다:

1. **conda 격리** (`env -u PYTHONPATH -u CONDA_PREFIX -u CONDA_DEFAULT_ENV`)
   code-server 터미널은 RoboStack `/opt/ros_env`(humble) conda를 자동 activate한다. 그 PYTHONPATH가
   Kit python(3.11)을 `/opt/ros_env`의 python3.12 rclpy로 보내 깨뜨린다(`_rclpy_pybind11.cpython-311
   not present`). conda 변수를 벗기면 Kit이 자기 **internal rclpy**(cpython-311)를 로드해 정상 동작.

2. **`LD_LIBRARY_PATH=/isaac-sim/exts/isaacsim.ros2.bridge/<distro>/lib`**
   C++ 브리지가 시작 시 여기서 `librmw_fastrtps_cpp.so`를 dlopen한다. 없으면 "RMW implementation not
   installed"로 브리지만 shutdown된다(Isaac Sim 본체는 살아있음). 5.1=`humble`, 6.0=`jazzy`.

ROS2 CLI(`ros2 …`)는 인터랙티브 터미널에서 자동 activate되어 같은 컨테이너 안에서 DDS(FastDDS,
`rmw_fastrtps_cpp`, domain 0)로 브리지와 토픽을 주고받는다(사이드카 불필요). distro/FastDDS가
브리지와 일치해야 한다(5.1=Humble/FastDDS 2.6).

---

## 6.0.0과의 차이 (참고)

6.0.0(Sim6.0)은 webrtc 확장 **10.x**(Kit 110)를 쓰며, **서버 relay candidate**를 만들 수 있어
양쪽이 coturn relay로 만난다 → hostPort/직접연결 없이도 비-hostNetwork에서 동작한다(ROS distro는
Jazzy). 5.1.0(webrtc 7.0.0)은 그 기능이 없어 위의 hostPort + 직접연결 구성이 필요하다.

---

## 트러블슈팅

| 증상 (브라우저 콘솔 / 서버 로그) | 원인 | 조치 |
|---|---|---|
| `StreamerNoStunResponsesReceived` | 서버가 candidate gathering 안 함 / 클라가 TURN 못 씀 | `publicEndpointAddress` 지정(서버 UDP 소켓 0개 → 생성), relay 강제 제거 |
| `StreamerNoNominatedCandidatePairs` | candidate(node-ip:port)가 pod에 안 닿음 | hostPort 30998 ↔ `fixedHostPort/publicEndpointPort/streamPort` **포트 일치**, relay 강제 제거 |
| `ROS2 Bridge startup failed` / `librmw_fastrtps_cpp.so cannot open` | C++ 브리지 LD_LIBRARY_PATH 누락 | `LD_LIBRARY_PATH=.../ros2.bridge/<distro>/lib` (isaacsim-ros2가 처리) |
| `Could not import rclpy` (Warning) | conda PYTHONPATH 오염 | conda 변수 unset (isaacsim-ros2가 처리) |
| 화면 안 뜨고 `NVST_CCE_DISCONNECTED` 반복 | kit 인스턴스 여러 개(좀비) 포트 충돌 | `pkill -9 -f kit/kit; pkill -9 -f runheadless` 후 1개만 기동 |

### 빠른 점검
```bash
# 서버가 미디어 UDP candidate 소켓을 열었는지 (연결 시도 중 확인)
ss -ulnp | grep -i kit            # 30998 UDP 보여야 정상

# hostPort 포워딩 존재 확인
kubectl -n <ns> get deploy <dep> -o jsonpath='{.spec.template.spec.containers[*].ports}'

# 정확한 실패 지점은 chrome://webrtc-internals 의 candidate-pair state 로 확인
```
