# 第五章：Timer Interrupt — SysTick 計時器中斷

## 本章目標

前幾章的任務切換都是「協作式」的：任務必須主動呼叫 `syscall()` 才能讓出 CPU。
如果一個任務不讓出，整個系統就卡住了。

**搶佔式（Preemptive）排程**的關鍵是：OS 需要一個**定時觸發的機制**，讓 CPU 定期回到核心，強制切換任務，不管任務想不想讓出。

本章的目標：設定 **SysTick 計時器**，讓它每隔一段時間自動觸發中斷，呼叫 `systick_handler`。
這是第六章搶佔式排程的基礎建設。

執行結果（每隔約 0.9 秒出現一行）：
```
Hello world!
Interrupt from System Timer
Interrupt from System Timer
Interrupt from System Timer
...
```

---

## 背景知識：中斷（Interrupt）是什麼？

### 中斷的概念

想像你正在看書，電話突然響了。你會：
1. **在當前頁夾書籤**（儲存目前狀態）
2. **去接電話**（處理中斷）
3. **接完電話回來**，從書籤繼續讀

CPU 的中斷機制幾乎一模一樣：

```
正常執行流程（while(1) 等待）：
  ...  →  指令 A  →  指令 B  →  指令 C  → ...
                            ↑
                     SysTick 計時到！
                            │
              ┌─────────────┘
              │  1. 硬體自動保存 r0-r3, r12, lr, pc, xpsr
              │     （Exception Frame 壓到堆疊）
              │
              ▼
         systick_handler()
              │  執行中斷處理
              │
              ▼
         返回指令 C 繼續執行
              │  （硬體自動從堆疊恢復暫存器）
              ▼
  ...  →  指令 C  →  指令 D  → ...
```

### 中斷 vs 一般函式呼叫

| 特性 | 一般函式呼叫（bl/bx） | 中斷（Exception） |
|------|----------------|----------------|
| 觸發方式 | 程式主動呼叫 | 硬體自動觸發（不可預期時間） |
| 保存狀態 | 只保存 r14（LR） | 硬體自動保存 8 個暫存器 |
| 執行位址 | 程式中指定 | 從向量表讀取 |
| 返回方式 | `bx lr` | EXC_RETURN（特殊 bx lr） |

---

## SysTick 計時器

### 什麼是 SysTick？

**SysTick（System Tick Timer）** 是 ARM Cortex-M 核心**內建**的計時器，每一顆 Cortex-M 晶片都有，不是 STM32 特有的。設計目的就是給 OS 提供定期中斷。

```
SysTick 原理：

  設定 LOAD = N（倒數起始值）

  每個時鐘週期：
  ┌──────────────────────────────────────────────────┐
  │  VAL = VAL - 1                                   │
  │                                                  │
  │  如果 VAL == 0：                                  │
  │    1. 重新載入 VAL = LOAD（自動重置）              │
  │    2. 設定 COUNTFLAG = 1                          │
  │    3. 如果 TICKINT 已啟用 → 觸發 SysTick 中斷    │
  └──────────────────────────────────────────────────┘

時序示意：
  N → N-1 → N-2 → ... → 2 → 1 → 0 → [中斷！] → N → N-1 → ...
                                       ↑
                               LOAD 重新載入，繼續倒數
```

### SysTick 暫存器

```c
/* SysTick 在記憶體中的位址（ARM 標準，每顆 Cortex-M 都在這裡） */
#define SYSTICK      0xE000E010
#define SYSTICK_CTRL (SYSTICK + 0x00)  // 控制暫存器
#define SYSTICK_LOAD (SYSTICK + 0x04)  // 重新載入值
#define SYSTICK_VAL  (SYSTICK + 0x08)  // 目前計數值
#define SYSTICK_CALIB (SYSTICK + 0x0C) // 校準暫存器（唯讀）
```

#### SYSTICK_CTRL 控制暫存器各位元

