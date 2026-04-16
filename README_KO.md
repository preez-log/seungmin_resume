# Delta Portfolio — Systems & Engine Programming

![USPTO](https://img.shields.io/badge/USPTO-Patent%20Pending-005A9C?style=for-the-badge)

Application #19/641,687 — Method and System for Audiovisual Synchronization and Render Latency Minimization via Hardware Clock Domain Bridging

<br/>

## Performance Metrics & System Logs

<img width="975" height="509" alt="image" src="https://github.com/user-attachments/assets/da00b32d-2b16-4574-a3df-d2f5b55873e9" />
<img width="3840" height="1080" alt="image" src="https://github.com/user-attachments/assets/e3373d5c-13de-4763-b3a8-f88d8c302a73" />
<img width="3840" height="1059" alt="image" src="https://github.com/user-attachments/assets/055ee063-5f73-4116-bc4c-cc370ec22fc0" />

<br/>

---

## Core Architecture & Features

<br/>

**코어 소스코드 비공개 안내**

본 레포지토리는 포트폴리오 쇼케이스용 공간입니다.
커스텀 D3D11 게임 엔진, 커널 레벨 가상 오디오 드라이버, AMD SVM 하이퍼바이저, 그리고 HFT 알고리즘 트레이딩 봇의
핵심 소스코드는 지적재산권(IP) 보호 및 보안상의 이유로 Private 처리되어 있습니다.
*상세 코드는 입사 지원 시 제출된 압축 파일(ZIP)을 통해 확인하실 수 있습니다.*

<br/>

### Delta Engine — Custom C++ / D3D11 Game Engine

> 상용 엔진의 추상화 오버헤드를 배제하고, 리듬게임 판정 정밀도와 렌더링 동기화를 극한까지 끌어올리기 위해 설계한 멀티스레드 로우레벨 엔진입니다.

**Architecture — 3-Thread Pipeline**

전체 루프가 세 스레드로 분리되며 뮤텍스 없이 동작합니다.

|      Thread     |   Core    |  Role  |
|-----------------|-----------|--------|
| Logic Thread    | Core 1    | 16,000 Hz 틱 — 입력 처리, 판정, 씬 업데이트 |
| Render Thread   | Main      | VBlank 세마포어 대기 → D3D11 Present       |
| VBlank Observer | Dedicated | D3DKMT 커널 레벨 VBlank 신호 후킹          |

- **Lock-Free Triple Buffering** : `RenderSnapshot` 3개 슬롯을 `atomic<int>` swap으로 관리. 뮤텍스 없이 Logic → Render 데이터 전달.

- **RDTSC 기반 나노초 타이밍** : `PrecisionClock`이 `__rdtsc()`와 QPC 교차 캘리브레이션으로 CPU 클럭 주파수를 측정, 이후 모든 타이밍을 나노초 단위로 처리. `waitUntil`은 2ms 이상이면 sleep, 그 이하는 `_mm_pause()` 스핀 전환.

- **SIMD 기반 병렬 판정** : `NotePool`을 SoA(Structure of Arrays) 레이아웃으로 설계, `_mm256_cmp_pd`로 4노트를 한 클럭에 Miss 판정. `[[likely]]` 속성으로 분기 예측 최적화.

- **2GB 메모리 아레나** : `VirtualAlloc`으로 2GB를 사전 예약, 플레이 중 동적 할당 제로. `NotePool`은 아레나에서 구조체 배열 단위로 바인딩.

- **Data-Oriented Scene Snapshot** : 씬이 매 프레임 `RenderSnapshot`에 상태를 기록, 렌더 스레드는 스냅샷만 읽어 그립니다. 씬 전환 시 `scene_id` 불일치로 스테일 프레임을 자동 스킵.

- **Lua 스킨 시스템** : `.luaskin` 파일을 2회 실행(1차: 옵션 주입, 2차: 전체 파싱)하는 방식으로 엔진 설정을 스킨에 반영. 로드타임에 정적 OP 컬링 및 포인터 사전 연결로 런타임 Map 조회 완전 제거.

- **BGA 비디오 디코딩** : Media Foundation을 별도 디코드 스레드에서 실행, Lock-Free 트리플 버퍼로 CPU→GPU 프레임 전달. 렌더 스레드는 `uploadToGPU`만 호출해 `Map/Unmap` 처리.

<br/>

---

### Delta Cast — Kernel-Level Virtual ASIO Driver

> ASIO 독점 모드로 인해 OBS 등 방송 소프트웨어가 오디오를 캡처할 수 없는 문제를 해결하기 위해 제작한 프록시 ASIO 드라이버입니다. 게임과 하드웨어 사이에 COM 프록시로 삽입되어, 오디오 콜백 내부에서 데이터를 Lock-Free 링 버퍼로 복제한 뒤 별도 스레드로 WASAPI Shared에 전송됩니다.

- **Zero-Latency Pass-Through** : ASIO 콜백(`bufferSwitch`) 내부에서 메모리 할당 없이 `ByteRingBuffer::Push`만 호출. 게임→하드웨어 경로에 추가 레이턴시 없음.

- **Lock-Free Ring Buffer** : 128KB `ByteRingBuffer` L/R 각 1개. `atomic<size_t>` write/read 인덱스로 뮤텍스 없이 콜백 스레드↔렌더 스레드 간 데이터 교환.

- **Dynamic Limiter** : -0.1dBFS 임계값 기반 릴리즈 계수 사전 계산(`exp(-1/(r*sr))`), 매 프레임 1회 게인 계산으로 클리핑 방지.

- **Virtual Backend** : 실제 오디오 인터페이스 없이도 작동하는 소프트웨어 클럭 ASIO 에뮬레이터. 링 버퍼 채움률(90% 오버런 / 10% 언더런)에 따라 ±10µs 적응형 클럭 조정.

- **Proxy Backend** : 기존 ASIO 하드웨어를 `CoCreateInstance`로 로드, 모든 API 호출을 투명하게 위임. Delta Cast 자신은 레지스트리에 `HKLM\SOFTWARE\ASIO\Delta_Cast ASIO`로 등록.

<br/>

---

### Delta HFT — Low-Latency Algorithmic Trading Bot (Binance Futures)

> 나노초 단위 레이턴시와 시스템 프로그래밍 역량을 금융 도메인에 적용한 실전 HFT 봇입니다. 장세(Market Regime)를 실시간으로 분류하고, 각 장세에 특화된 완전히 다른 전략 엔진을 동적으로 라우팅합니다.

**Architecture — 4-Core Pinned Design**

| Core |      Thread     | Role |
|------|-----------------|------|
| 0    | Config Watchdog | `inotify` 파일 감시 → `atomic<Config*>` pointer swap (Poor man's RCU) |
| 1    | Market Thread   | Binance WebSocket `bookTicker` + `aggTrade` 수신                      |
| 2    | Trade Thread    | 주문 체결/취소 WebSocket 수신                                          |
| 3    | Engine Loop     | 시세 처리 → 레짐 판독 → 전략 엔진 → 주문 관리                            |

- **Config Hot-Swap (Runtime RCU)** : `inotify`로 `config.json` 변경 감지 후 `g_active_config.exchange(new_config)`로 단 한 번의 원자 연산으로 전체 설정 교체. 봇 재시작 없이 실시간 파라미터 튜닝 가능.

- **Market Regime Detection** : 매 60초 변동성%, Taker 순델타, OU(Ornstein-Uhlenbeck) β계수를 조합해 장세를 CHOPPY / TRENDING / TOXIC 세 가지로 분류. 각 레짐에 완전히 다른 엔진을 라우팅.

- **RollingZEngine (횡보장)** : 롤링 Z-Score 평균회귀 + EMA OU 계수 필터 + OFI(Order Flow Imbalance) 3중 조건 진입. `RollingZScore`는 `sum` / `sum_sq` O(1) 업데이트, 부동소수점 누적 오차는 윈도우 순환 시 전체 재계산으로 리셋.

- **KinematicEngine (추세장)** : 가격을 물리 시스템으로 모델링(위치·속도·가속도). `PhysicsState`를 AVX2 `_mm256_fmadd_pd`로 256비트 레지스터 1개에 Kalman 업데이트. 가속도 + Jerk(가속도 변화율) + OBI 3중 조건으로 "가속 중인 추세"만 포착.

- **HawkesEngine (TOXIC)** : Hawkes Process 자기흥분 점 과정 모델. `hawkes_energy`는 이벤트 발생 시 `+alpha`, 시간 경과 시 `exp(-beta*dt)` 감쇠. 에너지 임계 초과 시 OBI 방향으로 진입해 충격 이후 여진(aftershock) 포착.

- **주문 상태 머신** : `NONE → PENDING_ENTRY → LONG/SHORT → PENDING_EXIT → PENDING_EMERGENCY` 6단계 상태 머신. Ghost Fix(웹소켓 유실 감지 후 REST 동기화), Soft Stop(지정가 추격) → Hard Stop(시장가) 2단계 손절, Trailing Stop 내장.

- **Zero-Allocation Hot Path** : Client Order ID를 `std::to_chars`로 스택 버퍼에 직접 생성. 주문 prefix 비교는 `strcmp` 대신 4바이트 정수 비교(`0x5f746e65 == "ent_"`). 주문 응답 파싱은 `simdjson` SIMD JSON 파서 사용.

- **안전한 종료** : SIGINT 1회 → `engine_armed = false`(신규 진입 차단), 지정가로 청산 시도. 10초 초과 시 시장가 강제 청산. SIGINT 2회 → 즉시 종료.

<br/>

---

### Delta Visor — AMD SVM Type-1 Hypervisor (Kernel Driver)

> 하드웨어 가상화 아키텍처의 동작 원리를 직접 검증하기 위해 AMD SVM 스펙을 기반으로 구현한 커널 드라이버 형태의 하이퍼바이저입니다. 실행 중인 Windows OS를 `VMRUN` 단 한 번으로 게스트 VM으로 전환하는 "Blue Pill" 방식을 채택합니다.

- **Blue Pill 가상화** : `DriverEntry`에서 `SetupGuestState`로 현재 CPU 전체 상태(CR/DR/GDT/IDT/MSR)를 VMCB에 스냅샷. VMCB.RIP를 `GuestResume` 레이블로, RSP를 현재 스택으로 설정 후 `VMRUN`. 게스트는 아무 일도 없었던 것처럼 `GuestResume`에서 재개.

- **DPC 기반 전 코어 동시 가상화** : `KeSetTargetProcessorDpc` + `KeInsertQueueDpc`로 시스템 전체 논리 코어에 DPC를 동시 디스패치, 각 코어를 독립적으로 가상화.

- **순수 x64 어셈블리 VM 루프 (`Svm.asm`)** : Non-volatile 레지스터 저장 → `MSR_VM_HSAVE_PA` 설정 → `VMRUN` → `clgi`(원자적 인터럽트 차단) → `HandleVmExit` 호출 → `HostLoop` 복귀. VMMCALL `0xDEADBEEF` 수신 시 `UnloadSequence`로 분기.

- **VMCB 4096바이트 완전 구현** : AMD APM Vol.2 Appendix B 기준 Control Area(0x000~0x3FF)와 State Save Area(0x400~0xFFF) 전체 레이아웃을 `#pragma pack(1)` 구조체로 정밀 구현. `static_assert(sizeof(VMCB) == 4096)` 컴파일 타임 검증.

- **GDT 파싱 및 x64 세그먼트 처리** : `FillSegmentDescriptor`가 TSS/LDT 등 시스템 세그먼트의 128비트 GDT 엔트리(x64 확장 포맷)를 `S` 비트로 감지해 상위 32비트 Base를 별도 추출. Granularity 비트 기반 Limit 확장 처리.

- **GS Base 보존** : VMEXIT 직후 Windows 커널이 사용하는 `MSR_GS_BASE`(KPCR 포인터)를 R15에 백업, 언로드 시 복원. GS Base 변조로 인한 커널 크래시 방지.

<br/>

---

### Delta Tracker — 실시간 손 추적 ML 파이프라인

> MediaPipe TFLite 모델을 C API로 직접 구동하는 실시간 손 추적 파이프라인. Python 없음, 프레임워크 추상화 없음. VTuber 모션 캡처 상업화를 목표로 설계된 기반 시스템.


- **커스텀 SSD 앵커 디코딩** : 모델의 [1, 2016, 18] 리그레서 출력에 맞는 Palm Detection 앵커 그리드를 처음부터 직접 생성. IoU + Containment 기반 NMS로 중복 검출 제거.

- **2단계 추론 파이프라인** : Palm Detection(192×192) → ROI 크롭 → Hand Landmark(224×224, 21 키포인트 × 3축). 트래킹 소실 / 주기 / 연속 랜드마크 실패 시 적응형 재검출, 안정적일 때는 전체 검출 스킵.

- **4단계 필터 체인** : 랜드마크 출력 → (1) 2D 칼만 필터(위치+속도 상태, SoA 레이아웃 2×2 공분산) → (2) Post-Kalman Gate(EMA 전파 전 미세 변동 차단) → (3) EMA 로우패스 → (4) 스냅 데드존(임계 미만 변화 완전 억제). 전 단계가 21 키포인트 × 3축에 대한 SoA 배치 연산.

- **Lock-Free 트리플 버퍼 아키텍처** : 캡처/추론 스레드가 TrackingSnapshot을 트리플 버퍼 atomic swap으로 기록. 렌더 스레드는 독립적으로 최신 스냅샷만 읽어 표시 — 추론 속도가 렌더를 블로킹하지 않음.

- **Zero-Allocation 추론 루프** : cv::Mat 프레임 간 재사용(프레임별 할당 없음). 스택 기반 padded_buffer_로 랜드마크 파싱 중 힙 할당 없음. NMS 벡터(det_buf_, nms_rects_) clear로 재사용.

- **FramePool** : Lock-Free 카메라 프레임 풀로 캡처 타이밍과 렌더 타이밍 분리. 렌더 스레드는 항상 최신 프레임을 티어링 없이 수신.

<br/>

---

## 🛠️ Tech Stack

|   Category  |        Stack        |
|-------------|---------------------|
| Language    | C++20, x64 Assembly (MASM)                                    |
| Graphics    | Direct3D 11, DirectWrite, Direct2D                            |
| Audio       | ASIO SDK, WASAPI (Exclusive / Shared), AVRT                   |
| System      | Windows WDM Kernel Driver, AMD SVM (Hardware Virtualization)  |
| Concurrency | Lock-Free Queue, Atomic, Triple Buffer, DPC, `std::jthread`   |
| SIMD        | AVX2 (`_mm256_fmadd_pd`, `_mm256_cmp_pd`, `_mm256_storeu_ps`) |
| Networking  | Binance WebSocket, simdjson                                   |
| Database    | SQLite (WAL mode, Prepared Statement)                         |
| Platform    | Windows 10/11 x64, Linux (Ubuntu)                             |
| ML / CV | TFLite C API, OpenCV, MediaPipe (Palm Detection, Hand Landmark)   |

<br/>

---

## Repository Structure

```
Delta_Engine/     — Custom D3D11 game engine + BMS rhythm game
Delta_Cast/       — Virtual ASIO driver + WASAPI loopback renderer
delta_hft/        — AMD64 Linux HFT trading bot (Binance Futures)
Delta_Visor/      — AMD SVM Type-1 hypervisor kernel driver (.sys)
Delta_Tracker/    — Real-time hand tracking ML pipeline (TFLite + OpenCV)
```
