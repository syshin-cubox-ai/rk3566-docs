# Firefly Core-3566JD4 개발 매뉴얼 정리 (한국어)

원문: [Firefly Wiki — Core-3566JD4 Manual](https://wiki.t-firefly.com/en/Core-3566JD4/index.html) · 문서 버전 **V2.0.2** (2022-05-10, Firefly Team)

RK3566 기반 Core-3566JD4 / AIO-3566JD4 개발 매뉴얼의 핵심 내용을 한국어로 재구성한 문서입니다.

## 목차

1. [제품 개요 및 하드웨어](#1-제품-개요-및-하드웨어)
2. [빠른 시작 (구성품 / 시리얼 디버그)](#2-빠른-시작)
3. [펌웨어 업그레이드](#3-펌웨어-업그레이드)
4. [Linux 펌웨어 빌드](#4-linux-펌웨어-빌드) — Ubuntu / Yocto / Buildroot
5. [Android 11.0 빌드](#5-android-110-빌드)
6. [드라이버](#6-드라이버)
7. [NPU (RKNN)](#7-npu-rknn)
8. [커널 / U-Boot](#8-커널--u-boot)
9. [액세서리](#9-액세서리)
10. [FAQ](#10-faq)
11. [인터페이스 정의 요약](#11-인터페이스-정의-요약)
12. [처음부터 재구축 — 펌웨어 플래싱 → RKNN 검증 (전체 런북)](#12-처음부터-재구축--펌웨어-플래싱--rknn-검증-전체-런북)

---

## 1. 제품 개요 및 하드웨어

Core-3566JD4는 **Rockchip RK3566** 쿼드코어 64bit Cortex-A55 프로세서(22nm) 기반의 AI 코어보드다. 듀얼코어 GPU와 고성능 NPU를 내장하며 최대 8GB RAM을 지원한다. 온보드 M.2, WiFi 2.4G/5G, 4G 모바일 네트워크를 지원하고, 스마트 NVR·클라우드 단말·IoT 게이트웨이·산업 제어 등에 적합하다. 실제 사용 시에는 **MB-JD4-RK3566 메인보드**와 결합한 **AIO-3566JD4** 형태로 동작한다.

### 지원 OS

| OS           | 커널 버전 | 지원 | 유지보수    |
| ------------ | --------- | ---- | ----------- |
| Linux        | 4.19      | O    | 중단        |
| Linux        | 5.10      | O    | 주 유지보수 |
| Android 11.0 | 4.19      | O    | 주 유지보수 |

- 소용량 메모리 버전은 기본 **Ubuntu** 설치, 대용량 메모리 버전은 기본 **Android** 설치.
- 다른 OS를 쓰려면 해당 펌웨어를 다운로드해 보드에 프로그래밍해야 한다.
- 펌웨어/자료 다운로드: `http://en.t-firefly.com/doc/download/123.html`

---

## 2. 빠른 시작

### 2.1 표준 구성품 (AIO-3566JD4)

- Core-3566JD4 코어보드 ×1
- MB-JD4-RK3566 메인보드 ×1
- 12V-2A 전원 어댑터 ×1
- USB Type-A ↔ Type-C 데이터 케이블 ×1

필요 시 추가로: HDMI 모니터/케이블, 유선 랜(100M/1000M)·라우터, USB 키보드/마우스, 적외선 리모컨(수신부 필요), 시리얼 디버그 어댑터 등.

### 2.2 시리얼 디버그

U-Boot / 커널 개발 시 부팅 로그 확인에 필수. 특히 GUI가 없을 때 유용하다.

**어댑터 선택 (칩별)**

| 칩     | 최대 baud | 권장   | 비고                          |
| ------ | --------- | ------ | ----------------------------- |
| CP2104 | 2 Mbps    | 권장   | 고속·안정성·내구성 우수       |
| CH340  | 2 Mbps    | 비권장 | 실제로 1.5Mbps 미달 사례 많음 |
| PL2303 | 1.2 Mbps  | 비권장 | 최대 baud < 1.5Mbps           |

> **주의:** AIO-3566JD4 기본 baud rate는 **1500000**. 일부 칩은 이 속도를 못 낸다. 같은 칩이라도 시리즈에 따라 다르므로 구매 전 확인.

**하드웨어 연결** (어댑터 4핀 중)

- 3.3V: 연결 불필요
- GND ↔ 보드 GND
- TXD ↔ 보드 RX
- RXD ↔ 보드 TX
- (입출력 안 되면 TX/RX를 바꿔서 시도)

**시리얼 파라미터**

- Baud: 1500000 / Data: 8 / Stop: 1 / Parity: none / Flow control: none

**Windows**: 드라이버(CH340/PL2303/CP210X) 설치 후 MobaXterm(무료판 권장) 또는 PuTTY/SecureCRT 사용. session→Serial, COM 포트 지정, Speed 1500000.

**Ubuntu (minicom 예시)**

```bash
sudo apt-get install minicom
ls /dev/ttyUSB*      # 예: /dev/ttyUSB0
sudo minicom
# Ctrl-A → Z (도움말) → O (설정) → Serial port setup
#   Serial Device: /dev/ttyUSB0
#   Bps/Par/Bits : 1500000 8N1
#   Hardware/Software Flow Control: No  (반드시 No)
# Save setup as dfl 로 기본값 저장
```

---

## 3. 펌웨어 업그레이드

### 3.1 동작 모드 개요

AIO-3566JD4는 **Normal 모드**(정상 부팅)와 **Upgrade 모드**(펌웨어 업그레이드)로 나뉜다. 부트 미디어는 eMMC / SDMMC.

**업그레이드 모드 3종 비교**

| 항목      | MaskRom 모드                                   | Loader 모드                          | SD 모드                  |
| --------- | ---------------------------------------------- | ------------------------------------ | ------------------------ |
| 진입      | 하드웨어 조작 필요                             | 키 또는 소프트웨어                   | 전원 인가 시 자동        |
| 연결      | USB                                            | USB                                  | TF(SD) 카드              |
| 전제 조건 | 하드웨어 조작                                  | uboot 정상 동작                      | 없음                     |
| 권장 상황 | 보드 부팅 불가 / Linux↔Android 교차            | 완전한 uboot 보유 / 파티션 개별 굽기 | 대량 생산 / 최종 고객용  |
| 장점      | 가장 기본적, 벽돌 복구 가능, 크로스시스템 지원 | 파티션 개별 굽기 가능                | 카드 꽂고 켜기만 하면 됨 |
| 단점      | 파티션 개별 굽기 어려움, 전체 지우고 굽기      | 완전한 loader 필요                   | 통합 펌웨어 합성 필요    |

- **MaskRom**: 부트로더 검증 실패/손상 시 BootRom이 진입. 벽돌 상태 복구용 "최후의 방어선".
- **Loader**: 부팅 시 `RECOVERY` 키 감지 + USB 연결 시 진입.
- **SD**: 부팅 가능한 SD-boot 업그레이드 펌웨어를 만들어 eMMC를 지우고 굽는 방식.

### 3.2 USB 케이블로 업그레이드

**펌웨어 종류 2가지**

- **통합 펌웨어(update.img)**: 파티션 테이블·부트로더·uboot·커널·시스템을 하나로 패키징. Firefly 공식 배포 형식. 굽으면 전체 파티션과 데이터가 갱신·삭제됨.
- **파티션 이미지**: 파티션별 개별 파일. 지정 파티션만 갱신, 개발 디버깅에 편리.

공개 펌웨어 패키지 구조 예:

```
XXXX_Android11_HDMI_XXXX
├── XXXX_Android11_HDMI_XXXX.img
├── linux/   Linux_Upgrade_Tool_v1.59.zip
└── windows/ DriverAssitant_v5.1.1.zip, RKDevTool_Release_v2.81.zip
```

**Windows 준비**

1. AndroidTool(RKDevTool) 사용. `config.ini`에서 `Selected=1`→`2`로 바꾸면 영어 표시.
2. `Release_DriverAssistant.zip`의 `DriverInstall.exe` 실행 → 먼저 **Driver uninstall** 후 **Driver install**.
3. 업그레이드 모드 진입:
   - 하드웨어: 전원 분리 → Type-C 연결 → `RECOVERY` 버튼 누른 채 약 2초 후 놓기.
   - 소프트웨어: 시리얼/adb 셸에서 `reboot loader`.
   - 장치관리자에 `Rockusb Device`가 보이면 성공.

**Linux 준비** (드라이버 설치 불필요)

```bash
unzip Linux_Upgrade_Tool_xxxx.zip
cd Linux_UpgradeTool_xxxx
sudo mv upgrade_tool /usr/local/bin
sudo chown root:root /usr/local/bin/upgrade_tool
sudo chmod a+x /usr/local/bin/upgrade_tool
```

> **Nor Flash 여부 확인 필수.** Nor Flash가 있으면 "Switching Upgrade Storage" 장을 참고해 저장소를 선택해야 한다.
> **extboot 주의:** Linux SDK v1.2.4a 이상은 extboot를 쓰므로 `boot.img` 대신 `extboot.img` 사용. 구버전 펌웨어에 extboot.img를 굽지 말 것. (Android은 무관)

SDK 버전 확인법: Buildroot `cat /etc/version`, Ubuntu `ffgo version`, 또는 `ls -l .repo/manifests/rk356x_linux_release.xml`.

**Windows 통합 펌웨어 굽기**: "upgrade firmware" 페이지 → firmware 버튼으로 update.img 열기 → upgrade.

**Windows 파티션 이미지 굽기**: "download image" 페이지 → Dev Partition → 파티션 체크 → 경로 확인 → Run.

**Linux 통합 펌웨어**

```bash
sudo upgrade_tool uf update.img
```

**Linux 파티션 이미지**

```bash
sudo upgrade_tool di -b     /path/to/boot.img
sudo upgrade_tool di -r     /path/to/recovery.img
sudo upgrade_tool di -m     /path/to/misc.img
sudo upgrade_tool di -u     /path/to/uboot.img
sudo upgrade_tool di -dtbo  /path/to/dtbo.img
sudo upgrade_tool di -p     paramater        # parameter 업그레이드
sudo upgrade_tool ul bootloader.bin          # bootloader 업그레이드
```

**Android fastboot (동적 파티션)**

```bash
adb reboot fastboot
sudo fastboot flash vendor vendor.img
sudo fastboot flash system system.img
sudo fastboot reboot
```

**업그레이드 FAQ**

- Loader 진입 실패 시 → MaskRom 강제 진입 시도.
- "Download Boot Fail"/에러 → 대개 USB 케이블 불량·저품질·PC USB 포트 구동력 부족. 포트 점검.
- Nor Flash + eMMC 동시 존재 시 MaskRom 후 저장소(Storage) 선택 필요.

### 3.3 MaskRom 모드 강제 진입

> 하드웨어 조작이 필요하고 위험이 따르므로, Loader 진입 실패 시에만 시도. **신중히 진행.**

1. 모든 전원 분리.
2. Type-C로 보드-PC 연결.
3. 금속 핀셋으로 보드의 지정 테스트 포인트 2곳을 단락(short)한 채 유지.
4. 전원 인가.
5. 수 초 후 단락 해제.

> Nor Flash가 있고 eMMC가 비어 있으며 Nor Flash에 파일이 구워져 있으면, **Nor Flash 근처 D0-GND 테스트 포인트**를 단락해 MaskRom 진입 후 "Switching Upgrade Storage" 절차로 업그레이드.

### 3.4 SD 카드로 업그레이드

업그레이드 카드 제작 툴로 MicroSD를 업그레이드 카드로 만든 뒤, 보드에 삽입·전원 인가하면 자동 업그레이드. 대량 생산·최종 고객용으로 편리. (Windows에서 카드 제작, Linux 미지원)

---

## 4. Linux 펌웨어 빌드

> 권장 환경: **Ubuntu 18.04**. 일반 사용자 계정으로 빌드(root 금지). SDK는 홈 아래 영문 경로에, VM 공유 폴더/비영문 경로 금지.

### 4.1 SDK 받기 (Linux 5.10)

```bash
sudo apt update
sudo apt install -y repo git python

mkdir ~/proj/rk356x_sdk-linux5.10
cd ~/proj/rk356x_sdk-linux5.10

# 전체 SDK
repo init --no-clone-bundle \
  --repo-url https://github.com/Firefly-rk-linux-utils/git-repo.git --no-repo-verify \
  -u https://github.com/Firefly-rk-linux/manifests.git -b master \
  -m rk356x_linux5.10_release.xml

# 또는 BSP (device/rockchip, docs, kernel, u-boot, rkbin, tools, 크로스 툴체인만)
#  -m rk356x_linux5.10_bsp_release.xml

# 코드 동기화
.repo/repo/repo sync -c --no-tags
.repo/repo/repo start firefly --all
```

**SDK 디렉터리 주요 구성**: `buildroot/`(루트파일시스템 빌드), `build.sh`(빌드 스크립트), `device/`(빌드 설정), `kernel/`, `u-boot/`, `prebuilts/`(크로스 툴체인), `rkbin/`, `tools/`.

**의존성 설치**

```bash
sudo apt-get install repo git ssh make gcc libssl-dev liblz4-tool \
expect g++ patchelf chrpath gawk texinfo diffstat binfmt-support \
qemu-user-static live-build bison flex fakeroot cmake \
unzip device-tree-compiler python-pip ncurses-dev python-pyelftools
```

(또는 Ubuntu 18.04 기반 **Docker** 이미지로 환경을 격리해 빌드 가능 — Dockerfile 제공)

### 4.2 Ubuntu 펌웨어 빌드

```bash
# 프리컴파일 설정 (보드 선택)
./build.sh firefly_rk3566_aio-3566-jd4_ubuntu_defconfig
```

주요 설정 항목:

- 디바이스 트리: `RK_KERNEL_DTS_NAME="rk3566-firefly-aiojd4"`
- 커널 defconfig: `RK_KERNEL_CFG="firefly_linux_defconfig"`
- 블루투스 UART: `RK_WIFIBT_TTY="ttyS8"`
- FIT 이미지 사용, extlinux(extboot) 방식 커널 로드

**액세서리별 defconfig** (`device/rockchip/rk3566_rk3568/`)

```
xxxx-ubuntu_defconfig         # HDMI + 단일 카메라 (기본)
xxxx-2cam_ubuntu_defconfig    # HDMI + 듀얼 카메라
xxxx-mipi_ubuntu_defconfig    # MIPI + 카메라
```

**루트파일시스템 준비**: Ubuntu rootfs(64bit) 다운로드 → 압축 해제 → `prebuilt_rootfs/`로 이동 후 `rk356x_ubuntu_rootfs.img`로 심볼릭 링크.

**빌드 명령**

```bash
./build.sh all          # 전체 자동 빌드 → output/update/
./build.sh uboot        # u-boot/uboot.img
./build.sh extboot      # kernel/extboot.img
./build.sh updateimg    # 통합 펌웨어 → rockdev/pack/
```

**파티션 테이블(parameter)**: `device/rockchip/rk3566_rk3568/parameter-ubuntu-fit.txt`. CMDLINE의 `mtdparts`에서 `0x크기@0x시작(파티션명)` 형식. 단위는 block(1 block = 512 Byte).

### 4.3 Yocto 펌웨어 빌드

```bash
repo init --no-clone-bundle \
  --repo-url https://gitlab.com/firefly-linux/git-repo.git \
  -u https://gitlab.com/firefly-linux/manifests.git -b master \
  -m rk356x_yocto_kirkstone_release.xml
.repo/repo/repo sync -c
```

**빌드 예시** (레이어 추가 후)

```bash
source oe-init-build-env
# bitbake-layers add-layer ... (meta-oe, meta-python, meta-networking 등)
MACHINE=aio-3566-jd4-kernel5-10 bitbake core-image-minimal
MACHINE=aio-3566-jd4-kernel5-10 bitbake core-image-weston   # wayland (display.conf 수정 필요)
```

- 제공 이미지: `core-image-minimal`, `-minimal-xfce`, `-sato`, `-weston`, `-x11`.
- 이미지 교체 시 이전 것을 `-c clean` 후 재빌드.
- 빌드 속도: `firefly-rk356x-kernel5-10.conf`의 `BB_NUMBER_THREADS`, `PARALLEL_MAKE` 조정.
- 결과물: `build/tmp/deploy/images/<board>/` (`.wic`, `update.img`).
- 기본 로그인: **root / firefly**, 일반 사용자 **firefly / firefly**.

### 4.4 Buildroot 펌웨어 빌드

```bash
./build.sh firefly_rk3566_aio-3566-jd4_buildroot_defconfig
./build.sh all           # 전체 → output/update/
./build.sh uboot         # u-boot/uboot.img
./build.sh extboot       # kernel/extboot.img
./build.sh recovery      # output/recovery
./build.sh buildroot     # 루트파일시스템 (일반 사용자로!)
./build.sh updateimg     # 통합 펌웨어 → output/update/
```

파티션 테이블: `parameter-buildroot-fit.txt`.

---

## 5. Android 11.0 빌드

**요구 사항**: DDR 최소 2GB, eMMC 최소 16GB.

### 5.1 SDK 준비

- 클라우드 디스크에서 `Firefly-RK356X_Android11.0_git` 다운로드 → `md5sum`으로 검증 → `7z`로 해제 → `git reset --hard`.
- 비공유·영문 경로에서만 해제.
- 원격 저장소(bundle) 업데이트: `git clone .../rk356x-android11-bundle.git .bundle` → `.bundle/update` → `git rebase FETCH_HEAD`.

### 5.2 제품별 컴파일 (첫 빌드는 반드시 전체 빌드)

```bash
# HDMI (공개 빌드)
./FFTools/make.sh -d rk3566-firefly-aiojd4 -j8 -l rk3566_firefly_aiojd4-userdebug
./FFTools/mkupdate/mkupdate.sh -l rk3566_firefly_aiojd4-userdebug

# MIPI 디스플레이 DM-M10R800 V2
./FFTools/make.sh -d rk3566-firefly-aiojd4-mipi101_M101014_BE45_A1 -j8 -l rk3566_firefly_aiojd4_mipi-userdebug

# 듀얼 카메라 / RK628D(HDMI→MIPI CSI)는 dts에서 include할 dtsi 교체 후 빌드
#   rk3566-firefly-aiojd4-cam-8ms1m.dtsi  (단일 카메라, 기본)
#   rk3566-firefly-aiojd4-cam-2ms2m.dtsi  (듀얼 카메라)
#   rk3566-firefly-aiojd4-tf-hdmi-mipi-rk628.dtsi (RK628D)
```

### 5.3 수동 빌드

```bash
export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH

# 커널
cd kernel/
make ARCH=arm64 firefly_defconfig android-11.config rk356x.config
make ARCH=arm64 BOOT_IMG=../rockdev/Image-rk3566_firefly_aiojd4/boot.img rk3566-firefly-aiojd4.img -j8

# uboot
cd u-boot/ && ./make.sh rk3566

# Android
source build/envsetup.sh
lunch rk3566_firefly_aiojd4-userdebug
make installclean && make -j8 && ./mkimage.sh

# 통합 펌웨어 패키징
./FFTools/mkupdate/mkupdate.sh -l rk3566_firefly_aiojd4-userdebug
```

---

## 6. 드라이버

### 6.1 GPIO

- 5개 뱅크(GPIO0~GPIO4), 각 뱅크는 A0~A7, B0~B7, C0~C7, D0~D7.
- **핀 계산식**: `pin = bank*32 + number`, `number = group*8 + X` (A=0,B=1,C=2,D=3).
  - 예) GPIO4_D5 → bank=4, group=3, X=5 → number=29 → pin=157.
- DTS 표기: `<&gpio4 29 ...>` 또는 매크로 `<&gpio4 RK_PD5 ...>`.

**sysfs로 수동 조작**

```bash
echo 157 > /sys/class/gpio/export
cat /sys/class/gpio/gpio157/direction   # in
cat /sys/class/gpio/gpio157/value
```

**DTS 예시**

```dts
gpio_demo: gpio_demo {
    status = "okay";
    compatible = "firefly,rk356x-gpio";
    firefly-gpio     = <&gpio0 12 GPIO_ACTIVE_HIGH>;         /* GPIO0_B4 */
    firefly-irq-gpio = <&gpio4 29 IRQ_TYPE_EDGE_RISING>;     /* GPIO4_D5 */
};
```

**주요 API**: `of_get_named_gpio_flags`, `gpio_is_valid`, `gpio_request`/`gpio_free`, `gpio_direction_input/output`, `gpio_get_value`, `gpio_to_irq`, `request_irq`/`free_irq`.

**인터럽트 트리거 타입**: `IRQ_TYPE_EDGE_RISING/FALLING/BOTH`, `IRQ_TYPE_LEVEL_HIGH/LOW`.

**핀 리유즈(멀티플렉싱)**: 한 핀이 GPIO/I2C/SPI 등 여러 func를 가질 수 있음. 다른 기능으로 점유 중이면 해당 노드를 `disabled` 처리해야 함. 런타임 전환은 `pinctrl-names`(default/gpio) + `pinctrl_lookup_state`/`pinctrl_select_state`로 처리.

**FAQ 요약**

- GPIO request 시 MUX가 강제로 GPIO로 전환됨 → 다른 모듈이 안 쓰는지 확인.
- IO 읽은 값이 0x00000000/0xffffffff → CLK가 꺼졌을 수 있음(CRU_CLKGATE_CON\* 확인).
- 핀 전압 이상 → IO 전압원·IO-Domain 설정 확인.
- 반복 제어 시 `gpio_direction_output` 대신 `gpio_set_value` 권장(전자는 내부 mutex로 인터럽트 컨텍스트에서 오류·낭비).

### 6.2 UART

지원: **UART3, UART5**(기본 비활성, SPI와 멀티플렉싱), **RS232(UART9)**, **RS485(UART0)**. 각 UART는 256-byte FIFO.

- UART3/UART5: TTL 레벨, RS232: RS232 레벨, RS485: RS485 레벨.

**디바이스 노드**

```
RS485 → /dev/ttyS0
RS232 → /dev/ttyS9
UART3 → /dev/ttyS3
```

**DTS** (`rk3566-firefly-aiojd4.dtsi`): `&uart0/&uart3/&uart9`를 `okay`로, 각 `pinctrl-0` 지정.

**RS485 테스트 예시** (호스트는 kermit, 9600 8N1, flow none)

```bash
echo "firefly RS485 test..." > /dev/ttyS0   # 송신
cat /dev/ttyS0                              # 수신
```

### 6.3 ADC

- **TS-ADC**(온도 센서): 2채널, 클럭 < 800KHz.
- **SAR-ADC**: 6채널 단선 10bit, 클럭 < 13MHz. 커널 IIO 서브시스템으로 제어.

**DTS 예시**

```dts
adc_demo: adc_demo {
    status = "okay";                 // 사용 시 okay
    compatible = "firefly,rk356x-adc";
    io-channels = <&saradc 5>;       // 채널 인덱스
};
```

**전압 환산식**: `Vref/(2^n − 1) = Vresult/raw`

- 예) Vref=1.8V, n=10, raw=568 → `Vresult = 1800mV * 568 / 1023`.

**주요 API**: `iio_channel_get`, `iio_channel_release`, `iio_read_channel_raw`.

**전체 SARADC 값 조회**

```bash
cat /sys/bus/iio/devices/iio\:device0/in_voltage*_raw
```

> **주의**: 드라이버 로드 시점이 saradc 초기화 이후여야 함. `module_init` 대신 `late_initcall()` 등 낮은 우선순위 사용.

### 6.4 PWM

- 드라이버: `kernel/drivers/pwm/pwm-rockchip.c`.
- **DTS 예시**

```dts
pwm_demo: pwm_demo {
    status = "okay";
    compatible = "firefly,rk356x-pwm";
    pwms = <&pwm1 0 10000 1>;   // pwm1 / period 10000ns / polarity 1
    duty_ns = <5000>;            // duty 활성 시간(ns)
};
```

- **API**: `pwm_request`, `pwm_free`, `pwm_config(pwm, duty_ns, period_ns)`, `pwm_enable`, `pwm_disable`.
- **디버그**: `cat /sys/kernel/debug/pwm`로 등록 상태 확인.

### 6.5 그 밖의 드라이버 (원문 참조 권장)

- **Camera**: MIPI CSI. Full/Split 모드 설정, 단일(CAM-8MS1M)/듀얼(CAM-2MS2MF) 카메라, Android 카메라 앱·Linux 프리뷰, IQ 파일.
- **I2C**: I2C 디바이스/드라이버 정의·등록.
- **IR**: 적외선 리모컨 설정, 커널 드라이버, 사용법.
- **LCD**: MIPI / EDP DTS 설정.
- **LED**: 디바이스 제어, trigger 제어.
- **RTC**: RTC 드라이버, 인터페이스 사용법.

---

## 7. NPU (RKNN)

RK3566 NPU는 **최대 1 TOPS** 성능. **RKNN SDK**로 프로그래밍하며, RKNN-Toolkit2로 변환한 `.rknn` 모델을 배포·가속한다. 지원 플랫폼·최신 정보는 [airockchip/rknn-toolkit2](https://github.com/airockchip/rknn-toolkit2/) 참고 권장.

### 7.1 RKNN 모델

- `.rknn` 확장자. RK3566에서 바로 실행. 데모는 `rknpu2/examples`.
- **YOLOv5 데모 실행 예시** (Android 기준, Linux는 `rknn_yolov5_demo_Linux`)

```bash
cd /data/rknn_yolov5_demo_Android/
chmod 777 rknn_yolov5_demo
export LD_LIBRARY_PATH=./lib
./rknn_yolov5_demo model/RK3566-RK3568/yolov5s-640-640.rknn ./model/bus.jpg resize ./out.jpg
# 참고: SDK 2.3.0 기준, 1회 추론 약 67ms, 10회 평균 약 59ms
```

### 7.2 Non-RKNN 모델

- Caffe/TensorFlow 등은 RKNN-Toolkit2로 `.rknn`으로 변환 후 실행.

### 7.3 RKNN-Toolkit2

PC와 Rockchip NPU에서 **모델 변환·추론·성능/메모리 평가·양자화 오차 분석**을 제공.

- 변환 지원: Caffe / TensorFlow / TFLite / ONNX / Darknet / PyTorch → RKNN.
- 양자화: asymmetric_quantized-8(및 -16, 단 -16 추론 미지원), 하이브리드 양자화.
- 시뮬레이터 내장(PC에서 NPU 없이 추론 가능), 실제 NPU 연결 실행도 지원.

**환경**: Ubuntu 18.04(x64)+ , Python 3.6/3.8 (Windows/macOS/Debian 미지원). PC 전용 설치.

**설치 (virtualenv 권장, Python 3.6 예시)**

```bash
sudo apt install virtualenv
sudo apt-get install python3 python3-dev python3-pip \
  libxslt1-dev zlib1g zlib1g-dev libglib2.0-0 libsm6 libgl1-mesa-glx libprotobuf-dev gcc
virtualenv -p /usr/bin/python3 venv
source venv/bin/activate
pip3 install -r doc/requirements_cp36-*.txt
sudo pip3 install packages/rknn_toolkit2*cp36*.whl
python3 -c "from rknn.api import RKNN"   # 오류 없으면 성공
```

**실제 NPU 연결 실행 준비**: 보드에 `librknnrt.so` 갱신 + `rknn_server` 실행.

- Android: `adb push ... librknnrt.so /vendor/lib(64)`, `rknn_server /vendor/bin`, 재부팅.
- Linux: `rknn_server`를 `/usr/bin`, `librknnrt.so`를 `/usr/lib`에 push 후 실행.
- 데모의 `test.py`에서 `rknn.config(..., target_platform='rk3566')`, `rknn.init_runtime(target='rk3566')`로 수정.

**상세 문서**: SDK 내 `Rockchip_RKNPU_User_Guide_RKNN_API_*.pdf`, `Rockchip_User_Guide_RKNN_Toolkit2_*.pdf`.

> 위 7.3은 **PC(x86) 기준 원문 설명**이다. RKNN-Toolkit2 2.x부터는 aarch64 휠이 제공되어 **보드 위에서 직접 변환·추론**이 가능하다. 실제 Core-3566JD4 보드에서 진행한 설치·변환·테스트 절차는 [12장](#12-처음부터-재구축--펌웨어-플래싱--rknn-검증-전체-런북) 참고.

---

## 8. 커널 / U-Boot

### 8.1 커널 커스터마이즈

Firefly 커널은 모든 기능이 켜져 있지 않음(예: USB CAN 미포함). 필요 기능은 직접 활성화 후 재빌드.

```bash
cd SDK/kernel
# .config 생성
make ARCH=arm64 firefly_linux_defconfig      # Linux
# make ARCH=arm64 firefly_defconfig           # Android
make ARCH=arm64 menuconfig                    # 기능 선택 (* build-in, M module)

# 설정 저장
make ARCH=arm64 savedefconfig
mv defconfig arch/arm64/configs/firefly_linux_defconfig   # (Android: firefly_defconfig)

# 빌드 (SDK 루트)
./build.sh xxxx.mk
./build.sh kernel        # 결과: SDK/kernel/boot.img
```

- 소수 기능 추가 시 후속 설치가 필요 없는 **build-in(Y)** 권장.
- 굽기: "파티션 이미지 업그레이드" 방식 참고.

### 8.2 U-Boot (원문 참조)

소개, 컴파일, 이미지 굽기(Flash Image), 새 Loader 업그레이드 검증, u-boot 커맨드라인 진입, 2차 Loader 등. (세부 명령은 원문 `uboot_introduction.html` 참고)

---

## 9. 액세서리

- **변환 모듈**: USB to TTL 시리얼 모듈.
- **스크린 모듈**: DM-M10R800 V2 MIPI 모듈.
- **카메라 모듈**: CAM-8MS1M(단안), CAM-2MS2MF(양안).
- **무선 모듈**: EC20 4G / EC200T 4G / GNSS 모듈.
- **전원 어댑터**: 12V.
- **리모컨**: 적외선 리모컨(파라미터·키코드).
- **SATA 어댑터**: AIO-3566JD4에 SATA 저장장치 연결(M.2 PCIe 멀티플렉싱). SATA 활성화 필요.
- **HDMI to MIPI CSI 보드**: RK628D 기반.

---

## 10. FAQ

- **커널 빌드 시 "IO-Domain Checklist" 대화상자**: 새 dts용 `.tmp.domain` 파일을 복사해 대응. **IO-Domain은 변경 금지**(최악의 경우 IO 손상).
- **RK3566 듀얼 디스플레이**: VOP가 "동일 소스 듀얼 디스플레이"라 주/보조 화면 방향·해상도를 동일하게 맞춰야 늘어짐(stretch) 방지.
- **커널/buildroot 설정 변경이 컴파일 후 반영 안 됨**: `.config`는 임시 파일이라 덮어써짐. `make ARCH=arm64 savedefconfig` 후 defconfig를 configs로 이동해 저장.
- **HDMI 4K 표시 안 됨**: 자동 4K 전환 미지원(기본 1080P). 수동 설정:
  ```
  setprop persist.sys.resolution.main 3840x2160@60
  setprop sys.display.timeline 1   # 설정할 때마다 +1
  ```
- **3.5mm 이어폰 이상**: 현재 **미국 표준(CTIA)만 지원**. 국내 표준(OMTP)은 하드웨어 비호환(좌우 채널 이중 소리).
- **부팅 이상·반복 재시작**: 전원 전류 부족 가능. **12V / 2.5A~3A** 전원 사용.
- **Ubuntu 기본 계정**: 사용자 `firefly` / 비밀번호 `firefly`, 슈퍼유저 전환 `sudo -s`.
- **SN·MAC 기록**: Windows는 RKDevInfoWriteTool(설정에서 RPMB 선택), Linux는 Buildroot `BR2_PACKAGE_VENDOR_STORAGE` 활성화 후 `vendor_storage` 명령. (eMMC erase 시 기록 데이터도 삭제됨)
  ```bash
  vendor_storage -w VENDOR_SN_ID -t string -i cad895bedb8ee15f
  vendor_storage -w VENDOR_LAN_MAC_ID -t string -i AABBCCDDEEFF
  ```
- **Ubuntu 이어폰 소리 없음**: PulseAudio Volume Control → Configuration에서 작동 사운드카드 선택, 나머지 끄기.
- **Android 로그 수집**: 개발자 옵션 활성화(빌드 번호 7회 탭) → Android LogSave 켜기 → `/data/vendor/logs`에 logcat·kmsg 생성.
- **Android boot.img 단독 컴파일 문제**: `BOOT_IMG=...` 지정 필요, 미지정 시 recovery 모드로 부팅.
  ```bash
  make ARCH=arm64 BOOT_IMG=../rockdev/Image-rk3566_firefly_aiojd4/boot.img rk3566-firefly-aiojd4.img -j8
  ```

---

## 11. 인터페이스 정의 요약

Core-3566JD4가 제공하는 주요 인터페이스:

- 전원 인터페이스, 12V 전원 입력
- USB3.0 ×1(host), USB2.0 ×4(seat), Type-C ×1(host/device)
- HDMI ×1, MIPI 스크린 ×2, EDP 스크린 ×1, MIPI 카메라 ×1
- 1G 이더넷 ×1, 2.4/5G WiFi ×1(+안테나), Bluetooth ×1
- MIC ×1, 3.5mm 이어폰잭 ×1, 스피커 ×1
- IR ×1, TF 카드 슬롯 ×1, SIM 슬롯 ×1
- 전원키/리셋키/Recovery키 각 ×1, Debug 시리얼 ×1
- 산업용 시리얼(RS485/RS232/TTL) ×1
- MINI-PCIE ×1(옵션 5G M.2 NGFF), M.2 PCIE2.0 ×1, SATA ×1(M.2와 멀티플렉싱)
- Nor Flash ×1

---

## 12. 처음부터 재구축 — 펌웨어 플래싱 → RKNN 검증 (전체 런북)

보드에 펌웨어를 새로 구운 뒤 RKNN 환경을 구축하고 YOLO11로 검증하기까지의 전체 절차다.

> **작업 환경 전제**
>
> - **`[PC]` 표시 단계는 Windows에서 PowerShell로 실행한다.** (Linux·Mac 호스트 절차는 이 장에서 다루지 않는다 — 필요하면 [3장](#3-펌웨어-업그레이드) 참고)
> - **`[보드]` 표시 단계는 보드의 Ubuntu 셸(bash)에서 실행한다.** SSH 또는 시리얼로 접속한 상태를 가정한다.

### 12.1 [PC] 준비물 내려받기 & 압축 풀기

- [다운로드 페이지](http://en.t-firefly.com/doc/download/123.html)에서 한 폴더에 모두 다운로드한다.
- 해당 폴더에서 `터미널에서 열기`를 클릭하여 powershell을 연다.
- 툴킷과 Model Zoo는 **버전을 맞춘다**(둘 다 2.3.2).

| 파일 (ABC순)                                             | 크기   | 용도                           |
| -------------------------------------------------------- | ------ | ------------------------------ |
| `AIO-3566-JD4_Ubuntu22.04-Xfce-r31127_v1.4.0d_251124.7z` | 587 MB | **굽는 대상** + RKDevTool 동봉 |
| `DriverAssitant_v5.13.zip`                               | 9.8 MB | USB 드라이버                   |
| `rknn_model_zoo-2.3.2.zip`                               | 225 MB | 검증용 예제                    |
| `rknn-toolkit2-2.3.2.zip`                                | 1.6 GB | aarch64 휠 + 보드 런타임 `.so` |

```powershell
Get-ChildItem *.7z | ForEach-Object { bz x -y -o:. $_.FullName }
Move-Item "AIO-3566-JD4_Ubuntu22.04-Xfce-r31127_v1.4.0d_251124\tools\windows\RKDevTool_Release_v3.19.zip" "."
Get-ChildItem *.zip | ForEach-Object { Expand-Archive $_.FullName -DestinationPath . -Force }
```

### 12.2 [PC] 플래싱 도구 설치

**USB 드라이버 설치**

- `DriverAssitant_v5.13\DriverInstall.exe`를 **관리자 권한으로 실행**
- `Uninstall Driver`를 먼저 누르고, 그다음 `Install Driver`

**RKDevTool 실행**

- `RKDevTool_v3.19_for_window\config.ini`에서 `Selected=1` → `2`로 바꿔 영문 표시.
- `RKDevTool_v3.19_for_window\RKDevTool.exe`를 **관리자 권한으로 실행**

### 12.3 [보드] Upgrade 모드 진입

Loader 모드가 가장 간단하다. ([3.1](#31-동작-모드-개요) 모드 비교 참고)

- **소프트웨어**: 보드가 부팅되는 상태라면 시리얼/SSH에서 아래 명령 입력 후 Type-C로 PC 연결
  ```bash
  sudo reboot loader
  ```
- **하드웨어**: 전원 분리 → Type-C로 PC 연결 → `RECOVERY` 버튼을 누른 채 전원 인가 → 약 2초 후 놓기.

확인 — 장치관리자에 `Rockusb Device`가 잡히고, RKDevTool 하단에 "Found One LOADER Device"가 표시된다.

### 12.4 [PC→보드] 통합 펌웨어 굽기

RKDevTool(GUI)에서 진행한다.

1. `Upgrade Firmware` 탭 선택
2. `Firmware` 버튼 → 펌웨어 이미지 열기
   ```plain
   D:\rknn\AIO-3566-JD4_Ubuntu22.04-Xfce-r31127_v1.4.0d_251124\AIO-3566-JD4_Ubuntu22.04-Xfce-r31127_v1.4.0d_251124.img
   ```
3. 로드되면 상단에 **Firmware Version / Loader Version / 칩 정보(RK3568)** 가 표시된다.
4. `Upgrade` 클릭
5. 완료되면 보드가 자동 재부팅된다.

**자주 걸리는 것**

| 증상                       | 원인 / 조치                                                            |
| -------------------------- | ---------------------------------------------------------------------- |
| `Download Boot Fail`       | USB 케이블 품질·PC 포트 구동력 문제가 대부분. 케이블·포트 교체         |
| 중간에 멈추거나 실패 반복  | USB 허브·연장선 경유. **PC 후면 USB 3.0 포트에 직결**                  |
| Nor Flash + eMMC 동시 존재 | MaskRom 진입 후 Storage 선택 필요 ([3.2](#32-usb-케이블로-업그레이드)) |

### 12.5 [보드] 첫 부팅 및 기본 설정

기본 계정은 `firefly` / `firefly`. 시리얼(1500000 8N1, [2.2](#22-시리얼-디버그)) / 데스크톱 환경 / SSH로 접속한다.

```bash
ssh firefly@<보드IP>        # 예: ssh firefly@172.16.151.220
```

**GUI 끄기 (콘솔 부팅)** — 2 GB 보드에서는 데스크톱을 끄고 작업한다.

```bash
sudo systemctl set-default multi-user.target && sudo reboot
```

- 실측: 메모리 사용률 **28 % → 6 %** (1957 MB 중 약 548 MB → 117 MB). 가용 1.8 Gi 확보.
- 되돌리기: `sudo systemctl set-default graphical.target && sudo reboot`
- 잠깐만 끄려면(재부팅 없이): `sudo systemctl stop lightdm`

**현황 확인**

```bash
ip a          # IP 확인
free -h       # RAM / swap — GUI 끈 뒤 확인
df -h /       # 저장공간
```

**패키지 확인**

```bash
sudo apt update
apt list --installed unzip libgl1 libglib2.0-0 openssh-server
```

- **`sudo apt upgrade`(전체 업그레이드)는 하지 말 것.** 보드 전용 커널·NPU 드라이버·`librknnrt.so`를 배포판 패키지가 덮어쓸 수 있다.

**[PC] 파일 전송** — 툴킷 zip 1.6 GB를 통째로 보내지 말고 3개만(약 44 MB).

```powershell
$src = "rknn-toolkit2-2.3.2"
New-Item -ItemType Directory -Force "stage" | Out-Null
Copy-Item "$src\rknpu2\runtime\Linux\librknn_api\aarch64\librknnrt.so" "stage\"
Copy-Item "$src\rknn-toolkit2\packages\arm64\arm64_requirements_cp312.txt" "stage\"
Copy-Item "$src\rknn-toolkit2\packages\arm64\rknn_toolkit2-2.3.2-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl" "stage\"

scp (Get-ChildItem "stage").FullName firefly@<보드IP>:~/
# scp (Get-ChildItem "stage").FullName firefly@172.16.151.220:~/
```

### 12.6 [보드] NPU 런타임(`librknnrt.so`) 버전 확인 — **가장 중요**

출고 이미지의 런타임이 구버전이면 2.3.2로 변환한 `.rknn`이 로드 단계에서 실패한다. **규칙: 런타임 버전 ≥ 변환에 쓴 툴킷 버전.**

```bash
sudo cat /sys/kernel/debug/rknpu/version                      # 커널 드라이버
strings /usr/lib/librknnrt.so | grep -i "librknnrt version"   # 런타임
```

실측(Xfce v1.4.0d 기준) — 드라이버 `v0.9.8`(충족), 런타임 **`1.6.0`(교체 필요)**. 최신 펌웨어라도 런타임은 구버전으로 묶여 있으니 반드시 확인한다.

```bash
sudo cp /usr/lib/librknnrt.so /usr/lib/librknnrt.so.1.6.0.orig   # 백업
sudo cp ~/librknnrt.so /usr/lib/librknnrt.so
sudo ldconfig
strings /usr/lib/librknnrt.so | grep -i "librknnrt version"
# → librknnrt version: 2.3.2 (429f97ae6b@2025-04-09T09:09:27) 이면 통과
```

- `ldconfig.real: /lib/librknnrt.so is not a symbolic link` 경고는 무시해도 된다.
- 버전 불일치 증상: `rknn_init` 실패, `RKNN_ERR_MODEL_INVALID`.
- 온디바이스 변환·추론만 한다면 `rknn_server`는 필요 없다(PC 원격 타깃 방식에서만 사용).

### 12.7 [보드] uv + RKNN-Toolkit2 설치

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc

uv venv --python 3.12 ~/venvs/rknn2
source ~/venvs/rknn2/bin/activate

uv pip install "setuptools<82" -r ~/arm64_requirements_cp312.txt
uv pip install ~/rknn_toolkit2-2.3.2-cp312-cp312-manylinux_2_17_aarch64.manylinux2014_aarch64.whl

python -c "from rknn.api import RKNN; print('OK')"
```

- `OK`가 찍히면 완료.
- `setuptools<82`를 먼저 고정하지 않으면 12.8 변환 단계에서 실패한다.

### 12.8 [보드] YOLO11로 동작 검증

**① [PC] Model Zoo 전송** (225 MB)

```powershell
scp "rknn_model_zoo-2.3.2.zip" firefly@<보드IP>:~/
# scp "rknn_model_zoo-2.3.2.zip" firefly@172.16.151.220:~/
```

**② [보드] 압축 풀기 및 모델 다운로드**

```bash
cd ~
unzip -q rknn_model_zoo-2.3.2.zip -d ~/workspace/
cd ~/workspace && mv rknn_model_zoo-2.3.2 rknn_model_zoo
cd rknn_model_zoo/examples/yolo11/model
sh download_model.sh
```

**③ [보드] ONNX → RKNN 변환**

```bash
source ~/venvs/rknn2/bin/activate
cd ~/workspace/rknn_model_zoo/examples/yolo11/python

python convert.py ../model/yolo11n.onnx rk3566
ll ../model/
# 사용법: convert.py <onnx_model> <platform> [i8/fp] [output_path]
# 결과: ../model/yolo11.rknn (약 4.6 MB, 기본 i8 양자화)
```

약 70초 소요(OpFusing 약 20초 + Quantizating 약 48초). **로그에 섞여 나오는 아래 메시지는 모두 무해하다.**

| 메시지                                                          | 의미                                                                    |
| --------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `E RKNN: Unkown op target: 0`                                   | 알 수 없는 목표 연산자가 없다는 뜻(`rknn building done` 확인)           |
| `Failed to detect devices under /sys/class/drm/card*`           | onnxruntime의 GPU 탐색 실패. 변환과 무관                                |
| `W build: found outlier value`                                  | 양자화 정확도 경고. 정확도가 아쉬우면 `convert.py ... rk3566 fp`로 비교 |
| `W build: The default input/output dtype ... changed to 'int8'` | i8 양자화 시 정상 안내                                                  |

**④ [보드] 추론 실행 (NPU)**

```bash
python yolo11.py --model_path ../model/yolo11.rknn --target rk3566 --img_save
ll result/
```

정상 출력 예 (`../model/bus.jpg`):

```plain
person @ (108 236 223 535) 0.902
person @ (213 240 284 509) 0.851
person @ (476 230 559 521) 0.820
person @ (79 358 118 516) 0.427
bus  @ (93 136 554 439) 0.948
Detection result save to ./result/bus.jpg
```

`./result`에 박스가 그려진 이미지가 생성된다.

### 참고 링크

- 매뉴얼 홈: https://wiki.t-firefly.com/en/Core-3566JD4/index.html
- 자료/펌웨어 다운로드: http://en.t-firefly.com/doc/download/123.html
- RKNN Toolkit2: https://github.com/airockchip/rknn-toolkit2/
- RKNN Model Zoo: https://github.com/airockchip/rknn_model_zoo

> 이 문서는 원문 매뉴얼의 요지를 정리한 것으로, 세부 코드·이미지·최신 다운로드 링크는 원문을 직접 확인하는 것이 정확하다. 일부 하위 페이지(Camera/I2C/IR/LCD/LED/RTC 세부 코드, U-Boot 상세, Firefly Linux/Android 사용자 가이드, 각 액세서리 세부 스펙)는 분량 관계로 핵심만 요약했다.
