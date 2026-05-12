# 第三章：Context Switch 2 — System Call 讓任務回到核心

## 本章目標

第二章留下了一個重大問題：使用者任務切換過去之後，**永遠回不來**。
本章引入 **System Call（系統呼叫）** 機制，讓使用者任務可以主動「讓出控制權」，把執行權歸還給 OS 核心，並且能夠多次來回切換。

執行結果示意：
```
OS: Starting...
OS: Calling the usertask (1st time)
usertask: 1st call of usertask!
usertask: Now, return to kernel mode
OS: Return to the OS mode!
OS: Calling the usertask (2nd time)
usertask: 2nd call of usertask!
usertask: Now, return to kernel mode
OS: Return to the OS mode!
OS: Going to infinite loop...
```

---

## 背景知識：ARM 異常（Exception）機制

### 什麼是異常（Exception）？

ARM Cortex-M 把「需要打斷正常執行流程的事件」統稱為**異常（Exception）**，包括：

```
異常類型
├── 同步異常（程式主動觸發）
│   ├── SVC（SuperVisor Call）← 本章重點
│   ├── HardFault
│   └── UsageFault / BusFault 等
│
└── 非同步異常（硬體觸發）
    ├── SysTick（定時器）← 第五章
    ├── 外部中斷（按鈕、UART 等）
    └── NMI（不可遮蔽中斷）
```

### 異常進入時的硬體自動行為

這是 ARM 最重要的機制之一。當任何異常發生時，**硬體自動**把以下暫存器推入當前使用的堆疊（進入前是 PSP，就推到 PSP；進入前是 MSP，就推到 MSP）：

```
異常進入前（使用者任務執行中，PSP 堆疊）：

    PSP 高位址
    ┌───────────┐
    │  ... 其他 │
    └───────────┘  ← PSP（異常前）

異常進入後（硬體自動 push 8 個暫存器）：

    PSP 高位址
    ┌───────────┐
    │   xPSR    │  ← 程式狀態暫存器
    │   PC      │  ← 觸發異常時的返回位址（SVC 後的下一條指令）
    │   LR      │  ← 原來的 LR
    │   r12     │
    │   r3      │
    │   r2      │
    │   r1      │
    │   r0      │
    └───────────┘  ← PSP（異常後，往下移了 8*4 = 32 bytes）
```

這個動作叫做「**Exception Frame（異常框架）**」。硬體幫我們保存 r0-r3, r12, lr, pc, xpsr。

### EXC_RETURN 魔法值

當 ARM 在 Handler Mode 中執行 `bx lr`，如果 LR 的值是以下特殊值，觸發**異常返回**：

| 值 | 名稱 | 返回到 | 使用的堆疊 |
|----|------|-------|---------|
| `0xFFFFFFF1` | HANDLER_MSP | Handler Mode | MSP |
| `0xFFFFFFF9` | THREAD_MSP | Thread Mode | MSP |
| `0xFFFFFFFD` | THREAD_PSP | Thread Mode | PSP |

異常返回時，CPU 自動從堆疊中 pop 出 exception frame（r0-r3, r12, lr, pc, xpsr），恢復使用者任務執行。

---

## SVC（SuperVisor Call）指令

```asm
svc 0    ; 觸發 SVC 異常，立刻進入 svc_handler
```

`syscall.S` 只有三行：

```asm
.global syscall
syscall:
    svc 0    ; 觸發 SVC 異常 → 跳到 svc_handler
    nop      ; SVC 返回後執行這裡（然後 bx lr 返回呼叫者）
    bx lr
```

使用者任務呼叫 `syscall()` → 執行 `svc 0` → 觸發異常 → CPU 跳到 `svc_handler`（從向量表找到位址）。

---

## 核心組語深入解析：`context_switch.S`

本章的 context_switch.S 包含兩個函式，都需要仔細理解。

### `activate`：核心進入使用者任務

