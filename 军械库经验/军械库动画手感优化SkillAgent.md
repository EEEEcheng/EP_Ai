# 军械库动画手感优化 SkillAgent 经验

## 核心结论

今天沉淀出的核心方法是：主动作的 gameplay 状态不要提前结束，只在动作尾段提前恢复普通移动表现；战术冲刺必须等主动作完全结束后，再交给现有状态机判断是否进入。

这套经验适用于网易 ModSDK 军械库类项目中的客户端动画、输入手感、移动姿态权重、跑射、换弹、draw、滑铲、战术冲刺等优化。

## 可沉淀成的 Skill 名称

`modsdk-animation-feel-optimizer`

## 建议触发场景

当用户提出以下需求时，应使用这套经验：

- 优化枪械动画手感、跑射手感、切枪手感、换弹手感。
- 调整 `epGun.py`、`animationWeighSystem.py`、`epTacticalSprint.py`、`epBlunt.py`。
- 处理跑步、行走、战术冲刺、滑铲、换弹、draw、开火之间的动画权重冲突。
- 希望只改客户端表现，不改服务端位移、物理、速度修饰符或网络同步协议。

## SkillAgent 的 SKILL.md 草案

可以把下面内容复制成一个 Codex Skill 的 `SKILL.md`。

```markdown
---
name: modsdk-animation-feel-optimizer
description: Optimize Minecraft China NetEase ModSDK client-side weapon animation, input feel, movement presentation, reload/draw/fire transitions, tactical sprint blending, slide interaction, and animation weight state machines. Use when working on epGun.py, animationWeighSystem.py, epTacticalSprint.py, epBlunt.py, or similar ModSDK weapon movement/animation feel tasks.
---

# ModSDK Animation Feel Optimizer

Use this skill for client-side animation and input-feel optimization in NetEase ModSDK weapon mods.

## Core Rules

- Read the existing movement and animation system before editing.
- Keep Python 2.7 compatibility: no f-strings, type hints, async/await.
- Consult ModSDK docs before using or changing API/event calls.
- Prefer client-side animation/input changes only; do not alter server physics or add network events unless explicitly requested.
- Preserve existing user changes and avoid unrelated refactors.
- Do not leave generated test files, caches, logs, or temporary validation files behind.

## Files To Inspect First

- `epGun.py`: main weapon input, fire, reload, draw, inspect, tick state.
- `animationWeighSystem.py`: walk/run/tactical animation weights and movement presentation resolver.
- `epTacticalSprint.py`: tactical sprint phase, visual blocker, stamina, blend behavior.
- `epBlunt.py`: slide/blunt action animation and interaction with reload/fire.

## Implementation Pattern

When optimizing action feel:

1. Identify the state variable that owns the action duration, such as `reloadTick`, `boltSpeed`, `inpectTick`, or action-specific timers.
2. Add named constants for timing values instead of new magic numbers.
3. Separate visual presentation from gameplay/physics state.
4. For action tail recovery, use a one-time flag or token:
   - restore normal walk/run before the action fully ends;
   - keep tactical sprint blocked until the action fully ends;
   - clear flags on death, weapon switch, reload start, action abort, and item cut.
5. Prefer dedicated movement presentation helpers such as:
   - reload/draw late move restore;
   - skip tactical sprint while the main action is still active;
   - full resolver only after action completion.

## Tactical Sprint Rules

- During main actions, hide tactical sprint animation weight.
- For reload/draw tail recovery, restore only normal walk/run.
- Re-enter tactical sprint only after the main action fully ends and the state machine confirms sprint input is still active.

## Important Project Lessons

- `boltSpeed` can mean gun draw/bolt action for guns, but item use duration for items. Restrict gun-only logic with `nowType == 0`.
- `reloadTick` should still block normal run testing. Tail recovery should use a dedicated presentation path instead of changing reload state.
- Draw/reload tail recovery should use `skipTactical=True` or an equivalent path.
- Old timers must use tokens or be cancelled on action abort to avoid affecting a newly equipped weapon.
- Sliding reload/fire changes should not forcibly stop server slide movement unless explicitly requested.

## Validation

After edits:

- Run `python -m py_compile` on changed Python files.
- Run `git diff --check`.
- Check for generated `__pycache__` or temporary files and delete them.
- Report unrelated dirty files but do not modify them.
```