```
SYSTICK_CTRL（32 位元，只用低 3 位）：

  bit 16：COUNTFLAG（唯讀）
    ─── 計數到 0 後自動設為 1，讀取後清零

  bit 2：CLKSOURCE（時鐘來源）
    0 = 使用外部時鐘（AHB / 8）
    1 = 使用處理器時鐘（AHB）

  bit 1：TICKINT（中斷使能）
    0 = 計數到 0 不觸發中斷（只設 COUNTFLAG）
    1 = 計數到 0 觸發 SysTick 中斷

  bit 0：ENABLE（計時器使能）
    0 = SysTick 停止
    1 = SysTick 開始計數
```

---

## 程式碼解析

### `main()` 中的 SysTick 設定

```c
void main(void)
{
    usart_init();
    print_str("Hello world!\n");

    /* ── SysTick 三步設定 ─────────────────────────────── */

    /* Step 1：設定重新載入值（決定中斷頻率） */
    *SYSTICK_LOAD = 7200000;
    //             ↑ 每 7,200,000 個時鐘週期觸發一次中斷
    //               8 MHz 時鐘 → 7,200,000 / 8,000,000 = 0.9 秒

    /* Step 2：清零計數值（從 0 開始，立刻載入 LOAD） */
    *SYSTICK_VAL = 0;

    /* Step 3：設定 CTRL = 0x07 */
    *SYSTICK_CTRL = 0x07;
    //             = 0b 0000 0111
    //                  ↑↑↑
    //                  │││ bit 0 = 1：ENABLE（啟動計數）
    //                  ││  bit 1 = 1：TICKINT（啟用中斷）
    //                  │   bit 2 = 1：CLKSOURCE（使用處理器時鐘）

    while (1); /* 什麼都不做，等待中斷 */
}
```

### 計算中斷頻率

```
時鐘頻率 = 8 MHz = 8,000,000 Hz（每秒 8 百萬個週期）

LOAD = 7,200,000

中斷週期 = LOAD / 時鐘頻率
         = 7,200,000 / 8,000,000
         = 0.9 秒

中斷頻率 = 1 / 0.9 ≈ 1.11 Hz（每秒約 1.1 次中斷）
```

不同 LOAD 值的效果：

| LOAD 值 | 中斷間隔（8MHz 時鐘） | 說明 |
|--------|---------------------|------|
| 8,000 | 1 ms | 標準 OS tick（1 kHz） |
| 80,000 | 10 ms | 100 Hz |
| 800,000 | 100 ms | 10 Hz |
| 7,200,000 | 0.9 s | 本章設定 |
| 8,000,000 | 1 s | 每秒一次 |

### SysTick 中斷處理函式

```c
void __attribute__((interrupt)) systick_handler(void)
{
    print_str("Interrupt from System Timer\n");
}
```

`__attribute__((interrupt))` 告訴 GCC：這是一個中斷處理函式。
GCC 會自動在函式進入/退出時加入正確的 push/pop 指令，並使用 EXC_RETURN 代替普通的 `bx lr` 返回。

---

## 完整的向量表：`startup.c`

本章向量表比前幾章完整得多，包含了 ARM Cortex-M3 的標準異常：

```c
__attribute((section(".isr_vector")))
uint32_t *isr_vectors[] = {
    (uint32_t *) &_estack,            // [0]  初始 SP 值
    (uint32_t *) reset_handler,       // [1]  Reset
    (uint32_t *) nmi_handler,         // [2]  NMI
    (uint32_t *) hardfault_handler,   // [3]  Hard Fault
    (uint32_t *) memmanage_handler,   // [4]  MemManage（MPU 違規）
    (uint32_t *) busfault_handler,    // [5]  BusFault（匯流排錯誤）
    (uint32_t *) usagefault_handler,  // [6]  UsageFault（無效指令等）
    0,                                // [7]  保留
    0,                                // [8]  保留
    0,                                // [9]  保留
    0,                                // [10] 保留
    (uint32_t *) svc_handler,         // [11] SVC ← 第三章以後需要
    0,                                // [12] 保留
    0,                                // [13] 保留
    (uint32_t *) pendsv_handler,      // [14] PendSV（OS 排程常用）
    (uint32_t *) systick_handler,     // [15] SysTick ← 本章重點！
};
```

### 向量表的記憶體佈局

