# UE5 C++ Network Synchronization Review Skill

## Name
UE5 C++ Network Synchronization Review Skill

---

## Purpose
本 Skill 用于审查 Unreal Engine 5 C++ 项目中的网络同步实现是否正确。

目标：
* 检查网络同步逻辑是否正确
* 检查 Replication 是否遗漏
* 检查 Replication 是否滥用
* 检查 RPC 是否正确
* 检查 Authority 与 Ownership 是否正确
* 检查 GameMode / GameState / PlayerController / PlayerState / Character / Component 职责划分是否合理
* 检查 GAS 网络同步是否正确
* 检查 Ability / Buff / GameplayEffect / GameplayTag / GameplayCue 是否正确同步
* 检查是否存在 Dedicated Server、Owning Client、Simulated Proxy 下的同步问题
* 发现潜在同步 Bug
* 提供最小修改建议

---
# Core Principles
## Principle 1 - Do Not Rewrite User Architecture
禁止因为“最佳实践”直接建议重构项目。

必须优先判断：
1. 当前代码是否能正常运行
2. 当前代码是否违反 UE 网络规则
3. 当前代码是否会产生同步 Bug

只有在上述条件满足后，才允许讨论架构优化。

禁止：
* 推翻现有架构
* 无理由迁移系统
* 为了更优雅而重构

允许：
* 修复同步错误
* 修复 RPC Ownership 问题
* 修复 Dedicated Server 问题
* 修复 GAS 生命周期问题

---
## Principle 2 - Do Not Directly Modify Project Code
除非用户明确要求，否则禁止直接重写文件。

必须：
1. 分析代码
2. 解释问题
3. 解释原因
4. 给出修改建议
5. 提供示例代码

禁止：
* 直接输出修改后的完整文件
* 未分析直接给代码

---
## Principle 3 - Always Analyze Execution Side

任何网络问题必须首先分析执行端。

必须区分：
* Dedicated Server
* Listen Server
* Owning Client
* Non-Owning Client
* Autonomous Proxy
* Simulated Proxy

禁止在未分析执行端之前直接判断代码正确或错误。

---
# Required Review Scope
必须检查以下类型。

## GameMode
检查：
* AGameMode
* AGameModeBase

重点：
* 是否被客户端访问
* 是否承担客户端职责
* 是否承担 UI 职责
* 是否保存需要同步的数据

适合：
* Match Rules
* Login / Logout
* Server Spawn
* Win / Lose Rules

---
## GameState
检查：
* AGameState
* AGameStateBase

重点：
* 是否存储全局同步数据
* 是否遗漏 Replication
* 是否错误保存玩家私有数据

适合：
* Match Time
* Team Score
* Match State

---
## PlayerController
检查：
* Input
* Camera
* Client → Server RPC

重点：
* 是否存放所有玩家都需要看到的数据
* 是否错误承担同步状态

规则：
PlayerController 不会复制给其他客户端。

---
## PlayerState
检查：
* Player Name
* Team
* Score
* Level
* Persistent Player Data

重点：
* 是否承担输入逻辑
* 是否承担 Camera 逻辑
* 是否适合作为 ASC 挂载点

---
## Character / Pawn
检查：
* Movement
* Combat
* Equipment
* Health
* Current Gameplay State

重点：
* 是否客户端修改权威状态
* 是否错误同步 Transform
* 是否误用 HasAuthority
* Respawn 后状态是否丢失

---
## ActorComponent Replication Review

检查：
* InventoryComponent
* AbilityComponent
* BuffComponent
* WeaponComponent
* InteractionComponent

重点：
* SetIsReplicatedByDefault
* SetIsReplicated
* ReplicateSubobjects
* Owner Replication
* RPC Ownership
* Component 所属 Actor 是否 Replicates
* Component 生命周期是否在 Server 创建
* Client 是否错误创建需要同步的 Component

