# Delta Engine & Delta Cast & SVM Hypervisor

> ** Notice: 코어 소스코드 비공개 안내**
> 본 레포지토리는 포트폴리오 쇼케이스용 공간입니다. 
> 커스텀 D3D11 엔진, 커널 레벨 가상 오디오 드라이버, 그리고 AMD SVM 하이퍼바이저의 핵심 소스코드는 지적재산권(IP) 보호 및 보안상의 이유로 Private 처리되어 있습니다. 
> *상세 코드는 입사 지원 시 제출된 압축 파일(ZIP)을 통해 확인하실 수 있습니다.*

<br/>

## 📊 Performance Metrics & System Logs
<img width="975" height="509" alt="image" src="https://github.com/user-attachments/assets/da00b32d-2b16-4574-a3df-d2f5b55873e9" />
<img width="3840" height="1080" alt="image" src="https://github.com/user-attachments/assets/e3373d5c-13de-4763-b3a8-f88d8c302a73" />
<img width="3840" height="1059" alt="image" src="https://github.com/user-attachments/assets/055ee063-5f73-4116-bc4c-cc370ec22fc0" />

<br/>

## ⚙️ Core Architecture & Features

### Delta Cast (Virtual Audio Driver & A/V Sync)
OS 오디오 믹서의 지연 현상을 원천 차단하기 위해 구현한 가상 오디오 드라이버입니다.
* WASAPI Core Audio API 직결 : 커스텀 오디오 렌더러 구현.
* VBlank Hooking : OS 디스플레이 서브시스템(D3DKMT)의 VBlank 신호를 직접 후킹.
* Kalman Filter 지터 보정 : 오디오 샘플링 레이트와 렌더링 프레임 틱을 실시간 동기화.

### Delta Engine (C++ / D3D11 Custom Engine)
상용 엔진의 추상화 오버헤드를 배제하고 설계한 극한의 로우레벨 멀티스레드 엔진입니다.
* Lock-Free 렌더링 파이프라인 : 뮤텍스 없이 캐시 라인 패딩(`alignas`)과 메모리 오더링을 활용한 트리플 버퍼링 스냅샷 패턴 적용.
* SIMD 기반 병렬 판정 : AVX2(`_mm256`) 명령어와 `[[likely]]` 속성을 통한 분기 예측 최적화.
* Data-Oriented Design (SoA) : 통짜 메모리 아레나(Arena)를 구축하여 동적 할당 오버헤드 원천 차단.

### Delta Visor (AMD SVM Hypervisor)
멀티스레드 제어와 하드웨어 아키텍처 분석을 위해 스크래치부터 구현한 Type-2 하이퍼바이저입니다.
* **DPC 기반 멀티코어 가상화**: Windows 커널의 DPC(Deferred Procedure Call)를 활용하여 시스템의 모든 논리 코어를 독립적으로 가상화 및 초기화.
* **VM Exit/Entry 제어**: 순수 x64 어셈블리를 활용한 Context Switching 및 MSR(GS Base) 제어.