```asm
.global activate
activate:
    /* ─── 1. 保存核心狀態到 MSP ───────────────────────────── */
    mrs ip, psr
    push {r4, r5, r6, r7, r8, r9, r10, r11, ip, lr}
    ;     ↑ 這個 push 使用 MSP（我們在核心 Thread Mode）

    /* ─── 2. 從使用者堆疊載入 r4-r11 和 lr ─────────────────── */
    ldmia r0!, {r4, r5, r6, r7, r8, r9, r10, r11, lr}
    ;          ↑ 從 r0 指向的位址依序載入 9 個值，r0 自動往後移
    ;          ldmia = Load Multiple Increment After

    msr psp, r0
    ;         ↑ 把更新後的 r0（已跳過 9 個值）設為 PSP
    ;           這樣 PSP 指向 exception frame 的位置（如果第二次呼叫）

    /* ─── 3. 判斷是否需要切換到使用者模式 ─────────────────── */
    mov r0, #0xfffffff0
    cmp lr, r0            ; 比較 lr 與 0xFFFFFFF0
    ;
    ; 第一次呼叫：lr = &usertask（很小的位址，< 0xFFFFFFF0）
    ;   → "ls"（lower or same）條件成立，執行下面三條
    ;
    ; 第二次呼叫：lr = 0xFFFFFFFD（EXC_RETURN，> 0xFFFFFFF0）
    ;   → "ls" 條件不成立，跳過下面三條

    ittt ls               ; if-then-then-then (條件執行下 3 條)
    movls r0, #3
    msrls control, r0     ; CONTROL = 3（Unpriv + PSP）← 只在第一次執行
    isbls                 ; ISB：指令同步屏障，確保 CONTROL 更新生效

    /* ─── 4. 跳到使用者任務 ───────────────────────────────── */
    bx lr
    ; 第一次：lr = &usertask，直接跳到函式
    ; 第二次：lr = 0xFFFFFFFD，觸發 EXC_RETURN → CPU 從 exception frame 恢復
```

### `svc_handler`：使用者任務返回核心

```asm
.type svc_handler, %function
.global svc_handler
svc_handler:
    /* ─── 1. 保存使用者狀態到 PSP ─────────────────────────── */
    mrs r0, psp
    ;        ↑ 讀取 PSP（此時 PSP 已被硬體往下移，存著 exception frame）

    stmdb r0!, {r4, r5, r6, r7, r8, r9, r10, r11, lr}
    ;          ↑ 在 exception frame 之前，再額外保存 r4-r11 和 lr
    ;          stmdb = Store Multiple Decrement Before（往低位址存）
    ;          r0 更新為新的堆疊頂端

    ; 現在 r0 = 使用者堆疊的新頂端（包含了完整的使用者狀態）

    /* ─── 2. 從 MSP 恢復核心狀態 ──────────────────────────── */
    pop {r4, r5, r6, r7, r8, r9, r10, r11, ip, lr}
    ;    ↑ 使用 MSP（Handler Mode 永遠用 MSP）
    msr psr, ip           ; 恢復 PSR

    /* ─── 3. 返回核心 ─────────────────────────────────────── */
    bx lr
    ; 此時 lr = activate 呼叫者（main）的返回位址
    ; r0 = 使用者堆疊的新頂端 ← 這就是 activate() 的返回值！
```

> **關鍵設計**：`svc_handler` 返回時 r0 仍指向使用者堆疊頂端。
> 依照 ARM 呼叫規約，r0 就是函式的返回值，因此 `activate()` 回傳了更新後的 PSP 指標。

---

## `ldmia` 和 `stmdb` 指令

這兩個是 ARM 的「批次記憶體搬移」指令：

