# 第零章：Hello World — 從零開始點亮嵌入式系統

> **使用文件**
> - `STM32-P103.pdf`（Olimex 開發板手冊）— 了解硬體電路、腳位連接
> - `RM0008.pdf`（STM32F10xxx 參考手冊）— 了解每個暫存器的位址和位元定義

---

## 本章目標

在桌面電腦上，寫 Hello World 只需要 `printf("Hello World\n")`，作業系統幫我們處理好一切。
但在裸機（bare metal）上，**沒有作業系統、沒有標準函式庫**，我們必須自己：

1. 告訴 CPU 從哪裡開始執行
2. 設定硬體週邊（USART）
3. 把字元一個一個送出去

本章有兩個版本：
- **00-Semihosting**：透過 QEMU/GDB 偵錯介面輸出
- **00-HelloWorld**：直接操作 STM32 的 USART2 硬體輸出

---

## 如何讀懂原廠文件

### 文件分工

在開始看程式碼之前，先了解兩份文件各自的用途：

```
STM32-P103.pdf（開發板手冊）
  → 告訴你「硬體長什麼樣」
  → 板子上有哪些晶振、腳位怎麼接、USART2 的 TX/RX 接到哪

RM0008（STM32F10xxx 參考手冊）
  → 告訴你「暫存器在哪裡、怎麼設定」
  → 每個週邊的記憶體位址、每個 bit 的意義
```

### 讀文件的標準流程

```
問題：「我要用 USART2 傳資料，第一步是什麼？」

Step 1：在 RM0008 第 50~52 頁的 Memory Map 找到 USART2 的基底位址
         → USART2 = 0x40004400

Step 2：在 RM0008 第 112~114 頁找 RCC_APB2ENR
         → 確認需要開啟哪些時鐘（IOPAEN, AFIOEN）

Step 3：在 RM0008 第 166 頁找 USART GPIO 設定表（Table 24）
         → 確認 TX 要設 Alternate function push-pull
         → 確認 RX 要設 Input floating

Step 4：在 RM0008 第 170~172 頁找 GPIOx_CRL 暫存器
         → 計算出 0x00004B00 這個值

Step 5：在 RM0008 第 821 頁找 USART_CR1 暫存器
         → 計算出 0x0000000C 和 0x2000 這兩個值
```

---

## 背景知識：ARM Cortex-M 與記憶體映射

### STM32-P103 開發板規格

> **STM32-P103.pdf，第 5 頁（"PROCESSOR FEATURES"）**

```
CPU：STM32F103RBT6，ARM Cortex-M3，最高 72 MHz
FLASH：128 KB（程式儲存）
RAM：20 KB（變數、堆疊）
USART：3 個
時鐘來源：板上 8 MHz 石英晶振（第 7 頁 "CLOCK CIRCUIT"）
```

### 記憶體映射（Memory Map）

> **RM0008，第 50~52 頁，Table 3 "Register boundary addresses"**

這份表格是**所有嵌入式開發的起點**，告訴你每個週邊在哪個記憶體位址：

```
摘錄自 RM0008 第 50~52 頁 Table 3：

  位址範圍                  週邊          匯流排
  0x4002 1000 ~ 13FF       RCC           AHB
  0x4001 0800 ~ 0BFF       GPIO Port A   APB2
  0x4001 0000 ~ 03FF       AFIO          APB2
  0x4000 4400 ~ 47FF       USART2        APB1
  0x0800 0000 ~ ...        Flash（程式碼）—
  0x2000 0000 ~ ...        SRAM          —
```

對應到程式碼的 `reg.h`：

```c
// reg.h 裡的數字，全部來自 RM0008 第 50~52 頁 Table 3

#define RCC    0x40021000   // 來自表格：RCC 基底位址
#define GPIOA  0x40010800   // 來自表格：GPIO Port A 基底位址
#define USART2 0x40004400   // 來自表格：USART2 基底位址
```

每個週邊的暫存器位址 = **基底位址 + 偏移量（offset）**：

