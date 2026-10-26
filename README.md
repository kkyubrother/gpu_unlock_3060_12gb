# GPU 언락 — RTX 3060 12GB GA106 실험적 포트

**2026년 10월 20일, RTX 3060 12GB LHR 실물 카드 한 장에서 펌웨어 실행·언락 후 NVIDIA 드라이버 인계·텐서 연산 성능 상승을 확인했습니다.** 단순 빌드나 모의 테스트 결과가 아니라 실제 GPU에서 얻은 결과입니다. 다만 다른 보드의 호환성, 장시간 안정성, 실제 애플리케이션의 성능 향상을 보장하는 범용 완성품은 아닙니다. 2026년 10월 26일에는 다른 보드·커널·드라이버·VBIOS 조합의 두 번째 카드 한 장에서도 같은 결과를 확인했습니다(11절).

이 포트는 `Zagitta/gpu-unlock`의 커밋 `0ebdb1c6308b6a2072be5bf04513472942541956`을 기반으로 합니다. 아래 실기 검증에는 이 저장소의 커밋 **`e58f77a227387614437da6f7fa618b88a7bb94d7`**으로 만든 모듈을 사용했습니다.

## 1. 검증 환경

- 그래픽카드: NVIDIA RTX 3060 12GB LHR, GA106, `10de:2504`, 리비전 `a1`.
- VRAM: 12,288 MiB. 시험 당시 PCI 주소: `0000:01:00.0`.
- CPU / 메인보드: Intel Core i7-6700K / ASRock B150M Pro4.
- 운영체제 / 커널: Ubuntu 26.04.1 LTS / `7.0.0-38-generic`.
- NVIDIA 드라이버: **580.178.04**. 비교 시험에서 드라이버를 교체하지 않았습니다.
- 언락에 사용한 고정 펌웨어 입력: NVIDIA **610.43.03** 계열.
- 벤치마크: 기존 CUDA 12.8 라이브러리, cuBLAS 12.8.5, 전력 제한 170W.
- 성공 조건: 내장그래픽을 주 화면으로 설정. Linux에서 내장그래픽 `boot_vga=1`, RTX 3060 `boot_vga=0` 확인.
- 사용자에게 안내하고 적용한 UEFI 설정: `Advanced → Chipset Configuration → Primary Graphics Adapter = Onboard`, `IGPU Multi-Monitor = Enabled`. 모니터는 메인보드 영상 출력 사용.
- 실제 사용한 모듈의 SHA-256: `5e8d9e484fed2f3e140987eb4d5258b2637a513f316e07690d0ea4ae7c1a058c`.

**검증 범위는 위 카드 한 장과 위 조합, 그리고 11절의 두 번째 카드와 그 조합입니다.** 모든 GA106, 모든 LHR 카드, 모든 RTX 3060 12GB 또는 다른 드라이버에서도 된다는 의미는 아닙니다.

## 2. 실제 성능 비교

동일한 cuBLAS `cublasGemmEx` 연산·측정 방식으로 비교했습니다. 행렬 크기는 `8192 × 8192 × 8192`, 라운드당 32회, 총 7라운드의 중앙값입니다. FP16/BF16은 FP32 누산·출력을 사용하고, TF32는 명시적인 `FAST_TF32` 계산 모드를 사용했습니다. 희소성 가속을 사용하지 않았습니다.

| 연산 | 적용 전 TFLOPS | 적용 후 TFLOPS | 배율 | 상승률 |
|---|---:|---:|---:|---:|
| FP16 입력 / FP32 누산 | 26.893 | 49.873 | 1.854배 | +85.45% |
| BF16 입력 / FP32 누산 | 26.892 | 50.646 | 1.883배 | +88.33% |
| TF32 | 13.497 | 26.459 | 1.960배 | +96.03% |

측정 조건과 한계:

