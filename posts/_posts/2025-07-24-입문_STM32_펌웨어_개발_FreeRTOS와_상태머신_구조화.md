---
layout: post
title: "STM32 펌웨어 개발 – FreeRTOS와 상태머신 구조화"
description: >
  STM32 보드에서 FreeRTOS와 상태머신을 이용해 펌웨어를 구조화하는 방법
sitemap: true
hide_last_modified: true
---

STM32 보드에서 펌웨어를 작성할 때 **FreeRTOS 태스크 + 상태머신(FSM, Finite State Machine)** 조합은 코드 구조를 명확히 하고 유지보수를 쉽게 만들어 줍니다.
이 글에서는 CubeMX로 생성한 프로젝트를 기반으로 주기적으로 실행되는 모니터링 태스크를 만들고, 그 내부 로직을 상태머신으로 관리하는 기본 패턴을 소개합니다.

## FreeRTOS: 태스크 생성과 주기 실행

CubeMX(또는 STM32CubeIDE)에서 FreeRTOS를 활성화하면 기본 헤더/구성파일이 생성됩니다. 아래 코드는 `exampleTask`라는 태스크를 하나 만들고 50ms 주기로 실행되도록 하는 예시입니다.

```c
const osThreadAttr_t exampleTask_attributes = {
  .name = "exampleTask",
  .stack_size = 256 * 4,
  .priority = (osPriority_t) osPriorityNormal,
};
/* Definitions for controlTask */
osThreadId_t controlTaskHandle;
```

### stack_size

- 단위는 바이트입니다(CMSIS-RTOS v2 규격).
- FreeRTOS 내부에서는 워드(4바이트) 단위로 변환되므로, 위 예시는 256워드(≈1KB) 스택을 갖는 태스크가 됩니다.
- 얼마나 줘야 하는지는 태스크에서 사용하는 **지역변수/함수 호출 깊이/printf 사용 여부** 등에 따라 다릅니다. 처음엔 넉넉히(예: 1024B~2048B) 주고, 실행 중 `uxTaskGetStackHighWaterMark()`(FreeRTOS API) 등으로 여유를 측정해 줄여 가세요.

#### stack_size 와 FreeRTOS 힙

`osThreadNew()`를 위처럼 사용하면 스택 메모리와 **TCB(Task Control Block)**는 FreeRTOS 힙에서 동적 할당됩니다.
즉 `stack_size` 값(바이트 단위)만큼의 스택을 `configTOTAL_HEAP_SIZE` 안에서 가져가므로, CubeMX에서 FreeRTOS 설정 창의 **Total heap size** 값을 충분히 크게 잡아야 합니다.

예) heap_4 사용, Total heap size = 20KB 인 프로젝트에서

- monitoringTask: 1KB
- 다른 태스크 3개: 1KB씩 = 3KB
- 큐/세마포어/타이머 등 내부 오브젝트 ~2KB
  - → 총 6KB 사용 → 여유 14KB (OK)
    > 정적 할당을 쓰고 싶다면 CubeMX에서 Support static allocation을 켜고, osThreadAttr_t에 .stack_mem에 미리 준비한 배열 포인터를 넣으면 힙을 쓰지 않고 생성할 수 있습니다.

### priority 설명

- `osPriorityNormal`, `osPriorityAboveNormal`, `osPriorityHigh` … 처럼 여러 단계가 있습니다. **값이 높을수록 더 자주 스케줄**되고 낮은 태스크를 선점(preempt)합니다.
- 처음에는 대부분의 일반 작업을 `osPriorityNormal`로 두고, **시간 제약이 있는 태스크**(예: 통신 수신 처리)를 한 단계 올려 `osPriorityAboveNormal` 정도로 설정하면 됩니다.
- 너무 많은 태스크를 높은 우선순위로 올리면 오히려 낮은 태스크가 기아 상태가 되거나 `Idle` 훅이 돌지 못해 전력 관리가 안될 수 있습니다.

### osThreadId_t (controlTaskHandle)

- `osThreadNew()`로 태스크를 생성하면 반환되는 **핸들(ID)**을 저장하는 변수입니다.
- 이후 `osThreadTerminate(controlTaskHandle)` 같은 API를 호출하거나, 디버깅/모니터링(상태 조회)에 사용합니다.

```c
  exampleTaskHandle = osThreadNew(Monitoring_Task_Start, NULL, &exampleTask_attributes);
```

태스크는 `osThreadNew()`로 생성합니다. (CMSIS-RTOS v2 래퍼가 FreeRTOS의 `xTaskCreate()`를 감싸고 있습니다.)

### 태스크 본체

`Monitoring_Task_Start()` 안에서 실제 반복 루프를 돌립니다.
고정된 주기를 정확히 맞추고 싶다면 `vTaskDelayUntil()`을 사용하는 것이 좋습니다. 이 함수는 **“지난 깨어난 시각 + 주기”까지 블록**하므로, 루프 내부에서 처리 시간이 약간 변동해도 평균 주기가 일정하게 유지됩니다.

