# 第一章：改進版 Hello World — 完整的啟動初始化

> **使用文件**
> - `STM32-P103.pdf`（Olimex 開發板手冊）— 確認 FLASH/RAM 容量與時鐘來源
> - `RM0008`（STM32F10xxx 參考手冊）— RCC 時鐘暫存器、Flash ACR、記憶體佈局

---

## 本章目標

第零章的程式跳過了很多重要初始化步驟，在真實硬體上可能失效。本章補足這些關鍵步驟：

1. **修正 FLASH 位址**：從 `0x00000000`（QEMU 簡化值）改為 `0x08000000`（真實 STM32 位址）
2. **初始化 .data 段**：把儲存在 FLASH 的初始值複製到 RAM
3. **清零 .bss 段**：把未初始化的全域變數歸零
4. **初始化時鐘系統**：從 HSI（內部 RC）切換到 HSE（外部晶振）
5. **完整向量表**：加入堆疊指標初始值、NMI、HardFault

---

## 第一步：找到正確的記憶體位址

### STM32F103 的 FLASH 真實位址

第零章用 `ORIGIN = 0x00000000`，這是 QEMU 的特殊映射，在真實硬體上不正確。

> **RM0008，第 54 頁，Table 5 "Flash module organization (medium-density devices)"**

```
Block       Name    Base address
Main memory Page 0  0x0800 0000 - 0x0800 03FF   1K
            Page 1  0x0800 0400 - 0x0800 07FF   1K
            ...
            Page 127 0x0801 FC00 - 0x0801 FFFF  1K
```

所有 STM32F103 medium-density（包含 STM32F103RBT6）的 Flash 從 `0x08000000` 開始。

> **STM32-P103.pdf，第 5 頁（"PROCESSOR FEATURES"）**
> ```
> FLASH 128KB    → 共 128 頁，每頁 1KB
> RAM 20KB       → 起始位址 0x20000000
> ```

因此連結腳本改為：

```ld
MEMORY
{
    FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 128K   /* 從 RM0008 第 54 頁 */
    RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 20K    /* 從 RM0008 第 53 頁 */
}
```

---

## 第二步：理解 C 程式的記憶體段

### 各段說明

```
FLASH（唯讀，斷電不消失）        RAM（可讀寫，斷電消失）
┌─────────────────────────┐      ┌──────────────────────────┐
│  .isr_vector（向量表）   │      │  .data（有初始值的全域變數）│
├─────────────────────────┤      │  → 開機時從 FLASH 複製過來 │
│  .text（程式碼）         │      ├──────────────────────────┤
│                         │      │  .bss（無初始值全域變數）  │
├─────────────────────────┤      │  → 開機時清零              │
│  .rodata（字串常數等）   │      ├──────────────────────────┤
├─────────────────────────┤      │  Stack（堆疊，向下生長）   │
│  .data 的初始值備份     │◄──複製│  ↓ 0x20005000（_estack）  │
└─────────────────────────┘      └──────────────────────────┘
 ↑ _sidata（LMA）                  ↑ _sdata … _edata（VMA）
```

### 什麼是 LMA vs VMA？

嵌入式系統特有的概念：

| | LMA (Load Memory Address) | VMA (Virtual Memory Address) |
|--|--------------------------|------------------------------|
| 意思 | 資料**儲存**在哪裡 | 程式**執行時**從哪裡訪問 |
| `.data` 的例子 | FLASH（斷電保留） | RAM（可修改） |

```c
// 這個全域變數，編譯器在 FLASH 裡保留一份 42
// 但執行時 count 變數住在 RAM 裡
int count = 42;
```

如果沒有 startup.c 的初始化，count 在 RAM 中的值是**亂數**而不是 42。

### 連結腳本的完整設計

