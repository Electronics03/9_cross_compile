좋다. 그럼 **PDF1(라즈베리파이 헤더 점검) → PDF2(Ubuntu에서 커널 컴파일)** 순서로 가자. 

순서가 중요하다. PDF1을 먼저 끝내야 라즈베리파이가 v7l 커널로 부팅되어 있고, 그 상태에서 PDF2 컴파일 결과물을 SD카드에 덮어쓰면 깔끔하게 검증할 수 있다.

---

## Phase 1 — 라즈베리파이에서 (PDF1)

### Step 1-1. 커널 버전 확인
라즈베리파이 터미널에서:
```
uname -r
```
**기대:** `6.12.75+rpt-rpi-v7l` (끝이 v7l)

v8로 나오면 멈추고 알려달라. config.txt 수정 필요.

### Step 1-2. build 심볼릭 링크 확인
```
ls -ld /lib/modules/$(uname -r)/build
```
**기대:** `... -> /usr/src/linux-headers-6.12.75+rpt-rpi-v7l` 화살표가 보여야 한다.

### Step 1-3. 모듈 빌드 파일 두 개 확인
```
ls /lib/modules/$(uname -r)/build/include/config/auto.conf
ls /lib/modules/$(uname -r)/build/Module.symvers
```
**기대:** 두 명령 다 에러 없이 파일 경로가 그대로 출력.

만약 Step 1-2나 1-3에서 `No such file` 에러가 나면 멈추고 알려달라.

---

## Phase 2 — Ubuntu Host PC에서 커널 컴파일 시작 (PDF2)

Phase 1이 끝났으면 우분투 가상머신으로 돌아온다.

### Step 2-1. Host PC 환경 사전 점검 (중요)

컴파일 전에 디스크 여유 공간과 RAM부터 본다. 우분투 터미널에서:
```
df -h /
free -h
nproc
```

**판단 기준:**
- `df -h /` 결과의 `Avail`이 **최소 15GB 이상** 있어야 안전 (소스 + 빌드 결과)
- `free -h`의 `total`이 **2GB 이상**, 가능하면 4GB 이상이면 좋다
- `nproc`은 사용 가능한 CPU 코어 수 (컴파일 병렬도)

세 명령의 출력을 알려달라. 부족하면 컴파일 시작 전에 VirtualBox 설정에서 RAM/디스크 늘려야 한다.

### Step 2-2. Root 권한 획득
```
sudo su
```
프롬프트가 `$` → `#` 으로 바뀐다. 이후 모든 명령은 root로 실행한다.

### Step 2-3. 작업 디렉터리 생성 및 이동
```
mkdir -p /work/achro-em
cd /work/achro-em
pwd
```
**기대 출력:**
```
/work/achro-em
```

### Step 2-4. 빌드 도구 설치 (인터넷 필요, 5~10분)
```
apt update
apt install -y git bc bison flex libssl-dev make libc6-dev libncurses5-dev build-essential gcc-arm-linux-gnueabihf
```
중간에 패키지 다운로드 진행 표시가 쭉 나오고 마지막에 프롬프트(`#`)로 돌아오면 OK.

설치 끝났으면 크로스 컴파일러 버전 확인:
```
arm-linux-gnueabihf-gcc --version
```
첫 줄에 `arm-linux-gnueabihf-gcc (Ubuntu ...)` 같은 게 보이면 OK.

### Step 2-5. 커널 소스 클론 (5~10분, 약 1.5GB 다운로드)
```
cd /work/achro-em
git clone --depth=1 --branch rpi-6.1.y https://github.com/raspberrypi/linux.git
```

`--depth=1`은 히스토리 없이 최신 스냅샷만 받는 옵션이라 용량을 크게 줄여준다.

다 받으면 확인:
```
cd linux
git branch
```
**기대:**
```
* rpi-6.1.y
```

`ls`도 한번 쳐봐서 `arch/`, `kernel/`, `mm/`, `drivers/` 같은 디렉터리들이 보이면 PDF1에서 봤던 그 커널 소스 트리가 맞다.

---

## 여기서 일단 멈춘다

이 다음 단계는:
- 환경변수 설정 (export ARCH, CROSS_COMPILE, KERNEL)
- defconfig 로드 (make bcm2711_defconfig)
- **본 컴파일 (make -j$(nproc) zImage modules dtbs)** — 30분~1시간+ 소요

본 컴파일은 시간이 오래 걸려서 중간에 끊으면 손해다. 그 전에 환경 점검 단계를 확실히 끝내고 가는 게 좋다.

---

## 지금 할 일

순서대로 진행하면서 **각 단계 결과를 알려달라:**

1. **Phase 1** (라즈베리파이): Step 1-1, 1-2, 1-3 결과
2. **Phase 2-1** (우분투): `df -h /`, `free -h`, `nproc` 출력

특히 Phase 2-1의 디스크/메모리 여유가 부족하면 컴파일 도중 죽으니까 먼저 점검해야 한다.

진행하면서 막히는 게 있으면 어느 Step에서 어떤 메시지가 나왔는지 그대로 복사해서 알려달라.