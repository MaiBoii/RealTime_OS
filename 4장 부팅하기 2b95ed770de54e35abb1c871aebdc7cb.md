# 4장. 부팅하기

(4장부터는 부록을 정독하고 와야 독해가 가능하다. 안 그러면 한국말인데 뭔 소린지 모름.)

## 부팅이란?

- 흔히 쓰이는 뜻 : 시스템에 전원이 들어가서 모든 초기화 작업을 마치고 펌웨어가 대기(idle) 상태가 될 때까지를 말함.
- 이 책에서 : 시스템에 전원이 들어가고 ARM 코어가 **리셋 익셉션 핸들러**를 모두 처리한 다음에 우리가 C언어 코드 짤 때 #include <stdio.h> 이랑 main() 함수로 넘어가기 전 그 사이 부분을 다룸.

---

## 실행 파일에서 메모리 (섹션)

- **.text 영역**: 코드가 있는 공간, **용량은 적지만 빠른 메모리**에 배치해야함. 제깍제깍 움직여야하니까
- **.data 영역**: 초기화한 전역 변수가 있는 공간, 전역 변수를 선언할 때 초기 값을 할당해서 선언하면 해당 전역 변수의 저장공간은 여기서 할당됨, **속도가 느려도 용량이 큰 메모리**에 할당해야함.
- **BSS 영역**: 초기화 하기 전에 전역 변수가 있는 공간, 빌드 완료되어 생성된 바이너리 파일에는 심벌과 크기만 들어 있음. **속도가 느려도 용량이 큰 메모리**에 할당해야함.

---

## 익셉션 벡터 테이블 만들기

```c
.text
	.code 32

	.global vector_start
	.global vector_end

	vector_start:
		LDR		PC, reset_handler_addr //Program Counter에다 해당 주소를 집어 넣음.
		LDR		PC, undef_handler_addr
		LDR		PC, svc_handler_addr
		LDR		PC, pftch_abt_handler_addr
		LDR		PC, data_abt_handler_addr
		B		.
		LDR		PC, irq_handler_addr
		LDR		PC, fiq_handler_addr

		reset_handler_addr: 		.word reset_handler //PC에 해당 주소가 들어가면 명령어 실행
		undef_handler_addr: 		.word dummy_handler //나머지는 일단 무한 루프 시켜줌.
		svc_handler_addr: 		.word dummy_handler //dummy라는 뜻에서 알 수 있듯 하는 거 없음
		pftch_abt_handler_addr: 	.word dummy_handler
		data_abt_handler_addr:  	.word dummy_handler
		irq_handler_addr:		.word dummy_handler
		fiq_handler_addr:		.word dummy_handler
	vector_end:

	reset_handler:  //그리고 해당 익셉션에서 할 행동 정의한다.
		LDR		R0, =0x10000000
		LDR		R1, [R0]

	dummy_handler:
		B .
.end
// 결과값은 3장에서 했던거랑 같음.
```

## 익셉션 핸들러 만들기

- 익셉션 핸들러를 만들기 메모리 맵 만들기 위해 동작 모드별 스택 주소를 각 뱅크드 레지스터 SP에 설정하기
- 따로 설정하지 않은 다른 메모리는 나빌로스에서 관리할 예정이라 함.
- 동작 모드별 스택이 모두 설정되고 나면 더 이상 어셈블리 쓸 필요없이 C언어 main() 함수로 진입할 수 있음.

```c
//메모리 시작 주소
#define INST_ADDR_START     0 
#define USRSYS_STACK_START  0x00100000
#define SVC_STACK_START     0x00300000
#define IRQ_STACK_START     0x00400000
#define FIQ_STACK_START     0x00500000
#define ABT_STACK_START     0x00600000
#define UND_STACK_START     0x00700000
#define TASK_STACK_START    0x00800000
#define GLOBAL_ADDR_START   0x04800000
#define DALLOC_ADDR_START   0x04900000

//스택 사이즈 : 
#define INST_MEM_SIZE       (USRSYS_STACK_START - INST_ADDR_START)
#define USRSYS_STACK_SIZE   (SVC_STACK_START - USRSYS_STACK_START)
#define SVC_STACK_SIZE      (IRQ_STACK_START - SVC_STACK_START)
#define IRQ_STACK_SIZE      (FIQ_STACK_START - IRQ_STACK_START)
#define FIQ_STACK_SIZE      (ABT_STACK_START - FIQ_STACK_START)
#define ABT_STACK_SIZE      (UND_STACK_START - ABT_STACK_START)
#define UND_STACK_SIZE      (TASK_STACK_START - UND_STACK_START)
#define TASK_STACK_SIZE     (GLOBAL_ADDR_START - TASK_STACK_START)
#define DALLOC_MEM_SIZE     (55 * 1024 * 1024)

//스택 TOP 주소 : 메모리 시작주소 + 스택 사이즈로 스택의 TOP 주소를 지정할 수 있음. 
#define USRSYS_STACK_TOP    (USRSYS_STACK_START + USRSYS_STACK_SIZE - 4)
#define SVC_STACK_TOP       (SVC_STACK_START + SVC_STACK_SIZE - 4)
#define IRQ_STACK_TOP       (IRQ_STACK_START + IRQ_STACK_SIZE - 4)
#define FIQ_STACK_TOP       (FIQ_STACK_START + FIQ_STACK_SIZE - 4)
#define ABT_STACK_TOP       (ABT_STACK_START + ABT_STACK_SIZE - 4)
#define UND_STACK_TOP       (UND_STACK_START + UND_STACK_SIZE - 4)
```