```ld
ENTRY(reset_handler)

MEMORY
{
    FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 128K
    RAM (rwx)  : ORIGIN = 0x20000000, LENGTH = 40K
}

SECTIONS
{
    .text :
    {
        KEEP(*(.isr_vector))   /* 向量表放最前面 */
        *(.text)
        *(.text.*)
        *(.rodata)             /* 字串常數也在 FLASH */
        _sidata = .;           /* 記下 .data 初始值在 FLASH 的起點 */
    } >FLASH

    /* AT(_sidata) 告訴連結器：.data 的 LMA = _sidata（在 FLASH） */
    .data : AT(_sidata)
    {
        _sdata = .;            /* .data 在 RAM 的起點（VMA） */
        *(.data)
        *(.data*)
        _edata = .;            /* .data 在 RAM 的終點（VMA） */
    } >RAM                     /* VMA 在 RAM */

    .bss :
    {
        _sbss = .;
        *(.bss)
        _ebss = .;
    } >RAM

    /* 堆疊從 RAM 頂端開始，向下生長 */
    _estack = ORIGIN(RAM) + LENGTH(RAM);   /* = 0x20000000 + 40K */
}
```

---

## 第三步：完整向量表設計

### ARM Cortex-M3 的標準向量表結構

ARM 架構規定向量表的固定格式（對應的 ARM 文件是 PM0056 Cortex-M3 程式設計手冊）：

```c
__attribute((section(".isr_vector")))
uint32_t *isr_vectors[] = {
    /* [0] */ (uint32_t *) &_estack,          // 初始 MSP（從連結腳本的 _estack）
    /* [1] */ (uint32_t *) reset_handler,     // Reset：開機後第一個執行
    /* [2] */ (uint32_t *) nmi_handler,       // NMI：不可遮蔽中斷
    /* [3] */ (uint32_t *) hardfault_handler  // HardFault：嚴重錯誤
};
```

**索引 [0]** 不是函式位址，而是 **MSP 的初始值**。CPU 開機時，自動把這個值載入 SP 暫存器，設定好堆疊。

```
向量表在 FLASH（位址 0x08000000）：

0x08000000  [&_estack]         CPU 讀取，載入 MSP（= 0x20005000）
0x08000004  [reset_handler]    CPU 跳到此位址執行
0x08000008  [nmi_handler]      NMI 觸發時跳到此
0x0800000C  [hardfault_handler] HardFault 觸發時跳到此
```

---

## 第四步：啟動程式（startup.c）完整解析

### Reset Handler 的工作

```c
void reset_handler(void)
{
    /* ① 複製 .data 段（FLASH → RAM）*/
    uint32_t *idata_begin = &_sidata;   // FLASH 中的初始值（LMA）
    uint32_t *data_begin  = &_sdata;    // RAM 中的目的位址（VMA）
    uint32_t *data_end    = &_edata;
    while (data_begin < data_end)
        *data_begin++ = *idata_begin++;

    /* ② 清零 .bss 段 */
    uint32_t *bss_begin = &_sbss;
    uint32_t *bss_end   = &_ebss;
    while (bss_begin < bss_end)
        *bss_begin++ = 0;

    /* ③ 初始化時鐘系統 */
    rcc_clock_init();

    /* ④ 進入主程式 */
    main();
}
```

---

## 第五步：時鐘初始化（RM0008 第 99~103 頁）

### 為什麼需要時鐘初始化？

> **STM32-P103.pdf，第 7 頁（"CLOCK CIRCUIT"）**
> ```
> 板上有一顆 8 MHz 石英晶振（HSE）
> 內部 PLL 可以把頻率倍增到最高 72 MHz
> ```

STM32 開機預設使用**內部 RC 振盪器（HSI，8 MHz）**，但它精確度差（±1%）。
`rcc_clock_init()` 把系統切換到**外部晶振（HSE，8 MHz）**，精確度更高。

### 相關暫存器

**RCC_CR（Clock Control Register）** — 位址 `RCC + 0x00`
> **RM0008，第 99~100 頁，Section 7.3.1**

```
bit 17：HSERDY（唯讀）— 硬體設為 1 表示 HSE 已穩定
bit 16：HSEON        — 軟體寫 1 啟動 HSE
bit  1：HSIRDY（唯讀）— HSI 穩定旗標
bit  0：HSION        — 啟動 HSI（開機預設已啟用）
```

**RCC_CFGR（Clock Configuration Register）** — 位址 `RCC + 0x04`
> **RM0008，第 101~103 頁，Section 7.3.2**