## 今天这次成功的具体经验

### 跑步开火

- 跑步开火延迟应集中成常量，例如 `RUN_FIRE_START_DELAY`。
- 松开开火时要立即取消待开火 timer，并立即清理 `v.tac.has_fire`。
- 不要等待旧的 return timer 才恢复移动表现。

### 换弹尾段恢复移动

- `reloadTick` 本身继续代表换弹动作未结束。
- 当 `reloadTick <= 15`，也就是最后约 `0.5s`，如果玩家仍在移动或奔跑，可以提前淡入普通 walk/run。
- 这一步不能提前进入 tactical sprint。
- 换弹完全结束后，再用完整移动状态机判断是否进入战术冲刺。

### Draw 尾段恢复移动

- 刚切换枪械进入 draw 时，draw 仍应阻塞战术冲刺。
- draw 最后 `0.5s` 可以提前恢复普通行走/奔跑表现。
- draw 完全结束后，清理 draw presentation 状态，再让战术冲刺状态机判断是否进入。
- `boltSpeed` 不能被强行提前清零，否则会影响枪械动作和射击节奏。

### 战术冲刺

- 主动作期间，战术冲刺动画权重要淡出或被视觉阻塞。
- 换弹、draw、检视等主动作不应保持战术冲刺第一人称姿态。
- 尾段恢复移动时，只恢复普通跑/走。
- 战术冲刺恢复必须晚于主动作结束。

### 滑铲

- 滑铲中的空弹换弹可以隐藏滑铲动画权重，但不应强制中断服务端滑铲位移。
- 滑铲有子弹开火不应走奔跑开火延迟。
- 手动换弹时，滑铲非空弹继续拦截，空弹允许进入换弹。

## 给其它工作区使用的方法

这份经验不要绑定某一个固定绝对路径。后续如果上传 GitHub、云盘或移动到其它机器，只需要让 Codex 读取“当前实际所在路径”的这份文档即可。

下面用 `<经验文档路径>` 表示这份文件当前所在的位置，例如：

```text
<经验文档路径>\军械库动画手感优化SkillAgent.md
```

如果是从 GitHub 克隆下来的，可以是：

```text
<repo根目录>\军械库经验\军械库动画手感优化SkillAgent.md
```

如果是从云盘下载的，可以是：

```text
<下载目录>\军械库动画手感优化SkillAgent.md
```

### 方法一：当作普通经验文档使用

在其它工作区开工前，把“当前实际路径”告诉 Codex：

```text
请先阅读 <经验文档路径>\军械库动画手感优化SkillAgent.md，然后按里面的规则分析这个军械库 Mod 的动画/运动系统。
```

这种方式最灵活，适合 GitHub、云盘、U 盘、临时下载目录等路径会变化的情况。

### 方法二：跟随仓库使用相对路径

如果这份经验文档跟项目仓库一起存放，推荐用相对路径：

```text
请阅读 ./军械库经验/军械库动画手感优化SkillAgent.md，并按其中的 SkillAgent 规则处理本项目。
```

或者在 `AI_WORK_TARGET` 中写：

```text
请使用 ./军械库经验/军械库动画手感优化SkillAgent.md 中的军械库动画手感优化经验。核心规则：主动作状态不提前结束，只在尾段恢复普通移动表现；战术冲刺等动作完全结束后再由状态机恢复。
```

这样即使项目换电脑、换盘符、从 GitHub 重新 clone，只要相对路径不变，就能继续使用。

### 方法三：安装成 Codex 全局 Skill

如果希望跨所有工作区自动复用，推荐把 `SKILL.md 草案` 部分保存为全局 Skill：

```text
%USERPROFILE%\.codex\skills\modsdk-animation-feel-optimizer\SKILL.md
```

在 Windows 上通常等价于：

```text
C:\Users\你的用户名\.codex\skills\modsdk-animation-feel-optimizer\SKILL.md
```

