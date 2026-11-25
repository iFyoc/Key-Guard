# Key Guard（VolumeKeyGuard）

系统进程级 Xposed/LSPosed 模块：让指定应用**完全无法访问音量键与音量调节能力**，且目标应用零感知。

## 功能

- **物理按键拦截**：`KEYCODE_VOLUME_UP` / `KEYCODE_VOLUME_DOWN` / `KEYCODE_VOLUME_MUTE` 等，事件在 system_server 入队前被静默丢弃
- **自定义按键**：UI 可添加任意 KEYCODE（预置常用键，也支持手动输入名称或数值）
- **程序调用拦截**：目标应用经 Binder 调用 `AudioService` 的音量写入方法（`adjustSuggestedStreamVolume`、`setStreamVolume` 等）被静默 no-op，方法正常返回、不抛异常
- **音量条 UI 抑制**：拦截来源为黑名单应用时不弹系统音量面板，SystemUI 侧另有防御层
- **目标应用多选**：全部已安装应用列表 + 搜索，勾选即生效（无需重启目标应用）
- **液态玻璃 UI**：动态渐变背景 + 磨砂玻璃卡片，模糊强度、透明度、圆角、高光、色调、动画全部可实时自定义

## 架构（为什么不进目标应用进程）

```
模块 App（配置端）──写入──▶ SharedPreferences("vkguard_prefs".raw, 行式 key=value)
                                   │ XSharedPreferences + 1.5s TTL 热加载
        ┌──────────────────────────┼───────────────────────────┐
        ▼ system_server            ▼ com.android.systemui      ▼ 模块自身
 PhoneWindowManager             VolumeDialogControllerImpl   ModuleStatus
  interceptKeyBeforeQueueing      Callback.onShowRequested    .isModuleActive()
  interceptKeyBeforeDispatching   （防御层，失败自动放行）      → true（UI 状态显示）
 AudioService 音量写入方法
  （Binder uid → 包名归属判断）
 AudioService.showSystemUI
```

- 目标应用进程**不被装载任何 Hook**，无法感知模块存在
- 前台应用识别：优先 `WindowManagerService.mCurrentFocus`（零延迟），兜底 `ActivityTaskManager.getFocusedRootTaskInfo()`（300ms TTL 缓存）
- 所有拦截均为"静默吞掉/正常返回"，绝不抛异常
- 所有 Hook 点 try/catch + Xposed 日志，单点失败自动降级，不误伤系统（系统 UID <10000 的调用永不拦截）

## 构建

1. Android Studio（Hedgehog 或更新）打开本目录，等待 Gradle 同步（依赖 `de.robv.android.xposed:api:82`，仓库 `https://api.xposed.info/` 已配置在 settings.gradle）
2. `Build → Build Bundle(s)/APK(s) → Build APK(s)`，或命令行 `./gradlew assembleDebug`
3. 单元测试：`./gradlew test`（纯逻辑核心 core 包，JUnit 5，无需设备）

## 安装与激活（需 Root + LSPosed）

1. 安装 APK
2. LSPosed 管理器 → 模块 → 启用「Key Guard」
3. **作用域勾选：`系统框架 (android)` 和 `系统界面 (com.android.systemui)`**（Manifest 已内置 xposedscope，LSPosed 通常会自动预选）
4. 重启设备
5. 打开模块 App，顶部显示「● 模块已生效」即成功

## 真机验证清单

| # | 操作 | 预期 |
|---|------|------|
| 1 | 未开总开关，目标应用内按音量键 | 音量正常变化 |
| 2 | 开总开关+勾选目标应用，目标应用前台按音量键 | 音量不变、无音量面板、应用无任何报错 |
| 3 | 切到桌面/其他应用按音量键 | 音量正常变化 |
| 4 | 目标应用内调用 `AudioManager.adjustVolume(...)`（可用测试 App 或日志验证） | 音量不变、无异常抛出 |
| 5 | 目标应用退到后台瞬间按键 | 不拦截（前台判断实时） |
| 6 | UI 修改按键/应用列表后不重启，1.5 秒内生效 | 新规则立即生效 |
| 7 | 外观滑杆拖动 | 模糊/透明度/圆角/高光实时变化 |

## 排查（debug 七步法入口）

- Hook 日志：`adb shell "su -c 'cat /data/adb/lspd/log/modules.log'" | grep VolumeKeyGuard`（LSPosed 路径因版本而异，也可在 LSPosed 管理器→日志查看）
- 常见现象：
  - 状态显示「未激活」→ 作用域没勾系统框架，或没重启
  - 按键拦不住 → 看日志 `hookPhoneWindowManager failed`，多为 ROM 改了 PhoneWindowManager 类名（MIUI 等），把日志发出来即可适配
  - 音量面板仍弹出 → SystemUI 防御层日志 `SystemUI hooks installed, entries=0` 表示该 ROM 类结构不同，属可选层，不影响核心拦截

## 已知边界

- 蓝牙/耳机线控音量事件不走 PhoneWindowManager 按键路径，如需拦截可扩展 `AudioService` 层（当前程序调用拦截已覆盖大部分场景）
- 厂商 ROM（MIUI/ColorOS/OneUI）系统类名可能有差异，Hook 端已做多签名兜底 + 静默降级，遇到具体机型把 Xposed 日志发我即可适配

## 目录结构

```
app/src/main/java/com/volumeguard/xp/
├── core/          纯逻辑（无 Android 依赖，已单测 16/16 通过）
│   ├── ConfigCodec.java    行式配置编解码
│   ├── BlockPolicy.java    拦截决策唯一事实来源
│   └── KeyNames.java       KEYCODE 名称/数值互转
├── hook/          Hook 端（system_server / SystemUI）
│   ├── MainHook.java       入口（assets/xposed_init）
│   ├── SystemServerHook.java 按键 + AudioService + 音量条拦截
│   ├── SystemUIHook.java   音量面板防御层
│   ├── ConfigBridge.java   配置热加载
│   ├── ForegroundWatch.java 前台包名探测（双路径）
│   └── CallerResolver.java Binder uid → 包名（带缓存）
└── ui/            液态玻璃界面
    ├── MainActivity.java   主页（开关/按键/应用/外观）
    ├── AppPickerActivity.java 应用多选
    ├── GlassCard.java      玻璃卡片控件
    ├── AnimatedBackgroundView.java 动态渐变背景（API31+ RenderEffect 模糊）
    ├── AppearanceManager.java 外观参数应用器
    ├── AppConfig.java      UI 端配置读写
    └── ModuleStatus.java   激活标记
```
