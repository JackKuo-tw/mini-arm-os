# 第四章：Multitasking — 兩個任務輪流執行

## 本章目標

第三章實現了核心和一個使用者任務之間的雙向切換。本章把這個能力**擴展到多個任務**，讓兩個使用者任務能夠輪流執行。

這就是**協作式多工（Cooperative Multitasking）**的雛形：任務主動呼叫 `syscall()` 讓出 CPU，OS 決定下一個要執行哪個任務。

執行結果示意：
```
OS: Starting...
OS: First create task 1
task1: Created!
task1: Now, return to kernel mode
OS: Back to OS, create task 2
task2: Created!
task2: Now, return to kernel mode

OS: Start multitasking, back to OS till task yield!
OS: Activate next task
task1: Executed!
task1: Now, return to kernel mode
OS: Back to OS
OS: Activate next task
task2: Executed!
task2: Now, return to kernel mode
OS: Back to OS
OS: Activate next task
task1: Executed!
...（無限交替）
```

---

## 背景知識：Exception Frame 全貌

### 完整的任務堆疊快照

執行 SVC 並保存完整狀態後，使用者任務的堆疊包含兩層：

```
使用者任務的堆疊（低位址在上）：

  ← PSP 指向這裡（svc_handler 的 stmdb 之後）
┌─────────────────────────────────────────────┐
│  r4   │  軟體保存（svc_handler stmdb）       │
│  r5   │                                     │
│  r6   │                                     │
│  r7   │                                     │
│  r8   │                                     │
│  r9   │                                     │
│  r10  │                                     │
│  r11  │                                     │
│  LR   │  = EXC_RETURN（0xFFFFFFFD）          │
├─────────────────────────────────────────────┤
│  r0   │  硬體自動保存（Exception Frame）     │
│  r1   │                                     │
│  r2   │                                     │
│  r3   │                                     │
│  r12  │                                     │
│  LR   │  （原使用者 LR，syscall 的返回位址） │
│  PC   │  = syscall() 中 svc 後的 nop 位址   │
│  xPSR │                                     │
└─────────────────────────────────────────────┘  ← 高位址
```

當 OS 要恢復任務時，`activate` 只需做：
1. `ldmia r0!, {r4-r11, lr}` — 恢復軟體保存的部分，lr = 0xFFFFFFFD
2. `msr psp, r0` — PSP 指向 exception frame
3. `bx lr`（0xFFFFFFFD）— 硬體自動從 exception frame 恢復 r0-r3, r12, lr, pc, xpsr

---

## 本章新增：`create_task` 函式

本章最關鍵的新增是 `create_task`，它讓 OS 能夠**從零開始建立一個任務**，而不需要任務自己先跑一次才能被管理。

### 設計思路

我們希望讓第一次呼叫 `activate(task_stack)` 的行為，跟之後每次 `activate` 的行為**完全一致**，都透過 EXC_RETURN 機制進入。

這樣 `activate` 本身就不需要第三章那個複雜的 `ittt ls` 判斷了！

### `create_task` 程式碼解析