```
bits 3:2：SWS（唯讀）— 目前系統時鐘來源狀態
          00 = HSI  01 = HSE  10 = PLL
bits 1:0：SW         — 選擇系統時鐘來源
          00 = HSI  01 = HSE  10 = PLL
bits 7:4：HPRE       — AHB 分頻（0xxx = 不分頻）
bits 13:11：PPRE2    — APB2 分頻（0xx = 不分頻）
bits 10:8：PPRE1     — APB1 分頻（0xx = 不分頻）
```

### 時鐘初始化步驟對照

```
程式碼                              RM0008 頁碼  說明
─────────────────────────────────────────────────────────
*RCC_CR |= 0x00000001               第 100 頁   確保 HSION=1（保持 HSI 備用）
*RCC_CFGR &= 0xF8FF0000             第 101 頁   清除 SW, HPRE, PPRE1/2, ADCPRE
*RCC_CR &= 0xFEF6FFFF               第 99 頁    關閉 HSEON, CSSON, PLLON
*RCC_CIR = 0x009F0000               第 103 頁   清除所有時鐘中斷旗標

*RCC_CR |= RCC_CR_HSEON             第 100 頁   啟動 HSE（HSEON=1）
do { HSEStatus = *RCC_CR & HSERDY } 第 99 頁    輪詢 HSERDY，等 HSE 穩定
while (HSEStatus == 0 && counter != TIMEOUT)

*FLASH_ACR |= FLASH_ACR_PRFTBE      第 54 頁    開啟 Flash 預取緩衝
*FLASH_ACR |= FLASH_ACR_LATENCY_0   第 54 頁    0 wait states（8MHz 不需要等待）

*RCC_CFGR |= HPRE_DIV1              第 102 頁   AHB 不分頻（HPRE=0000）
*RCC_CFGR |= PPRE2_DIV1             第 102 頁   APB2 不分頻（PPRE2=000）
*RCC_CFGR |= PPRE1_DIV1             第 102 頁   APB1 不分頻（PPRE1=000）

*RCC_CFGR &= ~RCC_CFGR_SW           第 103 頁   清除目前時鐘選擇
*RCC_CFGR |= RCC_CFGR_SW_HSE        第 103 頁   SW=01，選擇 HSE
while (SWS != 0x04)                  第 103 頁   等 SWS=01（確認切換完成）
```

### `SWS != 0x04` 這個值怎麼算？

> **RM0008，第 103 頁，bits 3:2 SWS 說明**

```
SWS[1:0] = 01（HSE 已成為系統時鐘）
         = 二進位 01，放在 bits 3:2 位置
         = 0b 0000 0100 = 0x04

程式碼：
while ((*RCC_CFGR & RCC_CFGR_SWS) != 0x04);
                                     ↑
                         0x04 = SWS 欄位值為 01，
                         代表 HSE 已成功切換為系統時鐘
```

### 時鐘初始化流程

```
開機預設：HSI（內部 8 MHz）→ SYSCLK

rcc_clock_init() 執行後：

  ┌─────────────────────────────────────────────┐
  │             STM32 時鐘樹                     │
  │                                             │
  │  外部晶振 HSE（8 MHz）                       │
  │      │                                      │
  │      │ HSEON=1，等待 HSERDY=1               │
  │      ▼                                      │
  │  SW=01 → SYSCLK = HSE = 8 MHz              │
  │      │                                      │
  │   HPRE=0                                   │
  │      ▼                                      │
  │   HCLK（AHB） = 8 MHz                      │
  │      │                                      │
  │   ┌──┴──┐                                   │
  │ PPRE1=0 PPRE2=0                             │
  │   │       │                                 │
  │ PCLK1   PCLK2                               │
  │（APB1）  （APB2）                            │
  │  USART2  GPIOA, USART1                      │
  └─────────────────────────────────────────────┘
```

---

## Flash ACR 是什麼？

**Flash ACR（Access Control Register）** — 位址 `0x40022000`
> **RM0008，第 54 頁，Flash module organization 的暫存器列表**

當 CPU 時鐘很快時，Flash 讀取速度跟不上，需要插入「等待週期（wait states）」。

