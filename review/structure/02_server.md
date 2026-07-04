# Snapserver 분석

## 역할

오디오 소스를 읽어 인코딩 후 클라이언트에 배포한다. 동시에 JSON-RPC 컨트롤 서버를 통해 볼륨·그룹·스트림을 제어할 수 있는 관리 인터페이스를 제공한다.

---

## 코어 서버 파일

| 파일 | 역할 |
|------|------|
| `snapserver.cpp` | 진입점 — CLI 파싱, 설정 로드, 데몬화 |
| `server.cpp` | 전체 조율 — 스트림 서버 + 컨트롤 서버 통합 |
| `config.cpp` | snapserver.conf 파싱 |
| `server_settings.hpp` | 서버 설정 구조체 (포트, SSL, 인증 등) |
| `authinfo.hpp` | Basic Auth / JWT 인증 처리 |
| `jwt.cpp` | JWT 토큰 생성 및 검증 |
| `image_cache.hpp` | 앨범 아트 캐시 |

---

## 스트림 서버 (클라이언트 배포)

| 파일 | 역할 |
|------|------|
| `stream_server.cpp` | TCP 포트 1704 — 클라이언트 연결 수락, PCM 청크 배포 |
| `stream_session.cpp` | 개별 클라이언트 스트림 세션 (기본 클래스) |
| `stream_session_tcp.cpp` | TCP 스트림 세션 구현 |
| `stream_session_ws.cpp` | WebSocket 스트림 세션 구현 |

---

## 컨트롤 서버 (JSON-RPC 관리)

| 파일 | 역할 |
|------|------|
| `control_server.cpp` | JSON-RPC 서버 (TCP 1705 / HTTP 1780 / WS 1788) |
| `control_session_tcp.cpp` | TCP 컨트롤 세션 |
| `control_session_http.cpp` | HTTP POST 및 WebSocket 컨트롤 세션 |
| `control_session_ws.cpp` | WebSocket 전용 세션 처리 |
| `control_requests.cpp` | RPC 요청 핸들러 (setVolume, getStatus 등) |

### 지원 포트

| 포트 | 프로토콜 | 용도 |
|------|---------|------|
| 1704 | TCP | 오디오 스트림 |
| 1705 | TCP | JSON-RPC 컨트롤 |
| 1780 | HTTP / WebSocket | 웹 컨트롤 |
| 1788 | WebSocket (SSL) | 보안 웹 컨트롤 |

---

## 스트림 소스 (`streamreader/`)

모든 소스는 `PcmStream` 기반 클래스를 상속하며 공통 인터페이스를 구현한다.

| 파일 | 소스 유형 | 설명 |
|------|----------|------|
| `pcm_stream.cpp` | 기반 클래스 | 공통 인터페이스 정의 |
| `stream_manager.cpp` | 매니저 | URI 파싱 후 소스 인스턴스 생성 |
| `pipe_stream.cpp` | Named Pipe | `/tmp/snapfifo` 등 FIFO 파일 |
| `alsa_stream.cpp` | ALSA | 리눅스 사운드 카드 직접 캡처 |
| `tcp_stream.cpp` | TCP | 네트워크 스트림 수신 |
| `file_stream.cpp` | 파일 | WAV / FLAC 파일 재생 |
| `process_stream.cpp` | 프로세스 | 외부 프로세스 stdout 캡처 |
| `librespot_stream.cpp` | Spotify | librespot 프로세스 연동 |
| `airplay_stream.cpp` | AirPlay | AirPlay 수신기 |
| `meta_stream.cpp` | 메타 스트림 | 여러 스트림 믹싱 |
| `jack_stream.cpp` | JACK | JACK 오디오 서버 연동 |
| `pipewire_stream.cpp` | PipeWire | PipeWire 오디오 캡처 |
| `asio_stream.hpp` | ASIO 추상화 | 비동기 스트림 기반 인터페이스 |

### URI 형식 예시

```ini
[stream]
source = pipe:///tmp/snapfifo?name=Radio&sampleformat=48000:16:2&codec=flac
source = alsa:///?name=Line-In&device=hw:0,0
source = tcp://0.0.0.0:4953?name=MPD&mode=server
source = spotify:///librespot?name=Spotify
source = file:///home/user/music.wav?name=File
```

---

## 인코더 (`encoder/`)

팩토리 패턴으로 인코더를 생성한다. 모든 인코더는 `Encoder` 기반 클래스를 상속한다.

| 파일 | 코덱 | 특징 |
|------|------|------|
| `flac_encoder.cpp` | FLAC | 무손실, **기본값** |
| `opus_encoder.cpp` | Opus | 손실, 저지연 |
| `ogg_encoder.cpp` | Ogg/Vorbis | 손실 |
| `pcm_encoder.cpp` | PCM | 무압축 |
| `null_encoder.cpp` | 없음 | 테스트용 패스스루 |
| `encoder_factory.cpp` | 팩토리 | URI의 codec 파라미터로 인스턴스 결정 |

---

## 스트림 메타데이터 및 제어

| 파일 | 역할 |
|------|------|
| `metadata.cpp` | 트랙 정보, 앨범 아트 처리 |
| `properties.cpp` | 스트림 재생 상태, 속성 |
| `stream_control.cpp` | 재생/일시정지/다음 트랙 제어 |
| `watchdog.cpp` | 스트림 상태 모니터링, 재시작 처리 |
| `control_error.cpp` | 스트림 제어 에러 정의 |

---

## 플러그인

`server/etc/plug-ins/` 에 스트림 제어 플러그인을 배치할 수 있다. 스크립트 기반으로 외부 플레이어(MPD, Mopidy 등)를 제어하는 용도로 활용한다.
