# 第二章：Context Switch — 從核心模式切換到使用者模式

> **使用文件**
> - `RM0008`（STM32F10xxx 參考手冊）— 記憶體映射、開機模式
> - `PM0056`（STM32F10xxx Cortex-M3 程式設計手冊）— CONTROL 暫存器、PSP/MSP、執行模式
>   （RM0008 第 1 頁「Related documents」提及，可在 st.com 取得）
>
> **本章的關鍵知識主要來自 PM0056，不在 RM0008 中**。
> RM0008 描述 STM32 的週邊，PM0056 描述 ARM Cortex-M3 核心本身的機制。

---

## 本章目標

第一章讓程式能夠正確啟動、輸出 Hello World。本章開始建構 OS 的核心能力：
讓 CPU 從「OS 核心」切換到「使用者任務」執行，這是多工作業系統的基礎。

---

## 如何讀文件找到這些資訊

### RM0008 vs PM0056 的分工

```
RM0008（STM32F10xxx 參考手冊）          PM0056（Cortex-M3 程式設計手冊）
────────────────────────────────        ────────────────────────────────
描述 STM32 的「週邊」                    描述 ARM「核心」本身
  ・RCC（時鐘）                            ・CPU 暫存器（r0~r15）
  ・GPIO                                   ・PSR（狀態暫存器）
  ・USART                                  ・CONTROL 暫存器
  ・Timer / SPI / I2C ...                  ・MSP / PSP（堆疊指標）
                                           ・Thread Mode / Handler Mode
                                           ・SVC 異常
                                           ・EXC_RETURN 機制
```

> **查找規則**：如果問題是「這個週邊怎麼設定」→ 查 RM0008；
> 如果問題是「CPU 本身的行為」→ 查 PM0056。

### 如何在 RM0008 裡找到 PM0056 的線索

> **RM0008，第 1 頁（Introduction）**
> ```
> Related documents：
>   STM32F10xxx Cortex-M3 programming manual (PM0056)
> ```

PM0056 描述所有 ARM Cortex-M3 晶片共用的核心行為，不只是 STM32。

---

## 背景知識：CPU 開機與記憶體映射

### 開機模式（Boot Mode）

> **RM0008，第 60~62 頁，Section 3.4 "Boot configuration"**

STM32 有三種開機模式，透過 BOOT0/BOOT1 腳位選擇（STM32-P103.pdf 第 7 頁說明了 B0_H/B0_L 跳線）：

```
BOOT1  BOOT0  開機位置
 x      0     從 Flash（0x08000000）開機  ← 正常情況
 0      1     從系統 ROM（燒錄器）開機
 1      1     從 SRAM（0x20000000）開機
```

關鍵段落（RM0008 第 61 頁）：
> *"The CPU fetches the top-of-stack value from address 0x0000 0000, then starts code execution from the boot memory starting from 0x0000 0004."*

**CPU 永遠從 0x00000000 讀取**，但 STM32 有一個「位址別名（Aliasing）」機制：
- 當 BOOT0=0 時，0x00000000 被映射到 Flash 的 0x08000000
- 因此我們的向量表放在 0x08000000，CPU 從 0x00000000 讀到的其實是 Flash 的內容

```
CPU 看到的                  實際對應
0x00000000 ───────────────→ 0x08000000（Flash 起始，向量表）
0x00000004 ───────────────→ 0x08000004（reset_handler 位址）
...
0x20000000 ───────────────→ 0x20000000（SRAM，無別名）
```

這也解釋了為什麼 01-HelloWorld 把 FLASH 改成 `ORIGIN = 0x08000000`：連結腳本要告訴 linker 程式碼真實存放的位置，而 CPU 開機讀取的別名位址是自動處理的。

---

## ARM Cortex-M3 的執行模式（PM0056）

以下概念全部來自 **PM0056**，是理解 context switch 的基礎。

### Thread Mode 和 Handler Mode

```
┌─────────────────────────────────────────────────────────┐
│          ARM Cortex-M3 執行模式（PM0056）                │
│                                                         │
│  Thread Mode（一般執行）          Handler Mode（中斷）   │
│  ─────────────────────           ─────────────────────  │
│  • 正常程式執行時使用             • 任何中斷/異常發生時  │
│  • 可以是 Privileged 或          • 永遠是 Privileged    │
│    Unprivileged                  • 永遠使用 MSP         │
│  • 使用 MSP 或 PSP               • 無法在這裡降低權限   │
└─────────────────────────────────────────────────────────┘
```

