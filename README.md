# kgdb-with-vscode-yocto

UART(KGDBOC)로 Yocto 커널을 GDB/VSCode에서 디버깅

---
# 개요

- 연결 방식: UART(KGDBOC)
- 멈추는 방법: SYSRQ-g (Ctrl-C 대신)
- 부팅 시 nokaslr 또는 커널 menuconfig에서 CONFIG_RANDOMIZE_BASE=n 지정해주면 좋음
- 권장 사항 : 디버깅용 시리얼 포트와 콘솔용 시리얼 포트를 분리하는게 좋음
---
# 커널 설정 (menuconfig)

```Kconfig
CONFIG_GDB_SCRIPTS = y
CONFIG_KGDB = y
CONFIG_KGDB_SERIAL_CONSOLE = y
CONFIG_MAGIC_SYSRQ = y
CONFIG_DEBUG_INFO = y
CONFIG_FRAME_POINTER = y
CONFIG_DEBUG_INFO_REDUCED = n
CONFIG_RANDOMIZE_BASE = n   # 부팅 파라미터에 nokaslr 사용
```

**추가 설명**
- CONFIG_GDB_SCRIPTS=y면 vmlinux-gdb.py 실행해서 gdb 변수 리스트 불러올 수 있음
- CONFIG_RANDOMIZE_BASE or bootargs에 nokaslr를 넘겨주면 커널 이미지 주소 랜덤화 안됨

---
# (WSL2) USB-Serial COM 포트 넘겨주기 (powershell - 관리자 권한)

## 장치 확인
```powershell
# usb 리스트 확인

usbipd list
```
리스트로 busid 확인하기

## usb wsl에 붙이기
```powershell
# usb bind
usbipd bind --busid <your-busid>

# usb wsl 연결
usbipd attach --wsl --busid <your-busid>
```

- 확인한 busid에서 com 포트에 해당하는 busid 넘겨주기

- 리눅스 쪽에서 시리얼 드라이버가 필요할 수 있음

## usb 빼기

```powershell
usbipd detach --busid <your-busid>
```

---
# KGDB 진입 (타깃 보드)

## 1. 런타임(이미 부팅 후)
```bash
# kgdb 시리얼 포트 설정 및 활성화
echo <board-uart>,115200 > /sys/module/kgdboc/parameters/kgdboc

# kgdb 커널을 멈추는 명령
# gdb로 vmlinux를 실행하면 stop 대신 아래 bash 스크립트를 사용한다.
# gdb에서는 커널 실행을 멈출 수 없다
echo g > /proc/sysrq-trigger
```

- board-uart : 시리얼 uart 포트

- board-uart 예: ttyLP0, ttymxc0 등 보드 콘솔 이름

- **주의)** break 포인트 삽입은 gdb가 멈춘 상태에서만 가능함
  
- 따라서 "echo g > /proc/sysrq-trigger" 이후 break point 삽입, 그 다음 continue

- 그래서, 콘솔 포트, kgdboc 포트 분리 하는것 추천


## 2. 부팅 할 때

**uboot**

```uboot
# 주의 : bootcmd에 의해서 bootargs가 덮어 씌워질 수 있으니 bootcmd도 설정하는거 권장함

setenv bootargs 'console=<board-uart>,115200 root=<rootdev> rw rootwait kgdboc=<board-uart>,115200 kgdbwait nokaslr'
setenv bootcmd "fatload mmc 0:1 <addr1> <Kernel Image>;fatload mmc 0:1 <addr2> <your dtb>;booti <addr1> - <addr2>"
```
boot

- board-uart : uart 시리얼 포트
- rootdev : 루트파일시스템 경로
- kgdbwait : gdb 붙을 때까지 커널 부팅 대기
- nokaslr : 커널 부트 이미지 랜덤화 x

---
# GDB (WSL2 GDB 사용)

**bash**

```bash
bitbake <your image> -c do_populate_sdk
```

- sdk 빌드하고 설치함. 여기서 환경 변수 가져와서 $GDB 사용할 것임

```bash
# 환경 변수 셋업
source /path/to/sdk/environment

#gdb 실행
$GDB <path to your dir>/vmlinux -tui
```

## yocto에서 빌드된 sdk의 gdb 사용하기


**gdb**

```gdb
# vmlinux 경로 지정, 위에서 $GDB <path to your dir>/vmlinux 열었으면 안해도 됨
# vmlinux 파일은 yocto build 디렉토리의 build/tmp/work/.../vmlinux 
file <path to your dir>/vmlinux

# Yocto/디버그 경로 매핑, 커널 소스는 build/tmp/work-shared/.../kernel-source에 있음
set substitute-path /usr/src/kernel <path to your kernel source>

# 시리얼 속도
set serial baud 115200

# 타깃 접속 (시리얼 장치 파일)
target remote <path to your debug serial>
```

---
# vscode 설정

**launch.json**

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "KGDB over Serial (ttyACM0)",
      "type": "cppdbg",
      "request": "launch",
      "program": "/home/leejunggyun/yocto/frdm/frdm-imx93/tmp/work/imx93frdm-poky-linux/linux-imx/6.6.36+git/build/vmlinux",
      "cwd": "${workspaceFolder}",
      "MIMode": "gdb",
      "miDebuggerPath": "/opt/frdm-imx93/sysroots/x86_64-pokysdk-linux/usr/bin/aarch64-poky-linux/aarch64-poky-linux-gdb",
      "stopAtEntry": false,
      "sourceFileMap": {
        "/usr/src/kernel": "/home/leejunggyun/yocto/frdm/frdm-imx93/tmp/work/imx93frdm-poky-linux/linux-imx/6.6.36+git/git"
      },
      "setupCommands": [
        { "text": "set serial baud 115200" },
        { "text": "set substitute-path /usr/src/kernel /home/leejunggyun/yocto/frdm/frdm-imx93/tmp/work/imx93frdm-poky-linux/linux-imx/6.6.36+git/git" },
        { "text": "target remote /dev/ttyACM0" }     
      ],
      "logging": {                                   
        "engineLogging": true,
        "trace": true,
        "traceResponse": true
      }
    }
  ]
}

```

1. program 지정
2. miDebuggerPath 지정
3. sourceFileMap 지정
4. setupCommands 지정
    - set serial baud <baud 설정>
    - set substitute-path /usr/src/kernel <path to kernel source>
    - target remote <tty시리얼>
5. logging 디버그 로그 (선택)

---
# 결과

<img width="1280" height="1391" alt="image" src="https://github.com/user-attachments/assets/d5ec655a-ade9-4118-802a-526ebaa9620f" />

## break point 삽입

<img width="1275" height="1386" alt="image" src="https://github.com/user-attachments/assets/2c2e4557-cb17-4016-a9e5-699a1163fb2f" />

- 동작 중에 break point 삽입하면 비활성화 됨 (gdb가 안멈췄기 때문)

- break point 새로 삽입하려면

1. echo g > /proc/sysrq-trigger (콘솔용 시리얼, 디버깅용 시리얼을 분리해둔 경우 콘솔용 시리얼에서 입력)
2. arch_kgdb_breakpoint 에 멈추는데 이 상태에서 break point를 삽입하거나, 혹은 2번을 실행하기 이전에 삽입해두면 됨

## 삽입 성공한 모습
<img width="2559" height="1388" alt="image" src="https://github.com/user-attachments/assets/1fe9f906-c302-430d-8598-fc3e4fc62c8d" />
