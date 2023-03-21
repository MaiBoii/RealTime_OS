# 5장.UART

(공부할 때 이거 열어서 같이 보세용)

PL011 레지스터 목록 (feat. ARM 인포센터)

[https://developer.arm.com/documentation/ddi0183/g/programmers-model/summary-of-registers?lang=en](https://developer.arm.com/documentation/ddi0183/g/programmers-model/summary-of-registers?lang=en)

PL011 UARTDR 설명 (feat. ARM 인포센터)

[https://developer.arm.com/documentation/ddi0183/g/programmers-model/register-descriptions/data-register--uartdr](https://developer.arm.com/documentation/ddi0183/g/programmers-model/register-descriptions/data-register--uartdr)

## UART란?

- Universal Asynchronous Receiver/Transmitter 의 약자.
- 해석하면 **범용 비동기화 송수신기**라고 하는데 그래도 뭔 소린지 모르겠으니 그냥 유아트라고 읽으셈.
- UART는 보통 콘솔 입출력용으로 사용하는데 문자 통신용으로 만든 프로토콜은 아니지만 **터미널을 UART 포트와 연결하면 UART를 통해서 받은 아스키 코드를 그 코드에 해당하는 문자로 화면에 출력할 수 있기 때문임.** 그러면 명령행 인터페이스 쓰듯이 임베디드 시스템을 제어할 수 있음. 물론 펌웨어에 관련된 기능은 만들어져 있어야함.
- 위 링크에 보면 정말 다양한 레지스터들이 많음. 그만큼 수많은 기능들을 하려면 할 수 있지만 일단 지금은 기본적인 기능인 입력과 출력만 사용할 것임.
- 그럼 이를 어떻게 코드로 구현하느냐? 두 가지 방법이 있음.
1. C언어 매크로를 이용하는 방법
2. 구조체를 이용하는 방법

필자는 후자가 좋다고하니 우리는 실리콘벨리 천룡인 이만우 씨를 얌전히 따라가도록 합니다.

 

```c
//PL011의 레지스터 헤더 UART.h 의 코드

#ifndef HAL_RVPB_UART_H_
#define HAL_RVPB_UART_H_

typedef union UARTDR_t
{
    uint32_t all;
    struct {
        uint32_t DATA:8;    // 7:0
        uint32_t FE:1;      // 8
        uint32_t PE:1;      // 9
        uint32_t BE:1;      // 10
        uint32_t OE:1;      // 11
        uint32_t reserved:20;
    } bits;
} UARTDR_t;

typedef union UARTRSR_t
{
    uint32_t all;
    struct {
        uint32_t FE:1;      // 0
        uint32_t PE:1;      // 1
        uint32_t BE:1;      // 2
        uint32_t OE:1;      // 3
        uint32_t reserved:28;
    } bits;
} UARTRSR_t;

typedef union UARTFR_t
{
    uint32_t all;
    struct {
        uint32_t CTS:1;     // 0
        uint32_t DSR:1;     // 1
        uint32_t DCD:1;     // 2
        uint32_t BUSY:1;    // 3
        uint32_t RXFE:1;    // 4
        uint32_t TXFF:1;    // 5
        uint32_t RXFF:1;    // 6
        uint32_t TXFE:1;    // 7
        uint32_t RI:1;      // 8
        uint32_t reserved:23;
    } bits;
} UARTFR_t;

typedef union UARTILPR_t
{
    uint32_t all;
    struct {
        uint32_t ILPDVSR:8; // 7:0
        uint32_t reserved:24;
    } bits;
} UARTILPR_t;

typedef union UARTIBRD_t
{
    uint32_t all;
    struct {
        uint32_t BAUDDIVINT:16; // 15:0
        uint32_t reserved:16;
    } bits;
} UARTIBRD_t;

typedef union UARTFBRD_t
{
    uint32_t all;
    struct {
        uint32_t BAUDDIVFRAC:6; // 5:0
        uint32_t reserved:26;
    } bits;
} UARTFBRD_t;

typedef union UARTLCR_H_t
{
    uint32_t all;
    struct {
        uint32_t BRK:1;     // 0
        uint32_t PEN:1;     // 1
        uint32_t EPS:1;     // 2
        uint32_t STP2:1;    // 3
        uint32_t FEN:1;     // 4
        uint32_t WLEN:2;    // 6:5
        uint32_t SPS:1;     // 7
        uint32_t reserved:24;
    } bits;
} UARTLCR_H_t;

typedef union UARTCR_t
{
    uint32_t all;
    struct {
        uint32_t UARTEN:1;      // 0
        uint32_t SIREN:1;       // 1
        uint32_t SIRLP:1;       // 2
        uint32_t Reserved1:4;   // 6:3
        uint32_t LBE:1;         // 7
        uint32_t TXE:1;         // 8
        uint32_t RXE:1;         // 9
        uint32_t DTR:1;         // 10
        uint32_t RTS:1;         // 11
        uint32_t Out1:1;        // 12
        uint32_t Out2:1;        // 13
        uint32_t RTSEn:1;       // 14
        uint32_t CTSEn:1;       // 15
        uint32_t reserved2:16;
    } bits;
} UARTCR_t;

typedef union UARTIFLS_t
{
    uint32_t all;
    struct {
        uint32_t TXIFLSEL:3;    // 2:0
        uint32_t RXIFLSEL:3;    // 5:3
        uint32_t reserved:26;
    } bits;
} UARTIFLS_t;

typedef union UARTIMSC_t
{
    uint32_t all;
    struct {
        uint32_t RIMIM:1;   // 0
        uint32_t CTSMIM:1;  // 1
        uint32_t DCDMIM:1;  // 2
        uint32_t DSRMIM:1;  // 3
        uint32_t RXIM:1;    // 4
        uint32_t TXIM:1;    // 5
        uint32_t RTIM:1;    // 6
        uint32_t FEIM:1;    // 7
        uint32_t PEIM:1;    // 8
        uint32_t BEIM:1;    // 9
        uint32_t OEIM:1;    // 10
        uint32_t reserved:21;
    } bits;
} UARTIMSC_t;

typedef union UARTRIS_t
{
    uint32_t all;
    struct {
        uint32_t RIRMIS:1;  // 0
        uint32_t CTSRMIS:1; // 1
        uint32_t DCDRMIS:1; // 2
        uint32_t DSRRMIS:1; // 3
        uint32_t RXRIS:1;   // 4
        uint32_t TXRIS:1;   // 5
        uint32_t RTRIS:1;   // 6
        uint32_t FERIS:1;   // 7
        uint32_t PERIS:1;   // 8
        uint32_t BERIS:1;   // 9
        uint32_t OERIS:1;   // 10
        uint32_t reserved:21;
    } bits;
} UARTRIS_t;

typedef union UARTMIS_t
{
    uint32_t all;
    struct {
        uint32_t RIMMIS:1;  // 0
        uint32_t CTSMMIS:1; // 1
        uint32_t DCDMMIS:1; // 2
        uint32_t DSRMMIS:1; // 3
        uint32_t RXMIS:1;   // 4
        uint32_t TXMIS:1;   // 5
        uint32_t RTMIS:1;   // 6
        uint32_t FEMIS:1;   // 7
        uint32_t PEMIS:1;   // 8
        uint32_t BEMIS:1;   // 9
        uint32_t OEMIS:1;   // 10
        uint32_t reserved:21;
    } bits;
} UARTMIS_t;

typedef union UARTICR_t
{
    uint32_t all;
    struct {
        uint32_t RIMIC:1;   // 0
        uint32_t CTSMIC:1;  // 1
        uint32_t DCDMIC:1;  // 2
        uint32_t DSRMIC:1;  // 3
        uint32_t RXIC:1;    // 4
        uint32_t TXIC:1;    // 5
        uint32_t RTIC:1;    // 6
        uint32_t FEIC:1;    // 7
        uint32_t PEIC:1;    // 8
        uint32_t BEIC:1;    // 9
        uint32_t OEIC:1;    // 10
        uint32_t reserved:21;
    } bits;
} UARTICR_t;

typedef union UARTDMACR_t
{
    uint32_t all;
    struct {
        uint32_t RXDMAE:1;  // 0
        uint32_t TXDMAE:1;  // 1
        uint32_t DMAONERR:1;// 2
        uint32_t reserved:29;
    } bits;
} UARTDMACR_t;

typedef struct PL011_t
{
    UARTDR_t    uartdr;         //0x000
    UARTRSR_t   uartrsr;        //0x004
    uint32_t    reserved0[4];   //0x008-0x014
    UARTFR_t    uartfr;         //0x018
    uint32_t    reserved1;      //0x01C
    UARTILPR_t  uartilpr;       //0x020
    UARTIBRD_t  uartibrd;       //0x024
    UARTFBRD_t  uartfbrd;       //0x028
    UARTLCR_H_t uartlcr_h;      //0x02C
    UARTCR_t    uartcr;         //0x030
    UARTIFLS_t  uartifls;       //0x034
    UARTIMSC_t  uartimsc;       //0x038
    UARTRIS_t   uartris;        //0x03C
    UARTMIS_t   uartmis;        //0x040
    UARTICR_t   uarticr;        //0x044
    UARTDMACR_t uartdmacr;      //0x048
} PL011_t;

#define UART_BASE_ADDRESS0       0x10009000
#define UART_INTERRUPT0          44

#endif /* HAL_RVPB_UART_H_ */
```

- 존나 길어서 짜증나기는 한데 무엇보다 가장 빡치는건 씹C 저자가 공용체(union)는 백 번 중에 한 번 볼까말까 한다고 해서 걍 대충 흝고 넘겼는데 여기서 나와버리는 바람에 오랜만에 씹C를 꺼내서 봐야했다는 것이다.
- 일단 중요한건 **각 레지스터별로 모든 비트 오프셋은 구조체의 비트 멤버 변수 선언을 사용해서 정의**한다는 것임. 각각의 구조체를 하나의 큰 구조체로 묶어서 만들 수 있음. 씹C에 있던 구조체 안에 구조체 만들기 기억나지요? 다 묶어 줍시다.
- 일단 RealView PB에서 UART의 기본 주소는 0x10009000 이랍니다. 그리고 해당 레지스터의 각 비트를 ARM사에서 친절하게 적어준 비트 오프셋대로 정의한다면 코드를 구현할 수 있다!
- 오프셋에 맞춰서 하위 구조체를 배치해 놓았으니 UART 하드웨어의 베이스 주소만 할당하면 나머지 레지스터는 구조체 메모리 접근 규칙에 따라 이름으로 접근가능해서 컴파일러가 알아서 구조체 정의에 맞춰서 계산해줌. **메모리 주소로 접근 가능한 레지스터를 구조체로 추상화 한것이니 여따 실제 메모리 주소를 지정하면 해당 메모리에 있는 데이터를 구조체 모양대로 읽어오게 되는 것!**
- 이걸 hal이라는 디렉터리를 만들어서 넣어줍시다. 참고로 HAL 은 Hardware Abstraction Layer 라는 뜻으로 쉽게 말해서 **하드웨어랑 소프트웨어(기능 코드) 사이에서 작동하는 번역기 모임** 같은거라고 보면 됨.
- 그리고 이 hal 디렉토리 안에다가 각 하드웨어들을 제어할 수 있는 변수를 선언해 줄거임. Regs.c 라고 RealView PB의 레지스터를 선언한 것들을 다 여따 모아줄 예정임.

## UART 공용 인터페이스

- 개발 하드웨어야 각자 알아서 돌아가더라도 이들을 사용하는 코드는 공용 인터페이스를 통해서 같은 방식으로 사용할 수 있어야 하기 때문에 일종의 디바이스 드라이버 같은것이 필요하다.
- 정식으로 상용화된 OS에서는 수많은 디바이스 드라이버 레이어가 필요하기 때문에 복잡한 만큼 범용적으로 쓸 수 있어야 하기 때문에 제작 난이도가 높다. 하지만 우리는 그런건 아니니까 적당한 범용성만 감당하도록 API만 정의해 놓고 해당 API를 각자의 하드웨어가 구현하는 식으로만 구현해보자.
- HAL이 API이다. (UART API, Timer API, GPIO API) 그리고 해당 API를 사용하는 개발보드 대용이 우리가 qemu로 구현하고 있는 RealViewPB이다.
-