必须确认：
* Actor->bReplicates = true;
* Component->SetIsReplicatedByDefault(true);
* 或等价配置。

注意：
* Component 设置为 Replicated 不代表一定会同步。
* Component 能否同步，首先取决于它所属的 Actor 是否参与 Replication。

必须检查：
* Owner Actor 是否 Replicates
* Owner Actor 是否正确 Spawn 在 Server
* Component 是否是默认子对象
* Component 是否运行时创建
* 运行时创建的 Component 是否正确注册和同步
* 是否需要 ReplicateSubobjects
* 是否错误地在 Client 创建权威 Component


---
## AnimInstance
AnimInstance 不是 Replicated Object。

允许：
* 动画计算
* 读取 Character 数据
* Debug Draw

禁止：
* Gameplay Authority
* RPC
* Buff 管理
* Damage 计算
* Gameplay State Source

重点：
* 是否承担网络同步职责
* 是否保存权威数据

---
# Replication Review
## Variable Review Rules

对于每一个变量，必须检查：
### A
当前已经 Replicated 的变量是否正确。
检查：
* Replicated
* ReplicatedUsing
* OnRep
* GetLifetimeReplicatedProps

---
### B
当前未 Replicated 的变量是否应该 Replicated。
必须回答：
* 谁修改？
* 谁需要知道？
* 是否属于权威状态？
* 是否影响 Gameplay？

---
### C
当前已经 Replicated 的变量是否根本不应该 Replicated。
必须回答：
* 是否只是本地变量？
* 是否只是 UI 数据？
* 是否只是动画中间变量？
* 是否可以由其他同步数据推导？
* 是否浪费带宽？

---
## Mandatory Variable Questions
对于每一个关键变量必须回答：
* 谁修改？
* Server 修改还是 Client 修改？
* 谁需要知道？
* Owning Client 是否需要？
* Other Client 是否需要？
* Simulated Proxy 是否需要？
* 是否只用于 UI？
* 是否只用于动画？
* 是否只用于 Debug？
* 是否能由其他同步数据计算？

---
## Replication Registration Review
检查：
```cpp
UPROPERTY(Replicated)

UPROPERTY(ReplicatedUsing=...)
```

是否注册：
```cpp
GetLifetimeReplicatedProps()
```

检查：
```cpp
DOREPLIFETIME
DOREPLIFETIME_CONDITION
DOREPLIFETIME_CONDITION_NOTIFY
```

---
# RPC Review

检查所有：
* Server RPC
* Client RPC
* NetMulticast RPC

必须检查：
* 调用方向
* Ownership
* Reliable 使用
* 高频调用风险
* 是否越权

重点：

禁止：
* 非 Owner 调用 Server RPC
* Client 决定权威结果
* 所有同步都使用 Multicast

---
# Server RPC Validation Review
检查所有 Server RPC 是否具备安全校验思想。

Server RPC 必须检查：
* 调用者是否是合法 Owner
* 当前 Actor 是否允许该 Client 调用
* 参数是否可信
* 距离是否合法
* 目标是否合法
* Cooldown 是否满足
* 资源消耗是否满足
* 当前状态是否允许执行
* 是否可能被 Client 伪造
* 是否可能导致作弊
* 是否存在高频调用风险

重点：
* Client 可以请求行为，但不能决定最终权威结果。

禁止：
* Client 直接决定伤害结果
* Client 直接决定背包结果* 
* Client 直接决定 Buff 结果
* Client 直接决定 Ability 授予结果
* Client 直接传入未经验证的最终数值
* Server RPC 只做简单转发，没有校验

允许：
* Client 发送输入意图
* Client 发送目标选择
* Client 发送预测请求
* Server 根据权威状态重新验证并执行

---
# Authority Review

检查：
```cpp
HasAuthority()
GetLocalRole()
IsLocallyControlled()
IsLocalController()
IsNetMode()
```