```
FLASH 0x08000000：

偏移  位址          內容             說明
+0   0x08000000   &_estack         初始 MSP（CPU 開機後設定 SP）
+4   0x08000004   reset_handler    CPU 從這裡開始執行
+8   0x08000008   nmi_handler
+12  0x0800000C   hardfault_handler
+16  0x08000010   memmanage_handler
+20  0x08000014   busfault_handler
+24  0x08000018   usagefault_handler
+28  0x0800001C   0（保留）
+32  0x08000020   0
+36  0x08000024   0
+40  0x08000028   0
+44  0x0800002C   svc_handler      SVC 異常 → 這裡
+48  0x08000030   0
+52  0x08000034   0
+56  0x08000038   pendsv_handler   PendSV 異常 → 這裡
+60  0x0800003C   systick_handler  SysTick 計時到 → 這裡！
```

---

## 弱符號與預設處理函式：`__attribute__((weak, alias))`

```c
void default_handler(void)
{
    while (1);  // 預設行為：無窮迴圈（方便除錯時發現問題）
}

/* 使用 weak alias 讓所有未實作的 handler 都指向 default_handler */
void nmi_handler(void)        __attribute((weak, alias("default_handler")));
void hardfault_handler(void)  __attribute((weak, alias("default_handler")));
void memmanage_handler(void)  __attribute((weak, alias("default_handler")));
void busfault_handler(void)   __attribute((weak, alias("default_handler")));
void usagefault_handler(void) __attribute((weak, alias("default_handler")));
void svc_handler(void)        __attribute((weak, alias("default_handler")));
void pendsv_handler(void)     __attribute((weak, alias("default_handler")));
void systick_handler(void)    __attribute((weak, alias("default_handler")));
```

這個模式有兩個屬性：

**`weak`（弱符號）**：
- 如果其他地方定義了同名的強符號，以強符號優先
- 如果沒有其他定義，使用這個弱符號

**`alias("default_handler")`**：
- 讓這個函式名稱成為 `default_handler` 的別名
- 不佔用額外的程式碼空間

```
效果示意：

  startup.c 定義的弱符號：
    systick_handler → weak alias → default_handler

  hello.c 定義的強符號：
    systick_handler（有實作）← 連結器選這個！

  最終結果：
    向量表 [15] → hello.c 的 systick_handler
                  （覆蓋了 startup.c 的 weak alias）
```

這樣的設計讓開發者可以：
- 只實作需要的 handler（其他自動使用 default_handler）
- 不用修改 startup.c 就能加入新的中斷處理

---

## 完整執行流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    SysTick 執行流程                              │
│                                                                 │
│  開機 → reset_handler → main()                                  │
│                                                                 │
│  1. SYSTICK_LOAD = 7,200,000                                    │
│  2. SYSTICK_VAL  = 0（清零）                                    │
│  3. SYSTICK_CTRL = 0x07（啟動計數 + 啟用中斷 + 處理器時鐘）     │
│  4. while(1)  ← CPU 在這裡空轉                                  │
│                                                                 │
│  ── 0.9 秒後 ──────────────────────────────────────────────    │
│                                                                 │
│  SysTick 計數到 0！硬體動作：                                   │
│  5. VAL 重置為 LOAD（繼續倒數）                                 │
│  6. 觸發 SysTick Exception（優先級 -1，很高）                   │
│  7. 硬體自動把 {r0,r1,r2,r3,r12,lr,pc,xpsr} 壓入 MSP          │
│     （pc = while(1) 中斷的位址）                                │
│  8. 從向量表 [15] 讀取 systick_handler 位址                    │
│  9. 跳到 systick_handler()                                      │
│                                                                 │
│  systick_handler():                                             │
│  10. print_str("Interrupt from System Timer\n")                 │
│  11. 返回（EXC_RETURN）                                         │
│                                                                 │
│  12. 硬體從 MSP 恢復 {r0,r1,r2,r3,r12,lr,pc,xpsr}             │
│      pc = while(1) 中斷的位址 ← 繼續執行                       │
│                                                                 │
│  ── 再過 0.9 秒 ────────────────────────────────────────────   │
│  重複步驟 5-12...                                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 中斷優先級