![images_embeddedjune_post_88c10907-e4d3-45ac-b4d0-c0632b7ac1fd_image.png](4%E1%84%8C%E1%85%A1%E1%86%BC%20%E1%84%87%E1%85%AE%E1%84%90%E1%85%B5%E1%86%BC%E1%84%92%E1%85%A1%E1%84%80%E1%85%B5%202b95ed770de54e35abb1c871aebdc7cb/images_embeddedjune_post_88c10907-e4d3-45ac-b4d0-c0632b7ac1fd_image.png)

```c
//ARMv7AR.h : ARM의 cpsr에 값을 할당하여 동작모드를 바꿀 수 있음.
#define ARM_MODE_BIT_USR 0x10
#define ARM_MODE_BIT_FIQ 0x11
#define ARM_MODE_BIT_IRQ 0x12
#define ARM_MODE_BIT_SVC 0x13
#define ARM_MODE_BIT_ABT 0x17
#define ARM_MODE_BIT_UND 0x1B
#define ARM_MODE_BIT_SYS 0x1F
#define ARM_MODE_BIT_MON 0x16
```

- cpsr의 M영역 값은 0x10이기 때문에 이 값을 펌웨어에서 0x1F로 바꾸면 SYS 모드로 변경된다. 이러다 하드웨어에서 FIQ가 발생하면 0x11로 바뀜.

```c
//4장 기준 MemoryMap.h의 최종형태, SYS모드로 바꿀 경우

#include "ARMv7AR.h"
#include "MemoryMap.h"

.text
    .code 32

    .global vector_start
    .global vector_end

    vector_start:
        LDR PC, reset_handler_addr
        LDR PC, undef_handler_addr
        LDR PC, svc_handler_addr
        LDR PC, pftch_abt_handler_addr
        LDR PC, data_abt_handler_addr
        B   .
        LDR PC, irq_handler_addr
        LDR PC, fiq_handler_addr

        reset_handler_addr:     .word reset_handler
        undef_handler_addr:     .word dummy_handler
        svc_handler_addr:       .word dummy_handler
        pftch_abt_handler_addr: .word dummy_handler
        data_abt_handler_addr:  .word dummy_handler
        irq_handler_addr:       .word dummy_handler
        fiq_handler_addr:       .word dummy_handler
    vector_end:

    reset_handler:
        MRS r0, cpsr.  //MRS: 후자를 전자에 임시저장함.
        BIC r1, r0, #0x1F   // 
        ORR r1, r1, #ARM_MODE_BIT_SVC
        MSR cpsr, r1
        LDR sp, =SVC_STACK_TOP

        MRS r0, cpsr
        BIC r1, r0, #0x1F
        ORR r1, r1, #ARM_MODE_BIT_IRQ
        MSR cpsr, r1
        LDR sp, =IRQ_STACK_TOP

        MRS r0, cpsr
        BIC r1, r0, #0x1F
        ORR r1, r1, #ARM_MODE_BIT_FIQ
        MSR cpsr, r1
        LDR sp, =FIQ_STACK_TOP

        MRS r0, cpsr
        BIC r1, r0, #0x1F
        ORR r1, r1, #ARM_MODE_BIT_ABT
        MSR cpsr, r1
        LDR sp, =ABT_STACK_TOP

        MRS r0, cpsr
        BIC r1, r0, #0x1F
        ORR r1, r1, #ARM_MODE_BIT_UND
        MSR cpsr, r1
        LDR sp, =UND_STACK_TOP

        MRS r0, cpsr
        BIC r1, r0, #0x1F
        ORR r1, r1, #ARM_MODE_BIT_SYS
        MSR cpsr, r1
        LDR sp, =USRSYS_STACK_TOP

				BL   main

    dummy_handler:
        B .
.end
```

- 왜 번거롭게 스택의 탑에다가 지정 해놓냐?
- 왜냐면 스택은 높은 주소에서 낮은 주소로 자라는 특징이 있기 때문임. 왜냐?
1. 낮은 주소에서 높은 주소 방향으로 자란다면 용량이 다 찰 경우 커널 영역을 침범할 위험이 있기 때문이다. OS에서 가장 중요한 부분이기 때문에 버퍼 오버 플로우가 일어나면 큰일남.
2. 공유 라이브러리 영역에서 서로 마주보는 형태로 자란다면 메모리 낭비를 줄일 수 있기 때문. (이라는데 마주보는거랑 메모리 낭비 효율성이랑 뭔 상관인지 모르겠네) 

- 암튼 이렇게 메모리맵 만들어놓고 조건에 맞게 빌드하면 실행파일을 만들 수 있음.
- 끝부분에 브랜치 명령어(BL)을 추가하면 어셈블리에서 C언어 main()으로 넘어갈 수 있음. 단, 다른 파일끼리 넘어가려면 레이블을 .global로 선언해야함.

```c
#include "stdint.h"

void main(void)
{
    uint32_t* dummyAddr = (uint32_t*)(1024*1024*100);
    *dummyAddr = sizeof(long);
}
```