- 연산 코드, 행렬 크기, 반복 횟수, 라이브러리 버전, 관련 환경변수, NVIDIA 드라이버와 전력 제한이 일치하는 것을 확인했습니다.
- 적용 전 전용 측정기는 그대로 보존했습니다. 적용 후 측정본은 실행 상태 검사와 결과의 단계 표기만 변경했으며, 연산·측정 부분이 동일한지 회귀 테스트했습니다.
- 입력은 모두 1인 행렬이며, 각 자료형에서 출력 앞 16개 값이 기대값 `8192`인지 검사해 통과했습니다. **전체 출력 또는 다양한 입력에 대한 포괄적 수치 정확성 검증은 아닙니다.**
- 적용 전에는 외장그래픽이 화면을 담당했고, 적용 후에는 내장그래픽이 화면을 담당했습니다. 클럭과 온도도 고정하지 않았으므로, 표시한 상승률은 이 두 조건에서 관측한 비교값입니다.
- 벤치마크 후 확인한 커널 로그에는 NVIDIA Xid 오류가 없었고, 측정 프로그램 종료 후 GPU 메모리 사용량은 12 MiB로 돌아왔습니다.
- **GEMM 처리량이 약 두 배에 가까워졌다는 결과이지, ComfyUI·게임·모든 AI 작업이 같은 비율로 빨라진다는 뜻은 아닙니다.** 장시간 안정성도 아직 검증하지 않았습니다.

## 3. 초기 실패 원인과 해결 과정

### 3.1 처음에는 언락 펌웨어 실행 전 BME 검사에서 중단

외장그래픽이 주 화면이던 시험에서 PCI Command는 다음과 같았습니다.

- GRUB 진단 항목 진입: `0x7`.
- GRUB의 커널·initrd 적재 후 Linux 인계 직전: `0x7`.
- Linux 초기 PCI 열거의 `pci=earlydump`: `0x7`.
- 언락 모듈 진입 검사: `0x7`, `PCI_COMMAND_MASTER` 비트가 설정된 상태.

`kernel/gpu_unlock/unlock_610.c`의 `unlock_transaction_claim()`은 초기 BME가 켜져 있으면 `-EBUSY`를 반환합니다. 실제 실패 로그는 다음과 같습니다.

```text
unlock_transaction_claim error=-16
last PCI sample: capability=0 offset=0x4 value=0x7 result=0
firmware_started=0
gpu_unlock: unlock failed (-16)
```

즉, **초기 진입 조건이 맞지 않아 중단된 것이며, V67 펌웨어 실행 실패·LHR 차단·후속 WPR 검사 실패로 판정할 상황이 아니었습니다.** 모듈은 NVIDIA보다 먼저 실행됐지만 이미 BME가 켜져 있었습니다. GRUB에서도 켜져 있었으므로, 해당 부팅에서 Linux NVIDIA 드라이버가 처음 켰다는 설명은 맞지 않았습니다.

사용자가 AC 전원을 분리한 뒤 켠 시험에서도 같은 결과였습니다. 그때 UEFI 기본값 로드 메시지도 보고됐습니다. 전지 방전은 사용자 추정이며 별도로 확인하지 않았고, UEFI 설정이 유지된 전원 분리 대조 시험으로 볼 수는 없습니다. **전원 분리만으로 다음 부팅의 초기 BME 상태가 보장되지는 않았습니다.**

### 3.2 내장그래픽을 주 화면으로 바꾼 후 초기 상태가 달라짐

UEFI에서 내장그래픽을 주 화면으로 지정한 뒤, 같은 검증 모듈을 별도 initramfs의 1회 부팅 항목으로 실행했습니다. **BME를 강제로 끄는 코드를 추가하거나 기존 거부 검사를 삭제하지 않았습니다.**

- 내장그래픽 `0000:00:02.0`: `boot_vga=1`.
- RTX 3060 `0000:01:00.0`: `boot_vga=0`.
- GRUB 진입 / Linux 인계 직전 / Linux 초기 PCI 읽기: 모두 **`0x0`**.
- 모듈 진입: `claim: command=0x0`으로 기존 BME 검사를 통과.

이후 FWSEC 실행, 선택자 설정, FLR, 후속 검사와 정리가 진행됐습니다. 실제 로그의 주요 부분은 다음과 같습니다.