```
ldmia r0!, {r4, r5, ..., lr}
  Load Multiple, Increment After
  從 r0 開始，依序載入多個暫存器
  每次載入後 r0 += 4
  感嘆號 (!) = 更新 r0 的值

  r0 = 0x20001000
  執行後：r4 = mem[0x20001000]
          r5 = mem[0x20001004]
          ...
          lr = mem[0x20001020]
          r0 = 0x20001024

stmdb r0!, {r4, r5, ..., lr}
  Store Multiple, Decrement Before
  先 r0 -= 4，再存值（向低位址生長，像 push）
  感嘆號 (!) = 更新 r0 的值

  r0 = 0x20001024
  執行後：r0 = 0x20001000
          mem[0x20001000] = r4
          mem[0x20001004] = r5
          ...
          mem[0x20001020] = lr
```

---

## 完整執行流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                    完整雙向切換流程（第一次）                         │
│                                                                     │
│  OS main()                                                          │
│    usertask_stack_start[8] = &usertask    ← 預設 lr                 │
│    activate(usertask_stack_start) ────────────────────────┐         │
│                                                           │         │
│                                                           ▼         │
│  activate():                                                        │
│    push {r4-r11, ip, lr} → MSP ← 保存核心狀態                       │
│    ldmia r0!, {r4-r11, lr}   ← 從使用者堆疊載入，lr = &usertask     │
│    msr psp, r0               ← 設定 PSP                             │
│    lr < 0xFFFFFFF0 → CONTROL = 3（切換到 Unpriv+PSP）               │
│    bx lr                     ← 直接跳到 usertask()                  │
│                                                                     │
│  usertask():                    （Unprivileged + PSP）               │
│    print_str("usertask: 1st call...")                               │
│    syscall() → svc 0 ─────────────────────────────────────┐        │
│                                                            │        │
│  SVC 異常進入：                                             │        │
│    硬體自動把 r0-r3, r12, lr, pc, xpsr 壓入 PSP 堆疊        │        │
│    跳到 svc_handler ←─────────────────────────────────────          │
│                                                                     │
│  svc_handler():                 （Handler Mode，使用 MSP）           │
│    mrs r0, psp                 ← r0 = 使用者 PSP                    │
│    stmdb r0!, {r4-r11, lr}    ← 保存 r4-r11+lr 到使用者堆疊         │
│    pop {r4-r11, ip, lr} ← MSP ← 恢復核心狀態                        │
│    bx lr                       ← 返回核心（activate 的呼叫者）       │
│                                  r0 = 使用者堆疊新頂端               │
│                                                                     │
│  OS main():  ← 核心恢復！                                            │
│    usertask_stack_start = (return value = 使用者堆疊頂端)            │
│    print_str("OS: Return to the OS mode!")                          │
│    usertask_stack_start = activate(usertask_stack_start) ─────┐    │
│                                                               │    │
│  activate()（第二次）：                                        │    │
│    push {r4-r11, ip, lr} → MSP                                │    │
│    ldmia r0!, {r4-r11, lr} ← 從使用者堆疊載入                  │    │
│      r4-r11 = 使用者之前的 r4-r11                              │    │
│      lr = 0xFFFFFFFD（EXC_RETURN，stmdb 存進去的）              │    │
│    msr psp, r0 ← PSP 指向 exception frame                     │    │
│    lr = 0xFFFFFFFD > 0xFFFFFFF0 → 不切換 CONTROL               │    │
│    bx lr（0xFFFFFFFD）→ 觸發 EXC_RETURN ──────────────────────     │
│                                                                     │
│  CPU 異常返回：                                                      │
│    從 PSP 的 exception frame pop r0-r3, r12, lr, pc, xpsr          │
│    PC = syscall() 的 nop 之後（svc 0 的下一條指令）                  │
│    ← 使用者任務從 syscall() 返回繼續執行                             │
│                                                                     │
│  usertask():（繼續執行）                                            │
│    print_str("usertask: 2nd call...")                               │
│    ...                                                              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 使用者堆疊記憶體佈局的演變

### 初始狀態（`activate` 第一次前）