| | Thread Mode（特權） | Thread Mode（非特權） | Handler Mode |
|--|--------------------|--------------------|--------------|
| 開機後預設 | ✓ | — | — |
| 進入方式 | 開機預設 / 特殊設定 | 寫 CONTROL 暫存器 | 任何中斷/異常 |
| 可用堆疊 | MSP 或 PSP | 通常 PSP | 只有 MSP |
| 能改 CONTROL | ✓ | ✗（觸發 UsageFault） | ✓ |

### 兩個堆疊指標（PM0056）

ARM Cortex-M3 有**兩個**堆疊指標，這是支援 OS 的硬體機制：

```
MSP（Main Stack Pointer）
─────────────────────────────────────────────
  • OS 核心和中斷處理使用
  • 開機後預設使用
  • 初始值來自向量表 [0]（startup.c 設定的 &_estack）
  • 永遠由 OS 掌控，使用者程式不能干擾

PSP（Process Stack Pointer）
─────────────────────────────────────────────
  • 使用者任務使用
  • 需要 OS 設定後才能用
  • 每個任務可以有自己的 PSP
```

為什麼要兩個堆疊？
- 使用者任務的堆疊溢位（stack overflow）不會覆蓋到 OS 核心堆疊
- OS 可以分配固定大小的使用者堆疊空間，更安全

### CONTROL 暫存器（PM0056）

控制目前執行狀態的 ARM 特殊暫存器（只用低 2 位）：

```
CONTROL 暫存器位元定義（PM0056）：

  bit 1（SPSEL）：堆疊選擇
    0 = Thread Mode 用 MSP（開機預設）
    1 = Thread Mode 用 PSP

  bit 0（nPRIV）：特權等級
    0 = Privileged（特權，開機預設）
    1 = Unprivileged（非特權，使用者模式）
```

本章用 `CONTROL = 3`（二進位 `0b11`）：
- bit1 = 1：切換到 PSP
- bit0 = 1：切換到 Unprivileged

讀寫 CONTROL 的方法：
```asm
mrs r0, control      ; 讀 CONTROL → r0
msr control, r0      ; r0 → CONTROL
```

---

## 程式碼設計：如何初始化使用者任務堆疊

### 目標行為

我們希望 `activate(stack_ptr)` 執行後，CPU 直接跳到 `usertask()` 函式，
且在 `usertask()` 眼中，就像是從 OS 正常「呼叫」的一樣（有正確的堆疊狀態）。

### 使用者堆疊的預設佈局

```c
// os.c 中的設定
unsigned int usertask_stack[256];                          // 1KB 使用者堆疊
unsigned int *usertask_stack_start = usertask_stack + 256 - 16;
//                                              ↑
//                        256 個 uint，從高位址往下 16 個位置開始

usertask_stack_start[8] = (unsigned int) &usertask;
//                  ↑
//     第 9 個位置（index 8）放入 usertask 函式的位址
//     這個位置對應到 context_switch.S 的 pop 順序中的 lr
```

為什麼是 index [8]？來自 `context_switch.S` 的 pop 指令：

```asm
pop {r4, r5, r6, r7, r8, r9, r10, r11, lr}
;    [0]  [1]  [2]  [3]  [4]  [5]  [6]   [7]  [8]
;                                               ↑
;                                      第 9 個（index 8）→ lr
```

記憶體佈局：

```
usertask_stack_start 指向這裡：
  [index 0]  r4 = 0
  [index 1]  r5 = 0
  [index 2]  r6 = 0
  [index 3]  r7 = 0
  [index 4]  r8 = 0
  [index 5]  r9 = 0
  [index 6]  r10 = 0
  [index 7]  r11 = 0
  [index 8]  lr = &usertask  ← 關鍵！pop 後 lr = 函式位址
  [index 9~15]              （未使用，activate 後 PSP 指向 [9]）
```

---

## 核心組語深入解析：`context_switch.S`

### 完整程式碼與逐行說明