ARM Cortex-M3 的 SysTick 是「系統異常」，優先級可以設定，預設較高：

```
系統異常優先級（數字越小越高）：
┌─────────────────────────────────────────────────────┐
│  -3：Reset（最高，不可設定）                         │
│  -2：NMI（不可設定）                                │
│  -1：HardFault（不可設定）                          │
│   0：MemManage / BusFault / UsageFault（可設定）     │
│   ...                                               │
│   n：SysTick（可設定，本章使用預設值）               │
│   ...                                               │
│  最低：普通中斷（NVIC 管理）                         │
└─────────────────────────────────────────────────────┘
```

中斷巢狀（Nested Interrupt）：高優先級中斷可以打斷低優先級的中斷處理函式。
本章不涉及這個複雜情境，SysTick 是唯一的中斷來源。

---

## 與第六章的關係

本章是「練習彈鋼琴前先認識琴鍵」：

```
第五章（本章）：         第六章（Preemptive）：
SysTick 觸發            SysTick 觸發
    │                       │
    ▼                       ▼
systick_handler         systick_handler
    │                       │
    │ 只是印字串             │ 觸發 OS 排程器！
    │                       │ 保存當前任務狀態
    │                       │ 選擇下一個任務
    │                       │ 恢復下一個任務
    ▼                       ▼
繼續原本的程式           下一個任務繼續執行
```

---

## `__attribute__((interrupt))` 說明

```c
void __attribute__((interrupt)) systick_handler(void)
{
    print_str("Interrupt from System Timer\n");
}
```

加上這個屬性後，GCC 生成的組語（大致）：

```asm
systick_handler:
    push {r4, r5, r6, r7, r8, r9, r10, r11}   ← 保存 callee-saved 暫存器
    ; ... 執行 print_str ...
    pop  {r4, r5, r6, r7, r8, r9, r10, r11}   ← 恢復暫存器
    bx   lr                                    ← lr = EXC_RETURN，觸發異常返回
```

不加這個屬性而用 `bx lr` 返回的話，`lr` 會是一般的函式呼叫返回位址，不是 EXC_RETURN，行為不正確。

> 在後續章節中，手工撰寫的 svc_handler 等都是在組語層面直接處理 EXC_RETURN，不依賴這個屬性。

---

## 本章新增的暫存器定義

`reg.h` 新增了 SysTick 的定義：

```c
#define SYSTICK         ((__REG_TYPE) 0xE000E010)
#define SYSTICK_CTRL    ((__REG) (SYSTICK + 0x00))
#define SYSTICK_LOAD    ((__REG) (SYSTICK + 0x04))
#define SYSTICK_VAL     ((__REG) (SYSTICK + 0x08))
#define SYSTICK_CALIB   ((__REG) (SYSTICK + 0x0C))
```

注意 SysTick 的位址 `0xE000E010` 在**系統控制空間（System Control Space）**，這是 ARM Cortex-M 的標準位址，不是 STM32 特有的週邊。這也是為什麼每一顆 ARM Cortex-M 晶片的 SysTick 操作方式都相同。

---

## 本章重點整理

| 概念 | 說明 |
|------|------|
| SysTick | ARM Cortex-M 內建的 24 位元倒數計時器，專為 OS 設計 |
| SYSTICK_LOAD | 設定倒數起始值，決定中斷頻率 |
| SYSTICK_CTRL | 控制使能、中斷使能、時鐘來源 |
| 中斷向量表 | 完整的 16 個系統異常，向量表是 OS 與硬體的溝通介面 |
| `weak` 屬性 | 可被覆蓋的符號，讓 startup.c 提供預設 handler |
| `alias` 屬性 | 讓多個名稱指向同一個函式，節省程式碼空間 |
| `__attribute__((interrupt))` | 告訴 GCC 生成中斷函式的正確進入/退出程式碼 |
| 中斷頻率計算 | 中斷週期 = LOAD / 時鐘頻率 |
| Exception Frame | 中斷時硬體自動保存的 8 個暫存器，確保主程式不受影響 |