目录结构：

```text
.codex\skills\modsdk-animation-feel-optimizer\
  SKILL.md
```

安装后，在其它工作区可以直接说：

```text
使用 modsdk-animation-feel-optimizer，分析并优化这个 Mod 的跑射、换弹、draw 和战术冲刺手感。
```

或者：

```text
按 modsdk-animation-feel-optimizer 的规则，只改客户端动画表现，不改服务端物理。
```

### 方法四：GitHub/云端同步推荐结构

如果要上传 GitHub，建议把经验库做成稳定目录结构：

```text
ai-experience/
  modsdk-animation-feel-optimizer/
    SKILL.md
    references/
      eplus-animation-system.md
  军械库经验/
    军械库动画手感优化SkillAgent.md
```

使用时有两种选择：

```text
请阅读 <repo根目录>/军械库经验/军械库动画手感优化SkillAgent.md 后执行任务。
```

或者把真正的 Skill 目录复制/同步到：

```text
%USERPROFILE%\.codex\skills\modsdk-animation-feel-optimizer
```

### 方法五：让 AI 自己定位经验文档

如果不确定路径，可以告诉 Codex 搜索文件名：

```text
请在当前机器或当前仓库中查找 “军械库动画手感优化SkillAgent.md”，找到后先阅读，再按其中规则执行。
```

如果是在当前仓库内，Codex 可以优先用：

```powershell
rg --files | rg "军械库动画手感优化SkillAgent\.md"
```

## 推荐实践

最稳的方式是两层都做：

1. 经验文档放 GitHub 或云盘，作为长期经验库，路径可以变化。
2. 真正要频繁复用时，把 `SKILL.md` 安装到 `%USERPROFILE%\.codex\skills\modsdk-animation-feel-optimizer\SKILL.md`，让 Codex 在所有工作区自动触发。

经验库适合人读和继续补充；Codex Skill 适合自动触发和跨工作区复用。

## 2026-08-16 补充：run_start/run_end 时间轴与快速切换平滑

### 核心问题

当近战/道具/枪械第一人称姿态从旧 `ep_walk_co` 状态机迁移到 Python 权重混合层后，`run_start`、`run_end` 不再通过动画控制器“进入状态”来自然重置 `query.anim_time`。

因此，如果资源动画仍使用：

```json
"anim_time_update": "query.anim_time + query.delta_time * ..."
```

在常驻权重层中就容易出现：

- `run_start` 或 `run_end` 停在旧时间点；
- 第二次起跑/收跑不从正确帧播放；
- 玩家快速“奔跑 -> 不奔跑 -> 奔跑 -> 不奔跑”时，动画反复从 0 帧硬切，产生跳跃感。

### 正确做法

把 `run_start`、`run_end` 当作“一次性姿态动画”，和滑铲/趴下类似处理：

1. 注册独立 Molang 时间变量：
   - `query.mod.ep_time_run_start`
   - `query.mod.ep_time_run_end`
2. 资源动画中改为读取显式时间：
   - `run_start`: `"anim_time_update": "query.mod.ep_time_run_start"`
   - `run_end`: `"anim_time_update": "query.mod.ep_time_run_end"`
3. Python 在触发起跑/收跑时推进时间轴，而不是依赖 `query.anim_time`。
4. 不要改循环 `run` 的时间推进。循环跑步仍应按地面速度/权重推进。

### 动画长度策略

不要假设所有 `run_start/run_end` 长度一致。

优先级建议：

1. 当前物品配置 `data.postureAnimationLength.run_start/run_end`。
2. 已知资源动画名长度映射，例如 `animation.m4.run_start`、`animation.sgs.run_end`、`animation.bullet_box.run_end`。
3. 默认值：
   - `run_start`: `0.2917`
   - `run_end`: `0.6667`

弹药箱这类前置包道具如果自定义了：

```json
["ep_run_start", "animation.bullet_box.run_start"],
["ep_run_end", "animation.bullet_box.run_end"]
```

应补充：

```json
"postureAnimationLength": {
  "run_start": 0.5333,
  "run_end": 0.8833
}
```

### 快速切换平滑规则