```asm
.thumb
.syntax unified

.global activate
activate:                         ; 接受一個參數 r0 = usertask_stack_start

    /* ── 第一部分：保存核心狀態到 MSP ──────────────────── */
    mrs ip, psr                   ; 讀取 PSR（程式狀態暫存器）到 r12(ip)
    ;   ↑ mrs = Move to Register from Special register（PM0056 指令）
    ;   PSR 包含 N/Z/C/V 旗標等執行狀態，切換回來時需要恢復

    push {r4, r5, r6, r7, r8, r9, r10, r11, ip, lr}
    ;    ↑ 把 10 個暫存器 push 到 MSP（此時是核心的堆疊）
    ;      包含 r4-r11（被呼叫者保存的暫存器）
    ;      ip（PSR）和 lr（activate 的返回位址 = main 裡呼叫 activate 後的下一條指令）

    /* ── 第二部分：切換到使用者模式 ──────────────────────── */
    msr psp, r0                   ; 設定 PSP = r0（使用者堆疊起點）
    ;   ↑ msr = Move to Special register from Register（PM0056 指令）

    mov r0, #3                    ; r0 = 0b11
    msr control, r0               ; CONTROL = 3（SPSEL=1, nPRIV=1）
    ;                               → Thread Mode 改用 PSP，且切換到 Unprivileged

    /* ── 第三部分：從使用者堆疊 pop，跳到任務 ─────────────── */
    pop {r4, r5, r6, r7, r8, r9, r10, r11, lr}
    ;   ↑ 此時 CONTROL[SPSEL]=1，pop 使用 PSP（使用者堆疊）！
    ;     讀取 9 個值：r4-r11 = 0, lr = &usertask

    bx lr                         ; 跳到 lr 指向的位址 = usertask()
    ;  ↑ bx = Branch and eXchange（PM0056 指令）
    ;    如果 lr 是普通函式位址 → 直接跳轉
    ;    如果 lr 是 EXC_RETURN 值 → 觸發異常返回（本章不用，第三章才用）
```

### 關鍵轉折點：`msr control, r0` 之後

```
執行 msr control, r0 之前：
  CONTROL = 0（Privileged + MSP）
  SP 指向 MSP（核心堆疊）

執行之後：
  CONTROL = 3（Unprivileged + PSP）
  SP 現在指向 PSP（使用者堆疊）！

因此下一條 pop 指令用的是 PSP
（從 usertask_stack_start 開始讀取暫存器）
```

---

## 完整切換流程

```
┌─────────────────────────────────────────────────────────────────┐
│                     完整執行流程                                  │
│                                                                 │
│  OS: main()                                                     │
│    ① 配置使用者堆疊                                              │
│       usertask_stack[256] 在 main 的 Stack Frame 上             │
│       usertask_stack_start[8] = &usertask                       │
│                                                                 │
│    ② usart_init()                                               │
│       設定 GPIOA, USART2（同第零章的 RCC / GPIO / USART 步驟）  │
│                                                                 │
│    ③ activate(usertask_stack_start) ─────────────────┐         │
│                                                      │         │
│  context_switch.S: activate()                        ▼         │
│    ④ mrs ip, psr                                                │
│       push {r4-r11, ip, lr} → MSP（保存核心狀態）              │
│                                                                 │
│    ⑤ msr psp, r0                                               │
│       （PSP = usertask_stack_start）                            │
│                                                                 │
│    ⑥ mov r0, #3  /  msr control, r0                            │
│       （切換到 Unprivileged + PSP）                             │
│                                                                 │
│    ⑦ pop {r4-r11, lr}  ← 從 PSP（使用者堆疊）pop              │
│       r4-r11 = 0, lr = &usertask                               │
│                                                                 │
│    ⑧ bx lr  →  跳到 usertask()                                 │
│                                                                 │
│  usertask():                   （Unprivileged + PSP）           │
│    ⑨ print_str("User Task #1\n")                               │
│      ↑ 雖然是 Unprivileged，但 USART2 仍可訪問                  │
│        （STM32 預設不啟用 MPU，記憶體保護尚未設定）             │
│                                                                 │
│    ⑩ while(1);  ← 永遠在這裡，回不到 OS                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 堆疊指標的狀態轉換

```
activate() 執行前：

  MSP（核心堆疊）         PSP（使用者堆疊，尚未設定）
  ┌──────────────┐         ─ 未使用 ─
  │ main() frame │
  │  local vars  │ ← MSP（目前 SP）
  └──────────────┘

activate() push 後（步驟④）：

  MSP（核心堆疊）         PSP（尚未設定）
  ┌──────────────┐
  │ main() frame │
  │  local vars  │
  ├──────────────┤
  │  r11         │
  │  r10         │
  │  ...         │
  │  r4          │
  │  ip（PSR）   │
  │  lr（返回址）│ ← MSP
  └──────────────┘

