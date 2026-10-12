# Capsicain 映射参考文档

> 本文档记录当前 `capsicain.ini` 中每条映射的底层工作原理、涉及的函数和修改的源文件，方便后续调整。

---

## 一、配置文件总览

**源文件**：`src/capsicain.ini`（CMake 构建时自动拷贝到 `bin/capsicain.ini`）  
**运行文件**：`cmake-build-debug-visual-studio/bin/capsicain.ini`

```
GLOBAL capsicainOnOffKey SCRLOCK
GLOBAL startInTraybar
GLOBAL startMinimized
GLOBAL ActiveConfigOnStartup 1

[CONFIG_1]
    REWIRE CAPS LCTRL ESC
    REWIRE LALT MOD9 NOP
    COMBO LSHF [RSHIFT] > key(CAPS)
    COMBO RSHF [LSHIFT] > key(CAPS)
    COMBO TAB  [MOD9] > alttab()
    COMBO H    [MOD9+LSHIFT] > key(LEFT)       ← Alt+Shift
    COMBO H    [MOD9+LCTRL]  > moddedKey(LEFT + LCTRL)  ← Alt+Caps
    COMBO H    [MOD9]        > moddedKey(LEFT)           ← Alt only
    ... (J/K/L/I/A/U/D 同理)
```

---

## 二、修饰符位掩码速查表

`src/modifiers.h` 定义了所有修饰符位掩码，`[.... .... .... ....]` 格式中 16 个字符从左到右对应 bit15→bit0：

```
格式: [1234 5678 9ABC DEFG]

位置 0   → bit15 (未使用)
位置 1   → bit14 = MOD15 (0x4000)
位置 2   → bit13 = MOD14 (0x2000)
位置 3   → bit12 = MOD13 (0x1000)
位置 4   → bit11 = MOD12 (0x0800)
位置 5   → bit10 = MOD11 (0x0400)
位置 6   → bit9  = MOD10 (0x0200)
位置 7   → bit8  = MOD9  (0x0100)  ← LAlt 的重映射目标
位置 8   → bit7  = RALT  (0x0080)
位置 9   → bit6  = RWIN  (0x0040)
位置 10  → bit5  = RCTRL (0x0020)
位置 11  → bit4  = RSHIFT(0x0010)
位置 12  → bit3  = LALT  (0x0008)
位置 13  → bit2  = LWIN  (0x0004)
位置 14  → bit1  = LCTRL (0x0002)  ← CapsLock 的重映射目标
位置 15  → bit0  = LSHIFT(0x0001)
```

**常用写法示例**：

| 含义 | 掩码写法 | 数值 |
|------|----------|------|
| MOD9 only | `[.... ...& .... ....]` | 0x0100 |
| MOD9 + LSHIFT | `[.... ...& .... ...&]` | 0x0101 |
| MOD9 + LCTRL | `[.... ...& .... ..&.]` | 0x0102 |
| RSHIFT only | `[.... .... ...& ....]` | 0x0010 |
| LSHIFT only | `[.... .... .... ...&]` | 0x0001 |

---

## 三、逐条映射解析

### 3.1 CapsLock：单击 = Esc，组合 = LCtrl

```
REWIRE CAPS LCTRL ESC
```

**涉及文件**：无修改（内置功能）

**底层流程**：
```
物理 CapsLock 按下
  → processRewireScancodeToVirtualcode()
    SC_CAPS (0x3A) → VK_LCTRL (0xA2)
  → processModifierState()
    设置 modifierDown |= BITMASK_LCTRL (0x0002)
  → detectTapping()
    如果短按后释放 → 发送 VK_ESC
    如果搭配其他键 → 保持 LCTRL
```

**关键函数**：
- `processRewireScancodeToVirtualcode()` — `capsicain.cpp` — 重映射 scancode → vcode
- `processModifierState()` — `capsicain.cpp` — 更新 modifierDown 位掩码
- `detectTapping()` — `capsicain.cpp` — 区分短按/长按

---

### 3.2 LAlt → 虚拟 MOD9

```
REWIRE LALT MOD9 NOP
```

**涉及文件**：无修改（内置功能）