```
Flash wait states（RM0008 相關章節說明）：
  0 WS：時鐘 0~24 MHz 適用     ← 我們 8 MHz 使用這個
  1 WS：時鐘 24~48 MHz 適用
  2 WS：時鐘 48~72 MHz 適用

程式碼：
*FLASH_ACR |= FLASH_ACR_PRFTBE;    // bit4 = 開啟預取緩衝（提高效能）
*FLASH_ACR |= FLASH_ACR_LATENCY_0; // bits 1:0 = 00，0 wait states
```

---

## 整章文件對照總表

| 程式碼或設計 | 數值/設定 | 來源文件 | 頁碼 | 說明 |
|------------|---------|---------|------|------|
| FLASH ORIGIN | `0x08000000` | RM0008 Table 5 | 第 54 頁 | Medium-density Flash 起始位址 |
| RAM ORIGIN | `0x20000000` | RM0008 §3.3.1 | 第 53 頁 | SRAM 起始位址 |
| RAM LENGTH | `40K`（lkd）/`20K`（實際） | STM32-P103.pdf | 第 5 頁 | STM32F103RBT6 有 20KB RAM |
| `_estack` | `RAM起點 + 大小` | — | — | 堆疊從 RAM 最高位址往下 |
| `RCC_CR` | `RCC + 0x00` | RM0008 §7.3.1 | 第 99 頁 | 時鐘控制暫存器 |
| `RCC_CR_HSEON` | bit16 | RM0008 §7.3.1 | 第 100 頁 | 啟動 HSE |
| `RCC_CR_HSERDY` | bit17 | RM0008 §7.3.1 | 第 99 頁 | HSE 就緒旗標 |
| `RCC_CFGR` | `RCC + 0x04` | RM0008 §7.3.2 | 第 101 頁 | 時鐘配置暫存器 |
| `RCC_CFGR_SW_HSE` | `0x01`（SW=01） | RM0008 §7.3.2 | 第 103 頁 | 選擇 HSE 為系統時鐘 |
| `RCC_CFGR_SWS` | `0x0C`（bits 3:2 mask） | RM0008 §7.3.2 | 第 103 頁 | 系統時鐘狀態 |
| `0x04`（等待 SWS） | SWS=01（HSE 已選） | RM0008 §7.3.2 | 第 103 頁 | SWS bits 3:2 = 0b01 = 0x04 |
| `FLASH_ACR` | `0x40022000` | RM0008 Table 5 | 第 54 頁 | Flash 存取控制暫存器 |
| `FLASH_ACR_PRFTBE` | bit4 | RM0008（Flash 章節） | — | 預取緩衝使能 |

---

## 00 vs 01 對照

| 比較項目 | 00-HelloWorld | 01-HelloWorld | 原因（來自文件） |
|---------|--------------|--------------|----------------|
| FLASH 位址 | `0x00000000` | `0x08000000` | RM0008 第 54 頁 Table 5 真實位址 |
| RAM 定義 | 無 | `0x20000000` | RM0008 第 53 頁 SRAM 起始位址 |
| .data 初始化 | 無 | 有 | C 語言標準要求 |
| .bss 清零 | 無 | 有 | C 語言標準要求 |
| 時鐘初始化 | 無（HSI） | HSE 8 MHz | STM32-P103 板上有 8 MHz 晶振 |
| 向量表 SP | 未設定 | `&_estack` | ARM 架構規定向量表 [0] 為初始 SP |
| NMI / HardFault | 無 | 有 | ARM Cortex-M3 規定的異常處理 |

---

## 本章重點整理

| 概念 | 說明 | 參考文件 |
|------|------|---------|
| FLASH 位址 | STM32F103 從 `0x08000000` 開始 | RM0008 第 54 頁 |
| LMA / VMA | 儲存位址 vs 執行位址，.data 段需要複製 | 連結腳本語法 |
| HSI | 內部 8 MHz RC 振盪器，開機預設，精確度差 | RM0008 第 100 頁 |
| HSE | 外部晶振，STM32-P103 上有 8 MHz，精確度高 | STM32-P103 第 7 頁 |
| SWS 旗標 | 確認時鐘切換完成，需等 SWS=01 | RM0008 第 103 頁 |
| Flash wait states | 高頻率時需要插入等待週期，8 MHz 用 0 WS | RM0008 第 54 頁 |
