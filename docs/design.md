# 설계와 검증

[프로젝트 소개·빌드·조작 방법으로 돌아가기](../README.md)

현재 소스의 실행 경로와 설정값을 기준으로 정리했습니다. 주파수·시간은 별도 표시가 없으면 코드에서 계산한 값이며 실측값이 아닙니다.

## 1. H-bridge 제어 패턴과 하드웨어

### 원리와 확인 범위

H-bridge는 모터 양단의 전압 극성을 바꿔 회전 방향을 제어하는 전력 구동 구조입니다. MCU는 제어 신호를 출력하고, 실제 모터 전류는 외부 드라이버의 전력단이 공급합니다. 이 프로젝트에서 확인한 구현은 **두 핀에 Low/PWM을 배분하는 MCU 측 제어**입니다.

두 입력형 H-bridge 드라이버의 예인 [TI DRV8833 데이터시트 §7.3.2](https://www.ti.com/lit/ds/symlink/drv8833.pdf)는 한 입력 Low·다른 입력 PWM으로 구동하고, 두 입력을 교환해 반대 방향을 구동하는 방식을 설명합니다. 현재 코드의 패턴은 이 방식과 대응합니다. DRV8833은 원리 비교를 위한 예이며 실제 사용 부품으로 확인된 것은 아닙니다.

`Motor_Forward()`는 PA0=Low·PA1=PWM, `Motor_Reverse()`는 PA0=PWM·PA1=Low를 적용합니다. F/R은 펌웨어가 붙인 방향 이름이므로 특정 드라이버의 Forward/Reverse 표기와 같다고 가정하지 않습니다. 실제 방향은 드라이버와 모터 연결에 따라 달라집니다.

Low/Low에서 coast인지 brake인지, PWM off 구간의 전류가 어떻게 순환하는지는 사용 드라이버의 진리표와 회로에 따라 확인해야 합니다. 드라이버 내부의 스위칭·보호 동작은 현재 확인 범위에 포함되지 않습니다. MCU 출력 전환 대기의 시간 조건은 다음 절에 정리했습니다.

### 핀과 실행 환경

대상은 STM32F411RE이며, HAL·RTOS 없이 CMSIS 레지스터 정의를 사용합니다. 핀의 Alternate Function은 [STM32F411 데이터시트](https://www.st.com/resource/en/datasheet/stm32f411re.pdf)의 매핑과 대조했습니다.

| 핀 | 설정 | 펌웨어에서의 역할 |
| --- | --- | --- |
| PA0 | AF2 / GPIO 출력 | TIM5 CH1 PWM 또는 Low, 드라이버 입력 1 |
| PA1 | AF2 / GPIO 출력 | TIM5 CH2 PWM 또는 Low, 드라이버 입력 2 |
| PA2 / PA3 | AF7, Pull-up | USART2 TX / RX |
| PC13 | 입력, Active Low 판정 | 사용자 버튼. 내부 pull 설정은 코드에 없음 |

외부 모터 드라이버의 모델·입력 진리표·전원·배선 기록은 저장소에 없습니다. PC13은 보드 회로의 비눌림 High 조건을 확인해야 합니다.

## 2. PWM과 방향 전환

HSI 16 MHz 기준으로 PLLM=8, PLLN=192, PLLP=4를 설정합니다.

```text
SYSCLK = 16 MHz / 8 × 192 / 4 = 96 MHz
HCLK = 96 MHz, PCLK1 = 48 MHz, PCLK2 = 96 MHz
TIM2 / TIM4 / TIM5 입력 클럭 = 2 × PCLK1 = 96 MHz
```

| 타이머 | PSC | ARR | 설정 결과 |
| --- | ---: | ---: | --- |
| TIM2 | 1919 | 149999, 3초 요청 시 | 50 kHz 카운트, 3초 원샷 |
| TIM4 | 1919 | 49 | 50 kHz 카운트, 1 ms 반복 인터럽트 |
| TIM5 | 47 | 99 | 2 MHz 카운트, 20 kHz PWM |

TIM2·TIM5는 32비트, TIM4는 16비트입니다. `TIM_MAX_PLS=0xFFFF`는 현재 사용하지 않습니다. 타이머 동작 기준은 [RM0383의 TIM2~TIM5 설명](https://www.st.com/resource/en/reference_manual/rm0383-stm32f411xce-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)입니다.

| 모터 상태 | PA0 | PA1 |
| --- | --- | --- |
| STOP | GPIO Low | GPIO Low |
| FORWARD | GPIO Low | TIM5 CH2 PWM |
| REVERSE | TIM5 CH1 PWM | GPIO Low |

`Motor_Forward()`·`Motor_Reverse()`는 먼저 두 핀을 GPIO Low로 전환하고 `Sys_Tick` 변화까지 기다린 뒤, CCR을 쓰고 필요한 핀을 AF로 전환합니다. 구동 중 듀티만 바꿀 때도 이 경로를 거칩니다.

`STOP_DELAY_MS=1`은 1 ms tick의 변화로 판정합니다. tick 위상에 따라 대기가 짧아질 수 있고, 인터럽트 지연으로 길어질 수도 있어 최소 1 ms의 출력 차단이나 축 정지를 보장하지 않습니다. TIM4가 진행하지 않으면 대기에서 빠져나오지 못합니다.

TIM5는 Downcounting PWM Mode 1, CCR preload를 사용합니다. CCR 계산은 `floor(99 × duty / 100)`이며 새 값은 update event에서 반영됩니다. [RM0383](https://www.st.com/resource/en/reference_manual/rm0383-stm32f411xce-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)의 비교 규칙상 현재 설정에서 `CCR=0`도 1카운트 High가 될 수 있어, STOP은 CCR 0이 아닌 GPIO Low 전환으로 구현합니다. AF 전환과 CCR 반영 시점이 같다고 가정해서는 안 됩니다.

## 3. 실행 경로와 소스 구성

```text
Reset → crt0.s: 초기 SP, GPIO 준비, .data 복사, .bss 초기화
      → Main: FPU → Clock_Init → USART2 → 표준 출력 설정
      → TIM2 → TIM4 → TIM5 → 버튼 GPIO → 모터 GPIO
      → USART2 RX 인터럽트 활성화 → 시작 로그 → STOP 출력 적용
      → 반복: UART 문자 처리 → 버튼 이벤트 처리
```

| 구현 위치 | 책임 |
| --- | --- |
| [main.c](../main.c), [motor.h](../motor.h) | 모터 상태형 정의와 메인의 상태·마지막 방향·듀티 관리 |
| [key.c](../key.c), [key.h](../key.h) | 버튼 폴링과 이벤트 생성 |
| [motor.c](../motor.c) | 상태를 방향별 GPIO/AF 출력으로 적용 |
| [timer.c](../timer.c), [timer.h](../timer.h) | 시간 기준·타임아웃·PWM 설정 |
| [uart.c](../uart.c), [exception.c](../exception.c) | UART 송수신과 인터럽트 처리 |
| [clock.c](../clock.c), [option.h](../option.h) | 클럭 설정과 클럭·메모리 상수 |
| [crt0.s](../crt0.s), [링커 스크립트](../rom_0x08000000.lds), [runtime.c](../runtime.c) | 시작 코드, 메모리 배치, 표준 입출력·heap 연결 |

링커는 Flash `0x08000000`부터 512 KiB, RAM `0x20000000`부터 128 KiB를 배치합니다. 초기 SP는 `0x20020000`입니다. `printf`는 `_write()`를 통해 USART2로 출력됩니다.

`system_stm32f4xx.c`도 빌드하지만 `SystemInit()` 및 `SystemCoreClockUpdate()`는 호출하지 않습니다. 따라서 `SystemCoreClock`은 초기값 16 MHz에 머물고, 현재 드라이버 계산은 `option.h`의 상수를 사용합니다. USART1과 EXTI 핸들러는 현재 입력 경로에서 사용하지 않습니다. `Sys_Tick`이라는 변수 이름과 달리 시간 기준은 Cortex-M SysTick이 아닌 TIM4입니다.

## 4. 입력과 상태 처리

### 모터 상태

`Main()`의 지역 변수 `MotorController`가 `state`·`last_dir`·`duty`를 보관합니다. STOP은 마지막 방향과 듀티를 유지하고, `F/R`은 현재·마지막 방향을 함께 변경합니다. 숫자 명령은 방향을 바꾸지 않습니다.

| 현재 상태 | `F` | `R` | `S` / 롱클릭 | 숏클릭 |
| --- | --- | --- | --- | --- |
| STOP | FORWARD | REVERSE | STOP | `last_dir`로 시작 |
| FORWARD | FORWARD | REVERSE | STOP | REVERSE |
| REVERSE | FORWARD | REVERSE | STOP | FORWARD |

같은 방향 명령을 다시 입력해도 출력을 재적용합니다. 메인은 매 반복에서 UART를 먼저, 버튼을 나중에 처리하므로 STOP 명령 직후 같은 반복의 숏클릭이 재시작으로 이어질 수 있습니다.

### 버튼 상태

`IDLE → PRESSED → HOLDING` 순서로 진행합니다. `PRESSED`에서는 첫 눌림 감지 후 20 ms가 지난 시점에 다시 입력을 읽고, 눌려 있으면 TIM2를 3초로 시작합니다. 중간 해제에 따라 시간을 다시 시작하지 않으므로 20 ms 연속 안정 상태를 보장하는 방식은 아닙니다.

`HOLDING`에서 타임아웃을 관측하면 롱클릭을 한 번 반환하고 `RELEASED`에서 해제를 기다립니다. 타임아웃 없이 해제를 관측하면 숏클릭을 반환합니다. 해제 디바운스는 없고, 타임아웃과 해제를 함께 관측하면 롱클릭이 우선합니다. 실제 누르기 시작부터 롱클릭 처리까지는 눌림 재확인과 메인 폴링 지연도 포함됩니다.

### ISR과 메인의 데이터 전달

| ISR | 수행 작업 | 메인에서의 사용 |
| --- | --- | --- |
| USART2 | `Uart_Data` 저장, `Uart_Data_In` 설정 | `Uart2_Read_Byte()`로 읽어 명령 해석 |
| TIM2 | 타이머 정지, `TIM2_Timeout_Flag` 설정 | 버튼 FSM에서 롱클릭 판정 |
| TIM4 | `Sys_Tick` 증가 | 버튼 재확인·모터 출력 전환 대기 |

모터 상태 변경과 출력 적용은 메인 문맥에서 수행합니다. 공유 변수의 `volatile`은 접근 생략을 막지만 데이터와 플래그를 하나의 원자적 동작으로 묶지는 않습니다. 특히 UART는 큐가 없는 1바이트 저장 방식으로, 덮어쓰기와 플래그 확인·해제·데이터 읽기 사이의 경쟁 조건이 있습니다.

## 5. 검증과 재현 범위

문서 정리 시점인 **2026-10-30**의 확인 범위입니다.

| 구분 | 확인 결과 | 범위 |
| --- | --- | --- |
| 펌웨어 빌드 | 성공 | Windows, Arm GNU Toolchain 15.2.Rel1 / GCC 15.2.1, `make -j1`. ELF·BIN 생성, 빌드 출력 경고 없음 |
| 정적 확인 | 완료 | 클럭·타이머 계산, 초기화·벡터 연결, 주요 함수 ELF 디스어셈블리 |
| 호스트 로직 검사 | 통과 | 실제 `main.c`·`key.c`를 임시 환경에 복사하고 GPIO·타이머·출력을 대체하여 명령·이벤트·시간 경계 검사 |
| 실기 측정 | 이번 검토에서 미수행 | Flash 기록·보드 실행, PWM 파형, 드라이버 출력, 회전·정지·방향 전환 전류 |

호스트 검사는 UART 입력 조합, 모터 상태 전이, 버튼의 20 ms 경계·타임아웃 우선순위·tick wraparound를 다뤘습니다. 실제 ISR 경쟁 조건이나 타이머 하드웨어를 재현한 검사가 아니며, 임시 검사 도구는 저장소에 포함되어 있지 않습니다.

빌드·조작 명령은 [README](../README.md#빌드와-사용)에 모았습니다. 현재 Makefile은 Windows 설치 경로를 포함하며, 각 object 대상의 recipe가 전체 소스를 함께 컴파일하므로 `-j1`을 사용합니다. `make run`에는 PATH에서 접근 가능한 `STM32_Programmer_CLI.exe`와 SWD 연결이 필요합니다.

## 6. 현재 구현의 제약

| 항목 | 코드에서 확인한 동작·제약 |
| --- | --- |
| UART 연속 수신 | 1바이트 저장과 경쟁 조건으로 명령 손실·중복 처리가 가능. ISR에 SR 상태 확인과 명시적인 overrun 복구가 없음 |
| 정지 응답 | 상태 변경 후 블로킹 로그를 출력하고 GPIO에 적용. STOP 로그는 출력 해제 완료나 실제 축 정지의 확인 응답이 아님 |
| 정지 유지 | 정지 latch 없음. 롱클릭 후 계속 누르고 있어도 UART `F/R`로 재구동 가능 |
| 초기 출력 | TIM5·AF 출력을 먼저 활성화하고 시작 로그 뒤 STOP 적용. 그 사이의 파형과 드라이버 반응은 미측정 |
| 속도·보호 | RPM 피드백, 전류·온도 감시, watchdog과 자동 보호·통신 단절 정지 경로 없음 |
| 예외 | 기본 벡터는 무한 루프. `_Invalid_ISR()`은 기본 벡터에 연결되지 않으며 예외 시 출력 해제 경로 없음 |

<details>
<summary>지원 코드와 설정에서 추가로 확인한 사항</summary>

- `Motor_Init()`의 pull-down 설정은 `PUPDR` bit 10~11, 즉 PA5에 적용됩니다. PA0·PA1이나 PC13의 노이즈 대책으로 설명할 수 없습니다.
- `crt0.s`에서 PA13·PA14를 잠근 후에는 GPIOA의 `LCKR`이 잠깁니다. 따라서 이후 UART 코드의 PA2·PA3 추가 잠금은 주석대로 성립하지 않는 것으로 해석됩니다. 근거: [RM0383의 GPIO lock 규칙](https://www.st.com/resource/en/reference_manual/DM00119316.pdf).
- `_sbrk()`는 실패 시 `NULL`을 반환하지만, 연결된 newlib `_sbrk_r`는 `(void *)-1`을 실패로 검사합니다. heap 하한·실제 stack과의 충돌 검사도 없습니다.
- 듀티 계산에 사용하는 `double`은 현재 ELF에서 `__aeabi_ddiv`·`__aeabi_dmul` 등의 소프트웨어 함수로 처리됩니다. FPU 활성화를 해당 연산의 하드웨어 가속으로 설명하지 않습니다.

</details>

드라이버 모델과 회로가 확인되면 Low/Low 동작, 초기·전환 파형, 전류와 실제 정지 시간을 별도로 측정해야 합니다.