```c
/* EXC_RETURN 魔法值 */
#define HANDLER_MSP  0xFFFFFFF1   // 返回 Handler Mode，使用 MSP
#define THREAD_MSP   0xFFFFFFF9   // 返回 Thread Mode，使用 MSP
#define THREAD_PSP   0xFFFFFFFD   // 返回 Thread Mode，使用 PSP  ← 我們需要這個

unsigned int *create_task(unsigned int *stack, void (*start)(void))
{
    // stack 目前指向陣列起始，往後跳到頂端
    // 扣掉 17 個槽位：9（軟體保存）+ 8（exception frame）
    stack += STACK_SIZE - 17;   // stack 指向預設好的「假堆疊」頂端

    // ── 軟體保存區（index 0-8）──────────────────────────────
    // index 0-7：r4-r11 = 0（不重要，任務開始時會設定自己的暫存器）

    // index 8：lr = THREAD_PSP（告訴 CPU：異常返回時去 Thread+PSP）
    stack[8] = (unsigned int) THREAD_PSP;     // = 0xFFFFFFFD

    // ── Exception Frame（index 9-16）───────────────────────
    // index 9-13：r0-r3, r12 = 0（任務開始時沒有參數）

    // index 14：lr = 0（使用者 lr，不重要）

    // index 15：PC = 任務函式位址（任務從這裡開始執行！）
    stack[15] = (unsigned int) start;

    // index 16：xPSR = 0x01000000（必須設定 T bit，表示 Thumb 模式）
    stack[16] = (unsigned int) 0x01000000;

    // 執行一次 activate：
    // - 從假堆疊的軟體保存區載入 r4-r11, lr=THREAD_PSP
    // - bx THREAD_PSP → 異常返回 → 從 exception frame 恢復
    // - PC = start → 任務開始執行！
    // - 任務呼叫 syscall() → 返回核心 → activate 回傳更新的 PSP
    stack = activate(stack);

    return stack;  // 返回任務的「目前 PSP 頂端」，供之後的 activate 使用
}
```

### `create_task` 建立的假堆疊佈局

```
stack（= user_stacks[n] + STACK_SIZE - 17）：

索引  偏移   內容
[0]  +0   = 0 （r4，初始不重要）
[1]  +4   = 0 （r5）
[2]  +8   = 0 （r6）
[3]  +12  = 0 （r7）
[4]  +16  = 0 （r8）
[5]  +20  = 0 （r9）
[6]  +24  = 0 （r10）
[7]  +28  = 0 （r11）
[8]  +32  = 0xFFFFFFFD （lr = THREAD_PSP）← EXC_RETURN
----- ldmia 載入以上 9 個，PSP 移到 [9] -----
[9]  +36  = 0 （r0）
[10] +40  = 0 （r1）
[11] +44  = 0 （r2）
[12] +48  = 0 （r3）
[13] +52  = 0 （r12）
[14] +56  = 0 （lr，使用者 lr）
[15] +60  = &start （PC）← 任務從這裡開始！
[16] +64  = 0x01000000 （xPSR，T bit = 1）
```

---

## `task_init`：為什麼需要它？

### 問題所在

理想的流程是：
```
activate(task_stack) → bx THREAD_PSP → 異常返回 → 任務執行
```

但是，**「異常返回」只能在 Handler Mode 中觸發**。如果在 Thread Mode 中執行 `bx 0xFFFFFFFD`，CPU 行為未定義，通常會觸發 HardFault。

上電後，CPU 預設在 **Thread Mode（Privileged，MSP）**，不是 Handler Mode。

### 解決方案

`task_init` 做了一件事：**讓系統先進入 Handler Mode 一次，建立正確的執行環境**，使得之後的 `activate` 呼叫能正確使用 EXC_RETURN。

```c
void task_init(void)
{
    unsigned int empty[32];
    task_init_env(empty + 32);  // 傳入一個臨時堆疊頂端
}
```

### `task_init_env` 組語解析

```asm
.global task_init_env
task_init_env:
    save_kernel_state         ; push {r4-r11, ip, lr} 到 MSP

    ; 切換到 Unpriv + PSP
    msr psp, r0               ; PSP = empty + 32（臨時堆疊）
    mov r0, #3
    msr control, r0           ; CONTROL = 3
    isb                       ; 指令同步屏障

    ; 呼叫 syscall → svc 0 → 觸發 SVC 異常 → 進入 Handler Mode！
    bl syscall

    bx lr                     ; 通常不會執行到這裡
```

執行過程：
1. `task_init_env` 儲存核心狀態到 MSP
2. 切換到 Unpriv + PSP（CONTROL = 3）
3. `bl syscall` → `svc 0` → SVC 異常觸發
4. 進入 `svc_handler`：保存使用者狀態，恢復核心狀態，`bx lr` 返回核心
5. 核心恢復，現在系統已建立好正確的 Handler-Thread Mode 轉換記錄