```
usertask_stack_start（= usertask_stack + 240）：

位址    值
[+0]   r4 = 0
[+4]   r5 = 0
[+8]   r6 = 0
[+12]  r7 = 0
[+16]  r8 = 0
[+20]  r9 = 0
[+24]  r10 = 0
[+28]  r11 = 0
[+32]  lr = &usertask  ← usertask_stack_start[8]
               ↑
               PSP 進入 activate 的 ldmia 後，這 9 個會被 pop，
               PSP 指向 [+36] 處
```

### `svc_handler` 執行後（第一次 syscall 後）

SVC 異常進入時，硬體先把 exception frame 壓入 PSP：

```
使用者堆疊（由低往高）：

  ← PSP（svc_handler stmdb 後的新頂端）
[+0 ]  r4 （使用者在 usertask 中的 r4）
[+4 ]  r5
[+8 ]  r6
[+12]  r7
[+16]  r8
[+20]  r9
[+24]  r10
[+28]  r11
[+32]  lr = 0xFFFFFFFD  ← EXC_RETURN（Handler Mode 進入時 LR 被設為此值）
  ← （原來 PSP 指向這裡，exception frame 往下）
[+36]  r0（使用者 r0，syscall 呼叫時的值）
[+40]  r1
[+44]  r2
[+48]  r3
[+52]  r12
[+56]  lr（使用者在 syscall 的返回位址）
[+60]  pc  = syscall 的 nop 之後（svc 後下一條）
[+64]  xpsr
高位址
```

第二次 `activate(usertask_stack_start)` 時：
- `ldmia` 載入 [+0] 到 [+32]：r4-r11, lr = 0xFFFFFFFD
- PSP 設為 [+36]（exception frame 起點）
- `bx 0xFFFFFFFD` → EXC_RETURN → CPU 從 [+36] 開始 pop，恢復 pc，繼續執行

---

## ittt 條件執行指令

ARM Thumb-2 的 IT（If-Then）指令塊，允許條件執行：

```asm
ittt ls      ; If-Then-Then-Then，條件為 ls（lower or same，無號小於等於）
movls r0, #3     ; 第 1 個 then：ls 成立才執行
msrls control, r0; 第 2 個 then：ls 成立才執行
isbls            ; 第 3 個 then：ls 成立才執行
```

`cmp lr, r0`（r0 = 0xFFFFFFF0）之後：
- 若 lr = `&usertask`（小值）→ lr < 0xFFFFFFF0 → ls 成立 → 執行 CONTROL 切換
- 若 lr = `0xFFFFFFFD`（EXC_RETURN）→ lr > 0xFFFFFFF0 → ls 不成立 → 跳過

---

## 本章 vs 上章的核心差異

| 比較項目 | 02-ContextSwitch-1 | 03-ContextSwitch-2 |
|---------|------------------|------------------|
| 使用者返回核心 | 不可能 | 透過 SVC 返回 |
| `activate` 返回值 | void | `unsigned int *`（更新的 PSP） |
| 核心-使用者切換次數 | 一次（單向） | 多次（雙向） |
| 使用者堆疊儲存內容 | 只有 r4-r11 + lr | r4-r11 + lr + exception frame |
| 觸發機制 | 直接 `bx lr` | 第二次用 EXC_RETURN |
| `svc_handler` | 不存在 | 負責保存使用者狀態、恢復核心 |

---

## 本章重點整理

| 概念 | 說明 |
|------|------|
| SVC 指令 | 軟體觸發的異常，讓使用者任務主動交出控制權 |
| Exception Frame | SVC 觸發時硬體自動保存的 8 個暫存器（r0-r3, r12, lr, pc, xpsr） |
| EXC_RETURN | Handler Mode 中 `bx lr` 的特殊值，觸發異常返回並恢復 exception frame |
| `ldmia` | 批次從記憶體載入暫存器，地址自動遞增 |
| `stmdb` | 批次存暫存器到記憶體，先遞減地址（像 push） |
| `activate` 返回值 | r0 = svc_handler 保存使用者狀態後的 PSP 頂端位址 |
| 兩段式切換 | 第一次：直接 `bx lr` 跳函式；第二次：EXC_RETURN 恢復 exception frame |