处理玩家快速切换奔跑状态时，不要简单重置时间轴：

- `run_start` 被 `run_end` 打断时：用 `1 - run_end当前进度` 作为新的 `run_start` 起点。
- `run_end` 被 `run_start` 打断时：用 `1 - run_start当前进度` 作为新的 `run_end` 起点。
- 同方向重复触发时：从当前进度继续，不要回到 0。
- 从循环 `run` 进入 `run_end` 时：`run_end` 从 0 开始。
- 从已有 `run` 权重回到 `run_start` 时：可以按 run 权重估算起始进度，避免从 idle 姿态硬跳。

这会让“奔跑 -> 不奔跑 -> 奔跑 -> 不奔跑 -> 奔跑”表现像在两个姿态之间往返过渡，而不是每次重新播放第一帧。

### Timer 与 token 规则

一次性姿态动画的自动淡出必须带 token：

- 每次触发 `run_start/run_end` 都更新 token。
- 延迟淡出时先检查 token 是否仍是当前播放。
- 旧 timer 不能影响新一轮动画。
- `run_end` 淡出等待时间应使用 `max(权重淡入时间, 剩余动画时间)`，避免长收跑动画播到中段就被淡掉。

### 资源镜像一致性

如果同一动画同时存在于资源动画文件和 `modconfigs/EP_JG_CAMERA` 或其它镜像配置中，要同步 `anim_time_update`，避免后续维护时两个副本不一致。

### 验证清单

完成后至少检查：

- 快速连按奔跑键：`run_start/run_end` 不抽跳、不硬回 0 帧。
- 起跑未播完就松开：`run_end` 从对应反向进度接上。
- 收跑未播完又按奔跑：`run_start` 从对应反向进度接上。
- 普通 `run` 循环仍按移动速度推进。
- 自定义长度的道具，例如弹药箱，不被默认枪械长度截断。
- Python 2.7 AST 或语法检查通过。
- JSON 使用 UTF-8 解析检查；PowerShell 默认编码可能误读中文名，必要时显式 `-Encoding UTF8`。
- 不留下 `.pyc`、`__pycache__`、临时测试文件。

### Skill 规则补充

以后处理军械库动画权重系统时，遇到“从状态机迁移到常驻权重层”的动画，优先检查它是否仍依赖 `query.anim_time`。

如果动画是一次性过渡层，并且需要重复触发或反向打断，应该优先改为：

- 独立 `query.mod.ep_time_xxx`；
- Python 显式推进；
- 当前进度续播/反向镜像；
- token 化 timer；
- 配置可覆盖动画长度。
## 2026-08-16 补充：旧近战/道具包 run_start/run_end 兼容回退

### 触发场景

当近战包或道具包接入新的 Python 运动权重混合系统后，如果切换到疾跑姿态时 `run_start`、`run_end` 仍然不播放，优先怀疑资源动画仍是旧格式，而不是只看 Python 权重是否切换成功。

典型受影响包包括：

- 近战包：军工4_求生之路、军工5_暗影突袭、军工14_APX传家包。
- 道具包：军工6_OL道具求生、军工11_堑壕战术、军工13_森罗物语。

这些独立包常见特征是：配置已经覆盖了 `ep_run_start` / `ep_run_end`，但资源动画内部仍依赖旧状态机的 `query.anim_time`，没有绑定 `query.mod.ep_time_run_start` / `query.mod.ep_time_run_end`。

### 关键诊断

不要只检查 `animationWeighSystem.py` 中的权重变量。需要同时检查三层：

1. 当前物品配置里的 `render.Animations` 是否覆盖了：
   - `ep_run_start`
   - `ep_run`
   - `ep_run_end`
2. 被覆盖的动画资源是否使用新时间轴：
   - `run_start`: `anim_time_update = query.mod.ep_time_run_start`
   - `run_end`: `anim_time_update = query.mod.ep_time_run_end`
3. 旧 `ep_walk_co` 是否还和新 `ep_co` / `melee_co` / `ep_item_co` 同时注册，导致姿态叠加。