**底层流程**：
```
物理 LAlt 按下
  → processRewireScancodeToVirtualcode()
    SC_LALT (0x38) → VK_MOD9 (0x109)
  → processModifierState()
    设置 modifierDown |= BITMASK_MOD9 (0x0100)
  → VK_MOD9 > 0xFF → sendCapsicainCodeHandler()
    虚拟键，不发送给 Windows
```

**为什么这样设计**：
- MOD9 是虚拟修饰键，**永远不会到达 Windows**
- 应用程序看不到 Alt 被按下 → Alt+HJKL 不会触发应用快捷键（Alt+H→Help 等）
- COMBO 规则通过检查 `modifierDown & BITMASK_MOD9` 来感知 Alt 状态

---

### 3.3 LShift + RShift → CapsLock 切换

```
COMBO LSHF [.... .... ...& ....] > key(CAPS)
COMBO RSHF [.... .... .... ...&] > key(CAPS)
```

**涉及文件**：无修改（内置功能）

**底层流程**：
```
按下 RShift → modifierDown |= BITMASK_RSHIFT (0x0010)
按下 LShift → vcode = SC_LSHIFT
  → processCombos()
    遍历 allMaps.modCombos
    匹配条件: vcode == SC_LSHIFT && (modifierDown & 0x0010) == 0x0010
    → 命中第一条 COMBO
  → 设置 resultingVKeyEventSequence = [{SC_CAPS, true}, {SC_CAPS, false}]
  → sendResultingKeyOrSequence()
    → playKeyEventSequence()
      → sendVKeyEvent(SC_CAPS, true)  → Windows 收到 CapsLock DOWN
      → sendVKeyEvent(SC_CAPS, false) → Windows 收到 CapsLock UP
```

**关键函数**：
- `processCombos()` — `capsicain.cpp:863` — 遍历 COMBO 规则，匹配修饰符掩码
- `playKeyEventSequence()` — `capsicain.cpp:1930` — 播放按键序列
- `parseKeywordCombo()` — `configUtils.cpp:407` — 解析 .ini 中 `> key(CAPS)` 语法

---

### 3.4 Alt+Tab → 原生窗口切换（重点）

```
COMBO TAB [.... ...& .... ....] > alttab()
```

**涉及文件（代码级别修改）**：
| 文件 | 修改 |
|------|------|
| `src/scancodes.h:187` | 新增 `VK_CPS_ALTTAB = 0x107` |
| `src/capsicain.cpp:118-119` | GlobalState 新增 `altTabPassthrough`, `altTabExitScancode` |
| `src/capsicain.cpp:314-329` | 主循环新增 passthrough 模式检查 |
| `src/capsicain.cpp:1867-1881` | sendCapsicainCodeHandler 新增 VK_CPS_ALTTAB case |
| `src/configUtils.cpp:534-541` | parseKeywordCombo 新增 `alttab` 函数解析 |

**底层流程**：
```
1. 按下 LAlt → REWIRE 为 MOD9 → modifierDown |= BITMASK_MOD9
2. 按下 Tab  → COMBO 匹配 [MOD9] → alttab()
   → parseKeywordCombo() 解析 "alttab"
   → 生成 strokeSeq = [{VK_CPS_ALTTAB, true}]

3. playKeyEventSequence() 发送序列
   → sendVKeyEvent({0x107, true})
   → 0x107 > 0xFF → sendCapsicainCodeHandler()
   → case VK_CPS_ALTTAB:
        interception_send(SC_LALT, DOWN)    ← 发送真实 LAlt 给 Windows
        Sleep(10)
        interception_send(SC_TAB, DOWN)     ← 发送 Tab 给 Windows
        globalState.altTabPassthrough = true  ← 进入透传模式
        globalState.altTabExitScancode = SC_LALT

4. 后续所有按键 → 主循环检测到 altTabPassthrough == true
   → InterceptionSendCurrentKeystroke()    ← 直接透传给 Windows
   → 跳过所有 REWIRE/COMBO 处理

5. 用户释放物理 LAlt
   → scancode == SC_LALT && !isDownstroke
   → InterceptionSendCurrentKeystroke()    ← 透传 LAlt UP → 关闭窗口切换器
   → altTabPassthrough = false
   → reset()                               ← 重置所有状态
```

