---
aliases:
  - GAS Prediction
tags:
  - 虚幻引擎
  - GAS

title: GAS系统-预测
date: 2025-08-01
author: jimmyzou
toc_number: true
decscription: 
---

GAS的客户端预测系统。**预测**即在网络游戏中，客户端先自行相应输入等事件，以获得更及时的操作反馈，预测系统需要的处理的问题包括：

1. 预测失败后预测改动的回滚。

2. 预测成功时，不会重复应用服务器下发的已经预测过的改动。

3. 预测链的处理，即一个预测的内容引发了其他更多的预测。

4. 预测的完整性，保证所有需要预测内容都正确的被预测了。

5. 预测的修改覆盖由服务器同步的状态。

---

### **1. GAS预测系统概述**

GAS的预测系统 对业务逻辑透明，在Ability执行的时候，会自动预测其中能预测的部分，并处理好预测相关的逻辑。也正因此，只有部分内容支持预测。

- 激活GA，包括链式激活，即一个预测的GA触发另外一个GA。

- 应用GE，只支持其中的 Attribute Modification 和 Gameplay Tag Modification，不支持 Attribute Executions，GE的移除和周期性效果也不支持预测。

- 蒙太奇。

- 位移，集成在移动组件中。

---

### **2. GAS预测系统实现**

预测系统的一个基础概念是 **预测键（Prediction Key）**，预测键本质是一个由客户端生成的唯一ID。客户端会将预测键发送给服务器，并将其预测的行为和预测产生的影响关联到这个预测键上。服务器可以回复给客户端**接受 / 拒绝**这个预测键，并且将服务器产生的影响也关联到这个预测键上。

**注意**：预测键总是可以从客户端发送到服务器，但是从服务器发送到客户端时，只会发给将这个预测键的发送方（将这个预测键发送给服务器的那个客户端），其他的客户端只会收到一个非法的预测键。相关逻辑在 `FPredictionKey::NetSerialize` 中实现。

---

#### - **GA的预测逻辑**

激活GA是首要的预测行为——它会生成最初的预测键。当客户端以预测的方式激活某个GA时，总是会显示的请求服务器，并且服务器会显示的回复。当一个GA被以预测的方式激活后（在服务器回复还没有发送之前），客户端存在”预测窗口“，在这个窗口期间，客户端可以自行预测GA产生的影响而不用询问服务器。预测窗口在调用 `ActivateAbility` 之后生效，在 `ActivateAbility` 的调用结束后失效。所以我们不能预测多帧的行为，在蓝图中的 `Timer` 或者 `Ability Task` 都会使预测窗口失效。

`AbilitySystemComponent` 使用一系列方法在客户端与服务器之间通信GA的激活：`TryActivateAbility` -> `ServerTryActivateAbility` -> `ClientActivateAbility(failed/succeed)`。

1. 客户端调用 `TryActivateAbility`，它生成一个预测键，并且调用`ServerTryActivateAbility`。

2. 客户端逻辑继续执行，将生成的预测键保存在 Ability 的 `ActivationInfo` 中并调用`ActivateAbility`。

3.  在 `ActivateAbility` 的调用结束之前，所有产生的影响都需要与预测键关联。

4. 服务器在 `ServerTryActivateAbility` 中决定GA是否执行成功，调用 `ClientActivateAbility(failed/succeed)` 并且将客户端发过来的预测键保存在 `UAbilitySystemComponent::ReplicatedPredictionKey`中。

5. 如果客户端收到了`ClientAbilityFailed`，则会立刻结束GA，并且回滚预测键关联的所有改动。

    1.  回滚通过 `FPredictionKeyDelegates` 和 `FPredictionKey::NewRejectedDelegate/NewCaughtUpDelegate/NewRejectOrCaughtUpDelegate` 注册回调来完成。

    2. `ClientAbilityFailed` 是唯一拒绝预测键的地方，所以我们的当前所有的预测都依赖于GA是否激活。

