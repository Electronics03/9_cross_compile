# 임베디드 시스템 설계실습 — 9주차 실습 명령어 정리

> **주제:** Linux Kernel 헤더 점검 + 커널 크로스 컴파일
> **환경:**
> - **Host PC:** Ubuntu (VirtualBox 가상머신, 8 CPU / 15GB RAM / 디스크 77GB 여유)
> - **Target Board:** Raspberry Pi 4B (Bookworm Legacy 32-bit, kernel 6.12.75+rpt-rpi-v7l)
> - **크로스 툴체인:** `arm-linux-gnueabihf-gcc`

---

## 목차

- [Phase 1. Raspberry Pi에서 커널 헤더 점검 (PDF1)](#phase-1)
  - [1-1. 커널 버전 확인](#1-1)
  - [1-2. config.txt 수정 (v8로 부팅됐을 때만)](#1-2)
  - [1-3. 커널 헤더 심볼릭 링크 확인](#1-3)
  - [1-4. 모듈 빌드 파일 확인](#1-4)
  - [1-5. 오류 처리](#1-5)
- [Phase 2. Ubuntu Host PC에서 커널 크로스 컴파일 (PDF2)](#phase-2)
  - [2-1. 환경 사전 점검](#2-1)
  - [2-2. Root 권한 + 작업 디렉터리](#2-2)
  - [2-3. 빌드 도구 설치](#2-3)
  - [2-4. 커널 소스 클론](#2-4)
  - [2-5. 환경 변수 설정](#2-5)
  - [2-6. 라즈베리파이 4 기본 설정 로드](#2-6)
  - [2-7. 본 컴파일](#2-7)
- [Phase 3. SD카드에 커널 기록](#phase-3)
  - [3-1. SD카드 마운트 확인](#3-1)
  - [3-2. 모듈 설치](#3-2)
  - [3-3. 커널 이미지 + DTB 복사](#3-3)
  - [3-4. 마운트 해제](#3-4)
- [Phase 4. 새 커널로 부팅 검증](#phase-4)
- [부록. 자주 쓰는 명령어 / 트러블슈팅](#부록)

---

## 개념 요약 — 무엇을 빌드하는가?

리눅스 시스템은 **3개 층**으로 이루어져있다.

```
┌──────────────────────────────┐
│   User Space — Applications  │  ← 앱 (vim, Chromium 등)
├──────────────────────────────┤
│   Root File System           │  ← /bin, /etc, /lib, /usr 등
├──────────────────────────────┤
│   Kernel  ← 이번 실습 빌드 대상 │
├──────────────────────────────┤
│   Hardware                   │
└──────────────────────────────┘
```

이번 실습 빌드 대상 / 비대상:

| 부품 | 빌드 여부 | 비고 |
|------|----------|------|
| 커널 (kernel7l.img) | ✓ | 직접 빌드해서 SD카드에 덮어씀 |
| 커널 모듈 (.ko) | ✓ | rootfs의 /lib/modules에 추가 |
| Device Tree (.dtb) | ✓ | bootfs로 복사 |
| Root File System | ✗ | 기존 SD카드 그대로 사용 |
| 부트로더 | ✗ | 라즈베리파이 펌웨어 |
| 유저 앱 | ✗ | apt로 이미 설치됨 |

크로스 컴파일러(`arm-linux-gnueabihf-gcc`)를 쓰는 이유: **x86 PC에서 ARM CPU용 바이너리를 만들기 위해**.

---

<a id="phase-1"></a>
## Phase 1. Raspberry Pi에서 커널 헤더 점검 (PDF1)

> **위치:** Raspberry Pi 보드 터미널 (`pi@raspberrypi:~$`)

<a id="1-1"></a>
### 1-1. 커널 버전 확인

```bash
uname -r
```

**기대 결과:**
```
6.12.75+rpt-rpi-v7l
```

| 결과 끝 | 의미 | 다음 단계 |
|--------|------|----------|
| `-v7l` | 32-bit 커널 정상 | 1-3으로 |
| `-v8` | 64-bit 커널 (문제) | 1-2로 |

<a id="1-2"></a>
### 1-2. config.txt 수정 (v8로 부팅됐을 때만)

```bash
# 부팅 가능한 커널 이미지 확인
ls -l /boot/firmware/kernel*.img

# config.txt 편집
sudo nano /boot/firmware/config.txt
```

`[all]` 항목 아래에 두 줄 추가:
```
[all]
arm_64bit=0
kernel=kernel7l.img
```

저장: `Ctrl+O` → Enter → `Ctrl+X`

```bash
sudo reboot
# 재부팅 후
uname -r
# → 6.12.75+rpt-rpi-v7l 로 변경 확인
```

<a id="1-3"></a>
### 1-3. 커널 헤더 심볼릭 링크 확인

```bash
ls -ld /lib/modules/$(uname -r)/build
```

**기대 결과:**
```
lrwxrwxrwx ... /lib/modules/6.12.75+rpt-rpi-v7l/build
              -> /usr/src/linux-headers-6.12.75+rpt-rpi-v7l
```

<a id="1-4"></a>
### 1-4. 모듈 빌드 파일 확인

```bash
ls /lib/modules/$(uname -r)/build/include/config/auto.conf
ls /lib/modules/$(uname -r)/build/Module.symvers
```

두 명령 다 에러 없이 경로가 출력되면 OK.

<a id="1-5"></a>
### 1-5. 오류 처리

#### (A) build 링크가 없거나 깨진 경우
```bash
ls /usr/src | grep linux-headers
sudo ln -s /usr/src/linux-headers-6.12.75+rpt-rpi-v7l \
           /lib/modules/6.12.75+rpt-rpi-v7l/build
```

#### (B) 헤더 패키지가 없는 경우
```bash
sudo apt update
sudo apt install raspberrypi-kernel-headers
```

#### (C) 부팅 실패 — config.txt 잘못 건드린 경우
SD카드를 PC/우분투VM에 꽂아 `bootfs` 파티션의 `config.txt`에서 다음 두 줄 삭제:
```
arm_64bit=0
kernel=kernel7l.img
```

---

<a id="phase-2"></a>
## Phase 2. Ubuntu Host PC에서 커널 크로스 컴파일 (PDF2)

> **위치:** Ubuntu 가상머신 터미널

<a id="2-1"></a>
### 2-1. 환경 사전 점검

```bash
df -h /     # 디스크 여유 (10GB+ 권장)
free -h     # RAM (4GB+ 권장)
nproc       # CPU 코어 수 (병렬 컴파일용)
```

**참고 — 본 실습 환경:**
| 항목 | 값 |
|------|-----|
| 디스크 여유 | 77GB ✓ |
| RAM | 15GB ✓ |
| CPU 코어 | 8 ✓ |
| 예상 컴파일 시간 | 약 20~40분 |

<a id="2-2"></a>
### 2-2. Root 권한 + 작업 디렉터리

```bash
sudo su
mkdir -p /work/achro-em
cd /work/achro-em
pwd
# → /work/achro-em
```

<a id="2-3"></a>
### 2-3. 빌드 도구 설치

```bash
apt update
apt install -y git bc bison flex libssl-dev make libc6-dev \
               libncurses5-dev build-essential gcc-arm-linux-gnueabihf
```

크로스 컴파일러 확인:
```bash
arm-linux-gnueabihf-gcc --version | head -1
# → arm-linux-gnueabihf-gcc (Ubuntu 11.x.x ...)
```

<a id="2-4"></a>
### 2-4. 커널 소스 클론

```bash
cd /work/achro-em
git clone --depth=1 --branch rpi-6.1.y https://github.com/raspberrypi/linux.git
cd linux
git branch
# → * rpi-6.1.y
ls
# → arch  drivers  fs  include  init  ipc  kernel  lib  mm  net ...
```

> `--depth=1`: 히스토리 없이 최신 스냅샷만 받음(용량 절감)

<a id="2-5"></a>
### 2-5. 환경 변수 설정

```bash
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-
export KERNEL=kernel7l

# 확인
echo $ARCH            # arm
echo $CROSS_COMPILE   # arm-linux-gnueabihf-
echo $KERNEL          # kernel7l
```

> ⚠ 이 환경 변수는 현재 터미널에서만 유효. 터미널 닫으면 다시 export 해야 한다.

<a id="2-6"></a>
### 2-6. 라즈베리파이 4 기본 설정 로드

```bash
make bcm2711_defconfig
ls -la .config
# → .config 파일 생성 확인
```

마지막 출력 줄에 `configuration written to .config`가 보이면 성공.

<a id="2-7"></a>
### 2-7. 본 컴파일 (20~40분)

```bash
make -j$(nproc) zImage modules dtbs
```

`-j$(nproc)` = 8코어 병렬 빌드.

빌드되는 세 가지:
- `zImage` — 압축된 커널 이미지 → `kernel7l.img`로 사용
- `modules` — 디바이스 드라이버 (.ko 파일 수천 개)
- `dtbs` — Device Tree Blob (하드웨어 정보)

성공 시 마지막 즈음 출력:
```
  Kernel: arch/arm/boot/zImage is ready
  ...
  LD [M]  drivers/.../xxx.ko
```

빌드 산출물 확인:
```bash
ls -la arch/arm/boot/zImage
ls arch/arm/boot/dts/*.dtb | head -5
ls arch/arm/boot/dts/overlays/*.dtbo | head -5
```

---

<a id="phase-3"></a>
## Phase 3. SD카드에 커널 기록

> **위치:** Ubuntu Host PC, root 권한, `/work/achro-em/linux` 디렉터리

### 사전 작업: SD카드 연결

1. 라즈베리파이에서 SD카드 빼기
2. SD카드 어댑터로 PC에 꽂기
3. VirtualBox 우측 하단 USB 아이콘 우클릭 → SD카드 리더 선택 (가상머신에 연결)

<a id="3-1"></a>
### 3-1. SD카드 파티션 확인

```bash
lsblk
```

**기대 결과:**
```
sda        ...   (호스트 PC 본체 — 건드리지 말 것)
└─sda1, sda2, sda5

sdb        7.4G   (← SD카드)
├─sdb1     ...   /media/사용자명/bootfs   ← FAT32 부트 파티션
└─sdb2     ...   /media/사용자명/rootfs   ← ext4 루트 파티션
```

> ⚠ **`sda`는 절대 건드리지 마라!** 호스트 우분투 본체다. SD카드는 `sdb` 또는 `mmcblk0` 등으로 잡힌다.

#### 자동 마운트 안 됐을 때 (수동 마운트)
```bash
mkdir -p /mnt/bootfs /mnt/rootfs
mount /dev/sdb1 /mnt/bootfs
mount /dev/sdb2 /mnt/rootfs
```

이 경우 아래 경로의 `/media/사용자명/`을 `/mnt/`로 바꿔서 진행한다.

<a id="3-2"></a>
### 3-2. 모듈 설치

> **사용자명**은 본인 우분투 사용자명(예: `park`). `whoami` 명령으로 확인.
> 본 실습 환경에서는 `/media/park/rootfs`.

```bash
# 자동 마운트인 경우
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- \
     INSTALL_MOD_PATH=/media/park/rootfs modules_install

# 수동 마운트인 경우
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- \
     INSTALL_MOD_PATH=/mnt/rootfs modules_install
```

<a id="3-3"></a>
### 3-3. 커널 이미지 + DTB + 오버레이 복사

```bash
# 1) 커널 이미지 (zImage → kernel7l.img로 이름 변경)
cp arch/arm/boot/zImage /media/park/bootfs/kernel7l.img

# 2) Device Tree Blob (.dtb)
cp arch/arm/boot/dts/*.dtb /media/park/bootfs/

# 3) Device Tree 오버레이
cp arch/arm/boot/dts/overlays/*.dtb* /media/park/bootfs/overlays/
cp arch/arm/boot/dts/overlays/README /media/park/bootfs/overlays/

# 4) 디스크 버퍼 강제 기록 (반드시!)
sync
```

> ⚠ `sync`는 빠뜨리면 안 된다. 디스크에 실제로 기록되지 않은 채 SD카드를 빼면 커널이 손상된다.

<a id="3-4"></a>
### 3-4. 마운트 해제

```bash
umount /media/park/bootfs
umount /media/park/rootfs

# 수동 마운트였다면
# umount /mnt/bootfs /mnt/rootfs
```

이후 SD카드 안전하게 분리.

---

<a id="phase-4"></a>
## Phase 4. 새 커널로 부팅 검증

1. SD카드를 라즈베리파이에 다시 꽂기
2. 전원 인가 → 부팅 대기
3. 부팅 후 터미널에서:

```bash
uname -r
# → 6.1.x-v7l+ 같은 새 빌드 버전이 보여야 한다

uname -a
# → 빌드 시각/호스트네임도 확인 가능

cat /proc/version
```

새 모듈 로드 확인:
```bash
ls /lib/modules/$(uname -r)
lsmod | head
```

부팅 성공 = **내가 빌드한 커널이 실제로 동작 중**.

부팅 실패 시 → Phase 1의 [1-5 (C)](#1-5) 복구 절차로 SD카드를 PC에 꽂아 `kernel7l.img`를 백업본으로 되돌리거나, 펌웨어 업데이트로 복구.

> 💡 **권장:** Phase 3 들어가기 전에 기존 `kernel7l.img`를 백업해두면 안전하다.
> ```bash
> cp /media/park/bootfs/kernel7l.img /media/park/bootfs/kernel7l.img.backup
> ```

---

## 부록

### 자주 쓰는 명령어 빠른 참조

| 목적 | 명령 |
|------|------|
| 커널 버전 확인 | `uname -r` |
| 커널 자세히 | `uname -a` |
| 빌드 정보 | `cat /proc/version` |
| 디스크 여유 | `df -h /` |
| RAM 상태 | `free -h` |
| CPU 코어 수 | `nproc` |
| 디스크 장치 트리 | `lsblk` |
| 모듈 목록 | `lsmod` |
| 사용자명 | `whoami` |

### 컴파일 다시 시작 / 정리

```bash
# 빌드 산출물만 삭제 (config 유지)
make clean

# config 포함 완전 초기화
make mrproper

# 멈춘 빌드 이어서 (이전 진행분 유지됨)
make -j$(nproc) zImage modules dtbs
```

### 환경변수 다시 export

터미널을 새로 열었을 때:
```bash
sudo su
cd /work/achro-em/linux
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabihf-
export KERNEL=kernel7l
```

### 트러블슈팅

| 증상 | 원인 / 해결 |
|------|------------|
| `make: command not found` | `apt install build-essential` 빠짐 |
| `arm-linux-gnueabihf-gcc: not found` | `apt install gcc-arm-linux-gnueabihf` 또는 환경변수 export 안 됨 |
| `*** No rule to make target 'bcm2711_defconfig'` | `linux/` 디렉터리가 아님. `cd /work/achro-em/linux` 확인 |
| 컴파일 중 `Killed` 메시지 | RAM 부족 → `-j` 값을 줄여서 재시도 (`-j4`, `-j2`) |
| `lsblk`에서 SD카드(sdb) 안 보임 | VirtualBox USB 패스스루 미설정 → 우측 하단 USB 아이콘에서 SD리더 선택 |
| 라즈베리파이 부팅 무한 루프 | Phase 1-5 (C) 복구 절차로 config.txt 또는 kernel7l.img 백업 복원 |
| `Permission denied` | `sudo su`로 root 권한 획득 |

### 작업 흐름 한 페이지 요약

```
[Raspberry Pi]                       [Ubuntu Host PC]
─────────────────                    ─────────────────────────
1. uname -r 확인
   (v7l 인지)
                                     2. sudo su
                                     3. mkdir /work/achro-em
                                        cd /work/achro-em
                                     4. apt install (도구)
                                     5. git clone (커널 소스)
                                     6. export 환경변수
                                     7. make bcm2711_defconfig
                                     8. make -j8 zImage modules dtbs
                                        ↓ (20~40분)
                                     [SD카드 PC에 꽂기]
                                     9. lsblk
                                     10. modules_install
                                     11. cp zImage → kernel7l.img
                                         cp *.dtb, overlays
                                     12. sync && umount
                                     [SD카드 빼기]
13. 라즈베리파이에 꽂기
14. 부팅
15. uname -r 확인
    (새 빌드 버전 출력)
```

---

## 참고

- 9주차 수업자료 (PDF) — Linux Kernel · Root File System
- 임베디드 리눅스 설계 및 응용 교재 — Chapter 7. Kernel
- Raspberry Pi 공식 커널 소스: https://github.com/raspberrypi/linux

---

> 작성: 박상혁 (20234690, 4조)
> 작성일: 2026-05-04