**为什么需要这个机制**：
- LAlt 被 REWIRE 为虚拟 MOD9，永远不发送给 Windows
- 普通的 `moddedKey(TAB + LALT)` 会瞬间发送 Alt+Tab（down/up 在一帧内完成）
- Windows 窗口切换器**需要 Alt 持续按住**才能循环选择
- Passthrough 模式让所有按键原样透传，用户可长按 Alt 并用 Tab 切换

---

### 3.5 Alt+HJKLUDIA → 光标移动

```
COMBO H [.... ...& .... ....] > moddedKey(LEFT + .... .... .... ....)
COMBO J [.... ...& .... ....] > moddedKey(DOWN + .... .... .... ....)
...
```

**涉及文件**：无修改（内置功能）

**底层流程**：
```
按下 LAlt → MOD9 → modifierDown |= BITMASK_MOD9
按下 H    → COMBO 匹配 [MOD9] → moddedKey(LEFT + .... .... .... ....)

parseFunctionModdedkey() 生成序列:
  1. VK_CPS_TEMPRELEASEKEYS, true   ← 释放所有已按下的修饰键
  2. (无修饰键要按下, + .... 全为 0)
  3. SC_LEFT, true                   ← LEFT DOWN
  4. SC_LEFT, false                  ← LEFT UP
  5. (无修饰键要释放)
  6. VK_CPS_TEMPRESTOREKEYS, false  ← 恢复所有修饰键

playKeyEventSequence():
  Step 1: TEMPRELEASE → keysDownSent 为空(MOD9 虚拟,未跟踪) → 无事发生
  Step 2: 跳过
  Step 3: sendVKeyEvent(SC_LEFT, true) → Sleep(5ms)
  Step 4: sendVKeyEvent(SC_LEFT, false)→ Sleep(5ms)
  Step 6: TEMPRESTORE → 无事发生

Windows 收到: 干净的 LEFT 键（不带任何修饰符）→ 光标左移一格
```

**为什么用 moddedKey 而不是 key()**：
- moddedKey 的 TEMPRELEASE 确保所有修饰符被释放后才发送目标键
- 虽然 MOD9 是虚拟的不会影响 Windows，但 moddedKey 是通用设计，如果是真实修饰键也能正确剥离
- `key()` 不释放修饰符，如果用在真实修饰符场景会有副作用

---

### 3.6 Alt+Shift+HJKLUDIA → 选中文本

```
COMBO H [.... ...& .... ...&] > key(LEFT)
```

**涉及文件**：无修改（纯配置层面）

**底层流程**：
```
按下 LShift → Windows 收到 LSHIFT DOWN（真实键，未被 REWIRE）
              modifierDown |= BITMASK_LSHIFT
按下 LAlt   → MOD9 → modifierDown |= BITMASK_MOD9
按下 H      → COMBO 匹配 [MOD9+LSHIFT] → key(LEFT)

parseKeywordCombo() 解析 "key(LEFT)" → 生成序列:
  1. SC_LEFT, true
  2. SC_LEFT, false

playKeyEventSequence():
  Step 1: sendVKeyEvent(SC_LEFT, true) → Sleep(5ms)
  Step 2: sendVKeyEvent(SC_LEFT, false)→ Sleep(5ms)

Windows 视角: LSHIFT 一直按住 + LEFT 被按下
           → Shift+Left → 选中一个字符

每次 H 自动重复: 再发送一次 LEFT down+up → 继续选中
```

**为什么这里用 key() 而不是 moddedKey()**：
| | moddedKey(LEFT+LSHIFT) | key(LEFT) |
|---|---|---|
| TEMPRELEASE | **释放物理 Shift**（发送 UP） | 无 |
| 每重复的 Sleep | 4 次 × 5ms = **20ms** | 2 次 × 5ms = **10ms** |
| LSHIFT 状态 | 每次被翻转两次 | 始终不变 |
| 体验 | 卡顿、松开后光标还在动 | **流畅** |

**物理 Shift 本身已被 Windows 感知**，所以直接发 LEFT 自然就是 Shift+Left，无需 moddedKey 多此一举地释放/恢复 Shift。

---

### 3.7 Alt+CapsLock+HL → 按单词移动