```
task_init 執行時序：

  Thread Mode (Priv + MSP)               Handler Mode (MSP)
        │                                      │
        │ task_init_env()                      │
        │                                      │
        │ save_kernel_state                    │
        │ CONTROL = 3 → 切換到 Unpriv+PSP     │
        │ bl syscall → svc 0 ─────────────────→│
        │                                      │ svc_handler
        │                                      │ 保存使用者狀態到PSP
        │                                      │ 恢復核心狀態從MSP
        │ ←──────────────────────────────────── bx lr
        │
  Thread Mode (Unpriv + PSP) ← 仍保持 CONTROL=3
```

---

## 簡化後的 `activate`

本章的 `activate` 比第三章**更簡單**，去掉了 `ittt ls` 判斷：

```asm
.macro save_kernel_state
    mrs ip, psr
    push {r4, r5, r6, r7, r8, r9, r10, r11, ip, lr}
.endm

.global activate
activate:
    save_kernel_state         ; 保存核心狀態到 MSP

    /* 從使用者堆疊載入軟體保存部分 */
    ldmia r0!, {r4, r5, r6, r7, r8, r9, r10, r11, lr}
    msr psp, r0               ; PSP 指向 exception frame

    /* 直接 bx lr，永遠是 EXC_RETURN（因為 create_task 保證 lr = 0xFFFFFFFD） */
    bx lr
```

這之所以能正確運作，是因為：
- `task_init` 確保系統已進入過 Handler Mode
- `create_task` 確保每個任務的初始堆疊中 lr = `THREAD_PSP`（0xFFFFFFFD）
- 每次 `svc_handler` 保存使用者狀態時，也會把 EXC_RETURN 值保存進去

---

## 多任務排程器（Round-Robin）

```c
int main(void)
{
    unsigned int user_stacks[TASK_LIMIT][STACK_SIZE];  // 每個任務獨立的 1KB 堆疊
    unsigned int *usertasks[TASK_LIMIT];               // 記錄每個任務的目前 PSP
    size_t task_count = 0;
    size_t current_task;

    usart_init();
    task_init();  // 必須在 create_task 前呼叫！

    // 建立兩個任務（各自執行一次初始化後返回核心）
    usertasks[0] = create_task(user_stacks[0], &task1_func);
    task_count += 1;
    usertasks[1] = create_task(user_stacks[1], &task2_func);
    task_count += 1;

    // 簡單的輪詢排程器（Round-Robin）
    current_task = 0;
    while (1) {
        // 啟動 current_task，取回更新的 PSP
        usertasks[current_task] = activate(usertasks[current_task]);

        // 換下一個任務（簡單的輪轉）
        current_task = current_task == (task_count - 1) ? 0 : current_task + 1;
    }
}
```

### 排程流程圖

```
OS: task_init()
    │
    ▼
OS: create_task(task1)
    ├── activate(假堆疊) → task1 執行一次 → syscall → 返回
    └── usertasks[0] = 任務 1 的 PSP 快照
    │
    ▼
OS: create_task(task2)
    ├── activate(假堆疊) → task2 執行一次 → syscall → 返回
    └── usertasks[1] = 任務 2 的 PSP 快照
    │
    ▼
OS: while(1) 排程迴圈
    │
    ├── current_task = 0
    │       │
    │       ▼
    │   activate(usertasks[0]) → task1 執行 → syscall → 返回
    │   usertasks[0] = 更新 PSP
    │
    ├── current_task = 1
    │       │
    │       ▼
    │   activate(usertasks[1]) → task2 執行 → syscall → 返回
    │   usertasks[1] = 更新 PSP
    │
    ├── current_task = 0  → ...（無限輪流）
    │
    ...
```

---

## 多任務記憶體佈局