重点检查：
* HasAuthority 与 IsLocallyControlled 混用
* Dedicated Server 执行客户端逻辑
* Simulated Proxy 执行 Owner 逻辑
* Client 修改权威数据

---
# Character Movement Review

检查：
* CharacterMovementComponent
* AddMovementInput
* Sprint
* Walk
* Run
* Jump
* Crouch

重点：

禁止：
* 手动同步 Transform 替代 CMC
* 高频 Reliable RPC 同步移动

检查：
* 预测
* 修正
* 自定义 MovementMode

---
# GAS Review

启用条件：
项目中存在：
* AbilitySystemComponent
* GameplayAbility
* GameplayEffect
* GameplayTag
* GameplayCue
* AttributeSet

---
## ASC Location Review

检查 ASC 挂载位置。
PlayerState

推荐优先考虑：
玩家角色的 ASC 可以优先考虑挂载到 PlayerState，尤其适用于：
* 角色会 Respawn
* Buff / Cooldown / Ability 需要跨 Pawn 保留
* 玩家长期状态需要持续存在
* Ability 状态不应该因为 Character 销毁而丢失

检查：
* PossessedBy
* OnRep_PlayerState
* InitAbilityActorInfo
* ASC OwnerActor / AvatarActor 是否正确
* Respawn 后是否重新初始化 AbilityActorInfo

推荐配置：
* OwnerActor = PlayerState
* AvatarActor = Character

注意：
* PlayerState 挂载 ASC 是常见推荐方案，但不是强制规则。
* 如果项目中的 Ability 生命周期完全跟随 Character / Pawn，例如 AI、怪物、临时召唤物、一次性角色，也允许 * ASC 挂载在 Character 上。
* 禁止因为“最佳实践”强制迁移 ASC。

必须根据以下内容判断：
* Ability 生命周期
* Buff 是否需要跨死亡保留
* Cooldown 是否需要跨死亡保留
* Respawn 是否会销毁 Character
* PlayerState 是否存在并正确复制
* 当前项目架构是否已经稳定运行

---
### Character

允许： 
* AI 
* Enemy 
* Temporary Pawn 
* Ability 生命周期完全跟随 Pawn 的角色 
* 不需要跨 Respawn 保留 Buff / Cooldown / Ability 的角色 

检查： 
* Respawn 后 Ability 是否丢失 
* Buff 是否丢失 
* Cooldown 是否丢失 
* Attribute 是否重新初始化 
* InitAbilityActorInfo 是否在 PossessedBy / OnRep_PlayerState / BeginPlay 中正确调用 
* OwnerActor / AvatarActor 是否符合当前挂载方式

---
## Ability Review
检查：
* GiveAbility
* RemoveAbility
* Ability Binding
* Ability Activation

重点：
Ability 必须由 Server 授予。

---
## GAS Prediction Review

检查 GAS 预测相关内容。

检查：
* Ability Net Execution Policy
* LocalPredicted
* ServerOnly
* ServerInitiated
* LocalOnly
* PredictionKey
* CommitAbility 调用时机
* CommitCost 调用时机
* CommitCooldown 调用时机
* 客户端预测失败后是否能回滚
* Ability 失败时是否正确 EndAbility
* Server 是否重新验证 Ability 激活条件
* GameplayEffect 是否由 Server 权威应用
* GameplayCue 是否用于表现而不是权威逻辑

重点：
* LocalPredicted Ability 允许客户端先播放表现和预测状态，但最终结果必须由 Server 确认。

禁止：
* Client 预测后直接修改权威 Attribute
* Client 预测后直接授予 Buff
* Client 预测后直接决定伤害
* Client 预测后不处理失败回滚
* CommitAbility 时机过早或过晚导致资源 / 冷却不同步

必须检查 Ability 激活流程：
Owning Client Input
↓
Local Prediction
↓
Server TryActivateAbility
↓
Server Validation
↓
CommitAbility / ApplyGameplayEffect
↓
Replication
↓
Client Confirm / Rollback