```text
claim: command=0x0 capability=0x112c8de1 BAR_mask=0x4b
pre-FLR selectors: ss0=0x88888888 expected=0x88888888 ss1=0x8 expected=0x8
PCIe FLR returned: result=0
post-FLR SEC2 reset released, GSP halted, BME=0 pending=0 WPR2=1ffffe00:00000000 SS0=88888888 SS1=00000008
restore PCI command: actual=0x0 original=0x0 mask=0x6
gpu_unlock: unlocked
```

그 뒤 NVIDIA 580.178.04가 정상 로드됐고, 위의 실제 CUDA 성능 향상까지 확인했습니다. 진단값은 실제 커널 명령행의 식별값, 변경된 부팅 ID, 재예약 전 이전 관측값 삭제, 정확한 PCI 식별값과 함께 확인했습니다. 저장된 과거 GRUB 값을 새 결과로 오인한 것이 아닙니다.

### 3.3 확인된 사실과 아직 남은 가설

**확인된 직접 실패 원인:** 초기 BME ON 상태에 대한 모듈의 안전 검사였습니다. **이번 환경에서는 내장그래픽을 주 화면으로 바꾼 후 이 진입 차단이 해소됐고 실제 언락·성능 상승으로 이어졌습니다.**

부팅 화면을 담당하는 GPU의 UEFI/GOP 초기화 경로가 차이를 만들었다는 해석을 강하게 뒷받침합니다. 다만 **GOP 드라이버의 특정 명령이 BME를 설정했다고 추적한 것은 아닙니다.** 보드 펌웨어, GOP, 더 이른 부트로더 초기화 중 정확한 설정 주체는 구분하지 못했습니다. 따라서 이를 단정적인 ‘GOP 버그’로 기록하지 않습니다.

이 결과는 초기 BME 강제 해제의 안전성을 입증하지도 않습니다. 프레임버퍼 제거는 GPU 전체 DMA 정지와 동의어가 아니며, 초기 BME 검사·cold-state 검사·DMA 수명 보호는 그대로 유지했습니다.

## 4. 실기 검증의 범위와 안전 주의사항

- **실제 검증 완료:** 이 카드와 11절의 두 번째 카드에서 펌웨어 경로 실행, FLR 후 목표 선택자 유지, 정상 NVIDIA 인계, 제한된 출력 샘플 검사와 GEMM 성능 비교.
- **미검증:** 1절과 11절 외의 다른 카드·보드·펌웨어·드라이버, 장시간 안정성, 다양한 입력의 전체 수치 검증, 실제 앱의 향상률, SS1만의 독립적 효과, 매 부팅 반복 재현성.
- 성공한 시험은 사용자가 ‘재부팅’으로 보고했습니다. 그 사실만으로 모든 warm reboot를 지원하거나, 별도 AC 분리 성공 시험을 완료했다고 주장하지 않습니다. 모듈의 실제 cold-state 검사를 통과하는지가 중요하며, 이미 초기화된 카드는 거부될 수 있습니다.
- 물리적 접근과 정상 Ubuntu 복구 항목을 확보하고 시험하십시오. GPU 정지나 화면 소실이 가능하며, 실패 시 전원을 완전히 꺼야 할 수 있습니다.
- 실행 중인 NVIDIA 드라이버를 강제로 내리거나, BME 거부 검사·장치 식별·메모리 보호 검사를 삭제해 진입을 강제하지 마십시오.
- 시험에서는 정상 커널·정상 initramfs·기본 부팅을 보존하고 별도 이미지와 1회 항목을 사용했습니다. 모듈이 udev/NVIDIA보다 먼저 실행되도록 실제 이미지의 순서와 내장된 모든 자산의 해시를 확인했습니다. 11절의 두 번째 시험은 패키지가 설치한 일반 initramfs와 GRUB 1회 항목을 사용했으며, 그 서버의 initramfs에 nvidia 모듈이 없음을 확인한 뒤 진행했습니다.
- **이 저장소의 문서 수정이 상시 자동 적용을 설정하지는 않습니다.** 검증 당시에도 1회 부팅을 사용했으며, 다음 일반 부팅에서 모듈이 자동 재실행된다고 가정하면 안 됩니다.
- AC를 분리하려면 운영체제가 완전히 종료된 다음에 하십시오. UEFI 설정이 초기화되는 시스템에서는 주 그래픽 설정이 유지됐는지 먼저 확인하십시오.

