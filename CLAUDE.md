# Snapcast — Claude 작업 지침

## 다이어그램

- Mermaid flowchart는 항상 `flowchart TD` (top-down) 방향으로 그린다.

---

## 마지막 작업 내용 (2026-06-30)

### 진행한 작업

1. **코드 구조 분석** — `review/` 디렉토리에 전체 코드 분석 문서 작성
   - `review/overview.md` — 전체 개요 및 데이터 흐름
   - `review/server.md` — Snapserver 상세 (스트림 소스, 인코더, 컨트롤 서버)
   - `review/client.md` — Snapclient 상세 (디코더, 플레이어, 시간 동기화)
   - `review/common.md` — 공유 라이브러리, 바이너리 프로토콜
   - `review/build.md` — 빌드 시스템 및 의존성

2. **Android HAL 통합 설계 검토** — `review/android_hal.md`

### 다음 작업 (Android HAL 통합)

**목표**: 기존 Primary HAL을 유지하면서 Snapcast 전용 신규 HAL 모듈 추가.

**확정된 설계 방향**:
- HAL 디바이스 타입: `AUDIO_DEVICE_OUT_IP`
- address 단위: **Snapcast 그룹/스트림명** (클라이언트 개별 IP 아님)
- address 식별자: `host_id` (MAC 기반, `common/message/hello.hpp`) 권장
- 동적 디바이스 등록: `dynamic="true"` + `set_parameters` (Android ~11) / AIDL `connectedExternalDevice` (Android 12+)

**미결 사항 (다음 작업)**:
1. Snapcast 그룹/스트림 기반 HAL 디바이스 매핑 세부 설계
2. HAL ↔ Snapserver IPC 채널 설계 (소켓 또는 binder)
3. Android 타겟 버전 결정 → HIDL / AIDL HAL 구현
4. `server/streamreader/tcp_stream.cpp` — 그룹별 다중 포트 확장 검토

**관련 파일**:
- `server/streamreader/tcp_stream.cpp` — 오디오 입력 TCP 소켓 (HAL이 연결할 대상)
- `server/streamreader/tcp_stream.hpp`
- `common/message/hello.hpp` — `host_id` 정의
- `review/android_hal.md` — 설계 검토 전문
