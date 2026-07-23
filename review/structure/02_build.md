# 빌드 시스템 및 의존성 분석

## 빌드 시스템

- **CMake** 3.14 이상
- **언어**: C++17
- **코드 스타일**: Google C++ Style Guide (`.clang-format` 설정)
- **정적 분석**: clang-tidy, cppcheck (`.clang-tidy` 설정)
- **Sanitizer**: AddressSanitizer, ThreadSanitizer, UBSanitizer 지원

---

## 빌드 타겟

```cmake
add_executable(snapserver)     # 서버 바이너리
add_executable(snapclient)     # 클라이언트 바이너리
add_library(common STATIC)     # 공유 정적 라이브러리
```

---

## CMake 빌드 옵션

### 컴포넌트 선택

| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `BUILD_SERVER` | ON | Snapserver 빌드 |
| `BUILD_CLIENT` | ON | Snapclient 빌드 |
| `BUILD_TESTS` | OFF | 테스트 빌드 |
| `BUILD_SHARED_LIBS` | OFF | 정적/공유 라이브러리 선택 |

### 코덱

| 옵션 | 기본값 | 설명 |
|------|--------|------|
| `BUILD_WITH_FLAC` | ON | FLAC 코덱 (기본 코덱) |
| `BUILD_WITH_VORBIS` | ON | Ogg/Vorbis 코덱 |
| `BUILD_WITH_OPUS` | ON | Opus 저지연 코덱 |

### 오디오 백엔드

| 옵션 | 플랫폼 | 설명 |
|------|--------|------|
| `BUILD_WITH_ALSA` | Linux | ALSA 오디오 |
| `BUILD_WITH_PULSE` | Linux | PulseAudio |
| `BUILD_WITH_PIPEWIRE` | Linux | PipeWire |
| `BUILD_WITH_JACK` | Linux | JACK 오디오 서버 |

### 네트워크 및 기타

| 옵션 | 설명 |
|------|------|
| `BUILD_WITH_SSL` | OpenSSL (HTTPS/WSS 지원) |
| `BUILD_WITH_AVAHI` | mDNS 서비스 디스커버리 (Linux) |

---

## 의존성

### 필수 의존성

| 라이브러리 | 버전 | 용도 |
|-----------|------|------|
| Boost | ≥ 1.74 | ASIO (비동기 I/O), program_options (CLI), process (프로세스 관리) |
| soxr | - | 오디오 리샘플링 (시간 동기화 속도 보정) |

### 코덱 의존성

| 라이브러리 | 코덱 |
|-----------|------|
| libFLAC | FLAC 무손실 |
| libogg | Ogg 컨테이너 |
| libvorbis | Vorbis 손실 |
| libopus | Opus 저지연 |

### 오디오 I/O 의존성

| 라이브러리 | 플랫폼 |
|-----------|--------|
| ALSA (libasound) | Linux |
| PulseAudio | Linux |
| PipeWire | Linux |
| JACK | Linux |
| CoreAudio (시스템) | macOS |
| WASAPI (시스템) | Windows |
| Oboe | Android |
| OpenSL ES (시스템) | Android |
| SDL2 | 크로스플랫폼 |

### 네트워크 / 기타

| 라이브러리 | 용도 |
|-----------|------|
| OpenSSL | HTTPS, WSS (TLS) |
| Avahi | mDNS 서비스 발견 (Linux) |
| Expat | XML 파싱 |

### 헤더 전용 (내장)

| 라이브러리 | 파일 |
|-----------|------|
| nlohmann/json | [common/json.hpp](../../common/json.hpp) |
| aixlog | [common/aixlog.hpp](../../common/aixlog.hpp) |
| popl | [common/popl.hpp](../../common/popl.hpp) |

---

## 플랫폼 지원

| 플랫폼 | 지원 |
|--------|------|
| Linux | 완전 지원 (ALSA, PulseAudio, PipeWire, JACK) |
| macOS | CoreAudio |
| Windows | WASAPI |
| Android | Oboe, OpenSL ES |
| FreeBSD | 지원 |
| OpenWrt | 크로스컴파일 |
| Raspberry Pi | 크로스컴파일 |
| Buildroot | 크로스컴파일 |
| webOS | SDL2 |

### Android 크로스컴파일

- 아래 빌드 스크립트를 사용한다.
  - [server/build_android.sh](../../server/build_android.sh)
  - [client/build_android.sh](../../client/build_android.sh)
  - [client/build_android_all.sh](../../client/build_android_all.sh)

---

## 빌드 예시 (Linux)

```bash
# 의존성 설치 (Debian/Ubuntu)
sudo apt-get install build-essential cmake \
    libboost-all-dev libssl-dev libsoxr-dev \
    libflac-dev libvorbis-dev libopus-dev \
    libasound2-dev libpulse-dev libavahi-client-dev

# 빌드
cmake -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j$(nproc)
```

---

## 패키징

- 배포 패키지 빌드 스크립트는 [extras/package/](../../extras/package/) 디렉토리에 있다.

| 디렉토리 | 패키지 형식 |
|----------|-----------|
| [extras/package/debian/](../../extras/package/debian/) | Debian/Ubuntu (.deb) |
| [extras/package/rpm/](../../extras/package/rpm/) | Fedora/CentOS (.rpm) |
| [extras/package/mac/](../../extras/package/mac/) | macOS Homebrew |

---

## CI/CD

- GitHub Actions 워크플로우는 [.github/](../../.github/) 디렉토리에 있다.
- 여러 플랫폼에서 빌드와 테스트를 자동으로 실행한다.