## 5. GA106 포트의 선택 조건과 안전 검사

- `gpu_unlock.experimental_ga106=1`은 기본값이 꺼진 별도의 실험 참여 옵션입니다.
- GA106은 `gpu_unlock.targets=`에 정확한 전체 PCI 주소를 명시해야 합니다.
- `targets=all`은 실험 옵션을 켜도 기존 RTX 3090만 선택합니다.
- 다른 RTX 3060 장치 ID, GA104 기반 3060, 노트북 변형과 `a1` 이외 리비전은 활성화하지 않았습니다. PCI ID·리비전만으로 VRAM 용량을 구분할 수 없으므로, 이 포트의 실기 검증 범위는 위 12GB 카드 한 장입니다.
- Booter, FLR 후 GFW와 PCI 검사는 모든 카드를 `10de:2204`로 간주하지 않고 원래 저장한 vendor/device를 비교합니다.
- 이미 연결된 드라이버, 초기 BME, cold-state, DMA 수명, 펌웨어 크기·정렬, 예약 메모리와 FLR 검사를 유지합니다. 실행 중인 드라이버를 내리는 도구는 없습니다.

시험에 사용한 GA106 명령행 옵션은 다음과 같습니다. **이 문자열만 추가한다고 올바른 initramfs 순서나 안전한 시험 환경이 구성되는 것은 아닙니다.**

```text
gpu_unlock.targets=0000:01:00.0 gpu_unlock.experimental_ga106=1
```

## 6. 공통 펌웨어 입력과 FWSEC 안전성 수정

GA106 전용 펌웨어를 임의로 만들지 않았습니다. NVIDIA 610.43.03의 `g_kernel_gsp_nvoc.c`에서 GA106과 GA102는 같은 `kgspGetBinArchiveBooterLoadUcode_GA102`, `kgspGetBinArchiveGspRmBoot_GA102`와 WPR 메타데이터 루틴을 사용합니다. 이 공통성은 초기 포팅 근거였으며, 실제 호환성 판단은 위 실기 결과와 구분합니다.

원본 확인 경로:

```text
NVIDIA/open-gpu-kernel-modules
610.43.03/src/nvidia/generated/g_kernel_gsp_nvoc.c
```

고정된 610.43.03 입력과 V67 페이로드는 유지했습니다. `IHaBiS02/ga107-unlock` 커밋 `5e6b3600bb01142891d43d62120727e96911d1a6`에서 제한된 DMEM 블록 올림과 DMA 노출 추적 수정을 가져왔습니다. 해당 코드는 GPL-2.0-only입니다.

- DMEM 전송 크기는 기존 이미지에 완전한 전송 범위가 있을 때만 256바이트 단위로 올립니다. 오버플로·정렬·범위 위반은 거부합니다.
- DMA 소스·베이스를 프로그래밍하기 직전에 버퍼가 펌웨어에 노출될 수 있다고 기록합니다. 형상 검사나 초기 큐 검사 전부터 노출됐다고 간주하지 않습니다.
- 노출 이후 실패하면 안전한 정지가 확인되지 않는 한 관련 자원을 보존합니다.
- 이 변경은 초기 BME를 정규화하거나 SS0/SS1 정책을 바꾸지 않습니다. 이번 검증 이미지에는 수정본 모듈이 실제 포함됐으며 해시로 확인했습니다.

## 7. 빌드와 호스트 모의 테스트

아래 커널 빌드는 모듈을 설치하거나 로드하지 않습니다.

```sh
make -C /lib/modules/$(uname -r)/build M="$PWD/kernel/gpu_unlock" modules
python3 -m unittest discover -s tests -v
```