```c
void Monitoring_Task_Start(void *argument)
{
  /* USER CODE BEGIN Monitoring_Task_Start */
	TickType_t xLastWakeTime;

	xLastWakeTime = xTaskGetTickCount();
  /* Infinite loop */
  for(;;)
  {
	Monitoring_Task_Main_Loop();

    vTaskDelayUntil(&xLastWakeTime, pdMS_TO_TICKS(50)); // Task period 50ms.
  }
  /* USER CODE END Monitoring_Task_Start */
}
```

> `xLastWakeTime` 포인터를 `vTaskDelayUntil()`에 넘기면 함수 내부에서 자동으로 다음 기준 시각으로 갱신해 줍니다. 만약 루프 수행 시간이 10ms라면 실제로 대기하는 시간은 40ms 정도가 되어 총 50ms 주기가 유지됩니다.

### 흔한 실수 & 팁

- **스택 크기 부족**: `configCHECK_FOR_STACK_OVERFLOW`를 활성화하고 `uxTaskGetStackHighWaterMark()`로 여유를 주기적으로 확인하세요.
- **Busy Loop**: `for(;;){...}` 안에서 `vTaskDelay`/`vTaskDelayUntil` 없이 계속 돌면 CPU를 독점합니다.
- **UART 등 블로킹 호출을 넣을 경우**: 가능하면 큐/세마포어로 비동기화하거나 DMA를 활용해 태스크가 오래 막히지 않게 합니다.

## 상태머신(FSM) 설계

주기적으로 실행되는 태스크 내부 로직이 커질수록 `if/else`나 무작위 함수 호출이 뒤섞여 “스파게티 코드”가 되기 쉽습니다. 이를 방지하기 위해 **상태(enum) + 전이** 구조를 도입합니다.

### 상태 정의

```c
typedef enum {
    MON_MAIN_STATE_INIT = 0,
    MON_MAIN_STATE_RUN = 10,
	MON_MAIN_STATE_WARNING = 20,
	MON_MAIN_STATE_ERROR = 30,
	MON_MAIN_STATE_RESET = 40,
} exampleTaskMainState_t;
// Define task status end
```

- 숫자를 띄워 두면(0,10,20,...) 디버깅 로그에서 육안 구분이 쉽고 중간에 새 상태를 끼워넣기 편합니다.
- 상태 추가 시 반드시 전이 로직에 `case`를 추가하거나 `default` 처리로 안전망을 만드세요.

### 메인 루프

```c
// Task main loop
void Monitoring_Task_Main_Loop()
{
	static exampleTaskMainState_t currentState = MON_MAIN_STATE_INIT;
	static exampleTaskMainState_t nextState;

	switch(currentState) {
		case MON_MAIN_STATE_INIT:
			// Initialization
			nextState = Monitoring_Task_Init();
			break;
		case MON_MAIN_STATE_RUN:
			// Run
			nextState = Monitoring_Task_Run();
			break;
		case MON_MAIN_STATE_ERROR:
			// Error
			nextState = Monitoring_Task_Error();
			break;
		case MON_MAIN_STATE_RESET:
			// Reset
			nextState = Monitoring_Task_Reset();
			break;
	}



	if(nextState != currentState){
		currentState = nextState;
		// Do Something;
	}

}
```

### 전이 함수 예시

각 상태별 함수는 현재 입력/센서 값 등을 검사하여 “다음 상태”를 반환합니다. 예:

```c
exampleTaskMainState_t Monitoring_Task_Init(void)
{
    if(Hardware_Init_OK())
        return MON_MAIN_STATE_RUN;
    else
        return MON_MAIN_STATE_ERROR;
}

exampleTaskMainState_t Monitoring_Task_Run(void)
{
    if(Temp_Is_OverThreshold())
        return MON_MAIN_STATE_WARNING;

    if(Critical_Error_Detected())
        return MON_MAIN_STATE_ERROR;

    return MON_MAIN_STATE_RUN; // 유지
}
```

### 상태머신을 쓰는 이점

1. **가독성**: 현재 시스템이 어떤 단계에 있는지 한눈에 파악.

2. **확장 용이**: 새로운 요구(예: `WARNING/HARD_RESET` 상태) 추가가 쉽다.

3. **테스트**: 각 상태 함수 단위로 유닛테스트 가능 (Mock 입력 → 예상 전이 확인).

4. **FreeRTOS와 자연스러운 결합**: 이벤트(큐/세마포어) 수신 시 상태 전이를 일관되게 처리.

### 추가 개선 아이디어

- **전이 테이블**: 상태×이벤트 매트릭스를 배열로 정의해 코드 변경 시 `switch` 분기 감소.

- **로그/디버깅**: 상태 변경 시 `printf("State: %d -> %d\r\n", old, new);` 또는 ITM 송출.

- **영속화**: 오류 복구 후 재부팅해야 할 경우 Flash에 마지막 상태 저장 후 부팅 시 복원.

## 마무리

이 글에서는 STM32 환경에서 FreeRTOS 태스크를 생성하고, `vTaskDelayUntil()`로 일정 주기를 확보한 뒤 상태머신 패턴으로 로직을 구조화하는 기본 틀을 살펴보았습니다.
다음 글에서는 **큐/세마포어로 외부 이벤트를 FSM에 주입**하거나, **에러 상태에서 Watchdog 연계**하는 확장 패턴을 다룰 예정입니다.

궁금한 점이나 다루었으면 하는 상태 추가 패턴이 있으면 댓글로 남겨 주세요!
