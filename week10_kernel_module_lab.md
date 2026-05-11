# Week 10 · Kernel Module 실습 정리

> 임베디드 시스템 설계실습 · 2026 Spring
> Part 1 · Chapter 9 — Kernel Module Programming

---

## 📁 실습 디렉터리 준비 (Host PC)

관리자 모드로 진입한 뒤, 실습 디렉터리를 생성합니다.

```bash
sudo su
mkdir -p ~/module_test
cd ~/module_test
```

---

## 🔹 실습 ① · `hello_module.c` 소스 코드 작성

```bash
nano hello_module.c
```

### `hello_module.c`

```c
/* hello_module.c */
#include <linux/kernel.h>
#include <linux/module.h>
#include <linux/init.h>

MODULE_LICENSE("GPL");

static int module_begin(void)
{
    printk(KERN_ALERT "Hello, Wellcome to module!!\n");
    return 0;
}

static void module_end(void)
{
    printk(KERN_ALERT "Goodbye, Exit module!!\n");
}

module_init(module_begin);
module_exit(module_end);
```

### 필수 헤더 3종

| 헤더 | 역할 |
|------|------|
| `linux/kernel.h` | `printk`, `KERN_*` 매크로 |
| `linux/module.h` | 모듈 등록 · 라이선스 매크로 |
| `linux/init.h` | `module_init` / `module_exit` |

---

## 🔹 실습 ② · `Makefile` 작성

```bash
nano Makefile
```

### `Makefile`

```makefile
# Makefile
obj-m := hello_module.o

KDIR := /lib/modules/$(shell uname -r)/build
PWD  := $(shell pwd)

default:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

> ⚠️ `Makefile`의 들여쓰기는 반드시 **Tab** 문자를 사용해야 합니다 (스페이스 X).

### 핵심 변수 설명

- **`obj-m`** : 모듈로 빌드할 대상 선언 (`obj-y`는 커널에 정적 포함)
- **`KDIR`** : 부팅 커널 헤더 트리 경로 (`/lib/modules/$(uname -r)/build`)
- **`M=$(PWD)`** : 외부 모듈 위치 지정 (external mode build)

---

## 🔹 실습 ③ · SCP 파일 전송 및 컴파일

### 1) Host PC에서 Raspberry Pi로 전송

```bash
# Target Board에 원격 접속하여 실습 디렉터리 생성
ssh pi@<raspberry-ip>
```

```bash
# Raspberry Pi 안에서 실행
mkdir -p /home/pi/module_test
exit
```

```bash
# Host PC로 돌아와서 scp로 파일 전송
scp hello_module.c Makefile pi@<raspberry-ip>:/home/pi/module_test/
```

> 💡 전송 전 라즈베리파이의 IP 주소(`hostname -I`)와 SSH 접속 가능 여부를 확인합니다.

### 2) Target Board(Raspberry Pi)에서 빌드

```bash
cd /home/pi/module_test
ls
# → hello_module.c  Makefile

# 빌드
make clean
make

# 결과물 확인
ls
# → hello_module.ko (및 .o, .mod, .mod.c, modules.order, Module.symvers 등)
```

✅ `hello_module.ko` 파일이 생성되면 빌드 성공.

---

## 🔹 실습 ④ · 모듈 적재 / 동작 확인 / 제거

```bash
# ① 모듈 적재
sudo insmod hello_module.ko

# ② 적재 확인
lsmod | grep hello
# → hello_module    12288    0

# ③ 커널 로그 확인
dmesg | tail
# → [ 1234.567890] Hello, Wellcome to module!!

# ④ 모듈 제거
sudo rmmod hello_module

# ⑤ 다시 로그 확인
dmesg | tail
# → [ 1240.123456] Goodbye, Exit module!!
```

### Checkpoint (문제 해결)

| 증상 | 원인 |
|------|------|
| `insmod` 거부 | version magic 불일치 |
| 로그 미출력 | `dmesg -w`로 실시간 확인 |
| `rmmod` 거부 | usage count ≠ 0 |

---

## 📚 모듈 관련 명령어 정리

| 명령어 | 역할 | 사용 예시 |
|--------|------|-----------|
| `insmod` | 커널 모듈을 직접 적재 (의존성 미처리) | `sudo insmod hello_module.ko` |
| `rmmod` | 실행 중인 커널 모듈 제거 | `sudo rmmod hello_module` |
| `lsmod` | 적재된 모듈 목록 · 크기 · usage count 표시 | `lsmod \| grep hello` |
| `depmod` | 모듈 간 의존성 정보 생성 · 갱신 | `sudo depmod -a` |
| `modprobe` | 의존성을 고려하여 모듈 적재/제거 | `sudo modprobe hello_module` |
| `modinfo` | 모듈의 라이선스 · 이름 · 버전 정보 표시 | `modinfo hello_module.ko` |

> 🔑 **Rule of Thumb**: 직접 만든 모듈은 `insmod` / `rmmod`로, 시스템 모듈은 `modprobe`로 다룬다.

---

## 🔍 Usage Count (참조 횟수) 확인

```bash
# 적재된 모듈 목록과 사용 카운트 확인
lsmod
# Module          Size    Used by
# hello_module    16384   2       ← count=2 이면 rmmod 거부됨

# 사용 중인 모듈 제거 시도 (실패 예시)
sudo rmmod hello_module
# → rmmod: ERROR: Module hello_module is in use
```

### Linux 6.x 함수 인터페이스 (참고)

```c
try_module_get(THIS_MODULE);   // 참조 카운트 증가
module_put(THIS_MODULE);       // 참조 카운트 감소
module_refcount(THIS_MODULE);  // 현재 참조 카운트 확인
```

---

## 🌐 (참고) 네트워크 디바이스 확인 명령

Part 2 · Device Driver 챕터에서 등장하는 네트워크 디바이스 확인 명령:

```bash
cat /proc/net/dev
```

출력 예시:
```
Inter-|   Receive                          |  Transmit
 face |bytes    packets errs drop fifo frame|bytes    packets
   lo:    8462      94    0    0    0     0   8462      94
 eth0: 1029384   8127    0    0    0     0  58302     742
 wlan0:  204812   1521    0    0    0     0  31210     412
```

---

## 📌 5단계 라이프사이클 요약

```
① 소스 작성 (hello_module.c)
        ↓
② 컴파일 (make)         → hello_module.ko 생성
        ↓
③ 적재 (insmod)         → module_init() 실행
        ↓
④ 동작 중 (running)     → printk → dmesg로 확인
        ↓
⑤ 제거 (rmmod)          → module_exit() 실행
```