`GPU_UNLOCK_ASSETS`에 생성된 자산 디렉터리를 지정하면 GA102와 **합성 12GiB GA106 시험 데이터**의 고정 자산 트랜잭션도 검사할 수 있습니다. 테스트의 PCI/MMIO/DMA와 ROM은 모의 데이터입니다. 실물 3060에서 읽어온 ROM·레지스터라고 해석하면 안 됩니다. 호스트 테스트와 위 실기 검증은 서로 다른 검증 단계입니다.

네이티브 패키지는 커밋된 `HEAD`를 사용합니다. 패키징 전에 검토된 소스가 커밋돼 있어야 합니다. 패키지 버전은 상위 프로젝트의 `1.0.0`을 유지합니다.

## 8. 패키지 생성과 설치

**다음 설치 명령은 운영체제의 모듈·initramfs 구성을 변경합니다. 빌드만 하는 명령과 구분하십시오.** 위 실기 검증은 아래 범용 설치 절차 전체가 아니라, 별도로 검증한 1회 시험 이미지에서 수행했습니다.

빌드는 고정된 NVIDIA/CachyOS 입력을 HTTPS로 내려받아 SHA-256을 검증하고, 빌드 디렉터리와 바이너리 패키지에만 펌웨어를 생성합니다. 커밋된 `HEAD`가 패키징 대상입니다.

```sh
# 의존성 준비·빌드·설치를 진행하며 관리자 인증이 필요합니다.
make install-arch   # makepkg -sif
make install-deb    # apt-get build-dep, dpkg-buildpackage, apt-get install
make install-rpm    # dnf builddep, rpmbuild, dnf install

# 설치하지 않고 빌드만 합니다.
make arch
make deb
make rpm
```

Debian/RPM 결과는 `build/deb`, `build/rpm`에 생성됩니다. 입력은 기본적으로 `~/.cache/gpu-unlock/inputs`에 캐시합니다. `make inputs` 또는 `INPUTS=`로 준비 경로를 지정할 수 있습니다. `dpkg-buildpackage` 자체는 네트워크를 사용하지 않으므로, 격리 빌드에는 `GPU_UNLOCK_INPUTS`로 준비된 캐시를 전달해야 합니다.

`make dist`, Debian 소스 패키지와 `.nosrc.rpm`에는 NVIDIA 아카이브나 생성 펌웨어가 포함되지 않습니다. 펌웨어 생성 과정은 [firmware/README.md](firmware/README.md)를 참고하십시오.

## 9. 부팅 순서와 실행 옵션

상위 패키징은 initramfs에 모듈과 펌웨어를 넣습니다.

- **mkinitcpio:** `/etc/mkinitcpio.conf.d/zz-gpu-unlock.conf`에서 `MODULES`의 `nvidia` 직전에 `gpu_unlock`을 넣습니다. 이미지에 `nvidia`가 없으면 목록 끝에 넣습니다. `-c <config>` 또는 프리셋의 `_config`를 사용하는 경우에는 해당 설정의 `MODULES`를 직접 확인해야 합니다.
- **initramfs-tools:** `/usr/share/initramfs-tools/modules.d/gpu-unlock.conf`를 사용합니다. **파일 존재만으로 udev보다 먼저 실행된다고 보장하지 마십시오.** 실기 시험에서는 직접 `insmod`를 호출하는 별도 초기 스크립트와 생성된 `ORDER`를 확인했습니다.
- **dracut:** `/usr/lib/dracut/dracut.conf.d/gpu-unlock.conf`를 사용합니다.

`softdep nvidia pre:` 규칙은 추가하지 않습니다. kmod가 먼저 읽은 같은 이름의 규칙만 적용해 다른 패키지의 `nvidia` 규칙과 충돌할 수 있기 때문입니다.

RTX 3090용 기존 선택 예시는 다음과 같습니다. GA106은 위의 별도 실험 옵션과 전체 주소가 모두 필요합니다.