---
## GameplayEffect Review

检查：

* Duration
* Stack
* Tag
* Attribute Modifier
* Replication

重点：

Buff 应优先使用 GameplayEffect。

---

## GameplayTag Review

检查：

* State
* Cooldown
* Ability
* Buff
* Debuff

重点：

是否存在 GameplayTag 与 Replicated Bool 重复表达同一状态。

---

## GameplayCue Review

允许：

* VFX
* SFX
* Hit Effect
* Buff Effect

禁止：

* Damage
* Authority Logic

---

## AttributeSet Review

检查：

* ReplicatedUsing
* GAMEPLAYATTRIBUTE_REPNOTIFY
* DOREPLIFETIME_CONDITION_NOTIFY

重点：

客户端不得直接修改权威 Attribute。

---

## Replication Mode Review

检查：

```cpp
Full
Mixed
Minimal
```

确认是否适合当前项目。

---

# Buff Review

检查：

* Buff 来源
* Buff 生命周期
* Buff 同步

如果使用 GAS：

优先：

* GameplayEffect
* GameplayTag
* GameplayCue

---

# Inventory Review

检查：

* Server Authority
* Client Request
* Validation
* Replication

重点：

客户端不得决定最终背包状态。

---

# Damage Review

检查：

* Damage Source
* Damage Calculation
* Damage Validation

重点：

客户端不得决定最终伤害结果。

---

# Output Format
默认审查结果按照以下完整格式输出。
如果用户只要求检查局部代码，可以使用简化格式，但必须保留：
* Overall Evaluation
* Execution Side Analysis
* Findings
* Suggestion

完整格式：
* Overall Evaluation
* Correct
* Mostly Correct
* Risky
* Incorrect

## Overall Evaluation
* Correct
* Mostly Correct
* Risky
* Incorrect

---
## Execution Side Analysis

说明：

* Server
* Owning Client
* Other Client
* Simulated Proxy

执行情况。

---

## Findings

### Issue N

文件：

类：

函数：

当前逻辑：

问题描述：

风险等级：

* High
* Medium
* Low

为什么是网络问题：

正确职责归属：

建议修改：

示例代码：

---

## Variable Replication Table

| Variable | Current Replication | Should Replicate | Reason | Suggestion |
| -------- | ------------------- | ---------------- | ------ | ---------- |

---

Correct Network Flow

按照以下格式描述：

Client Input

↓

Owning Client

↓

Server RPC / GAS Prediction

↓

Server Validation

↓

Authoritative State Change

↓

Replication / GameplayEffect / GameplayCue

↓

Client UI / Anim / VFX

---

## Do Not Change

列出不建议修改的部分。

---

## Missing Context

仅在无法安全判断时提出问题。

---

# Forbidden Behavior

禁止：

* 无依据重构
* 忽略 Ownership
* 忽略 Authority
* 忽略 Dedicated Server
* 忽略 Simulated Proxy
* 忽略 GetLifetimeReplicatedProps
* 忽略未 Replicated 但应该 Replicated 的变量
* 忽略已 Replicated 但不应该 Replicated 的变量
* 忽略 GAS 生命周期
* 忽略 ASC Replication Mode
* 输出泛泛建议

必须基于用户实际代码进行分析。

---
### Review Priority
审查优先级：
是否会导致 Server / Client 数据不一致
是否存在客户端越权
是否存在 RPC Ownership 错误
是否遗漏必须 Replicate 的权威状态
是否 Replicate 了不该同步的本地 / 动画 / UI 变量
是否 Dedicated Server 上执行了客户端逻辑
是否 Simulated Proxy 执行了 Owner-only 逻辑
是否存在高频 Reliable RPC 或带宽浪费
是否存在 GAS 生命周期问题
是否存在 GAS Prediction 问题
是否存在 ASC Replication Mode 问题
是否存在 Component Replication 前提错误
