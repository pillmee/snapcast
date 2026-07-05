# Snapcast — Claude 작업 지침

## 다이어그램

- Mermaid flowchart는 항상 `flowchart TD` (top-down) 방향으로 그린다.

---

## 마지막 작업 내용 (2026-07-05)

### 진행한 작업

1. **구조 문서 재정리** — `review/structure/`에 문서 2개 추가, 기존 문서의 중복/분산된 내용 이동
   - `review/structure/07_client_management.md` — 서버의 클라이언트 등록·그룹·스트림 배정·영속화 구조
   - `review/structure/08_control_stream_flow.md` — 서버-클라이언트 간 제어(JSON-RPC)·스트림(오디오) 전송 과정을 하나로 통합 (기존 `04_common.md`의 핸드셰이크/JSON-RPC API/PCM 청크 흐름, `03_client.md`의 클라이언트 메시지 처리 흐름을 이동)
   - `02_server.md`에 "실제 전송 메커니즘" 표 추가 — `pipe`/`alsa`/`tcp`/`process`/`spotify`/`airplay`/`pipewire`/`jack`/`meta` 등 스트림 scheme별로 실제 어떤 OS/IPC API(예: `snd_pcm_readi`, `boost::asio::async_read`, libpipewire 콜백 스레드, JACK RT 스레드 등)를 쓰는지 정리
   - `01_overview.md` 파일 목록 갱신

2. **`review/integration/android_hal.md` 설계 재검토**
   - "문제점 1" 다이어그램 오류 수정: HAL이 Snapserver 이벤트를 직접 IPC로 구독해 APM에 통지하는 것처럼 그려졌던 부분을 제거 — 실제 설계는 별도 자바 시스템 서비스가 컨트롤 플레인을 담당(문서 내 이미 확정된 "해결 방안"과 다이어그램을 일치시킴)
   - **그룹은 네트워크 홉이 아니라 config 상의 스트림 구독 매핑**이라는 점을 명확화 — 실제 오디오 전송은 `StreamServer::onChunkEncoded()`가 스트림당 한 번 인코딩된 버퍼를 그 스트림을 구독하는 각 클라이언트 세션에 개별 TCP unicast하는 구조 (그룹을 거쳐 재배포되는 게 아님)
   - `setWiredDeviceConnectionState()`를 바로 사용할 수 없는 이유 검토: hidden API(공개 SDK 아님), `MODIFY_AUDIO_SETTINGS_PRIVILEGED` 특권 권한 필요, 원래 "유선 액세서리"용으로 설계되어 `AUDIO_DEVICE_OUT_IP` 지원 여부가 Android 버전마다 다를 수 있음 → **Android 12+는 AIDL `IModule.connectedExternalDevice()`/`disconnectedExternalDevice()` 경로를 우선 검토할 것을 권고** (아직 문서에는 미반영, 다음 작업으로 이월)

### 다음 작업 (Android HAL 통합)

**목표**: 기존 Primary HAL을 유지하면서 Snapcast 전용 신규 HAL 모듈 추가.

**확정된 설계 방향**:
- HAL 디바이스 타입: `AUDIO_DEVICE_OUT_IP`
- address 단위: **Snapcast 그룹/스트림명** (클라이언트 개별 IP 아님)
- address 식별자: `host_id` (MAC 기반, `common/message/hello.hpp`) 권장
- 동적 디바이스 등록: `dynamic="true"` + `set_parameters` (Android ~11) / AIDL `connectedExternalDevice` (Android 12+)
- 컨트롤 플레인/데이터 플레인 분리: 자바 시스템 서비스가 Snapserver JSON-RPC 구독 + `AudioManager` 등록, HAL은 순수 TCP write만 담당

**미결 사항 (다음 작업)**:
1. `android_hal.md`에 `setWiredDeviceConnectionState` 관련 권고(Android 12+는 AIDL 경로 우선) 반영
2. Snapcast 그룹/스트림 기반 HAL 디바이스 매핑 세부 설계
3. 자바 시스템 서비스 구현: JSON-RPC 클라이언트, 그룹→address 상태 추적(신규/소멸 필터링), `AudioManager` 등록/해제
4. `setWiredDeviceConnectionState()`(또는 대체 API)의 `AUDIO_DEVICE_OUT_IP` 지원 여부·필요 권한을 타겟 Android 버전에서 검증
5. Android 타겟 버전 결정 → HIDL / AIDL HAL 구현
6. `server/streamreader/tcp_stream.cpp` — 그룹별 다중 포트 확장 검토

**관련 파일**:
- `server/streamreader/tcp_stream.cpp` — 오디오 입력 TCP 소켓 (HAL이 연결할 대상)
- `server/streamreader/tcp_stream.hpp`
- `common/message/hello.hpp` — `host_id` 정의
- `review/integration/android_hal.md` — 설계 검토 전문
- `review/structure/08_control_stream_flow.md` — 제어/스트림 전송 흐름 통합 문서
- `review/structure/07_client_management.md` — 클라이언트 등록/그룹 관리 구조