```
COMBO H [.... ...& .... ..&.] > moddedKey(LEFT + .... .... .... ..&.)
COMBO L [.... ...& .... ..&.] > moddedKey(RIGHT + .... .... .... ..&.)
```

**涉及文件**：无修改（纯配置层面）

**底层流程**：
```
按下 CapsLock → REWIRE 为 LCTRL → modifierDown |= BITMASK_LCTRL (0x0002)
按下 LAlt     → REWIRE 为 MOD9  → modifierDown |= BITMASK_MOD9  (0x0100)
按下 H        → COMBO 匹配 [MOD9+LCTRL] → moddedKey(LEFT + LCTRL)

parseFunctionModdedkey() 生成序列:
  1. VK_CPS_TEMPRELEASEKEYS, true   ← 释放 LCTRL
  2. SC_LCTRL, true                  ← 按下 LCTRL（moddedKey 指定）
  3. SC_LEFT, true                   ← LEFT DOWN
  4. SC_LEFT, false                  ← LEFT UP
  5. SC_LCTRL, false                 ← 释放 LCTRL
  6. VK_CPS_TEMPRESTOREKEYS, false  ← 恢复 LCTRL

Windows 收到: Ctrl+Left → 光标左移一个单词
```

**为什么这里必须用 moddedKey**：
- CapsLock 被 REWIRE 为 LCTRL，物理 CapsLock 键发送 LCTRL 给 Windows
- moddedKey 的 TEMPRELEASE 确保在发送 Ctrl+Left 时，LCTRL 状态是干净的
- 如果用 `key(LEFT)`，Windows 会同时看到真实的 LCTRL（来自 CapsLock REWIRE）和 MOD9（虚拟），可能导致意外行为

---

## 四、COMBO 匹配规则（顺序敏感！）

```
规则: 同一 vkey 的多个 COMBO，先匹配先胜，更具体的修饰符必须排在前面！

例: H 键的三个规则必须按此顺序:
  1. COMBO H [MOD9+LSHIFT] > key(LEFT)          ← 最具体: 2 个修饰符
  2. COMBO H [MOD9+LCTRL]  > moddedKey(LEFT+LCTRL) ← 次具体: 2 个修饰符
  3. COMBO H [MOD9]        > moddedKey(LEFT)        ← 最宽泛: 1 个修饰符

匹配逻辑 (processCombos):
  当 modifierDown = MOD9 | LSHIFT | LCTRL = 0x0103
  规则1: (0x0103 & 0x0101) == 0x0101 → 0x0101 == 0x0101 ✓ 匹配！break
  规则2,3: 不会执行

  当 modifierDown = MOD9 | LCTRL = 0x0102
  规则1: (0x0102 & 0x0101) == 0x0101 → 0x0100 ≠ 0x0101 ✗ 不匹配
  规则2: (0x0102 & 0x0102) == 0x0102 → 0x0102 == 0x0102 ✓ 匹配！break
  规则3: 不会执行

  当 modifierDown = MOD9 = 0x0100
  规则1: (0x0100 & 0x0101) == 0x0101 → 0x0100 ≠ 0x0101 ✗
  规则2: (0x0100 & 0x0102) == 0x0102 → 0x0100 ≠ 0x0102 ✗
  规则3: (0x0100 & 0x0100) == 0x0100 → 0x0100 == 0x0100 ✓ 匹配！
```

---

## 五、涉及的核心函数速查

| 函数 | 文件 | 行号 | 作用 |
|------|------|------|------|
| `processRewireScancodeToVirtualcode()` | capsicain.cpp | ~540 | REWIRE 重映射 scancode→vcode |
| `processModifierState()` | capsicain.cpp | ~820 | 更新 modifierDown 位掩码 |
| `processCombos()` | capsicain.cpp | 863 | 匹配 COMBO 规则 |
| `playKeyEventSequence()` | capsicain.cpp | 1930 | 播放按键序列（处理 VK_CPS_* 控制码） |
| `sendVKeyEvent()` | capsicain.cpp | 2140 | 发送单个按键（处理 keysDownSent 跟踪、宏录制） |
| `sendCapsicainCodeHandler()` | capsicain.cpp | ~1830 | 处理 vcode > 0xFF 的特殊码 |
| `parseKeywordCombo()` | configUtils.cpp | 407 | 解析 .ini 中 COMBO ... > func() 语法 |
| `parseFunctionModdedkey()` | configUtils.cpp | 349 | 解析 moddedKey() 生成 TEMPRELEASE 序列 |
| `parseModString()` | configUtils.cpp | 312 | 解析 `[&^t ....]` 修饰符字符串→位掩码 |
| `getModifierForBitmask()` | modifiers.cpp | 31 | 位掩码→scancode 反向查找 |
| `reset()` | capsicain.cpp | ~530 | 重置状态（退出 passthrough 时调用） |