```text
gpu_unlock.targets=all
gpu_unlock.targets=0000:21:00.0
gpu_unlock.targets=0000:21:00.0,0000:22:00.0
```

주소는 `lspci -D`의 전체 PCI 주소이며 GPU 번호와는 별개입니다. `all`과 개별 주소를 혼합하면 `all` 토큰은 무시되고 적격인 개별 주소만 처리합니다. 이미 드라이버가 연결됐거나 전원·BME·cold-state 조건을 만족하지 않는 카드는 거부됩니다. 상위 코드는 여러 적격 카드를 병렬 처리하지만, 이번 GA106 실기 검증은 한 장만 사용했습니다.

상위 코드에는 디스플레이 카드 처리 경로도 있습니다. 펌웨어 프레임버퍼는 FLR 직전에 제거하고 이후 NVIDIA가 화면을 인계합니다. **이 경로가 있다는 사실은 초기 BME ON 진입을 허용한다는 뜻이 아닙니다.** 이번 보드에서는 내장그래픽을 주 화면으로 바꿔 진입 조건을 만족시켰습니다.

결과 확인은 읽기 전용 명령으로 시작하십시오.

```sh
journalctl -k -b | grep -E 'gpu_unlock|post-FLR|FWSEC|NVRM'
lspci -nnk -s 0000:01:00.0
nvidia-smi
```

모듈이 로드됐다는 사실만으로 성공으로 판단하지 마십시오. 실제 실행 결과·정리·드라이버 인계와 CUDA 측정을 함께 확인해야 합니다. 펌웨어가 멈추지 않으면 관련 DMA 버퍼를 보존할 수 있으므로 전원을 완전히 끈 뒤 원인을 확인하십시오. 실행을 중단하려면 부팅 명령행의 참여 옵션을 제거하거나, 별도 시험 항목 대신 정상 부팅 항목을 사용하십시오.

## 10. 상위 프로젝트 결과와 구분

상위 프로젝트의 원래 검증 대상은 RTX 3090 `10de:2204`입니다. 상위 기록에는 CachyOS `7.2.8-1-cachyos` / NVIDIA `615.71.09`, 시험 카드 `0000:21:00.0`, 비적용 대조 카드 `0000:01:00.0`이 등장합니다. 이는 이 문서의 Ubuntu/RTX 3060 측정과 다른 시험이며, 이 포트에서 다시 측정한 결과가 아닙니다.

이 모듈은 V67/PLM 기법을 NVIDIA 드라이버 연결 전에 실행하고 SS0 `0x88888888`, SS1 `0x8`을 적용한 뒤 FLR을 수행합니다. UEFI 애플리케이션도, 수정된 NVIDIA 드라이버도 아닙니다. 이번 결과는 이 조합의 효과이며 **SS1의 독립적인 효과는 분리 검증하지 않았습니다.**

## 11. 두 번째 실기 검증: Ubuntu 24.04 / 커널 6.8 / NVIDIA 590 2-GPU 서버

**2026년 10월 26일(UTC), 다른 보드·커널·드라이버·VBIOS 조합의 RTX 3060 12GB LHR 두 번째 카드에서 같은 모듈 소스로 언락과 GEMM 성능 상승을 확인했습니다.** 1절의 시험과 독립적인 한 번의 부팅 결과이며, 아래 명시한 조합에 한합니다.

### 11.1 환경

