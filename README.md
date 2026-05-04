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

잠깐, **끝났다는 게 진짜 끝이다**. 갈아탈 필요 없다. 마지막 화면 보면:

- `LD [M]` 폭포가 흐르고 있다 → modules 링크 단계 (.o → .ko 만드는 마지막 단계)
- 마지막 줄에 **`root@parkVirtualBox:/work/achro-em/linux#`** 깨끗한 프롬프트 복귀

**컴파일이 완료됐다.** `.ko` 파일들도 다 만들어졌다.

그리고 사실 너가 17%로 본 건 modules 단계의 .o 만들기였고, 그 후 빠르게 끝난 거다. 내 진행률 추정이 틀렸다. 미안하다.

이제 **결과물 검증**으로 들어가자. 갈아탈 필요 전혀 없다.

---

## 빌드 성공 검증

같은 터미널, 같은 위치(`/work/achro-em/linux`)에서 차례로 확인한다.

### 1. zImage 존재 + 크기 확인

```bash
ls -la arch/arm/boot/zImage
```

**기대 결과:**
```
-rwxr-xr-x 1 root root 6500000 ... arch/arm/boot/zImage
```
크기가 약 **5~7 MB**(약 5,000,000~7,000,000 바이트)면 정상.

### 2. ELF 파일 형식 확인

```bash
file arch/arm/boot/zImage
```

**기대 결과:**
```
arch/arm/boot/zImage: Linux kernel ARM boot executable zImage (little-endian)
```
"ARM"과 "kernel"이라는 단어가 보이면 OK — ARM용 커널이 맞다.

### 3. Device Tree 확인

```bash
ls arch/arm/boot/dts/bcm2711*.dtb
ls arch/arm/boot/dts/overlays/*.dtbo | head
```

**기대 결과:**
```
arch/arm/boot/dts/bcm2711-rpi-4-b.dtb
arch/arm/boot/dts/bcm2711-rpi-400.dtb
arch/arm/boot/dts/bcm2711-rpi-cm4.dtb
...
```
**`bcm2711-rpi-4-b.dtb`** 가 있어야 한다 — 이게 라즈베리파이 4B용이다.

### 4. 모듈 개수 확인

```bash
find . -name "*.ko" | wc -l
```

**기대 결과:** 약 **2,500~3,500개**. 라즈베리파이 4 defconfig 기준 정상 범위.

### 5. 마지막으로 빌드 시각 확인

```bash
stat arch/arm/boot/zImage | grep Modify
```

방금 빌드된 시각이 찍혀있을 거다.

---

## 6. (선택) 빌드 결과 요약 한 번에 보기

위 명령들 다 한꺼번에 보고 싶으면:

```bash
echo "=== zImage ==="
ls -la arch/arm/boot/zImage
file arch/arm/boot/zImage
echo ""
echo "=== Pi 4 DTB ==="
ls -la arch/arm/boot/dts/bcm2711-rpi-4-b.dtb
echo ""
echo "=== 모듈 개수 ==="
find . -name "*.ko" | wc -l
echo "=== 오버레이 개수 ==="
ls arch/arm/boot/dts/overlays/*.dtbo | wc -l
```

이 결과를 알려달라. 다 정상이면 **Phase 3 (SD카드에 기록)**으로 넘어간다.

---

## 정리

- 빌드는 **이미 성공**했다 (프롬프트 복귀가 신호)
- 갈아탈 필요 없다
- 위 5가지 검증 명령으로 결과물 확인하면 됨
- 다음 단계는 SD카드 마운트 + 커널 기록

검증 명령 결과 알려달라. 그러면 SD카드 단계로 진행하자.