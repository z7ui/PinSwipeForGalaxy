# PIN 滑动输入 for Galaxy

三星 One UI 的锁屏数字密码，支持**按住不松手、划过即输入**：手指按在第一位数字上不抬起来，
一路划过去，按划过的顺序依次输入；**长按 ⌫ 清空**。

实测机型：**Galaxy S25 Ultra（SM-S9380）· One UI 8.0.5 · Android 16（SDK 36）**

> 这是**独立的一个软件**：跟 HyperOS 那个版本是两个项目、两个包名、分开发布。
> 两边都 hook SystemUI，所以同一台机器上只能装一个，否则一次划动会输入两遍。

---

## 一、它靠什么工作

普通 LSPosed 模块，作用域只有 **SystemUI**。不写系统分区、不需要重启 zygote。

挂点是拿真机的 SystemUI.apk 反编译出来的：

```
com.android.keyguard.NumPadKey#onTouchEvent(MotionEvent)   ← 数字键
com.android.keyguard.SecNumPadKey **extends** NumPadKey     ← 三星的键，没覆写 onTouchEvent
```

为什么挂这里：数字键一旦收到 ACTION_DOWN 就成了这次手势的 touch target，之后**每一个 MOVE
都会送到它手里**（哪怕手指早跑到别的键上）。从这里旁观能拿到整段手势的完整轨迹，
而且**不改参数、不吞事件**，原生逻辑一点不受影响。

输入动作用的是 ROM 自己的 performClick() —— 和它处理外接键盘时走的是同一条路
（KeyguardPinBasedInputView#onKeyDown → performNumberClick() → mButtons[i].performClick()），
所以不碰任何私有输入逻辑，也不绕过 isEnabled() 检查。

| 项 | 三星上是什么 |
| :--- | :--- |
| 数字键类 | SecNumPadKey extends NumPadKey（未覆写 onTouchEvent） |
| 键数组字段 | mButtons（**没有** getButtons()，HyperOS 才有） |
| ⌫ 字段 | mDeleteButton（NumPadButton，可见可点） |
| ⌫ 短按 | ROM 绑定为删一位 |
| ⌫ 长按 | ROM 绑定为**清空**（resetPasswordText(true, true)） |
| 图片 PIN | 用的是按键的 mDigit，所以图片 PIN 模式下一样能划 |

## 二、两个从真机数据里抠出来的规则

### 2.1 数字键只认中央 56%，但 ⌫ 用整个键

真机量出来的三星键盘几何：

```
键尺寸 240 x 240
1(365,1111) 2(720,1111) 3(1075,1111)
4(365,1413) 5(720,1413) 6(1075,1413)
7(365,1715) 8(720,1715) 9(1075,1715)
⌫(365,2017)           0(720,2017)
```

（注意三星的 **⌫ 在左下角**，HyperOS 在右下角。）

斜划时手指只要有一点弧度就会蹭到隔壁键，所以数字键的判定缩到中间那一块。
但**⌫ 是「按」的目标，不是「划过」的目标**，缩到中央会让手指够不着 —— 这条踩过坑：
缩到 56% 之后 ⌫ 只剩 134x134，短按和长按都摸不到。现在 ⌫ 用整个键判定。

### 2.2 相邻两位不会重复上报

手指停在中央区域的边界上会来回穿越，同一个键会被上报两次（HyperOS 上抓到过间隔 **5ms**
的重复，人不可能是这个速度）。规则：**连续两次上报同一个键就吞掉**；中间隔了别的键
（真的划回来）照报，所以 1→5→1 依然是 151。

### 2.3 有些密码必须分两笔

如果两个要输的数字之间的**直线上正好压着另一个数字**，那一段就不能一笔划：
手指确实从那个数字上过去了，任何算法都分不出你是想输它还是路过它。

```
1→2→3   一笔可以（中间那个 2 就是你想要的）
1→5→9   一笔可以
9→7     一笔会多出一个 8（8 在 9 和 7 中间）→ 在这里抬一下手指
```

## 三、安装

1. 装 APK
2. LSPosed 里启用「PIN 滑动输入 for Galaxy」，作用域勾 **SystemUI**（APK 里已经用 xposedscope 声明了，通常会自动勾上）
3. 重启 SystemUI 让它注入（**不是重启手机**）：su -c 'killall com.android.systemui'

## 四、先验证，再上锁屏

APK 里带一个演示页（桌面图标「滑动输入 · 演示」）：和锁屏同样布局的假键盘，用的是**同一个**
SwipePadController，挂点也一样。先在这里试：按住一位数字不松手划过去、单点确认不会翻倍、
按住 ⌫ 清空。手感对了再去真锁屏。

## 五、怎么确认真的生效

模块只在 hook / 接管 / 出错时写 LSPosed 日志（逐位日志默认关闭，把模块里的 LOG_TRIGGERS 改成 true
重新装就会每一位都留一条）：

su -c 'grep PinSwipe /data/adb/lspd/log/modules_*.log | tail -20'

正常的开头是：

```
PinSwipe: hooked com.android.keyguard.NumPadKey#onTouchEvent
PinSwipe: 接管 PIN 键盘，数字键 10 个，附加键 1 个
```

## 六、出事了怎么退

模块只做加法、绝不吞事件，最坏情况是「滑动没反应」或「多输一位」，不会让你点不动。

- 进 LSPosed 关掉这个模块，再 su -c 'killall com.android.systemui'
- 或者直接卸载这个 APK（它是普通应用，跟系统分区没关系）
- 输错密码会计入失败次数，测试时别拿一整排数字去划

## 七、没做的

- 只覆盖 **PIN**（数字密码）。图形密码 / 混合密码没做。
- 滑过 ⌫ 只删一位；按住不放才清空（不会连续重复清空）。
- 2.3 里说的「直线穿过中间数字」的图案做不到一笔，那是几何问题不是 bug。

## 八、文件

```
app/src/main/java/com/z7ui/pinswipe/
├─ PinSwipeModule.java      LSPosed 入口：挂 NumPadKey#onTouchEvent，按键 performClick()
├─ SwipePadController.java  ★ 划过即输入的状态机（中央判定 / 相邻去重 / 长按）
└─ DemoActivity.java        演示页：假键盘，先在真机验证手感

app/src/main/assets/xposed_init         LSPosed 入口类声明
app/src/main/res/values/arrays.xml      xposed_scope = com.android.systemui
```

## 九、构建

Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
& ..\android-env\build-apk.ps1 . -Task assembleDebug

---

作者 **Deepseek** · B站 **@z7ui**