---

## 六、关键函数调用的概念

### key() — 简单按键
```
key(LEFT) → 序列: LEFT DOWN, LEFT UP
不释放任何修饰符，适用于修饰符已由物理键提供（如 Shift+key）
```

### moddedKey() — 剥离修饰符后按键
```
moddedKey(LEFT + LCTRL) → 序列:
  TEMPRELEASE（释放所有修饰符）
  → LCTRL DOWN → LEFT DOWN → LEFT UP → LCTRL UP
  → TEMPRESTORE（恢复所有修饰符）
适用于需要精确控制哪些修饰符参与的情况
```

### alttab() — 进入透传模式
```
alttab() → 序列: VK_CPS_ALTTAB
触发后: 发送真实 LAlt → 进入 passthrough → 所有按键直接透传
退出条件: 物理 LAlt 释放
```

---

## 七、源代码修改记录

为支持 `alttab()` 功能，修改了以下文件（相对于原始 capsicain）：

| 文件 | 改动摘要 |
|------|----------|
| `src/scancodes.h:187` | 在 `VirtualCode` 枚举中新增 `VK_CPS_ALTTAB = 0x107` |
| `src/capsicain.cpp:118-119` | `GlobalState` 结构体新增 `bool altTabPassthrough` 和 `unsigned char altTabExitScancode` |
| `src/capsicain.cpp:314-329` | 主循环中 `capsicainOn` 检查后新增 passthrough 模式分支 |
| `src/capsicain.cpp:1867-1881` | `sendCapsicainCodeHandler` 新增 `case VK_CPS_ALTTAB` 处理 |
| `src/configUtils.cpp:534-541` | `parseKeywordCombo` 新增 `alttab` 函数名识别 |

**CMake 注意**：`CMakeLists.txt:68-69` 在每次构建时自动将 `src/capsicain.ini` 拷贝到 `bin/`，因此修改 .ini 时必须同时修改源文件和 bin 文件，或只修改源文件后重新构建。

---

## 八、完整需求映射表

| 物理操作 | 效果 | 配置行 | 核心函数 |
|----------|------|--------|----------|
| CapsLock 单击 | Esc | `REWIRE CAPS LCTRL ESC` | detectTapping |
| CapsLock 组合 | LCtrl | `REWIRE CAPS LCTRL ESC` | processRewireScancodeToVirtualcode |
| LShift + RShift | CapsLock 切换 | `COMBO LSHF/RSHF [SHIFT] > key(CAPS)` | processCombos |
| Alt+Tab | 窗口切换（可长按） | `COMBO TAB [MOD9] > alttab()` | **自定义**: VK_CPS_ALTTAB + passthrough |
| Alt+HJKL | 方向键 | `COMBO H [MOD9] > moddedKey(LEFT+....)` | moddedKey |
| Alt+IA | Home/End | `COMBO I/A [MOD9] > moddedKey(HOME/END+....)` | moddedKey |
| Alt+UD | PageUp/PageDown | `COMBO U/D [MOD9] > moddedKey(PGUP/PGDOWN+....)` | moddedKey |
| Alt+Shift+HJKL | 选中方向移动 | `COMBO H [MOD9+LSHIFT] > key(LEFT)` | key (利用物理 Shift) |
| Alt+Shift+IA | 选中到 Home/End | `COMBO I/A [MOD9+LSHIFT] > key(HOME/END)` | key |
| Alt+Shift+UD | 选中翻页 | `COMBO U/D [MOD9+LSHIFT] > key(PGUP/PGDOWN)` | key |
| Alt+Caps+HL | 按单词左右移动 | `COMBO H/L [MOD9+LCTRL] > moddedKey(LEFT/RIGHT+LCTRL)` | moddedKey |