```
RAM
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  OS 核心堆疊（MSP）                                      │
│  ┌─────────────────────────────────┐                    │
│  │ 每次 activate 儲存的核心狀態    │ ← MSP              │
│  └─────────────────────────────────┘                    │
│                                                         │
│  task1 的堆疊（user_stacks[0]，1KB）                    │
│  ┌─────────────────────────────────┐                    │
│  │ ...（未使用）                   │                    │
│  │ [軟體保存：r4-r11, lr]          │                    │
│  │ [Exception Frame: r0-r3, r12,  │                    │
│  │  lr, pc, xpsr]                 │ ← usertasks[0]     │
│  └─────────────────────────────────┘                    │
│                                                         │
│  task2 的堆疊（user_stacks[1]，1KB）                    │
│  ┌─────────────────────────────────┐                    │
│  │ ...（未使用）                   │                    │
│  │ [軟體保存：r4-r11, lr]          │                    │
│  │ [Exception Frame: r0-r3, r12,  │                    │
│  │  lr, pc, xpsr]                 │ ← usertasks[1]     │
│  └─────────────────────────────────┘                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

`usertasks[0]` 和 `usertasks[1]` 是兩個 `unsigned int *` 指標，各自記錄對應任務**目前堆疊頂端的位址**，這就是任務的「控制塊（TCB, Task Control Block）」的最簡形式。

---

## xPSR 的 T bit

```c
stack[16] = (unsigned int) 0x01000000;  // xPSR
```

xPSR（Extended Program Status Register）的 bit 24 是 **T bit（Thumb bit）**：

```
xPSR 位元結構：
  bit 31: N（負旗）
  bit 30: Z（零旗）
  bit 29: C（進位旗）
  bit 28: V（溢位旗）
  ...
  bit 24: T（Thumb 模式，必須為 1！）
  ...

0x01000000 = 0000 0001 0000 ... 0000
                   ↑
                  bit 24 = T = 1（Thumb 模式）
```

如果 T bit 為 0，CPU 嘗試以 ARM 模式執行 Thumb 程式碼，會立即觸發 UsageFault。

---

## 本章 vs 上章的差異

| 比較項目 | 03-ContextSwitch-2 | 04-Multitasking |
|---------|------------------|----------------|
| 任務數量 | 1 個 | 2 個（可擴展到 N 個） |
| `activate` 判斷 | ittt ls 判斷第一次/之後 | 無判斷，永遠 EXC_RETURN |
| 任務建立 | 手動設定堆疊 | `create_task()` 封裝 |
| 系統初始化 | 無 | `task_init()` 建立執行環境 |
| 排程 | 固定 2 次呼叫 | while(1) 輪詢迴圈 |
| 任務控制塊 | 單一 `usertask_stack_start` | `usertasks[]` 陣列 |

---

## 協作式 vs 搶佔式多工

本章實作的是**協作式多工（Cooperative Multitasking）**：

```
協作式（本章）：                    搶佔式（第六章）：

任務 A 執行                         任務 A 執行
    │                                   │
    │（主動呼叫 syscall）                │
    ▼                                   │← SysTick 中斷（強制打斷！）
任務 B 執行                         任務 B 執行
    │                                   │
    │（主動呼叫 syscall）                │← SysTick 中斷
    ▼                                   ▼
任務 A 執行                         任務 A 執行

優點：實作簡單，時序可預測         優點：任務無法獨佔 CPU
缺點：任務若不讓出 CPU，其他       缺點：實作複雜，需處理共享
      任務永遠無法執行              資源的同步問題
```

---

## 本章重點整理

| 概念 | 說明 |
|------|------|
| `create_task` | 建立假 exception frame，讓首次啟動與後續啟動行為一致 |
| `THREAD_PSP (0xFFFFFFFD)` | EXC_RETURN 值，指示「返回 Thread Mode，使用 PSP 恢復狀態」 |
| xPSR T bit | 必須設為 1，否則 CPU 嘗試 ARM 模式，觸發 UsageFault |
| `task_init` | 觸發一次 SVC 進入 Handler Mode，建立 EXC_RETURN 可用的環境 |
| `usertasks[]` | 儲存每個任務目前的 PSP 指標，最簡單的任務控制塊 |
| 協作式多工 | 任務主動呼叫 `syscall()` 讓出 CPU，OS 決定下一個任務 |
| Round-Robin | 最簡單的排程演算法：任務依序輪流執行 |
