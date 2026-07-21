# OS 설치확인서 및 인프라 구성 정보

- 작성일: 2026-07-21
- 대상 서버: VM 서버(192.168.47.115), vLLM 서버(192.168.47.185)

---

## 1. OS 설치 정보 총괄표

| 항목 | VM 서버 (192.168.47.115) | vLLM 서버 (192.168.47.185) |
|---|---|---|
| 호스트명 | SSIS-VM | ssis-vllm |
| OS | Ubuntu 24.04.4 LTS (Noble Numbat) | Ubuntu 24.04.4 LTS (Noble Numbat) |
| 커널 | 6.8.0-136-generic (2026-07-01 빌드) | 6.8.0-136-generic (2026-07-01 빌드) |
| 아키텍처 | x86-64 | x86-64 |
| 설치 방식 | Curtin(Ubuntu Server 표준 설치기) 기반 설치 흔적 확인 (`/etc/fstab` 주석) | 동일 |
| 타임존 | Etc/UTC (UTC+0) | Etc/UTC (UTC+0) |
| 로케일 | en_US.UTF-8 | en_US.UTF-8 (동일 계열 확인) |
| NTP 동기화 | 활성 (ntp.ubuntu.com, 동기화 정상) | 활성 (동기화 정상) |
| 자동 보안 업데이트 | 활성 (`APT::Periodic::Unattended-Upgrade "1"`) | 활성 (동일) |

---

## 2. 파티션 / LVM 구성

### 2.1 VM 서버 (192.168.47.115)
| 물리 디스크 | 파티션 | 용도 | 크기 |
|---|---|---|---|
| /dev/nvme0n1 | p1 (EFI) | /boot/efi | 1G |
| | p2 | /boot (ext4) | 2G |
| | p3 → LVM PV | ubuntu-vg → ubuntu-lv → `/` (ext4) | 1.8T |
| /dev/nvme1n1 | p1/p2 | 미사용 파티션(예비) | 1G/2G |
| | p3 → LVM PV | ubuntu-vg-1 → ubuntu-lv (별도 VG, 마운트 미지정) | 1.8T |
| swap.img | 파일 기반 스왑 | - | (fstab 등록) |

> 참고: 2번째 NVMe(nvme1n1)의 LVM(`ubuntu-vg-1`)이 파티션은 구성되어 있으나 현재 `/etc/fstab`/마운트에 연결되어 있지 않음 — 데이터 디스크 용도로 추가 구성 예정인지 확인 필요.

### 2.2 vLLM 서버 (192.168.47.185)
| 물리 디스크 | 파티션 | 용도 | 크기 |
|---|---|---|---|
| /dev/nvme0n1 | p1 (EFI) | /boot/efi | 1G |
| | p2 | /boot (ext4) | 2G |
| | p3 → LVM PV | ubuntu-vg → ubuntu-lv → `/` (ext4) | 1.8T |
| swap.img | 파일 기반 스왑 | - | (fstab 등록) |

단일 디스크(SK hynix P41 2TB) 구성, RAID 미적용.

---

## 3. 네트워크 구성

| 항목 | VM 서버 | vLLM 서버 |
|---|---|---|
| 운영 IP | 192.168.47.115/24 | 192.168.47.185/24 |
| NIC | Realtek 8126 5GbE (eno1) | Broadcom BCM57416 10G (eno2np1, host 모드) |
| 예비/미사용 NIC | - | eno1np0 (DOWN), USB NIC(enxbe3af2b6059f, DOWN) |
| 가상 네트워크 | Docker bridge 다수 (docker0 172.17.0.1, RAGFlow 전용 br-* 2개) | Docker bridge (docker0 172.17.0.1), vLLM 컨테이너는 host 네트워크 직접 사용 |

> IP 할당표는 사내 IP 관리대장/발주처 네트워크 설계서와 대조하여 공식 IP 할당 근거 문서로 별도 확정 권장.

---

## 4. 계정 / 접근 통제

| 항목 | VM 서버 | vLLM 서버 |
|---|---|---|
| 일반 계정 | master (uid=1000) | master (uid=1000) |
| sudo 권한 | master → sudo 그룹 소속, **비밀번호 없는 sudo(NOPASSWD) 설정 확인됨** | 동일 |
| SSH 포트 | 22 (grep 결과 미표시 → 기본값 22 적용 추정) | 22 (실측 확인) |
| SSH PermitRootLogin | 미확인(설정값 grep 안됨, 기본 정책 적용 추정) | without-password (비밀번호 방식 root 로그인 차단, 키 인증만 허용) |
| SSH PasswordAuthentication | 미확인(설정값 grep 안됨) | **yes (비밀번호 인증 허용)** |
| 호스트 방화벽(UFW) | **비활성(inactive)** | **비활성(inactive)** |