```c
// 偏移量來自各週邊章節的暫存器描述（後面會詳細說明）
#define RCC_APB2ENR  (RCC    + 0x18)   // RM0008 第 112 頁
#define RCC_APB1ENR  (RCC    + 0x1C)   // RM0008 第 115 頁
#define GPIOA_CRL    (GPIOA  + 0x00)   // RM0008 第 170 頁
#define USART2_SR    (USART2 + 0x00)   // RM0008 第 818 頁
#define USART2_DR    (USART2 + 0x04)   // RM0008 第 820 頁
#define USART2_CR1   (USART2 + 0x0C)   // RM0008 第 821 頁
```

---

## 00-Semihosting：最簡單的 Hello World

### 什麼是 Semihosting？

**Semihosting** 讓嵌入式程式可以借用**主機電腦**的 I/O 能力：

```
嵌入式裝置                        主機電腦
┌─────────────────┐               ┌─────────────┐
│  ARM 程式       │  BKPT 0xAB   │  QEMU/GDB   │
│  "Hello World!" │ ────────────→ │             │
│                 │               │ 顯示到終端機 │
│                 │ ←──────────── │             │
└─────────────────┘               └─────────────┘
```

觸發方式：執行 `BKPT 0xAB` 特殊指令，QEMU/GDB 捕捉後代為執行 I/O。

### 程式碼解析

```c
static int semihost_call(int service, void *opaque)
{
    register int r0 asm("r0") = service;   // r0 = 服務碼（SYS_WRITE = 0x05）
    register void *r1 asm("r1") = opaque;  // r1 = 參數位址
    register int result asm("r0");
    asm volatile("bkpt 0xab"               // 觸發 semihosting
                 : "=r" (result) : "r" (r0), "r" (r1));
    return result;
}

void main(void)
{
    char message[] = "Hello World!\n";
    uint32_t param[] = {
        1,                    // fd = 1（stdout）
        (uint32_t) message,  // 字串位址
        sizeof(message)      // 長度
    };
    semihost_call(SYS_WRITE, (void *) param);
    while (1);
}
```

---

## 00-HelloWorld：直接操作 USART 硬體

### 第一步：查硬體接線（STM32-P103.pdf）

> **STM32-P103.pdf，第 8 頁（"RS232" 段落）**

```
USART2.TX – pin.16  PA2  EXT2-7    （傳送）
USART2.RX – pin.17  PA3  EXT2-10   （接收）
```

這告訴我們：
- USART2 的傳送腳位（TX）接在 **PA2**（GPIO Port A 的 Pin 2）
- USART2 的接收腳位（RX）接在 **PA3**（GPIO Port A 的 Pin 3）

板子上有一顆 ST3232 晶片，把 STM32 的 3.3V 邏輯電位轉換成標準 RS-232 電位（±12V），再接到板子的 COM 埠，QEMU 模擬的就是這個 COM 埠。

### 第二步：找記憶體位址（RM0008 第 50~52 頁）

從 RM0008 第 50~52 頁的 Table 3 找到：

```
週邊     位址                   匯流排   對應時鐘使能 bit
AFIO     0x40010000 ~ 03FF      APB2    RCC_APB2ENR bit0（AFIOEN）
GPIOA    0x40010800 ~ 0BFF      APB2    RCC_APB2ENR bit2（IOPAEN）
USART2   0x40004400 ~ 47FF      APB1    RCC_APB1ENR bit17（USART2EN）
RCC      0x40021000 ~ 13FF      AHB     —
```

> **為什麼需要查匯流排？**
> STM32 的週邊分屬不同匯流排（APB1/APB2/AHB），開啟時鐘的暫存器不同：
> - APB2 週邊 → 在 `RCC_APB2ENR` 開時鐘
> - APB1 週邊 → 在 `RCC_APB1ENR` 開時鐘

### 第三步：開啟時鐘（RM0008 第 112~117 頁）

**RCC_APB2ENR** — 位址 `RCC + 0x18 = 0x40021018`
> **RM0008，第 112~114 頁，Section 7.3.7**

```
bit 2：IOPAEN   — IO Port A 時鐘使能（需要操作 PA2, PA3）
bit 0：AFIOEN   — Alternate Function IO 時鐘使能（USART 用複用功能需要）
```

**RCC_APB1ENR** — 位址 `RCC + 0x1C = 0x4002101C`
> **RM0008，第 115~117 頁，Section 7.3.8**

