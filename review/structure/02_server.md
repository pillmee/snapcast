# Snapserver 분석

## 역할

오디오 소스를 읽어 인코딩 후 클라이언트에 배포한다. 동시에 JSON-RPC 컨트롤 서버를 통해 볼륨·그룹·스트림을 제어할 수 있는 관리 인터페이스를 제공한다.

---

## 코어 서버 파일

| 파일 | 역할 |
|------|------|
| [snapserver.cpp](../../server/snapserver.cpp) | 진입점 — CLI 파싱, 설정 로드, 데몬화 |
| [server.cpp](../../server/server.cpp) | 전체 조율 — 스트림 서버 + 컨트롤 서버 통합 |
| [config.cpp](../../server/config.cpp) | snapserver.conf 파싱 |
| [server_settings.hpp](../../server/server_settings.hpp) | 서버 설정 구조체 (포트, SSL, 인증 등) |
| [authinfo.hpp](../../server/authinfo.hpp) | Basic Auth / JWT 인증 처리 |
| [jwt.cpp](../../server/jwt.cpp) | JWT 토큰 생성 및 검증 |
| [image_cache.hpp](../../server/image_cache.hpp) | 앨범 아트 캐시 |

---

## 스트림 서버 (클라이언트 배포)

| 파일 | 역할 |
|------|------|
| [stream_server.cpp](../../server/stream_server.cpp) | TCP 포트 1704 — 클라이언트 연결 수락, PCM 청크 배포 |
| [stream_session.cpp](../../server/stream_session.cpp) | 개별 클라이언트 스트림 세션 (기본 클래스) |
| [stream_session_tcp.cpp](../../server/stream_session_tcp.cpp) | TCP 스트림 세션 구현 |
| [stream_session_ws.cpp](../../server/stream_session_ws.cpp) | WebSocket 스트림 세션 구현 |

---

## 컨트롤 서버 (JSON-RPC 관리)

| 파일 | 역할 |
|------|------|
| [control_server.cpp](../../server/control_server.cpp) | JSON-RPC 서버 (TCP 1705 / HTTP 1780 / WS 1788) |
| [control_session_tcp.cpp](../../server/control_session_tcp.cpp) | TCP 컨트롤 세션 |
| [control_session_http.cpp](../../server/control_session_http.cpp) | HTTP POST 및 WebSocket 컨트롤 세션 |
| [control_session_ws.cpp](../../server/control_session_ws.cpp) | WebSocket 전용 세션 처리 |
| [control_requests.cpp](../../server/control_requests.cpp) | RPC 요청 핸들러 (setVolume, getStatus 등) |

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
| [pcm_stream.cpp](../../server/streamreader/pcm_stream.cpp) | 기반 클래스 | 공통 인터페이스 정의 |
| [stream_manager.cpp](../../server/streamreader/stream_manager.cpp) | 매니저 | URI 파싱 후 소스 인스턴스 생성 |
| [pipe_stream.cpp](../../server/streamreader/pipe_stream.cpp) | Named Pipe | `/tmp/snapfifo` 등 FIFO 파일 |
| [alsa_stream.cpp](../../server/streamreader/alsa_stream.cpp) | ALSA | 리눅스 사운드 카드 직접 캡처 |
| [tcp_stream.cpp](../../server/streamreader/tcp_stream.cpp) | TCP | 네트워크 스트림 수신 |
| [file_stream.cpp](../../server/streamreader/file_stream.cpp) | 파일 | WAV / FLAC 파일 재생 |
| [process_stream.cpp](../../server/streamreader/process_stream.cpp) | 프로세스 | 외부 프로세스 stdout 캡처 |
| [librespot_stream.cpp](../../server/streamreader/librespot_stream.cpp) | Spotify | librespot 프로세스 연동 |
| [airplay_stream.cpp](../../server/streamreader/airplay_stream.cpp) | AirPlay | AirPlay 수신기 |
| [meta_stream.cpp](../../server/streamreader/meta_stream.cpp) | 메타 스트림 | 여러 스트림 믹싱 |
| [jack_stream.cpp](../../server/streamreader/jack_stream.cpp) | JACK | JACK 오디오 서버 연동 |
| [pipewire_stream.cpp](../../server/streamreader/pipewire_stream.cpp) | PipeWire | PipeWire 오디오 캡처 |
| [asio_stream.hpp](../../server/streamreader/asio_stream.hpp) | ASIO 추상화 | 비동기 스트림 기반 인터페이스 |

### URI 형식 예시

```ini
[stream]
source = pipe:///tmp/snapfifo?name=Radio&sampleformat=48000:16:2&codec=flac
source = alsa:///?name=Line-In&device=hw:0,0
source = tcp://0.0.0.0:4953?name=MPD&mode=server
source = spotify:///librespot?name=Spotify
source = file:///home/user/music.wav?name=File
```

### 실제 전송 메커니즘

모든 스트림은 서버의 단일 `boost::asio::io_context` 이벤트 루프 위에서 동작한다(PipeWire 제외). "비동기"는 별도 스레드 없이 콜백 기반으로 처리됨을 의미한다.