- 그래픽카드(대상): NVIDIA RTX 3060 12GB LHR, GA106, `10de:2504`, 리비전 `a1`, 서브시스템 `10de:1536`, VBIOS `94.06.2F.00.89`, VRAM 12,288 MiB, PCI 주소 `0000:04:00.0`, `boot_vga=0`. UEFI에서 Resizable BAR가 켜져 있어 BAR1이 16 GiB였습니다.
- 그래픽카드(대조): GA104 기반 RTX 3060 `10de:2487` 리비전 `a1`(Gigabyte `1458:4074`), PCI 주소 `0000:01:00.0`, `boot_vga=1`로 화면 담당. 이 포트의 대상이 아니며 `targets=`에 넣지 않았습니다.
- CPU / 메인보드: AMD Ryzen 5 5600G / BIOSTAR B550M-SILVER (UEFI 5.17). 내장그래픽은 UEFI에서 비활성화되어 PCI에 보이지 않았습니다.
- 운영체제 / 커널: Ubuntu 24.04.4 LTS / `6.8.0-147-generic`. initramfs-tools `0.142ubuntu25.8`.
- NVIDIA 드라이버: **590.44.01 proprietary 커널 모듈**(NVIDIA CUDA apt 저장소의 `cuda-drivers`), GSP 펌웨어 사용(`nvidia-smi`의 GSP Firmware Version 590.44.01). 비교 시험에서 드라이버를 교체하지 않았습니다.
- Secure Boot: 비활성. dkms가 서버 MOK 키로 서명했지만 커널에는 그 키가 없어 `module verification failed` 메시지와 함께 taint 상태로 로드됐습니다.
- 모듈 소스: 이 저장소의 커밋 `b977a67fd8229500360e715e1714c2f01697babb`. `kernel/` 아래 코드는 1절의 검증 커밋 `e58f77a227387614437da6f7fa618b88a7bb94d7`과 동일하고, 그 사이 변경은 `firmware/` 문서뿐입니다.
- 패키지: 서버에서 `make deb`로 생성한 `gpu-unlock-dkms_1.0.0_all.deb`와 `gpu-unlock-firmware_1.0.0_all.deb`. Python 3.12.3, zstd 1.5.5, debhelper 13.14.1ubuntu5이며 `dh-sequence-dkms`는 NVIDIA 저장소의 `dh-dkms 1:3.3.0`이 제공했습니다. 빌드 중 호스트 테스트 5건이 통과했고, 생성된 펌웨어 5개의 SHA-256은 고정값과 일치했습니다.
- 실제 사용한 모듈: dkms가 `6.8.0-147-generic`용으로 빌드한 `gpu_unlock.ko`, SHA-256 `20fce4a9f69afda689690c25de3c023fc9595c5e5fdd6512b0de0fadfb5ab98f`(압축 해제 기준, 설치 파일은 `gpu_unlock.ko.zst`).
- 부팅 방식: 8절의 패키지 설치 그대로 initramfs-tools `modules.d` 경로를 사용했고 별도 시험 이미지는 만들지 않았습니다. initramfs를 풀어 확인한 `conf/modules`의 첫 항목이 `gpu_unlock`이었고, 이 서버의 initramfs에는 nvidia 모듈이 없어 udev가 먼저 nvidia를 연결할 수 없는 구성이었습니다. 1회 시험은 `/etc/grub.d/40_custom`에 같은 커널·initrd에 옵션만 더한 항목을 추가하고 `grub-reboot`로 선택했습니다.
- 명령행 옵션: `gpu_unlock.targets=0000:04:00.0 gpu_unlock.experimental_ga106=1`.

### 11.2 커널 로그

warm reboot 한 번으로 진입했고, 초기 PCI Command는 `0x3`(BME 꺼짐)이라 3.1절의 진입 차단이 발생하지 않았습니다. 대상 카드가 부팅 VGA가 아니었기 때문으로 해석하며, 이 보드의 펌웨어가 부팅 VGA가 아닌 카드의 BME를 켜지 않는다는 관찰 이상의 결론은 내리지 않습니다.

```text
claim: command=0x3 capability=0x10688de1 BAR_mask=0x4b
FWSEC: PROM bytes=1048576 signature=0xaa55
FWSEC signature selection: versions=0x7 fuse_version=2 sig_bytes=384
FWSEC result: scratch=0x0 expected_hi16=0
pre-FLR selectors: ss0=0x88888888 expected=0x88888888 ss1=0x8 expected=0x8
post-FLR GFW result: error=0 pci=0x250410de plm=0x8b8f boot=0x3ff cpuctl=0x10 budget_us=2050000
post-FLR SEC2 reset released, GSP halted, BME=0 pending=0 WPR2=1ffffe00:00000000 SS0=88888888 SS1=00000008
gpu_unlock: unlocked
```