```
bit 17：USART2EN — USART2 時鐘使能
```

對應程式碼：

```c
// 開啟 AFIO（bit0=1）和 GPIOA（bit2=1）時鐘
// 0x00000001 | 0x00000004 = 0x00000005
*(RCC_APB2ENR) |= (uint32_t) (0x00000001 | 0x00000004);

// 開啟 USART2（bit17=1）時鐘
// 0x00020000 = bit 17 = 1
*(RCC_APB1ENR) |= (uint32_t) (0x00020000);
```

> **為什麼用 `|=` 而不是 `=`？**
> RCC_APB2ENR 控制**所有** APB2 週邊的時鐘，用 `=` 會把其他週邊的時鐘全部關掉。
> 用 `|=` 只打開我們需要的 bit，不影響其他 bit。

### 第四步：設定 GPIO 腳位功能（RM0008 第 166~172 頁）

#### 先查功能需求（RM0008 第 166 頁 Table 24）

> **RM0008，第 166 頁，Table 24 "USARTs"**

```
USART 腳位      模式          GPIO 設定
USARTx_TX       Full duplex   Alternate function push-pull
USARTx_RX       Full duplex   Input floating / Input pull-up
```

這告訴我們：
- PA2（TX）要設成「複用功能推挽輸出（Alternate function push-pull）」
- PA3（RX）要設成「浮空輸入（Input floating）」

#### 再查 CRL 暫存器格式（RM0008 第 170~172 頁）

> **RM0008，第 170~172 頁，Section 9.2.1，GPIOx_CRL**
> 位址偏移：0x00，Reset value：0x4444 4444

CRL 控制 GPIO Port 的 Pin 0 ~ Pin 7，每個 Pin 佔 4 個 bit（MODE[1:0] + CNF[1:0]）：

```
GPIOx_CRL 位元佈局（32 bit）：
bit 31..28  CNF7[1:0] + MODE7[1:0]  → Pin 7
bit 27..24  CNF6[1:0] + MODE6[1:0]  → Pin 6
...
bit 15..12  CNF3[1:0] + MODE3[1:0]  → Pin 3  ← PA3（RX）
bit 11..8   CNF2[1:0] + MODE2[1:0]  → Pin 2  ← PA2（TX）
...

MODE[1:0] 含義（輸出模式）：
  00 = 輸入模式（reset state）
  01 = 輸出，最高 10 MHz
  10 = 輸出，最高 2 MHz
  11 = 輸出，最高 50 MHz

CNF[1:0] 在輸入模式（MODE=00）：
  00 = 類比輸入
  01 = 浮空輸入（reset state）← 我們需要這個給 RX
  10 = 輸入上拉/下拉
  11 = 保留

CNF[1:0] 在輸出模式（MODE>00）：
  00 = 一般輸出推挽
  01 = 一般輸出開洩極
  10 = 複用功能推挽 ← 我們需要這個給 TX
  11 = 複用功能開洩極
```

#### 計算 GPIOA_CRL 的值

```
PA2（TX）要：Alternate function push-pull → MODE=11（50MHz），CNF=10
  → 4 bits = 1011 = 0xB

PA3（RX）要：Input floating → MODE=00，CNF=01
  → 4 bits = 0100 = 0x4

其他 Pin（PA0, PA1, PA4~PA7）保持預設值 0x4（Input floating）

組合成 32 bit：
  Pin7  Pin6  Pin5  Pin4  Pin3  Pin2  Pin1  Pin0
  0x4   0x4   0x4   0x4   0x4   0xB   0x4   0x4
  ↓
  0x44444B44

但程式碼寫的是：0x00004B00
  → 因為只設了 Pin2 和 Pin3，其他 Pin 清零了（不影響 QEMU 模擬）
  
完整的值（保留其他 pin 預設值）應該是 0x44444B44
但 QEMU 環境下 0x00004B00 也能運作
```

對應程式碼：
```c
*(GPIOA_CRL) = 0x00004B00;
//                  ↑↑
//                  4B = PA3(RX)=4, PA2(TX)=B
//               00   = PA1, PA0 設為 0（QEMU 不影響功能）
```