注意：不同独立包的 JSON 结构可能不完全一致。运行时核心数据通常是顶层 `render`，不是 `data.render`；写扫描脚本时不要误判成 0 个配置。

### 安全修复优先级

如果要同时兼容多个旧独立包，优先改前置核心系统，不要直接批量重写外部包资源 JSON。

推荐策略：

1. 前置包先注册默认姿态动画。
2. 再读取当前物品 `render.Animations` 覆盖同名键。
3. 对近战/道具的 `ep_run_start` / `ep_run_end` 做兼容判断：
   - 已知在前置长度映射里的新格式动画，允许自定义覆盖。
   - 配置显式声明兼容标记，例如 `data.postureRunOneShotTimeDriven`，允许自定义覆盖。
   - 其它未知旧格式动画，回退到前置默认 `animation.default.run_start/end`。
4. `ep_run` 循环层可以尽量保留自定义动画；优先只回退一次性起跑/停跑层。

这样可以先解决“权重切到了但动画没播”的问题，同时避免对 6 个外部包做大规模格式化改写。

### 可选兼容标记

如果某个独立包已经把自定义 `run_start/run_end` 资源改成了新时间轴，可以在物品 `data` 中加入：

```json
"postureRunOneShotTimeDriven": true
```

或者按键单独声明：

```json
"postureRunOneShotTimeDriven": {
  "run_start": true,
  "run_end": true
}
```

这样前置核心就可以保留该物品的自定义起停动画，不再回退到默认动画。

### 权重系统隐藏坑

`_SetTargeWeigh()` 注册过渡时，必须立即把起始权重写入本地 `animation_weigh_dict`，再写 Molang：

```python
self.animation_weigh_dict[weigh_animation] = float(weigh_start)
molangComp.Set(molang_key, round(weigh_start, 4))
```

如果这行因为乱码注释、合并冲突或误删没有实际执行，快速切换姿态时会复用旧权重，表现为：

- 奔跑动画像叠了两层；
- 切枪 -> 切刀 -> 切枪 后 run 姿态飞掉；
- 快速奔跑/停跑时起止姿态跳帧。

### 外部包排查方法

如果用户指定的是 VSCode 工作区文件，不要假设 `.code-workspace` 所在目录就是 AddOn 包。应先读取 workspace 的 folders，定位真实 AddOn 根目录，再递归搜索：

```powershell
Get-ChildItem -LiteralPath <AddOn根目录> -Recurse -File -Filter *.json |
  Where-Object { $_.FullName -match 'modconfigs\\EP_JG_DATA' }
```

再从每个配置的顶层 `render.Animations` 中找 `ep_run_start/end`。

### 验证规则

完成修复后验证：

- 近战包切入疾跑：`run_start` 能播放，结束后进入 `run`。
- 松开疾跑：`run_end` 能播放，移动时回到 walk。
- 快速切枪、切刀、切回枪：不会叠加两层 run 动画。
- 道具包，尤其前置弹药箱与食物道具：疾跑起停不丢姿态。
- Python 2.7 `py_compile` 通过。
- 语法检查生成的 `.pyc` 必须删除，不留下 `__pycache__` 或临时扫描文件。

### 经验原则

当多个外部包都有旧格式资源时，优先提供“核心兼容 + 显式新格式 opt-in”机制。只有用户明确允许并接受格式化风险时，才批量迁移外部包 JSON。

## 2026-08-17 补充：战术冲刺速度退出与姿态音效更新

### 战术冲刺限制经验

当用户提出“射击、拉栓、瞄准、换弹不能战术冲刺”时，不要只淡出第一人称 `tactical_sprint` 权重。动画淡出只解决视觉冲突，真正的速度加成通常跟 `epPosture.moveState == MOVE_STATE_TACTICAL_SPRINT` 和服务端 speed modifier 绑定。

正确做法是把限制放在战术冲刺控制器本身：

1. `CanEnterPhase()` 入口处阻止进入战术冲刺。
2. `Update()` 中每帧低成本检查运行中阻塞，发现后主动退出 tactical moveState。
3. 开火入口不要再用“只隐藏动画但保留 phase”的路径。
4. 退出时必须走会更新 moveState 和 `SetMotion()` 的路径，让服务端移除战术冲刺速度 modifier。