6. 如果服务器GA执行成功，客户端需要等待属性同步将预测键的成功信息同步下来（执行成功的RPC会立即下发，但是属性同步可能会延迟）。一旦 `ReplicatedPredictionKey` 同步完成，客户端就可以回滚所有相关的预测改动了。 

    - 在 `FReplicatedPredictionKeyItem::OnRep` 中查看预测键的确认逻辑。

    - 在 `UAbilitySystemComponent::ReplicatedPredictionKeyMap` 中查看预测键实际是如何同步的。

    - 在 `~FScopedPredictionWindow` 中查看服务器是如果确认预测键的。

---

#### - **GE的预测逻辑**

GE跟随GA被一起预测，不会单独的确认或拒绝

1. GE只有在客户端有合法的预测键时才会被应用。

2. 如果GE被预测了，那么其相关的属性、GameplayCue、GameplayTag都会被一起预测。

3. 当一个 `FActiveGameplayEffect` 被创建的时候，会将预测键保存在里面 `FActiveGameplayEffect::PredictionKey`。

4. 服务器也会将预测键放到其创建的 `FActiveGameplayEffect` 中，并一起同步下来。

5. 当客户端收到包含合法的预测键的 `FActiveGameplayEffect` 时，会检查是否有相同预测键的对象存在，如果有则不会再次应用这个效果。这解决了预测的效果重复的问题。

6. 同时，`FReplicatedPredictionKeyItem::OnRep` 确认预测键时，被预测的效果会被移除，同时也会再次检查预测键来决定是否需要执行其移除逻辑。

`FActiveGameplayEffectsContainer::ApplyGameplayEffectSpec` 注册了预测键被确认时的逻辑。

在`FActiveGameplayEffect::PostReplicatedAdd`，`FActiveGameplayEffect::PreReplicatedRemove` 和 `FActiveGameplayCue::PostReplicatedAdd`中查看预测键是怎么与GE、GC关联上的。

---

#### - **属性预测逻辑**

为了解决属性预测和属性同步互相覆盖的问题，我们**预测改动而不是预测绝对值**。并且把瞬时的修改当作永久持续性的修改，来解决回滚的问题。

同时我们把服务器同步的值当作”基础值“而不是”最终值“，并且在同步完成后，重新计算”最终值“。

1. 在预测时，将瞬时修改当作永久持续性的修改来处理。见 `UAbilitySystemComponent::ApplyGameplayEffectSpecToSelf`。

2. 在属性同步下来时，**总是**调用 `RepNotify` 函数（而不是仅在值发生改变时调用，因为预测会提前改变属性）。使用`REPNOTIFY_Always`完成。

3. 在 `RepNofity` 函数中，调用 `AbilitySystemComponent::ActiveGameplayEffects` 来基于新的”基础值“更新”最终值“。使用 `GAMEPLAYATTRIBUTE_REPNOTIFY` 宏完成。

4. 其他的逻辑与GE同步类似：当预测键被确认时，预测的GE会被移除并回归到服务器下发的状态。

---
#### - **GC的预测逻辑**

GE中触发的GC已经包含在GE的预测逻辑中了。除此之外，GC也能单独使用，相关函数（`UAbilitySystemComponent::ExecuteGameplayCue` 等）也考虑了预测逻辑。

1. 在 `UAbilitySystemComponent::ExecuteGameplayCue` 函数中，如果是Authority的会直接广播，否则如果有合法的预测键，则会进行预测。

2. 在收到GC广播时，如果包含了合法的预测键，则不会执行，假设客户端之前已经预测过了。

---
#### - **Triggered Data预测逻辑**

Triggered Data 用于触发GA。其核心逻辑与 `ActivateAbility` 代码路径一致，区别在于GA并非由玩家输入触发，而是其他游戏逻辑事件驱动。客户端可以预测这些事件从而预测GA。

但是存在以下细节：

- 服务器不会等客户端通知，而是会同步的执行事件触发。服务器会维护一个通过预测触发过的GA列表。当收到触发的GA的 `TryActivate` 时，会检查是否已经有一个正在运行了，并返回相关信息。

- 回滚逻辑还没有处理。