msr psp + msr control 後（步驟⑤⑥）：

  MSP（核心，不動）        PSP（使用者）
  ┌──────────────┐          ┌──────────────┐
  │  [已保存的   │          │  r4 = 0      │
  │   核心狀態]  │          │  r5 = 0      │
  │              │ ← MSP   │  ...         │
  └──────────────┘          │  r11 = 0    │
                             │  lr=&task   │ ← PSP（目前 SP）
                             └──────────────┘

pop + bx lr 後（步驟⑦⑧）：

  MSP（核心，不動）        PSP（pop 9 個後往高位移動）
  ┌──────────────┐          ┌──────────────┐
  │  [已保存的   │          │  （已 pop）  │
  │   核心狀態]  │ ← MSP   │              │ ← PSP（移動到 [9] 之後）
  └──────────────┘          └──────────────┘

  lr = &usertask → bx lr → 進入 usertask()
```

---

## 從文件到程式碼：重要的位元操作

本章沒有直接操作 STM32 特有的週邊暫存器，關鍵的暫存器都是 ARM 核心的：

| 程式碼 | 對應概念 | 來源文件 | 說明 |
|-------|---------|---------|------|
| `mrs ip, psr` | 讀 PSR | PM0056 | Program Status Register |
| `msr psp, r0` | 設定 PSP | PM0056 | Process Stack Pointer |
| `msr control, r0` | 設定 CONTROL | PM0056 | bit0=nPRIV, bit1=SPSEL |
| `#3`（CONTROL 值） | Unprivileged + PSP | PM0056 | bit1=1: PSP, bit0=1: Unpriv |
| `push/pop` 使用 MSP/PSP | 自動根據 SPSEL | PM0056 | SPSEL 決定 SP 使用哪個 |
| `bx lr` | 跳轉或 EXC_RETURN | PM0056 | 普通位址→跳轉, 特殊值→異常返回 |

---

## 開機時的 0x00000000 vs 0x08000000

> **RM0008，第 60~61 頁，Section 3.4 "Boot configuration"**

這個細節解釋了 00-HelloWorld 和 01-HelloWorld 連結腳本中 FLASH 位址不同的原因：

```
00-HelloWorld：FLASH ORIGIN = 0x00000000
─────────────────────────────────────────
  利用了 QEMU 的簡化模擬（QEMU 把 0x00000000 = Flash 起始）
  在真實 STM32 上，正確的 Flash 位址應是 0x08000000

01-HelloWorld 之後：FLASH ORIGIN = 0x08000000
──────────────────────────────────────────────
  STM32 的 Boot Configuration（BOOT0=0）：
  ・CPU 從 0x00000000 讀取，但硬體把這個位址「別名」到 Flash（0x08000000）
  ・程式實際存在 0x08000000，連結腳本告訴 linker 真實位址
  ・CPU 仍從 0x00000000 讀到正確的 reset_handler

  RM0008 第 61 頁說明：
  "Boot from main Flash memory: the main Flash memory is aliased in
   the boot memory space (0x0000 0000), but still accessible from
   its original memory space (0x0800 0000)."
```

---

## Unprivileged 模式的保護範圍

切換到 Unprivileged 後，使用者任務在這個範例中**並未受到完整保護**（因為 STM32 的 MPU 未啟用），但 ARM 架構層面上限制了以下行為：

```
使用者任務（Unprivileged）無法做的事（PM0056 定義）：

❌ 修改 CONTROL 暫存器
     → msr control, r0 會觸發 UsageFault

❌ 修改 PRIMASK、FAULTMASK、BASEPRI
     → 無法關閉中斷

❌ 使用 MSR 寫入系統暫存器

✅ 一般計算、RAM 讀寫
✅ 訪問週邊（如 USART），如果 MPU 允許
✅ 呼叫 SVC（下一章的重點）
```

---

## 本章重點整理

| 概念 | 說明 | 參考文件 |
|------|------|---------|
| Thread Mode | ARM 一般執行模式，可以是特權或非特權 | PM0056 |
| Handler Mode | 中斷/異常執行模式，永遠是特權 | PM0056 |
| MSP | 核心和中斷使用的堆疊指標，初始值來自向量表 [0] | PM0056 |
| PSP | 使用者任務使用的堆疊指標，由 OS 設定和管理 | PM0056 |
| CONTROL 暫存器 | bit0=nPRIV, bit1=SPSEL，控制特權和堆疊選擇 | PM0056 |
| 開機別名 | STM32 BOOT0=0 時，0x00000000 自動映射到 Flash | RM0008 第 61 頁 |
| 堆疊預設佈局 | 在使用者堆疊特定位置放 lr = &usertask，pop 後 bx lr 跳轉 | 本章設計 |
