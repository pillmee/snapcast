# Android 통합 문서 개요

- `review/integration/`의 [android_hal.md](android_hal.md)와 [android_client.md](android_client.md)는 Android 기기가 맡을 수 있는 **두 가지 역할**을 각각 설명한다.
- 최종 목표는 한 기기가 두 역할을 모두 수행하는 것이다.
- 그래서 두 문서는 서로를 전제로 한다. 따로 떼어 읽을 수 없다.
- [network_discovery.md](network_discovery.md)는 두 역할 모두에 적용되는 근본 제약을 다룬다.
  - 이 제약은 "연결은 항상 클라이언트가 먼저 건다"는 Snapcast 프로토콜 규칙이다.

---

## 서버 역할 — [android_hal.md](android_hal.md)

**정의**: Android 기기가 **소스** 역할을 한다.

- 다른 앱이 재생 중인 오디오(`AudioTrack`/`MediaPlayer` 출력)를 캡처한다.
- 캡처한 오디오를 Snapcast 그룹(zone)으로 묶는다.
- 그 오디오를 네트워크의 다른 Snapclient들에게 배포한다.

- **통합 레벨**: HAL (AudioFlinger 라우팅 레벨)
  - 다른 앱이 이 오디오 경로를 "출력 대상"으로 선택할 수 있어야 한다.
  - 그래서 앱 레벨로는 구현할 수 없다.
- **핵심 설계**:
  - 신규 `audio.snapcast.so` HAL 모듈 (`AUDIO_DEVICE_OUT_IP`)
  - zone 슬롯 주소 체계
  - 컨트롤 플레인과 데이터 플레인의 분리
    - 컨트롤 플레인: Java 시스템 서비스 — JSON-RPC 구독, zone 풀 관리, `Group.SetStream`
    - 데이터 플레인: HAL — zone별로 고정 포트에 TCP write
- **상태**:
  - 아키텍처는 확정됨.
  - 구현은 아직 시작 전.
  - `setWiredDeviceConnectionState()`가 `AUDIO_DEVICE_OUT_IP`를 지원하는지 등 검증할 사항이 남음.

## 클라이언트 역할 — [android_client.md](android_client.md)

**정의**: Android 기기가 **소비자** 역할을 한다.

- 원격 Snapserver에 접속한다.
- 오디오를 수신한다.
- 자신의 스피커로 재생한다.
- 기존에 공개 배포된 앱 "snapdroid"와 같은 패턴이다.

- **통합 레벨**: 앱/포그라운드 서비스
  - 재생된 오디오를 다른 앱이 "소스"로 선택할 필요가 없다.
  - 그래서 HAL까지 내려갈 이유가 없다.
  - 기존 `libsnapclient.so`(Oboe 백엔드)를 JNI로 임베드하면 충분하다.
- **핵심 설계**:
  - 오디오 파이프라인(Controller/Decoder/Stream/OboePlayer)은 이미 프로덕션에서 검증됨.
  - 그래서 **변경 없이 그대로 재사용**한다.
  - 남은 작업은 서버 역할의 Java 시스템 서비스에 클라이언트 모드를 추가하는 것.
    - 클라이언트 모드 = `NsdManager` 디스커버리 + JNI 바인딩
- **상태**:
  - 검토는 완료됨.
  - 신규로 구현할 범위는 작음.
  - 서버 역할과 클라이언트 역할이 동시에 켜졌을 때 오디오 라우팅을 어떻게 중재할지는 아직 정책 결정 사항으로 남음.

## 연결 방향성 — [network_discovery.md](network_discovery.md)

- Snapcast는 비대칭 구조다.
  - 클라이언트가 먼저 연결을 걸어야만 서버가 그 존재를 안다.
  - 서버는 accept만 한다. 클라이언트는 connect만 한다.
  - mDNS도 서버만 광고한다. 클라이언트는 광고하지 않는다.
- 서버 역할은 이 제약의 영향을 받지 않는다. Android 자신이 서버이기 때문이다.
- 클라이언트 역할은 이 제약의 당사자다.
  - Android의 포그라운드 서비스가 죽으면, 서버는 그 사실조차 알 방법이 없다.
  - 서버가 먼저 재접속을 걸어줄 수도 없다.

---

## 역할 비교

| | 서버 역할 ([android_hal.md](android_hal.md)) | 클라이언트 역할 ([android_client.md](android_client.md)) |
|---|---|---|
| 오디오 방향 | Android → Snapserver (소스) | Snapserver → Android (재생) |
| 통합 레벨 | HAL (AudioFlinger 라우팅 레벨) | 앱/포그라운드 서비스 (JNI 임베드) |
| 신규 구현 규모 | 큼 — 신규 HAL 모듈 + Java 시스템 서비스 신규 작성 | 작음 — 기존 snapdroid 자산 재사용, Java 서비스에 모드 추가만 |
| 컨트롤 플레인 | Java 시스템 서비스 (zone 풀 관리, `Group.SetStream`) | Java 시스템 서비스 (`NsdManager` 디스커버리, JNI 바인딩) |
| 데이터 플레인 | 신규 HAL → Snapserver TCP (zone별 write) | `libsnapclient.so`(Controller + OboePlayer) ← Snapserver TCP |
| 최상위 미결 사항 | `setWiredDeviceConnectionState()`의 `AUDIO_DEVICE_OUT_IP` 지원 검증 | 서버·클라이언트 동시 활성 시 오디오 라우팅/포커스 중재 정책 |

- 두 문서가 공유하는 컨트롤 플레인은 "**Java 시스템 서비스**" 하나로 통합 관리하는 것을 권장한다.
  - 포그라운드 서비스, 배터리 최적화 예외를 한 곳에서 관리할 수 있다.
- 다만 물리적 출력이 하나뿐인 기기에서 두 역할이 동시에 켜질 때, 오디오 라우팅을 어떻게 중재할지는 아직 제품 정책 결정 사항으로 남아 있다.
  - 자세한 내용은 [android_client.md의 "검토 사항 2"](android_client.md#검토-사항-2-서버-역할과의-공존-모드-전환)를 참고.