이어서 NVIDIA 590.44.01이 두 카드를 정상 인계했고, 측정 후까지 Xid 오류는 없었습니다.

### 11.3 성능 비교

1절의 측정기는 저장소에 포함되지 않아 같은 방식의 별도 호스트 프로그램으로 측정했습니다. `cublasGemmEx`, 행렬 `8192 × 8192 × 8192`, FP16/BF16 입력에 FP32 누산·출력, TF32는 `CUBLAS_COMPUTE_32F_FAST_TF32`, 라운드당 20회 × 5라운드의 중앙값, cuBLAS 12.0.2(Ubuntu `nvidia-cuda-toolkit` 12.0), 전력 제한 170 W입니다. 입력은 모두 1이고 각 자료형에서 출력 앞 16개가 `8192`인지 검사해 통과했습니다. 적용 전 측정은 재부팅 직전 같은 드라이버에서 했고, 두 측정 모두 GPU를 쓰는 다른 프로세스를 멈춘 상태였습니다.

대상 카드 `0000:04:00.0`:

| 연산 | 적용 전 TFLOPS | 적용 후 TFLOPS | 배율 |
|---|---:|---:|---:|
| FP16 입력 / FP32 누산 | 27.350 | 49.301 | 1.803배 |
| BF16 입력 / FP32 누산 | 27.217 | 49.930 | 1.835배 |
| TF32 | 13.603 | 26.257 | 1.930배 |

비적용 대조 카드 `0000:01:00.0`(GA104)은 같은 재부팅 전후 FP16 28.728 → 28.628, BF16 28.728 → 28.627, TF32 14.355 → 14.307 TFLOPS로 변화가 없었습니다. 상승은 대상 카드에만 나타났습니다.

### 11.4 범위와 한계

- 이 결과로 추가 확인된 것: 커널 6.8 / Ubuntu 24.04, 590.44.01 proprietary(GSP) 드라이버 인계, NVIDIA `10de:1536` 보드와 VBIOS `94.06.2F.00.89`, Resizable BAR 활성 상태, 부팅 VGA가 다른 외장 GPU인 2-GPU 구성, 패키지의 initramfs-tools 경로.
- 여전히 미검증: 매 부팅 반복 재현성(이 시험도 한 번의 부팅), 장시간 안정성, 다른 보드·VBIOS·드라이버, AC 분리 후 cold boot. 시험 후 이 서버의 `GRUB_CMDLINE_LINUX_DEFAULT`에 같은 옵션을 넣어 상시 적용으로 바꿨지만, 그 뒤 부팅의 결과는 아직 기록하지 않았습니다.
- 4절의 안전 주의사항은 그대로 적용됩니다.

## 12. 검증 자료와 라이선스

로컬 시험 자료는 다음 경로에 보존했습니다. `build/` 아래 자료는 생성물이며, 원자료 전체가 이 소스 저장소에 배포된다는 의미는 아닙니다.

- 적용 전: `build/benchmarks/baseline-unpatched-20261019-220350.json`.
- 적용 후: `build/benchmarks/post-unlock-5ca2e7cf.json`.
- 계산된 비교: `build/benchmarks/comparison-5ca2e7cf.json`.
- 해당 부팅의 전체 커널 로그와 진단 결과: `build/deploy/combined_trial/boot-5ca2e7cf-2558-4543-a769-2681e78eb76f/`.
- 11절 두 번째 시험의 커널 로그와 측정 출력은 서버 밖에 별도로 보관했으며 이 저장소에 포함하지 않았습니다.

커널 모듈과 로컬 패키징은 GPL-2.0-only입니다. 생성 페이로드는 고정된 CMP 참조에서 유래하며, NVIDIA 펌웨어는 자체 조건을 따릅니다. [firmware/NOTICE.md](firmware/NOTICE.md)와 `licenses/`의 고지를 확인하십시오. 설명 문서는 한글로 작성하되 명령어·식별자·파일명·저작권 및 라이선스 원문은 변경하지 않습니다.