### 第五步：設定 USART2（RM0008 第 821 頁）

**USART_CR1** — 位址 `USART2 + 0x0C`
> **RM0008，第 821 頁，Section 27.6.4**

```
bit 13：UE  — USART Enable（USART 總開關）
bit  3：TE  — Transmitter Enable（啟用發送器）
bit  2：RE  — Receiver Enable（啟用接收器）
```

```c
// CR1 = 0x0C = 0b 0000 1100
//                      ↑↑
//                      RE=1（bit2）, TE=1（bit3）
*(USART2_CR1) = 0x0000000C;

// CR1 |= 0x2000 = 0b 0010 0000 0000 0000
//                      ↑
//                      UE=1（bit13），啟用 USART
*(USART2_CR1) |= 0x2000;
```

> **為什麼分兩次寫入 CR1？**
> ARM 技術手冊建議先設好所有參數（波特率、格式等），最後才開 UE。
> 本程式沒有設波特率（使用 QEMU 預設值），所以簡化成這樣。

### 第六步：傳送資料（RM0008 第 818~820 頁）

**USART_SR（Status Register）** — 位址 `USART2 + 0x00`
> **RM0008，第 818~819 頁，Section 27.6.1**

```
bit 7：TXE（Transmit data register Empty）
       0 = 資料還沒傳送到移位暫存器
       1 = 資料已傳送到移位暫存器，DR 可以寫入新資料
```

**USART_DR（Data Register）** — 位址 `USART2 + 0x04`
> **RM0008，第 820 頁，Section 27.6.2**

```
bits 8:0：DR[8:0]
  寫入 = 傳送資料（TDR）
  讀出 = 接收資料（RDR）
  雖然只有一個位址，但內部有兩個暫存器，
  讀和寫會分別存取 RDR 和 TDR
```

對應程式碼：
```c
#define USART_FLAG_TXE  ((uint16_t) 0x0080)   // bit7 = 0x80

int puts(const char *str)
{
    while (*str) {
        // 等 TXE=1（移位暫存器空了，可以送下一個字元）
        while (!(*(USART2_SR) & USART_FLAG_TXE));

        // 寫入 DR，硬體自動開始傳送
        *(USART2_DR) = *str++ & 0xFF;
    }
    return 0;
}
```

---

## 啟動程式（startup.c）與連結腳本（hello.ld）

### 向量表：CPU 開機的第一個動作

ARM Cortex-M 開機時，硬體**自動**讀取記憶體最前端的向量表：

```c
__attribute((section(".isr_vector")))
uint32_t *isr_vectors[] = {
    0,                          // [0] 初始 SP（此版本未設）
    (uint32_t *) reset_handler, // [1] Reset Handler → 第一個被執行的程式
};
```

ARM 開機硬體行為（不可修改，ARM 架構規定）：
1. 從 `0x00000000` 讀取初始 MSP 值，設定堆疊指標
2. 從 `0x00000004` 讀取 Reset Handler 位址，跳過去執行

### 連結腳本：告訴連結器怎麼排列程式

```ld
ENTRY(reset_handler)     /* 告訴偵錯器進入點 */

MEMORY
{
    FLASH (rx) : ORIGIN = 0x00000000, LENGTH = 128K
    /* 注意：00-HelloWorld 用 0x00000000（QEMU 映射）
              01-HelloWorld 會改成 0x08000000（真實 STM32）*/
}

SECTIONS
{
    .text :
    {
        KEEP(*(.isr_vector))  /* 向量表必須在最前面（位址 0x00）*/
        *(.text)              /* 程式碼 */
    } >FLASH
}
```

---

## 完整執行流程