推荐阻塞字段：

```python
('reloadTick', 'fireSpeed', 'boltSpeed', 'hasAim', 'leftClick')
```

如果右键瞄准在 `hasAim` 置位前有短暂输入窗口，可以额外判断：

```python
getattr(owner, 'rightClick', False) and getattr(owner, 'nowType', None) == 0
```

经验原则：

- “动作不能战术冲刺”必须同时处理视觉权重和速度状态。
- 如果 `ExitForMainAction(..., keepPhase=True)` 仍保留 `MOVE_STATE_TACTICAL_SPRINT`，服务端速度可能继续存在。
- 射击/拉栓/瞄准/换弹这类主动作应使用 `keepPhase=False` 或等价硬退出路径。
- 不需要新增网络事件；客户端退出 moveState 后沿用现有 `SetMotion()`/姿态同步即可。

### 姿态音效更新经验

运动姿态音效分两类处理：

1. 循环姿态声：走路、瞄准走路、奔跑、战术冲刺。
2. 一次性切换声：开始奔跑、结束奔跑。

循环姿态声适合放在 `move_camera_tracks.animation.json` 的 `sound_effects` 关键帧里，由 `CameraTrackMixer` 的关键帧跨越检测播放：

- `walk` / `walk_aiming`: `ep_jxk.walk`
- `run`: `ep_jxk.run`
- `tactical_sprint`: `ep_jxk.tactical_sprint`

一次性切换声不适合塞进 run 摄像机循环轨道，因为当前摄像机系统会把 `run_start/run_end` 权重合并成 run pose 采样。如果不区分，起跑/收跑阶段可能误触发循环 `ep_jxk.run`。

推荐把一次性声放在运动权重状态切换入口：

- 开始奔跑时播放 `ep_jxk.run_start`。
- 结束奔跑时播放 `ep_jxk.run_end`。
- 加短冷却，例如 `0.18s`，防止输入事件重复上报导致连响。
- 播放出口复用 `CameraTrackMixer.PlayMovePoseSound()` 或等价方法，继续受第一人称、持有军械库物品、姿态摄像机/姿态音效开关控制。

### CameraTrackMixer 音效规则

姿态音效最好仍走摄像机混合器统一出口，而不是在多个系统里直接 `PlayCustomMusic`：

- 关闭姿态摄像机/姿态摇晃设置时，应同时停止姿态循环声。
- 未持有军械库物品、第三人称、玩家死亡、无姿态权重时，不应继续采样和播声。
- 关键帧音效使用跨越检测和冷却，不要按帧重复播放。
- `run_start/run_end` 借用 run 摄像机轨道时，需要避免误播 `ep_jxk.run`，例如只有真实 `run` 权重大于阈值时才播放 run 循环声。

### 验证清单

完成类似改动后检查：

- 战术冲刺中开火：动画淡出，服务端速度加成也退出。
- 战术冲刺中拉栓/换弹/瞄准：不会重新进入战术冲刺速度。
- 动作结束后仍按住奔跑：按现有状态机恢复普通奔跑或战术冲刺。
- 开始奔跑只响 `ep_jxk.run_start`，奔跑循环响 `ep_jxk.run`。
- 松开奔跑只响 `ep_jxk.run_end`，移动时回到 `ep_jxk.walk`。
- 战术冲刺循环响 `ep_jxk.tactical_sprint`。
- 关闭姿态摄像机/姿态音效设置后，循环姿态声不残留。
- Python 2.7 语法检查和 JSON 解析检查通过。
- 检查并删除 `.pyc`、`__pycache__`、临时日志或扫描文件。

### Skill 规则补充

以后处理军械库运动姿态问题时，先判断用户说的“淡出”是哪一层：

- 只是视觉冲突：改权重淡入淡出即可。
- 涉及速度、体力、服务端 modifier：必须同步退出 posture/moveState。
- 涉及循环音效：优先走姿态摄像机/混合器统一出口。
- 涉及一次性姿态切换音效：放在输入/权重状态切换点，配冷却和环境条件。
