# kgdb-with-vscode-yocto

UART(KGDBOC)로 Yocto 커널을 GDB/VSCode에서 디버깅하는 최소 설정 모음.
<> 안의 값은 당신 환경에 맞게 교체하세요. 맨 아래에 실제 경로/장치 예시도 있습니다.

----------------------------------------------------------------
0) 개요
----------------------------------------------------------------
- 연결 방식: UART(KGDBOC)
- 멈추는 방법: SYSRQ-g (Ctrl-C 대신)
- 주소 일치: 부팅 시 nokaslr 또는 CONFIG_RANDOMIZE_BASE=n
- 권장 흐름: (1) 커널 정지 → (2) 브레이크 설정 → (3) 계속

----------------------------------------------------------------
1) 커널 설정 (menuconfig)
----------------------------------------------------------------
# 선택(편의): CONFIG_GDB_SCRIPTS = y
CONFIG_KGDB = y
CONFIG_KGDB_SERIAL_CONSOLE = y
CONFIG_MAGIC_SYSRQ = y
CONFIG_DEBUG_INFO = y
CONFIG_FRAME_POINTER = y
CONFIG_DEBUG_INFO_REDUCED = n
CONFIG_RANDOMIZE_BASE = n   # ← 또는 부팅 파라미터에 nokaslr 사용

# 팁
# - CONFIG_GDB_SCRIPTS=y면 GDB에서 lx-* 헬퍼 사용 가능
# - 전역 -Og는 비추. 필요 파일만 -Og/-fno-inline 등으로 빌드

----------------------------------------------------------------
2) (WSL2) USB-Serial 패스스루
----------------------------------------------------------------
# 장치 확인
usbipd list

# (처음 1회) 지정 버스ID 바인드
usbipd bind --busid <your-busid>

# WSL2에 붙이기
usbipd attach --wsl --busid <your-busid>

# 분리
usbipd detach --busid <your-busid>

----------------------------------------------------------------
3) KGDB 진입 (타깃 보드)
----------------------------------------------------------------
[A] 런타임(이미 부팅 후)
echo <board-uart>,115200 > /sys/module/kgdboc/parameters/kgdboc
echo 1 > /proc/sys/kernel/sysrq
echo g > /proc/sysrq-trigger

# <board-uart> 예: ttyLP0, ttymxc0 등 보드 콘솔 이름

[B] 부팅 중(더 이른 시점)
setenv bootargs 'console=<board-uart>,115200 root=<rootdev> rw rootwait kgdboc=<board-uart>,115200 kgdbwait nokaslr'
saveenv
boot

# 주의: 인자 구분자는 "공백"입니다. 세미콜론(;) 쓰지 말 것.

----------------------------------------------------------------
4) 호스트 GDB 세션 (순정 GDB)
----------------------------------------------------------------
# vmlinux(심볼 포함) 지정
(gdb) file <path to your dir>/vmlinux

# Yocto/디버그 경로 매핑(필요 시)
(gdb) set substitute-path /usr/src/kernel <path to your kernel source>

# 시리얼 속도
(gdb) set serial baud 115200
(gdb) show serial baud

# 타깃 접속 (시리얼 장치 파일)
(gdb) target remote <path to your debug serial>