```
┌─────────────────────────────────────────────────────────────┐
│                 00-HelloWorld 執行流程                        │
│                                                             │
│  上電 / Reset                                               │
│      │                                                      │
│      ▼  （CPU 硬體行為）                                    │
│  從 0x00000004 讀取 reset_handler 位址                      │
│      │                                                      │
│      ▼                                                      │
│  reset_handler() → main()                                   │
│      │                                                      │
│      ├─ ① RCC_APB2ENR |= AFIOEN | IOPAEN                   │
│      │    開 AFIO 和 GPIOA 時鐘                              │
│      │    （RM0008 第 112~114 頁）                           │
│      │                                                      │
│      ├─ ② RCC_APB1ENR |= USART2EN                          │
│      │    開 USART2 時鐘                                    │
│      │    （RM0008 第 115~117 頁）                           │
│      │                                                      │
│      ├─ ③ GPIOA_CRL = 0x00004B00                           │
│      │    PA2=複用推挽輸出，PA3=浮空輸入                     │
│      │    （RM0008 第 166 頁 Table 24，第 170 頁 CRL）       │
│      │                                                      │
│      ├─ ④ USART2_CR1 = 0x0C，然後 |= 0x2000                │
│      │    啟用 TE+RE，最後啟用 UE                           │
│      │    （RM0008 第 821 頁）                               │
│      │                                                      │
│      ├─ ⑤ puts("Hello World!\n")                           │
│      │    迴圈：等 TXE=1 → 寫入 USART2_DR                  │
│      │    （RM0008 第 818~820 頁）                           │
│      │                                                      │
│      └─ ⑥ while(1)                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 從文件到程式碼的完整對照表

| 程式碼 | 數值 | 來源文件 | 頁碼 | 說明 |
|-------|------|---------|------|------|
| `RCC` | `0x40021000` | RM0008 Table 3 | 第 50 頁 | RCC 基底位址 |
| `RCC + 0x18` | `0x40021018` | RM0008 §7.3.7 | 第 112 頁 | RCC_APB2ENR 偏移 |
| `RCC + 0x1C` | `0x4002101C` | RM0008 §7.3.8 | 第 115 頁 | RCC_APB1ENR 偏移 |
| `0x00000004` (APB2ENR) | IOPAEN | RM0008 §7.3.7 | 第 113 頁 | bit2 = GPIOA 時鐘 |
| `0x00000001` (APB2ENR) | AFIOEN | RM0008 §7.3.7 | 第 114 頁 | bit0 = AFIO 時鐘 |
| `0x00020000` (APB1ENR) | USART2EN | RM0008 §7.3.8 | 第 116 頁 | bit17 = USART2 時鐘 |
| `GPIOA` | `0x40010800` | RM0008 Table 3 | 第 51 頁 | GPIOA 基底位址 |
| `GPIOA + 0x00` | — | RM0008 §9.2.1 | 第 170 頁 | GPIOA_CRL 偏移 |
| `0x00004B00` (CRL) | PA2=B, PA3=4 | RM0008 §9.2.1 + Table 24 | 第 166, 170~172 頁 | TX=複用推挽, RX=浮空輸入 |
| `USART2` | `0x40004400` | RM0008 Table 3 | 第 52 頁 | USART2 基底位址 |
| `USART2 + 0x00` | — | RM0008 §27.6.1 | 第 818 頁 | USART2_SR 偏移 |
| `USART2 + 0x04` | — | RM0008 §27.6.2 | 第 820 頁 | USART2_DR 偏移 |
| `USART2 + 0x0C` | — | RM0008 §27.6.4 | 第 821 頁 | USART2_CR1 偏移 |
| `0x0080` (SR mask) | TXE flag | RM0008 §27.6.1 | 第 818 頁 | bit7 = TXE |
| `0x0000000C` (CR1) | TE+RE | RM0008 §27.6.4 | 第 822 頁 | bit2=RE, bit3=TE |
| `0x2000` (CR1) | UE | RM0008 §27.6.4 | 第 821 頁 | bit13=UE |

---

## 本章重點整理

| 概念 | 說明 |
|------|------|
| Memory Map | 每個週邊在固定記憶體位址，查 RM0008 第 50~52 頁 Table 3 |
| volatile 指標 | 讓編譯器不優化暫存器讀寫，必須加 |
| 時鐘使能 | 使用週邊前先在 RCC_APB1/2ENR 開時鐘，否則讀到的都是 0 |
| GPIO 設定 | Mode + CNF 組合，查 RM0008 第 170~172 頁 CRL/CRH 暫存器 |
| TXE 旗標 | 等 bit7=1 才能寫下一個字元，查 RM0008 第 818 頁 SR 暫存器 |