| scheme | 실제 전송 메커니즘 | 동기/비동기 |
|--------|---------------------|--------------|
| `pipe` | `mkfifo()` + `open(O_RDONLY\|O_NONBLOCK)` → `posix::stream_descriptor`로 감싸서 `AsioStream::do_read()`의 `async_read()` 사용 | 비동기 |
| `file` | 일반 파일을 `open(O_RDONLY\|O_NONBLOCK)` → 동일하게 `posix::stream_descriptor` + `async_read()` | 비동기 |
| `tcp` | `mode=server`: `tcp::acceptor::async_accept()` / `mode=client`: `tcp::socket::async_connect()` → 연결 후 동일한 `AsioStream::do_read()` 재사용 | 비동기 |
| `process` | Boost.Process로 외부 프로세스 실행(`bp::child(...)`), 자식의 stdout 파이프를 `O_NONBLOCK` 설정 후 `stream_descriptor`로 감싸 `AsioStream` 읽기 루프에 태움. stderr는 `async_read_until(..., "\n")`으로 별도 파싱(로그/워치독) | 프로세스 spawn만 동기, 이후 I/O는 비동기 |
| `spotify` / `librespot` | `ProcessStream` 상속 — `librespot ... --backend pipe` 실행, stdout에서 PCM 수신(메커니즘은 process와 동일, stderr 로그에서 트랙 메타데이터 파싱) | 비동기 |
| `airplay` | `ProcessStream` 상속 — `shairport-sync --output=stdout ...` 실행, stdout에서 PCM 수신. 추가로 별도 named pipe(`/tmp/shairmeta.*`)를 열어 메타데이터 XML을 비동기로 수신(Expat 파싱) | 비동기 |
| `alsa` | boost::asio 스트림 객체 없이 libasound 직접 호출: `snd_pcm_open(..., CAPTURE, NONBLOCK)` → `snd_pcm_readi()`로 폴링 읽기, `steady_timer`로 직접 페이싱 | 비동기(non-blocking 폴링, 스레드 없음) |
| `pipewire` | libpipewire 자체 이벤트 루프(`pw_main_loop_run`)를 전용 스레드에서 실행, `on_process` 콜백에서 버퍼 획득 후 `boost::asio::post(strand_, ...)`로 서버 io_context에 전달 | 별도 스레드(유일하게 진짜 백그라운드 스레드 사용) |
| `jack` | libjack 콜백 API — JACK이 자체 관리하는 실시간 오디오 스레드가 `readJackBuffers()` 콜백을 직접 호출, float→PCM 변환 후 처리 | JACK의 RT 스레드(Snapcast가 만든 스레드 아님) |
| `meta` | 실제 I/O 없음 — 다른 `PcmStream`들의 `Listener`로 등록되어 활성 소스의 `onChunkRead`를 중계/리샘플링만 함 | 해당 없음(순수 인메모리 라우팅) |

대부분의 소스(pipe/file/tcp/process 계열)는 [asio_stream.hpp](../../server/streamreader/asio_stream.hpp)의 공통 `AsioStream<ReadStream>` 템플릿을 통해 "청크 하나 읽기 → `steady_timer`로 다음 읽기 시각까지 대기"하는 동일한 페이싱 패턴을 공유한다. ALSA는 이 템플릿을 쓰지 않고 직접 폴링하며, PipeWire/JACK만 각 라이브러리의 자체 콜백/스레드 모델을 그대로 사용한다.

---

## 인코더 (`encoder/`)

팩토리 패턴으로 인코더를 생성한다. 모든 인코더는 `Encoder` 기반 클래스를 상속한다.

| 파일 | 코덱 | 특징 |
|------|------|------|
| [flac_encoder.cpp](../../server/encoder/flac_encoder.cpp) | FLAC | 무손실, **기본값** |
| [opus_encoder.cpp](../../server/encoder/opus_encoder.cpp) | Opus | 손실, 저지연 |
| [ogg_encoder.cpp](../../server/encoder/ogg_encoder.cpp) | Ogg/Vorbis | 손실 |
| [pcm_encoder.cpp](../../server/encoder/pcm_encoder.cpp) | PCM | 무압축 |
| [null_encoder.cpp](../../server/encoder/null_encoder.cpp) | 없음 | 테스트용 패스스루 |
| [encoder_factory.cpp](../../server/encoder/encoder_factory.cpp) | 팩토리 | URI의 codec 파라미터로 인스턴스 결정 |

---

## 스트림 메타데이터 및 제어

| 파일 | 역할 |
|------|------|
| [metadata.cpp](../../server/streamreader/metadata.cpp) | 트랙 정보, 앨범 아트 처리 |
| [properties.cpp](../../server/streamreader/properties.cpp) | 스트림 재생 상태, 속성 |
| [stream_control.cpp](../../server/streamreader/stream_control.cpp) | 재생/일시정지/다음 트랙 제어 |
| [watchdog.cpp](../../server/streamreader/watchdog.cpp) | 스트림 상태 모니터링, 재시작 처리 |
| [control_error.cpp](../../server/streamreader/control_error.cpp) | 스트림 제어 에러 정의 |

---

## 플러그인

[server/etc/plug-ins/](../../server/etc/plug-ins/) 에 스트림 제어 플러그인을 배치할 수 있다. 스크립트 기반으로 외부 플레이어(MPD, Mopidy 등)를 제어하는 용도로 활용한다.
