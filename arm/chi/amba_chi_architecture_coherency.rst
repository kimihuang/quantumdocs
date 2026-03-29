================================================================================
AMBA® CHI 一致性协议深度分析
================================================================================

.. This document provides a comprehensive analysis of the AMBA CHI Coherence Protocol
.. Based on: AMBA CHI Architecture Specification (IHI0050G)
.. Document version: G
.. Date of issue: Mar 2024
.. Copyright © 2014, 2017-2024 Arm Limited or its affiliates. All rights reserved.

.. contents::
   :depth: 4
   :backlinks: none
   :local:

-------------------------------------------------------
一致性协议整体概述
-------------------------------------------------------

1 一致性协议的基本概念
--------------------------

硬件一致性使得系统组件能够共享内存，而无需软件缓存维护来保持一致性。

**一致性定义：**

如果两个组件对同一内存位置的写入可以被所有组件以相同的顺序观察到，则该内存区域是一致的。

**核心特性：**

1. **写使无效协议（Write-Invalidate Protocol）**
   - 请求节点向共享缓存行写入时，必须先使所有其他副本失效
   - CHI协议采用写使无效机制

2. **缓存行粒度**
   - 一致性以64字节缓存行为粒度进行维护
   - 所有缓存操作都在缓存行级别进行

3. **主内存更新策略**
   - 主内存仅在缓存行副本不再被任何缓存持有时才需要更新
   - 一致性协议不要求主内存始终是最新的
   - 允许在缓存副本仍存在时更新主内存

4. **多点一致性保证**
   - 一致性协议确保所有请求者在任何给定地址位置观察到正确的数据值
   - 每次存储操作后，其他请求者可以获得数据的新副本用于本地缓存

2 一致性点的定义
------------------

**关键概念：**

- **PoC（Point of Coherence）**：一致性点
  - 所有能够访问内存的代理保证看到相同内存位置副本的点
  - 在典型的CHI系统中，PoC是互连中的HN-F

- **PoS（Point of Serialization）**：串行化点
  - 互连中确定来自不同代理的请求之间顺序的点

- **PoE（Point of Encryption）**：加密点
  - 内存系统中到达该点的任何写入都被加密的点

- **PoP（Point of Persistence）**：持久化点
  - 内存系统中在或超出一致性点的点，此时对内存的写入在系统断电时被维护，并在恢复供电时可靠地恢复

3 一致性协议的核心目标
--------------------------

1. **确保数据一致性**
   - 保证所有代理对同一内存位置的视图一致
   - 维护内存访问的全局顺序

2. **优化性能**
   - 减少不必要的内存访问
   - 支持缓存间数据转发
   - 允许部分缓存行状态

3. **支持多种缓存模型**
   - MESI缓存模型
   - MOESI缓存模型
   - 支持从任何缓存状态转发数据

4. **提供灵活性**
   - 支持目录和监听过滤器系统
   - 允许实现特定的优化

-------------------------------------------------------
缓存状态模型深度解析
-------------------------------------------------------

1 缓存状态的基础特征
----------------------

每个缓存状态基于以下缓存行特征：

**2.1.1 有效性（Valid/Invalid）**

- **Valid（有效）**：缓存行存在于缓存中
- **Invalid（无效）**：缓存行不存在于缓存中

**2.1.2 唯一性与共享性（Unique/Shared）**

- **Unique（唯一）**：缓存行仅存在于当前缓存中
- **Shared（共享）**：缓存行可能存在于多个缓存中

**2.1.3 清洁性与脏状态（Clean/Dirty）**

- **Clean（清洁）**：缓存没有更新主内存的责任
- **Dirty（脏）**：缓存行相对于主内存已被修改，缓存必须确保主内存最终被更新

**2.1.4 完整性（Full/Partial/Empty）**

- **Full（完整）**：所有字节都有效
- **Partial（部分）**：部分字节有效
- **Empty（空）**：没有字节有效

2 七态缓存模型详解
--------------------

CHI协议定义了七种缓存状态：

**2.2.1 I（Invalid）- 无效状态**

```
定义：缓存行不存在于缓存中
特性：
  - 不占用缓存资源
  - 不需要维护一致性
  - 任何访问都需要从内存或其他缓存获取数据
```

**2.2.2 UC（Unique Clean）- 唯一清洁**

```
定义：缓存行仅存在于当前缓存，且未被修改
特性：
  - 缓存行唯一存在于此缓存
  - 相对于内存未被修改
  - 可以修改而无需通知其他缓存
  - 响应请求数据的监听时：
    * 被请求时可以返回到Home
    * 被指示时可以直接转发给请求者
用途：
  - 读取操作的最终状态
  - 写入操作的准备状态
```

**2.2.3 UCE（Unique Clean Empty）- 唯一清洁空**

```
定义：缓存行处于唯一状态但没有有效的数据字节
特性：
  - 缓存行仅存在于当前缓存
  - 处于唯一状态但数据字节均无效
  - 可以修改而无需通知其他缓存
  - 响应请求数据的监听时：
    * 即使被请求也不能返回到Home
    * 即使被指示也不能直接转发给请求者
使用场景：
  - 提前获取写入权限以节省系统带宽
  - 请求者预期写入缓存行前获得空缓存行
```

**2.2.4 UD（Unique Dirty）- 唯一脏**

```
定义：缓存行仅存在于当前缓存，且已被修改
特性：
  - 缓存行唯一存在于此缓存
  - 相对于内存已被修改
  - 驱逐时必须写回到下一级缓存或内存
  - 可以修改而无需通知其他缓存
  - 响应请求数据的监听时：
    * 被请求时必须返回到Home
    * 被指示时期望直接转发给请求者
用途：
  - 写入操作的最终状态
  - 需要维护数据的最新版本
```

**2.2.5 UDP（Unique Dirty Partial）- 唯一脏部分**

```
定义：缓存行唯一，但部分字节有效
特性：
  - 缓存行唯一
  - 部分字节有效（可能包括无或全部字节）
  - 相对于内存已被修改
  - 驱逐时必须与下一级缓存或内存的数据合并
  - 可以修改而无需通知其他缓存
  - 响应请求数据的监听时：
    * 必须返回到Home
    * 即使被指示也不能直接转发给请求者
使用场景：
  - 部分写入操作
  - 需要与内存数据合并的缓存行
```

**2.2.6 SC（Shared Clean）- 共享清洁**

```
定义：其他缓存可能有该缓存行的共享副本
特性：
  - 可能相对于内存已被修改
  - 缓存没有责任在驱逐时写回缓存行
  - 修改前必须使共享副本失效并获取唯一所有权
  - 响应请求数据的监听时：
    * 如果RetToSrc位未设置，要求不返回数据
    * 如果RetToSrc位设置，期望返回数据
    * 被指示时期望直接转发给请求者
用途：
  - 只读数据共享
  - 多个处理器读取相同数据
```

**2.2.7 SD（Shared Dirty）- 共享脏**

```
定义：其他缓存可能有该缓存行的共享副本
特性：
  - 相对于内存已被修改
  - 驱逐时必须写回到下一级缓存或内存
  - 修改前必须使共享副本失效并获取唯一所有权
  - 响应请求数据的监听时：
    * 被请求时必须返回到Home
    * 被指示时期望直接转发给请求者
用途：
  - 共享修改的数据
  - 需要写回的共享缓存行
```

3 缓存状态转换规则
--------------------

**2.3.1 静默转换（Silent Transitions）**

缓存可以由于内部事件而改变状态，无需通知系统其余部分：

```
允许的静默转换：

1. 缓存驱逐（Cache Eviction）
   - UC → I
   - UCE → I
   - SC → I

2. 本地共享（Local Sharing）
   - UC → SC
   - UD → SD
   说明：RN-F将唯一缓存行指定为共享，实际上忽略了缓存行对RN-F仍保持唯一的事实

3. 存储操作（Store Operations）
   - UC → UD（完整或部分缓存行存储）
   - UCE → UDP（部分缓存行存储）
   - UD → UD（完整缓存行存储）
   - UDP → UD（填充缓存行的存储）

4. 缓存失效（Cache Invalidate）
   - UD → I
   - UDP → I
```

**2.3.2 转换约束**

- UC → UCE 转换不被允许
- 静默转换序列可以发生
- 导致缓存行处于UD、UDP或SC状态的任何静默转换可以进一步进行静默转换

4 空缓存行所有权
------------------

**定义：**
空缓存行是以唯一状态持有的缓存行，用于防止缓存行的其他副本存在。空缓存行中没有有效的数据字节。

**使用场景：**

1. **节省系统带宽**
   - 请求者在开始写入之前故意获取空缓存行
   - 期望写入缓存行的请求者可以获取具有存储权限的空缓存行，而不是获取缓存行的有效副本

2. **权限请求期间的转换**
   - 当请求存储权限时，请求者如果有缓存行副本可以转换到空状态
   - 在请求完成之前，该缓存行副本被失效
   - 结果是请求者具有存储权限的空缓存行

**空缓存行状态：**
- UCE（Unique Clean Empty）
- UDP（Unique Dirty Partial）

5 部分脏数据所有权
------------------

**工作原理：**
一旦获得不带数据的缓存行所有权，请求者被允许（但不要求）存储到缓存行。

**状态转换：**
- 如果请求者修改缓存行的部分，缓存行保持部分唯一脏状态
- 此缓存行状态为UDP（Unique Dirty Partial）

**处理要求：**
- 驱逐时必须与下一级缓存或内存的数据合并
- 以形成完整的有效缓存行

-------------------------------------------------------
请求类型详解
-------------------------------------------------------

1 请求分类
-----------

CHI协议请求按以下方式分类：

1. **读请求（Read Transactions）**
   - 向请求者提供数据响应

2. **无数据请求（Dataless Transactions）**
   - 不向请求者提供数据响应

3. **写请求（Write Transactions）**
   - 数据从请求者移动到完成者

4. **组合写请求（Combined Write Transactions）**
   - 数据从请求者移动并执行缓存维护操作

5. **原子请求（Atomic Transactions）**
   - 数据从请求者移动，某些请求类型向请求者提供数据响应

6. **填充请求（Stash Transactions）**
   - 数据在系统中移动以提高性能

7. **其他请求（Other Transactions）**
   - 不涉及系统中的数据移动
   - 用于辅助DVM维护
   - 用于预热后续读请求的内存控制器

2 读请求深入分析
------------------

**3.2.1 读请求的共同特性**

1. **数据响应要求**
   - 必须在完成响应中包含数据（MakeReadUnique除外）
   - MakeReadUnique的数据响应是可选的

2. **数据来源**
   - 可以来自Home节点、另一个请求节点或从属节点

3. **缓存行为**
   - 分配读事务：接收的数据如果缓存，必须以系统一致方式缓存
   - 非分配读事务（ReadNoSnp和ReadOnce*）：接收的数据不预期被缓存

4. **缓存状态变化**
   - 可以导致请求者的缓存状态变化
   - 可以导致系统中其他请求节点的缓存状态变化

**3.2.2 具体读请求类型**

**ReadNoSnp（非监听读）**
```
用途：请求节点对非监听地址区域的读请求
特性：
  - 或从Home节点到任何地址区域以获取所寻址数据的副本
  - 不触发监听操作
  - 数据不要求以一致方式缓存
初始状态：I
最终状态：I
```

**ReadNoSnpSep（非监听读-分离响应）**
```
用途：从Home节点到从属节点的读请求
特性：
  - 请求完成者仅发送数据响应
  - 用于使用分离的完成和数据响应来完成读事务
初始状态：I
最终状态：I
```

**ReadOnce（一次性读）**
```
用途：对监听地址区域的读请求，获取一致性数据的快照
特性：
  - 获取数据的一致性快照
  - 不强制其他缓存副本的状态变化
  - 适用于临时读取数据
初始状态：I
最终状态：I
```

**ReadOnceCleanInvalid（读-清理-失效）**
```
用途：对监听地址区域的读请求，获取一致性数据的快照
特性：
  - 建议（但不要求）清理并使其他缓存行的缓存副本失效
  - 如果脏副本被失效，必须写回内存
  - 用于数据仍有效但近期不使用的情况
  - 改善缓存效率，减少缓存污染
初始状态：I
最终状态：I
注意事项：
  - 清理和失效是提示，事务完成不保证清理或移除所有缓存副本
  - 不能用作CMO的替代
  - 可能导致缓存行释放，如果事务可能被其他代理用于独占访问需谨慎
```

**ReadOnceMakeInvalid（读-使失效）**
```
用途：对监听地址区域的读请求，获取一致性数据的快照
特性：
  - 建议（但不要求）使其他缓存行的缓存副本失效
  - 如果脏副本被失效，缓存行不需要写回内存
  - 如果失效提示被接受且脏副本未写回内存，所有缓存副本必须被失效
  - 用于确定缓存数据不会被再次使用的情况
  - 释放缓存并丢弃脏数据，避免不必要的WriteBack
初始状态：I
最终状态：I
注意事项：
  - 失效是提示，不能保证移除所有缓存副本
  - 不能用作CMO的替代
  - 可能导致脏缓存行丢失，必须严格限制在已知且无害的场景
  - 要求缓存行失效在读数据响应之前提交
  - 数据必须以I、UC或UD状态提供给请求者
  - 请求者必须忽略响应中的缓存状态
```

**ReadClean（读清洁）**
```
用途：对监听地址区域的读请求，获取缓存行的清洁副本
特性：
  - 用于将缓存行分配到不支持脏缓存行的缓存（如指令缓存）
  - 数据必须仅以UC或SC状态提供给请求者
初始状态：I（TagOp=Transfer时可为UC, UCE, SC, SD）
最终状态：SC
```

**ReadNotSharedDirty（读非共享脏）**
```
用途：对监听地址区域的读请求，执行缓存行加载
特性：
  - 数据必须仅以UC、UD或SC状态提供给请求者
  - 不允许SD状态
  - 当请求者不能接受SD状态数据时使用
初始状态：I, UCE
最终状态：SC
```

**ReadShared（读共享）**
```
用途：对监听地址区域的读请求，执行缓存行加载
特性：
  - 数据必须以UC、UD、SC或SD状态提供给请求者
  - 当请求者能够接受SD状态数据时使用
初始状态：I, UCE
最终状态：SC
```

**ReadUnique（读唯一）**
```
用途：对监听地址区域的读请求，执行缓存行存储
特性：
  - 数据必须仅以UC或UD状态提供给请求者
  - 获取唯一副本以便进行修改
初始状态：I, SC, UC, UCE, UD, SD
最终状态：UC或UD
```

**ReadPreferUnique（读首选唯一）**
```
用途：对监听地址区域的读请求，请求缓存行的唯一副本
特性：
  - 请求者偏好（但不要求）数据以唯一状态返回
  - 如果另一个请求节点当前正在使用相同地址执行独占序列，则以共享状态提供数据
  - 允许始终以共享状态向请求者提供数据
  - 用于提高独占序列的执行效率
初始状态：I, UC, UCE, UD, SD, SC
最终状态：SC或UC/UD
```

**MakeReadUnique（使读唯一）**
```
用途：对监听地址区域的读请求，请求缓存行的唯一副本
特性：
  - 典型用法：请求者有缓存行的共享副本，想要获取存储到缓存行的权限
  - 如果请求者收到失效监听且需要保留数据，则保证数据返回
  - 永远不需要重新发送请求以获取缓存行的唯一副本
初始状态：SC（请求者有共享副本时）
最终状态：UC或UD或SC（独占版本）
```

**3.2.3 读请求属性值**

关键属性包括：
- **Size**：请求大小（字节）
- **Excl**：独占属性
- **SnpAttr**：监听属性
- **MemAttr**：内存属性（A、C、D、E位）
- **Order**：排序要求
- **LikelyShared**：可能共享
- **ExpCompAck**：期望完成确认

3 无数据请求深入分析
----------------------

**3.3.1 无数据请求分类**

1. **所有权请求**
   - CleanUnique：将缓存状态更改为唯一以执行存储
   - MakeUnique：获取缓存行所有权且无数据响应

2. **驱逐请求**
   - Evict：指示清洁缓存行不再被请求节点缓存

3. **独立填充请求**
   - StashOnceUnique/StashOnceSepUnique：尝试将缓存行移动到目标缓存以支持存储
   - StashOnceShared/StashOnceSepShared：尝试将缓存行移动到目标缓存

4. **缓存维护事务（CMO）**
   - CleanShared：确保所有缓存副本更改为非脏状态
   - CleanSharedPersist：确保脏缓存副本写回到持久化点
   - CleanSharedPersistSep：支持分离的Comp和Persist响应
   - CleanInvalid：确保所有缓存副本失效
   - CleanInvalidPoPA：确保PoPA之前的所有缓存副本失效
   - MakeInvalid：确保所有缓存副本失效，允许丢弃脏副本

**3.3.2 CleanUnique（清理唯一）**
```
用途：对监听地址区域的请求，将请求者的缓存状态更改为唯一以执行存储
特性：
  - 典型用法：请求者有缓存行的共享副本，想要获取存储到缓存行的权限
  - 被监听缓存中的任何脏副本必须写回内存
初始状态：UC, UCE, SC, SD
最终状态：UC, UCE, UD
```

**3.3.3 MakeUnique（使唯一）**
```
用途：对监听地址区域的请求，获取缓存行所有权且无数据响应
特性：
  - 仅在请求者保证将存储到缓存行的所有字节时使用
  - 被监听缓存中的任何脏副本必须在不执行数据传输的情况下失效
初始状态：SC, SD, I
最终状态：UD
```

**3.3.4 Evict（驱逐）**
```
用途：指示清洁缓存行不再被请求节点缓存
特性：
  - 用于释放缓存资源
  - 不涉及数据移动
初始状态：I
最终状态：I
```

**3.3.5 填充请求**

**StashOnceUnique（填充一次唯一）**
```
用途：尝试将寻址缓存行移动到目标缓存以使目标能够存储该行
特性：
  - 包括另一个请求节点的有效节点ID作为填充目标
  - 包括可选的LPID
  - 建议（但不要求）监听其他代理
  - 目标请求节点的DataPull请求被视为ReadUnique请求
初始状态：I
最终状态：I
```

**StashOnceShared（填充一次共享）**
```
用途：尝试将寻址缓存行移动到目标缓存
特性：
  - 包括另一个请求节点的节点ID
  - 可以包括该节点内的LPID
  - 目标请求节点的DataPull请求被视为ReadNotSharedDirty请求
初始状态：I
最终状态：I
```

**3.3.6 缓存维护操作（CMO）详解**

**CleanShared（清理共享）**
```
完成响应确保：
  - 所有缓存副本更改为非脏状态
  - 任何脏副本写回内存
特性：
  - 用于确保数据持久化
  - 支持软件缓存管理
初始状态：SC, UC, I
最终状态：无变化（缓存状态字段被忽略）
```

**CleanSharedPersist（清理共享持久化）**
```
完成响应确保：
  - 所有缓存副本更改为非脏状态
  - 任何脏缓存副本写回到持久化点（PoP）
特性：
  - 用于非易失性内存环境
  - 确保数据在断电后可恢复
初始状态：SC, UC, I
最终状态：无变化
```

**CleanSharedPersistSep（清理共享持久化-分离）**
```
完成响应确保：
  - 所有缓存副本更改为非脏状态
  - 任何脏缓存副本写回到PoP
特性：
  - 功能类似于CleanSharedPersist但允许两个分离的响应
  - 当发送PCMO时，预期（但不要求）请求者使用CleanSharedPersistSep事务
  - 必须支持接收分离的Comp和Persist响应以及组合的CompPersist响应
初始状态：SC, UC, I
最终状态：无变化
```

**CleanInvalid（清理失效）**
```
完成响应确保：
  - 所有缓存副本失效
  - 要求任何缓存脏副本必须写回内存
特性：
  - 用于确保所有缓存中的数据被移除
初始状态：I
最终状态：I
```

**CleanInvalidPoPA（清理失效-PoPA）**
```
完成响应确保：
  - PoPA之前的所有缓存副本失效
  - 要求任何缓存脏副本必须写回到PoPA之外
特性：
  - 使一个PAS中位置的写入对其他物理地址空间可见
  - 可能需要其他物理地址空间中的额外缓存维护操作
初始状态：I
最终状态：I
```

**MakeInvalid（使失效）**
```
完成响应确保：
  - 所有缓存副本失效
  - 允许丢弃任何缓存脏副本
特性：
  - 不要求写回脏数据
  - 可以更快的失效操作
初始状态：I
最终状态：I
```

**CMO的共同特性：**

1. **响应字段处理**
   - Comp中的Resp字段值指示缓存状态，必须被请求者和Home忽略
   - Home和Subordinate之间的MemAttr值必须保留
   - Order字段不能断言

2. **事务排序**
   - 针对特定地址的CMO必须在之前可分配数据的所有先前事务完成后发送
   - 可分配数据的事务必须在之前的CMO完成后发送

3. **完成者行为**
   - 完成者被允许（但不要求）等待WriteData以发送Persist响应
   - 不支持持久化的内存位置的早期Persist示例

4. **传播要求**
   - CMO必须传播到所有基于可缓存性区分的缓存
   - CMO必须对以相同地址和相同PAS缓存的所有缓存行操作

5. **组合能力**
   - CMO被允许（但不要求）与相同地址的写事务组合

6. **Nonshareable_Cache_Maint属性**
   - 如果为True，CMO必须应用于页表中标记为可缓存的位置
   - 无论标记为Non-shareable还是Shareable
   - 必须传播到任何基于共享性区分其条目的缓存

**3.3.7 持久化CMO处理**

**Home处的持久化点**
```
如果Home是PoP，对于CleanSharedPersistSep：
  - Home被允许不将CleanSharedPersistSep请求转发到Subordinate
  - 当Home决定不转发时，必须向请求者返回Persist响应
  - 当Home发送Persist响应时，被允许（但不要求）与Comp组合
```

**Home下游的持久化点**
```
如果PoP在Home下游，对于CleanSharedPersistSep：
  - Home必须将请求发送到下游
  - Subordinate节点必须在保证请求被接受后仅返回Comp响应
  - 不返回RetryAck响应
  - Subordinate能够保证同一地址的所有先前写入到非易失性内存是持久化的
  - Subordinate必须返回Persist响应
  - Home接收到来自下游的Persist响应
  - Subordinate被允许（但不要求）返回组合的CompPersist响应
  - Home被允许（但不要求）等待Subordinate的Persist响应
```

**深度持久化CMO**
```
用于高可用性期望：
  - 系统需要保证关键操作数据即使在电源和备用电池同时失败时也能保留
  - 通过添加机制将先前的写入推送到PoDP来提供保证
  - 由PCMO事务上的Deep属性支持
```

4 写请求深入分析
------------------

**3.4.1 写请求分类**

1. **立即事务（Immediate Transactions）**
   - 也称为非CopyBack写事务
   - 从请求节点向Home节点传输数据，无需先获得数据的一致性所有权
   - 用于从Home节点向Subordinate节点传输数据

2. **CopyBack事务**
   - CopyBack事务移动一致数据从缓存到下一级缓存或内存
   - 不需要监听系统中的其他代理

**3.4.2 立即写请求**

**WriteNoSnpFull（非监听写-完整）**
```
用途：从请求节点向非监听地址区域写完整缓存行数据
特性：
  - 或从Home到任何地址区域的完整缓存行数据
  - 所有BE位必须断言
初始状态：I
最终状态：I
```

**WriteNoSnpPtl（非监听写-部分）**
```
用途：从请求节点向非监听地址区域写最多一个缓存行数据
特性：
  - 或从Home到任何地址区域的最多一个缓存行数据
  - BE位必须在指定数据大小的适当字节通道断言
  - 数据传输的其余部分取消断言
初始状态：I
最终状态：I
```

**WriteNoSnpDef（非监听写-可延迟）**
```
用途：从请求节点向非监听地址区域可延迟地写完整缓存行数据
特性：
  - 或从Home到任何地址区域的完整缓存行数据
  - 所有BE位必须断言
  - 允许多个可延迟写事务从同一请求者未完成
初始状态：I
最终状态：I
```

**WriteNoSnpZero（非监听写-零）**
```
用途：从请求节点向非监听地址区域写数据值为0，不传输数据字节
特性：
  - 或从Home到任何地址区域写数据值为0，不传输数据字节
初始状态：I
最终状态：I
```

**WriteUniqueFull（写唯一-完整）**
```
用途：对监听地址区域的写操作
特性：
  - 当缓存行在请求者处为Invalid时，将完整缓存行数据写入下一级缓存或内存
  - 所有BE位必须断言
初始状态：I
最终状态：I
```

**WriteUniquePtl（写唯一-部分）**
```
用途：对监听地址区域的写操作
特性：
  - 当缓存行在请求者处为Invalid时，写最多一个缓存行数据到下一级缓存或内存
  - BE位必须在指定数据大小的适当字节通道断言
  - 数据传输的其余部分取消断言
初始状态：I
最终状态：I
```

**WriteUniqueZero（写唯一-零）**
```
用途：对监听地址区域的写操作
特性：
  - 当数据值为0时写操作，不传输数据字节
初始状态：I
最终状态：I
```

**WriteUniqueFullStash（写唯一-完整-填充）**
```
用途：对监听地址区域的写操作
特性：
  - 当缓存行在请求者处为Invalid时，将完整缓存行数据写入下一级缓存或内存
  - 还包括向填充目标节点获取寻址缓存行的请求
  - 所有BE位必须断言
初始状态：I
最终状态：I
```

**WriteUniquePtlStash（写唯一-部分-填充）**
```
用途：对监听地址区域的写操作
特性：
  - 当缓存行在请求者处为Invalid时，写最多一个缓存行数据到下一级缓存或内存
  - 还包括向填充目标节点获取寻址缓存行的请求
  - BE位必须在指定数据大小的适当字节通道断言
初始状态：I
最终状态：I
```

**3.4.3 CopyBack写请求**

**WriteBackFull（写回-完整）**
```
用途：写回完整缓存行的脏数据到下一级缓存或内存
特性：
  - 所有BE位必须断言（当写数据不是CopyBackWriteData_I时）
初始状态：UD, SD
最终状态：I
```

**WriteBackPtl（写回-部分）**
```
用途：写回最多一个缓存行的脏数据到下一级缓存或内存
特性：
  - 所有适当的BE位（最多全部64）必须断言
初始状态：UD
最终状态：I
```

**WriteCleanFull（写清洁-完整）**
```
用途：写回完整缓存行的脏数据到下一级缓存或内存，并在缓存中保留清洁副本
特性：
  - 所有BE位必须断言（当写数据不是CopyBackWriteData_I时）
初始状态：UD, SD
最终状态：UC或SC
```

**WriteEvictFull（写驱逐-完整）**
```
用途：UniqueClean数据的写回到下一级缓存
特性：
  - 所有BE位必须断言（当写数据不是CopyBackWriteData_I时）
  - 缓存行必须不传播超出其监听域
初始状态：UC
最终状态：I
```

**WriteEvictOrEvict（写驱逐或驱逐）**
```
用途：清洁数据的写回到下一级缓存
特性：
  - WriteEvictFull和Evict合并为一个请求
  - Home节点被允许确定是否发送数据
  - 如果完成者不接受数据，可能不发送数据
  - 如果发送数据，则数据大小为缓存行长度
  - LikelyShared值指示初始状态：
    * 0 → UC
    * 1 → SC
初始状态：UC或SC
最终状态：I
```

**3.4.4 写请求特性**

1. **字节使能（BE）**
   - 每个写事务必须断言适当的BE位与数据
   - BE位指示哪些字节通道包含有效数据

2. **DWT（直接写传输）流**
   - 立即写请求，除WriteNoSnpZero和WriteUniqueZero外，允许DWT流
   - WriteNoSnpZero和WriteUniqueZero的DWT流从不允许

3. **缓存状态**
   - 立即写事务的初始状态为I
   - CopyBack事务有特定的初始状态要求

5 原子请求深入分析
--------------------

**3.5.1 原子操作概念**

**原子操作（Atomic Operation）**
```
定义：执行涉及多个数据值的函数，使得原始值的加载、函数的执行和更新值的存储以原子方式发生
含义：在操作期间没有其他代理能够访问该位置
```

**原子事务（Atomic Transaction）**
```
定义：用于将原子操作连同执行原子操作所需的数据值从一个系统中的代理传递到另一个代理的事务
目的：原子操作可以由需要执行操作的组件之外的组件执行
```

**3.5.2 原子事务类型**

CHI协议定义了四种原子事务类型：

**AtomicStore（原子存储）**
```
特性：
  - 发送单个数据值、地址和要执行的原子操作
  - 目标（Home节点或Subordinate节点）使用原子事务中提供的数据对指定地址位置执行所需操作
  - 目标返回不带数据的完成响应
  - 不向请求者返回地址位置的原始值
  - 支持8种操作
  - 适用于1字节、2字节、4字节或8字节数据大小

支持的操作：
  1. STADD: (TxnData + InitialData)
  2. STCLR: (InitialData AND (NOT TxnData))
  3. STEOR: (InitialData XOR TxnData)
  4. STSET: (InitialData OR TxnData)
  5. STSMAX: 如果((有符号INT)TxnData - (有符号INT)InitialData) > 0
  6. STSMIN: 如果((有符号INT)TxnData - (有符号INT)InitialData) < 0
  7. STUMAX: 如果((无符号INT)TxnData - (无符号INT)InitialData) > 0
  8. STUMIN: 如果((无符号INT)TxnData - (无符号INT)InitialData) < 0

初始状态：任何状态
最终状态：I
```

**AtomicLoad（原子加载）**
```
特性：
  - 发送单个数据值、地址和要执行的原子操作
  - 目标使用原子事务中提供的数据值对指定地址位置执行所需操作
  - 目标返回带数据的完成响应
  - 数据值是地址位置的原始值
  - 支持8种操作
  - 适用于1字节、2字节、4字节或8字节数据大小

支持的操作：
  1. LDADD: (TxnData + InitialData)
  2. LDCLR: (InitialData AND (NOT TxnData))
  3. LDEOR: (InitialData XOR TxnData)
  4. LDSET: (InitialData OR TxnData)
  5. LDSMAX: 如果((有符号INT)TxnData - (有符号INT)InitialData) > 0
  6. LDSMIN: 如果((有符号INT)TxnData - (有符号INT)InitialData) < 0
  7. LDUMAX: 如果((无符号INT)TxnData - (无符号INT)InitialData) > 0
  8. LDUMIN: 如果((无符号INT)TxnData - (无符号INT)InitialData) < 0

初始状态：任何状态
最终状态：I
```

**AtomicSwap（原子交换）**
```
特性：
  - 发送单个数据值（交换值）连同要操作的地址
  - 目标将地址位置的值与事务中提供的数据值交换
  - 目标返回带数据的完成响应
  - 数据值是地址位置的原始值
  - 支持1种操作
  - 适用于1字节、2字节、4字节或8字节数据大小

初始状态：任何状态
最终状态：I
```

**AtomicCompare（原子比较）**
```
特性：
  - 发送两个数据值（比较值和交换值）连同要操作的地址
  - 目标将地址位置的值与比较值比较：
    * 如果值匹配，目标将交换值写入地址位置
    * 如果值不匹配，目标不将交换值写入地址位置
  - 目标返回带数据的完成响应
  - 数据值是地址位置的原始值
  - 支持1种操作
  - 适用于2字节、4字节、8字节、16字节或32字节数据大小

初始状态：任何状态
最终状态：I
```

**3.5.3 原子事务的共同特性**

1. **数据响应**
   - 必须在完成响应中包含数据（AtomicStore除外）
   - AtomicStore的完成响应不包含数据响应
   - 返回数据时，入站数据大小必须与出站数据大小相同（AtomicCompare除外）
   - AtomicCompare中，入站数据大小必须是出站数据大小的一半
   - 响应数据中的数据值必须是地址位置的原始值
   - 接收的数据不能在请求者处缓存

2. **字节使能**
   - 出站数据消息的所有有效数据的BE位必须断言
   - 入站数据的BE位不适用，可以采用任何值

3. **缓存状态变化**
   - 请求可以导致系统中其他请求节点的缓存状态变化
   - 对等请求节点缓存状态在请求完成时必须为Invalid

4. **数据流**
   - 原子事务不能使用DMT或DWT流

**3.5.4 原子操作的本地执行**

如果请求者必须在缓存行的缓存副本上执行原子操作：

**缓存行状态为Unique时：**
```
- 可以在本地执行原子操作，无需生成原子事务
```

**缓存行状态为Shared但非Dirty时：**
```
选项1：
  - 生成ReadUnique、CleanUnique或MakeReadUnique以获取缓存行所有权
  - 在本地执行原子操作

选项2：
  - 使本地副本失效
  - 向互连发送原子事务
```

**缓存行状态为Shared Dirty时：**
```
选项1：
  - 生成ReadUnique、CleanUnique或MakeReadUnique
  - 获取缓存行所有权
  - 在本地执行操作

选项2：
  - WriteBack并使本地副本失效
  - 随后向互连发送原子事务
```

**所有情况下的可选行为：**
```
- 请求者被允许（但不要求）发送带有SnoopMe位设置的原子事务
- 以指示互连向请求者发送监听请求以失效（必要时提取缓存副本）
```

6 组合写请求深入分析
----------------------

**3.6.1 组合写请求概念**

组合写请求允许将写事务与缓存维护事务（CMO）组合，当两者都针对相同地址时。

**优势：**
```
1. 避免需要序列化写和CMO/PCMO事务
2. 减少事务开销
3. 提高性能效率
4. 在必须完成写操作才能启动CMO/PCMO事务的点特别有用
```

**3.6.2 允许的组合**

**Request Node到Home Node的组合：**

.. list-table:: Request Node到Home Node的组合
   :widths: 25 25 25 25 10
   :header-rows: 1

   * - 写类型
     - PCMO（分离持久响应）
     - 非持久CMO（CleanShared）
     - 非持久CMO（CleanInvalid/CleanInvalidPoPA）
     - 非持久CMO（MakeInvalid）
   * - WriteNoSnpFull
     - Y
     - Y
     - Y
     - -
   * - WriteNoSnpPtl
     - -
     - -
     - -
     - -
   * - WriteNoSnpDef
     - -
     - -
     - -
     - -
   * - WriteUniqueFull
     - Y
     - Y
     - -
     - -
   * - WriteUniquePtl
     - -
     - -
     - -
     - -
   * - WriteUniqueStashFull
     - -
     - -
     - -
     - -
   * - WriteUniqueStashPtl
     - -
     - -
     - -
     - -
   * - WriteBackFull
     - Y
     - Y
     - Y
     - -
   * - WriteBackPtl
     - -
     - -
     - -
     - -
   * - WriteCleanFull
     - Y
     - Y
     - -
     - -
   * - WriteEvictFull
     - -
     - -
     - -
     - -

**Home Node到Subordinate Node的组合：**

.. list-table:: Home Node到Subordinate Node的组合
   :widths: 25 25 25 25 10
   :header-rows: 1

   * - 写类型
     - PCMO（分离持久响应）
     - 非持久CMO（CleanShared）
     - 非持久CMO（CleanInvalid/CleanInvalidPoPA）
     - 非持久CMO（MakeInvalid）
   * - WriteNoSnpFull
     - Y
     - Y
     - Y
     - -
   * - WriteNoSnpPtl
     - -
     - -
     - -
     - -
   * - WriteNoSnpDef
     - -
     - -
     - -
     - -

**3.6.3 具体组合写请求**

**WriteNoSnpFull的组合：**
```
- WriteNoSnpFullCleanInv
- WriteNoSnpFullCleanSh
- WriteNoSnpFullCleanShPerSep
- WriteNoSnpFullCleanInvPoPA
```

**WriteNoSnpPtl的组合：**
```
- WriteNoSnpPtlCleanInv
- WriteNoSnpPtlCleanSh
- WriteNoSnpPtlCleanShPerSep
- WriteNoSnpPtlCleanInvPoPA
```

**WriteUniqueFull的组合：**
```
- WriteUniqueFullCleanSh
- WriteUniqueFullCleanShPerSep
```

**WriteUniquePtl的组合：**
```
- WriteUniquePtlCleanSh
- WriteUniquePtlCleanShPerSep
```

**WriteBackFull的组合：**
```
- WriteBackFullCleanInv
- WriteBackFullCleanSh
- WriteBackFullCleanShPerSep
- WriteBackFullCleanInvPoPA
```

**WriteCleanFull的组合：**
```
- WriteCleanFullCleanSh
- WriteCleanFullCleanShPerSep
```

**3.6.4 组合写请求的特性**

1. **适用性**
   - 允许用于CopyBack和Immediate写
   - WriteNoSnp(Excl)不允许与CMO组合
   - TagOp设置为Match不允许在Write*CMO中使用

2. **Home行为**
   - Home接收来自请求节点的组合写请求时，被允许向Subordinate节点转发
   - 如果写和CMO/PCMO都需要发送到下游
   - Home也被允许对这样的写请求使用DWT
   - 如果写是Immediate写且ExpCompAck设置为0

3. **分离处理**
   - 组合写请求的接收者被允许分离写和CMO请求并分别处理
   - 在这种情况下，CMO请求必须排序在写之后
   - 一旦请求被分离，完成者被允许在写和CMO事务之间插入对相同地址的请求

4. **字段值**
   - 组合请求中的Size字段值对应于写请求中的数据大小
   - CMO始终在64字节粒度上
   - 组合写请求中的MemAttr和SnpAttr字段值对应于写请求的内存属性
   - 当写和CMO事务被分离时，写事务继承原始组合请求的MemAttr和SnpAttr值
   - 分离的CMO事务的SnpAttr和Cacheable位必须设置为最广泛

**3.6.5 使用场景示例**

1. **WriteBack与CleanShared/CleanSharedPersist组合**
   - 支持在由CMO清理时将缓存行转换为Invalid的请求节点
   - 无论CMO类型如何

2. **WriteUnique与CleanShared组合**
   - 当CleanSharedPersist目标已知不支持持久化事务时
   - CleanSharedPersist请求需要由请求者转换为CleanShared

-------------------------------------------------------
监听请求详解
-------------------------------------------------------

1 监听请求概述
----------------

**生成时机：**
```
1. 响应来自请求节点的请求
2. 由于内部触发，如缓存或监听过滤器维护操作
```

**操作目标：**
```
1. SnpDVMOp之外的所有Snoop事务在RN-F的缓存数据上操作
2. SnpDVMOp事务在目标节点执行DVM维护操作
```

**选择标准：**
Home对监听的选择基于多个标准：

1. **请求要求的预期或允许的最终缓存状态**
   - 在请求者和被监听节点处

2. **避免丢失被监听缓存中存在的任何脏标签**

3. **用等效的转发监听替换非转发监听**（如果存在）

4. **转发监听仅允许发送到一个RN-F**

5. **填充监听仅允许发送到一个RN-F**

6. **允许（但不要求）对非监听地址位置进行监听**

2 监听请求类型详解
--------------------

**4.2.1 SnpOnceFwd / SnpOnce（一次性转发监听）**
```
用途：获取缓存行的最新副本，最好不更改被监听者处的缓存行状态
特性：
  - 非失效监听
  - 可以转发数据
  - 用于ReadOnce请求
```

**4.2.2 SnpStashUnique（填充唯一监听）**
```
用途：建议被监听者获取缓存行的唯一状态副本
特性：
  - 期望在填充目标的缓存处于唯一状态时不发送监听
  - 对于WriteUniqueFullStash和WriteUniquePtlStash：
    * 仅当被监听者没有缓存行副本时，才允许向填充目标发送监听
  - 被监听者不得在监听响应中返回数据
  - 允许监听响应包括Data Pull
  - 监听响应中的Data Pull请求被视为ReadUnique
  - 不得更改被监听者处的缓存行状态
```

**4.2.3 SnpStashShared（填充共享监听）**
```
用途：建议被监听者获取缓存行的共享状态副本
特性：
  - 期望在目标处缓存缓存行时不发送监听
  - 被监听者不得在监听响应中返回数据
  - 允许监听响应包括Data Pull
  - 监听响应中的Data Pull请求被视为ReadNotSharedDirty
  - 不得更改被监听者处的缓存行状态
```

**4.2.4 SnpCleanFwd / SnpClean（清理转发监听）**
```
用途：获取清洁状态的缓存行副本，同时将任何缓存副本保留在共享状态
特性：
  - 必须不将缓存行留在唯一状态
  - 非失效监听
  - 可以转发数据
```

**4.2.5 SnpNotSharedDirtyFwd / SnpNotSharedDirty（非共享脏转发监听）**
```
用途：获取共享清洁状态的缓存行副本，同时将任何缓存副本保留在共享状态
特性：
  - 必须不将缓存行留在唯一状态
  - 非失效监听
  - 可以转发数据
```

**4.2.6 SnpSharedFwd / SnpShared（共享转发监听）**
```
用途：获取共享状态的缓存行副本，同时将任何缓存副本保留在共享状态
特性：
  - 必须不将缓存行留在唯一状态
  - 非失效监听
  - 可以转发数据
```

**4.2.7 SnpUniqueFwd / SnpUnique（唯一转发监听）**
```
用途：获取唯一状态的缓存行副本，同时使任何缓存副本失效
特性：
  - 必须将缓存行更改为Invalid状态
  - 失效监听
  - 可以转发数据
```

**4.2.8 SnpPreferUniqueFwd / SnpPreferUnique（首选唯一转发监听）**
```
用途：获取唯一状态的缓存行副本，同时使任何缓存副本失效
特性：
  - Home预期使用SnpPreferUniqueFwd或SnpPreferUnique响应ReadPreferUnique
  - 被监听者的行为取决于是否正在执行独占序列
  - 失效监听
  - 可以转发数据
```

**4.2.9 SnpUniqueStash（唯一填充监听）**
```
用途：使被监听者处的缓存副本失效，并建议被监听者获取缓存行的唯一状态副本
特性：
  - 允许监听响应包括DataPull
  - 监听响应中的Data Pull请求被视为ReadUnique
```

**4.2.10 SnpCleanShared（清理共享监听）**
```
用途：移除被监听者处的任何脏缓存行副本
特性：
  - 必须不将缓存行留在脏状态
  - 非失效监听
```

**4.2.11 SnpCleanInvalid（清理失效监听）**
```
用途：使被监听者处的缓存行失效并获取任何脏副本
特性：
  - 也可以由互连在没有对应请求的情况下生成
  - 必须将缓存行更改为Invalid状态
  - 失效监听
```

**4.2.12 SnpMakeInvalid（使失效监听）**
```
用途：使被监听者处的缓存行失效并丢弃任何脏副本
特性：
  - 不得在监听响应中返回数据，脏数据被丢弃
  - 必须将缓存行更改为Invalid状态
  - 失效监听
```

**4.2.13 SnpMakeInvalidStash（使失效填充监听）**
```
用途：使缓存行的副本失效，并建议被监听者获取缓存行的唯一状态副本
特性：
  - 被监听者不得在监听响应中返回数据，脏数据必须被丢弃
  - 允许监听响应包括DataPull
  - 监听响应中的Data Pull请求被视为ReadUnique
```

**4.2.14 SnpQuery（查询监听）**
```
用途：探测请求节点处缓存行的状态
特性：
  - Home可以在没有来自请求者的对应请求的情况下发送SnpQuery监听
  - 监听响应必须包括目标被监听者处缓存行的精确状态
  - 被监听者不得在监听响应中返回数据
  - SnpQuery监听不得更改被监听者处的缓存行状态
```

**4.2.15 SnpDVMOp（DVM操作监听）**
```
用途：在互连处生成，由DVMOp请求启动
特性：
  - 单个DVMOp请求生成两个监听请求
  - 为两个监听请求返回单个监听响应
```

3 监听请求选择策略
--------------------

**4.3.1 监听数量**

Home节点可以：
```
1. 向多个RN-F发送非转发监听，包括向所有RN-F
2. 仅允许向一个RN-F发送转发监听
```

**失效监听要求：**
```
- 在失效的情况下，失效监听必须至少发送到所有其他缓存副本
```

**非失效监听要求：**
```
- 被允许（但不要求）向所有有缓存副本的RN-F发送监听
- 必须能够获取脏行副本
- 对于唯一脏缓存行状态，Home节点必须向有唯一脏缓存副本的RN-F发送监听
```

**带填充提示的写事务：**
```
- 对于WriteUniqueFullStash，对非填充目标节点的预期监听是SnpMakeInvalid
- 对于WriteUniquePtlStash，对非填充目标节点的预期监听是SnpCleanInvalid
```

**监听过滤器或目录：**
```
- 可以在互连中包含以支持监听过滤
```

**4.3.2 监听类型选择**

基于内部偏好和实现约束，Home节点可能偏好预期监听之一或用其他监听替换预期监听。

**预期监听对应表：**

| 请求类别 | 请求 | 预期监听 |
|----------|------|----------|
| Read | ReadNoSnp | None或SnpOnceFwd |
| Read | ReadNoSnpSep | n/a |
| Read | ReadOnce | SnpOnceFwd |
| Read | ReadOnceCleanInvalid | SnpUnique或SnpOnceFwd |
| Read | ReadOnceMakeInvalid | SnpUnique、SnpUniqueFwd或SnpOnceFwd |
| Read | ReadClean | SnpCleanFwd |
| Read | ReadNotSharedDirty | SnpNotSharedDirtyFwd |
| Read | ReadShared | SnpSharedFwd |
| Read | ReadUnique | SnpUniqueFwd |
| Read | ReadPreferUnique | SnpPreferUniqueFwd |
| Read | MakeReadUnique | SnpCleanInvalid或SnpUniqueFwd |
| Dataless | CleanUnique | SnpCleanInvalid |
| Dataless | MakeUnique | SnpMakeInvalid |
| Dataless | Evict | None |
| Dataless | CleanShared | SnpCleanShared |
| Dataless | CleanSharedPersist | SnpCleanShared |
| Dataless | CleanSharedPersistSep | SnpCleanShared |
| Dataless | CleanInvalid | SnpCleanInvalid |
| Dataless | CleanInvalidPoPA | SnpCleanInvalid |
| Dataless | MakeInvalid | SnpMakeInvalid |
| Dataless-stash | StashOnceUnique | SnpStashUnique |
| Dataless-stash | StashOnceSepUnique | SnpStashUnique |
| Dataless-stash | StashOnceShared | SnpStashShared |
| Dataless-stash | StashOnceSepShared | SnpStashShared |
| Write | WriteNoSnp | None |
| Write | WriteNoSnpDef | None |
| Write | WriteUniqueFull | SnpMakeInvalid |
| Write | WriteUniquePtl | SnpCleanInvalid或SnpUnique |
| Write | WriteUniqueZero | SnpMakeInvalid |
| Write-stash | WriteUniqueFullStash | SnpMakeInvalidStash |
| Write-stash | WriteUniquePtlStash | SnpUniqueStash或SnpMakeInvalidStash |
| Write-CopyBack | WriteBack | None |
| Write-CopyBack | WriteClean | n/a |
| Write-CopyBack | WriteEvictFull | n/a |
| Write-CopyBack | WriteEvictOrEvict | n/a |
| Atomic | AtomicStore | SnpUnique |
| Atomic | AtomicLoad | SnpUnique |
| Atomic | AtomicSwap | SnpUnique |
| Atomic | AtomicCompare | SnpUnique |
| Others | DVMOp | SnpDVMOp |
| Others | PCrdReturn | n/a |
| Others | PrefetchTgt | n/a |

**4.3.3 互连行为**

**监听过滤器/目录支持：**
```
1. 互连支持监听过滤器或目录以跟踪RN-F缓存中存在的缓存行状态
2. 跟踪可以详细到知道每个RN-F有缓存行副本
3. 也可以非特定到知道缓存行存在于RN-F缓存之一
4. 这种跟踪允许互连过滤对RN-F的不必要监听
```

**过滤示例：**
```
1. 监听过滤器指示缓存行不在任何RN-F缓存中：
   - 互连不发送监听请求

2. RN-F缓存中的缓存行已处于所需状态：
   - 例如：接收的请求是ReadShared且所有缓存行副本都处于SC状态
   - 互连不发送监听请求
```

**SnpMakeInvalid使用限制：**
```
互连在响应以下请求时不得使用SnpMakeInvalid监听请求：
  - WriteUniqueFull
  - WriteUniqueFullStash
  - MakeUnique
  - MakeInvalid

除非：
  - 事务TagOp值为Update
  - 互连可以确定被监听者不持有脏标签
```

**SnpMakeInvalid用于MakeReadUnique：**
```
互连在响应MakeReadUnique时不得使用SnpMakeInvalid监听请求，除非互连可以确定：
  - 请求者仍有缓存行副本且被监听者没有脏标签
  - 请求者已丢失缓存行副本且被监听者没有缓存行的脏副本
```

**自发监听：**
```
- 允许互连在没有来自请求节点的对应请求的情况下自发生成监听请求
- 例如：互连可以发送SnpUnique或SnpCleanInvalid请求，作为来自监听过滤器或互连缓存的向后失效的结果
```

**监听请求选择示例：**
```
对于WriteUniquePtl请求：
  - 可以发送SnpCleanInvalid或SnpUnique监听请求
  - 两者都使缓存行失效
  - 如果缓存行为脏，则响应中返回数据
  - 接收到所有监听响应后，将写数据与监听响应中接收的任何脏数据合并写入内存
  - 唯一区别是SnpUnique可以从UniqueClean（UC）状态返回数据，但SnpCleanInvalid不能
  - 使用SnpUnique可能导致不必要的数据传输
```

**监听替换：**
```
互连被允许：

1. ReadNotSharedDirty、ReadShared和ReadClean：
   - 使用SnpNotSharedDirty或SnpShared或SnpClean

2. ReadShared：
   - 使用SnpNotSharedDirtyFwd或SnpSharedFwd或SnpCleanFwd

3. ReadNotSharedDirty和ReadClean：
   - 使用SnpNotSharedDirtyFwd或SnpCleanFwd

4. ReadOnce事务：
   - 使用任何非转发、非失效监听类型

5. ReadOnceCleanInvalid和ReadOnceMakeInvalid：
   - 使用任何非转发监听类型，除了SnpMakeInvalid

6. ReadOnce和ReadOnceCleanInvalid：
   - 使用转发监听类型SnpOnceFwd

7. ReadOnceMakeInvalid：
   - 使用转发监听类型SnpUniqueFwd或SnpOnceFwd

8. WriteUniqueFullStash和WriteUniquePtlStash：
   - 如果目标请求节点没有缓存行，向目标请求节点发送SnpStashUnique或SnpMakeInvalidStash

9. 替换任何失效监听请求为SnpUnique或SnpCleanInvalid请求

10. 用对应的非转发类型替换转发监听
    - 接收者被允许将转发指示视为提示
    - 以协议兼容方式以对应的非转发版本响应监听
    - 无论是否使用MTE都允许
    - 转发监听不得用于小于64B的请求

11. MakeInvalid和WriteUniqueZero的SnpMakeInvalid使用：
    - 仅当Home知道被监听者没有脏标签时才允许
```

-------------------------------------------------------
响应类型详解
-------------------------------------------------------

1 响应分类
-----------

每个请求可以生成一个或多个响应。某些响应也可以包括数据。

响应分类：
```
1. 完成响应（B4.5.1）
2. WriteData响应（B4.5.2）
3. 监听响应（B4.5.3）
4. 其他响应（B4.5.4）
```

2 完成响应详解
----------------

**5.2.1 完成响应基本概念**

```
定义：
  - 所有事务（PCrdReturn和PrefetchTgt除外）都需要完成响应
  - 通常是完成者发送的最后一条消息，以结束请求事务
  - 请求者仍可以发送CompAck响应以结束事务

保证：
  - 完成保证请求已到达PoS或PoC
  - 在那里它将与系统中任何请求者对相同地址的请求进行排序
```

**5.2.2 读和原子事务完成**

**响应形式：**
```
1. 单个响应形式
   - 在RDAT通道上使用CompData操作码的响应

2. 两个分离响应形式
   - 一个在RSP通道上使用RespSepData操作码
   - 另一个在RDAT通道上使用DataSepResp操作码
```

**响应包含的Resp字段：**
```
1. Cache state（缓存状态）
   - 最终允许的缓存行在请求者处的状态
   - 对于所有读（ReadNoSnp和ReadOnce除外）

2. Pass Dirty（传递脏）
   - 指示更新内存的责任是否传递给请求者
   - Pass Dirty位的断言在响应名称中由_PD表示
```

**允许的读事务完成和Resp字段编码：**

.. list-table:: 允许的读事务完成和Resp字段编码
   :widths: 30 15 30 40
   :header-rows: 1

   * - 响应
     - Resp[2:0]
     - 最终缓存行状态
     - 说明
   * - CompData_I
     - 0b000
     - I
     - 指示不能保持一致性缓存行副本
   * - RespSepData_I
     - 0b000
     - 不适用
     - 缓存状态必须从DataSepResp响应确定
   * - CompData_UC
     - 0b010
     - UC, UCE, SC或I（响应中的缓存状态适用时）
     - 此响应也允许用于ReadNoSnp和ReadOnce事务，但缓存行不一致
   * - DataSepResp_UC
     - 0b010
     - UC, UCE, SC或I
     - 数据响应
   * - RespSepData_UC
     - 0b010
     - UC, UCE, SC或I
     - 完成响应
   * - CompData_SC
     - 0b001
     - SC或I
     - 脏缓存行的责任不被传递
   * - DataSepResp_SC
     - 0b001
     - SC或I
     - 数据响应
   * - RespSepData_SC
     - 0b001
     - SC或I
     - 完成响应
   * - CompData_UD_PD
     - 0b110
     - UD或SD
     - 脏缓存行的责任被传递
   * - DataSepResp_UD_PD
     - 0b110
     - UD或SD
     - 数据响应
   * - RespSepData_UD_PD
     - 0b110
     - UD或SD
     - 完成响应
   * - CompData_SD_PD
     - 0b111
     - SD
     - 脏缓存行的责任被传递

**使用分离Comp和Data响应时：**
```
- RespSepData也包括Resp字段，包含Cache state和Pass Dirty指示
- RespSepData中的Resp字段值必须是不适用并设置为0，或者与相应的DataSepResp相同
```

**5.2.3 无数据事务完成**

**响应形式：**
```
- 在CRSP通道上发送
- 使用Comp、CompPersist、CompCMO或CompStashDone操作码
```

**CompCMO响应：**
```
- 组合写事务中CMO和PCMO的完成消息
- CompCMO必须仅用于组合写事务
- CompCMO可以与Persist响应组合为CompPersist操作码
```

**Comp响应包含的Resp字段：**
```
1. Cache state（缓存状态）
   - 最终允许的缓存行在请求者处的状态
   - 对于CMO事务，缓存状态字段值被忽略，缓存状态保持不变
```

**允许的无数据事务完成和Resp字段编码：**

.. list-table:: 允许的无数据事务完成和Resp字段编码
   :widths: 20 15 40 20
   :header-rows: 1

   * - 响应
     - Resp[2:0]
     - 最终缓存行状态
     - 说明
   * - Comp_I
     - 0b000
     - I
     -
   * - Comp_UC
     - 0b010
     - UD, UC, UCE, SC或I
     -
   * - Comp_SC
     - 0b001
     - SC或I
     -
   * - Comp_UD_PD
     - 0b110
     - UD或SD
     - 脏缓存行的责任被传递

**5.2.4 写和原子事务完成**

**响应形式：**
```
- 在CRSP通道上发送
- 使用Comp或CompDBIDResp操作码
```

**响应特性：**
```
- 不使用写事务完成传达缓存状态信息或脏缓存行责任
- Comp或CompDBIDResp响应的Resp字段必须设置为0
  - WriteNoSnpDef事务除外
- 所有缓存状态信息和脏缓存行责任都与WriteData通信
```

**允许的写事务完成响应：**

1. **Comp**
```
用途：完成响应与DBIDResp或DBIDRespOrd响应分离时使用
```

2. **CompDBIDResp**
```
用途：完成响应与DBIDResp或DBIDRespOrd响应组合时使用
特性：
  - 所有CopyBack请求必须使用CompDBIDResp完成响应
  - Immediate写和AtomicStore可以：
    * 分别发送Comp和DBIDResp或DBIDRespOrd响应
    * 如果两者都准备好发送给请求者，可以机会性地组合两个响应并发送CompDBIDResp
```

**5.2.5 WriteNoSnpDef的特殊处理**

**允许的Resp和RespErr字段值组合：**

| RespErr[1:0] | Resp[2:0] | 响应 | 说明 |
|--------------|-----------|------|------|
| 00 | 000 | OK/Successful | 完成者支持WriteNoSnpDef事务且可延迟写成功 |
| 00 | 001 | OK/Unsupported | 完成者不支持WriteNoSnpDef事务。请求者必须完成事务流。完成者不得更新内存 |
| 00 | 010 | OK/Defer | 完成者支持WriteNoSnpDef事务。事务此时无法服务且不成功。内存位置不更新 |
| 10 | 000 | DERR | 数据错误 |
| 11 | 000 | NDERR | 非数据错误 |
| 其他 | 保留 | 保留 |  |

**HN-F作为目标Home节点：**
```
- HN-F被允许（但不预期）成为WriteNoSnpDef请求的目标Home节点
- 接收到意外WriteNoSnpDef请求的HN-F不得将WriteNoSnpDef请求转发到SN-F
- HN-F必须返回OK/Unsupported响应
```

**不支持可延迟写的Subordinate：**
```
- 如果Home节点检测到可延迟写针对不支持可延迟写的Subordinate
- 事务不得传播
- Home预期发送OK/Unsupported响应
- Home被允许发送NDERR响应而不是OK/Unsupported响应
```

**5.2.6 其他事务完成**

**DVM事务完成：**
```
- 始终使用Resp字段设置为0的Comp响应
```

3 WriteData响应详解
---------------------

**基本概念：**
```
定义：WriteData响应是写请求和DVMOp事务的一部分
流程：
  1. 请求者在接收到缓冲区可用以接受数据的保证后发送WriteData给完成者
  2. 缓冲区可用性通过从完成者发送的DBIDResp或DBIDRespOrd响应发出信号
```

**响应操作码：**

1. **CopyBackWriteData（CBWrData）**
```
用途：用于WriteBack、WriteClean、WriteEvictFull和WriteEvictOrEvict，以及CopyBack组合写事务
特性：
  - 将一致数据从请求者的缓存传输到互连
  - 包括在发送WriteData响应之前缓存行状态的指示
```

2. **NonCopyBackWriteData（NCBWrData）**
```
用途：用于WriteUnique和WriteNoSnp、WriteNoSnpDef，以及组合Immediate写事务，也用于DVMOp事务
特性：
  - 响应中的缓存状态必须为I
```

3. **NonCopyBackWriteDataCompAck（NCBWrDataCompAck）**
```
用途：用于Immediate写和组合写事务
特性：
  - 组合NonCopyBackWriteData和CompAck
  - 响应中的缓存状态必须为I
  - 指示事务已完成
```

4. **WriteDataCancel**
```
用途：通知完成者在写数据发送之前写请求被取消
特性：
  - 请求节点可以在WriteNoSnpPtl、WriteUniquePtl、WriteUniquePtlStash和相应组合写事务中发送WriteDataCancel而不是NonCopyBackWriteData
  - Home节点可以在WriteNoSnpFull、WriteNoSnpPtl和相应组合写事务中向Subordinate节点发送WriteDataCancel而不是NonCopyBackWriteData
  - 请求节点或Home节点可以在WriteNoSnpDef事务中发送WriteDataCancel而不是NonCopyBackWriteData，如果在发送NonCopyBackWriteData之前看到：
    * 到达的Comp响应
    * 到达的CompDBIDResp响应
  - 必不得用于对设备内存的写请求（WriteNoSnpDef事务期间除外）
  - 所有最初预期传输的数据包必须发送
  - WriteDataCancel消息中的BE字段值必须设置为零
  - 响应中的缓存状态必须为I
```

**响应包含的Resp字段：**
```
1. Cache state（缓存状态）
   - 指示在发送WriteData响应之前缓存行的状态
   - 此状态可以与发送原始事务请求时的缓存行状态不同
   - 如果在发送原始事务请求之后但在发送相应WriteData响应之前接收到对相同地址的监听请求

2. Pass Dirty（传递脏）
   - 指示更新内存的责任是否由请求者传递
   - Pass Dirty位的断言在响应名称中由_PD表示
```

**允许的WriteData响应、操作码和Resp字段编码：**

.. list-table:: 允许的WriteData响应、操作码和Resp字段编码
   :widths: 30 15 15 35 40
   :header-rows: 1

   * - 响应
     - DAT操作码
     - Resp[2:0]
     - 发送数据时的缓存行状态
     - 说明
   * - CopyBackWriteData_I
     - 0x2
     - 0b000
     - 不精确且必须被忽略
     - 指示CopyBack请求已被取消。响应中的数据必须为0且所有BE必须取消断言
   * - CopyBackWriteData_UC
     - 0x2
     - 0b010
     - UC
     - CopyBack请求对应的数据
   * - CopyBackWriteData_SC
     - 0x2
     - 0b001
     - SC
     - CopyBack请求对应的数据
   * - CopyBackWriteData_UD_PD
     - 0x2
     - 0b110
     - UD或UDP
     - CopyBack请求对应的数据。更新内存的责任被传递
   * - CopyBackWriteData_SD_PD
     - 0x2
     - 0b111
     - SD
     - CopyBack请求对应的数据。更新内存的责任被传递
   * - NonCopyBackWriteData
     - 0x3
     - 0b000
     - I
     - Immediate写请求对应的数据
   * - NonCopyBackWriteDataCompAck
     - 0xC
     - 0b000
     - I
     - Immediate写请求对应的数据和组合CompAck以指示事务已完成
   * - WriteDataCancel
     - 0x7
     - 0b000
     - I
     - 指示Immediate写请求已被取消。响应中的数据必须为0且所有BE必须取消断言

**缓存状态确定：**
```
- 写事务完成后请求者处的缓存行状态不从WriteData响应中的缓存状态信息确定
- 可以通过事务的操作码确定事务完成后缓存行是否保持有效：
  * WriteBack或WriteEvictFull事务必须处于I状态
  * WriteClean事务可以保持分配并处于清洁状态
```

4 监听响应详解
----------------

**5.4.1 监听响应形式**

**1. 不带数据的监听响应**
```
用途：当不需要数据传输时使用
特性：
  - 在SRSP通道上发送
  - 使用SnpResp操作码
  - 对于填充监听，可以包括DataPull请求
  - 始终用于对SnpDVMOp事务的响应
```

**2. 不带数据到Home和DCT的监听响应**
```
用途：当被监听者向请求者发送数据且不需要向Home传输数据时使用
特性：
  - 在SRSP通道上发送
  - 使用SnpRespFwded操作码
```

**3. 带数据的监听响应**
```
用途：当完整缓存行数据传输到Home时使用
特性：
  - 在WDAT通道上发送
  - 使用SnpRespData操作码
  - 对于填充监听，可以包括DataPull请求
```

**4. 带部分数据的监听响应**
```
用途：当部分缓存行数据传输到Home时使用
特性：
  - 在WDAT通道上发送
  - 使用SnpRespDataPtl操作码
  - 对于填充监听，可以包括DataPull请求
  - 当监听请求和缓存行状态的组合为以下情况时发送：
    * 任何SnpMakeInvalid之外的监听请求，且缓存行状态为UDP
```

**5. 带数据到Home和DCT的监听响应**
```
用途：当被监听者向请求者发送数据且也需要向Home传输数据时使用
特性：
  - 在DAT通道上发送
  - 使用SnpRespDataFwded操作码
```

**5.4.2 监听响应包含的字段**

**Resp字段：**
```
1. Cache state（缓存状态）
   - 发送监听响应后被监听节点处缓存行的最终状态

2. Pass Dirty（传递脏）
   - 指示更新内存的责任是否传递给请求者或互连
   - Pass Dirty必须仅对带数据的监听响应断言
   - Pass Dirty位的断言在响应名称中由_PD表示
```

**FwdState字段：**
```
- 适用于带DCT的监听响应
- 指示发送给请求者的CompData响应中的缓存状态和传递脏值
```

**信息充分性：**
```
监听响应属性传达足够的信息以供互连确定：
  1. 对初始请求者的适当响应
  2. 数据是否必须写回内存

监听响应属性也足以支持互连中的监听过滤器或目录维护
```

**重要区别：**
```
监听响应缓存状态信息提供发送监听响应后缓存行的状态。这与以下不同：

1. WriteData响应：
   - 缓存状态信息提供写数据时缓存行的状态

2. 读数据响应：
   - 缓存状态信息指示事务完成后缓存行的允许状态
```

**5.4.3 允许的非转发类型不带数据的监听响应**

.. list-table:: 允许的非转发类型不带数据的监听响应
   :widths: 30 20 15 20
   :header-rows: 1

   * - 响应
     - RSP操作码
     - Resp[2:0]
     - 缓存行状态
   * - SnpResp_I
     - 0x1
     - 0b000
     - I
   * - SnpResp_SC
     - 0x1
     - 0b001
     - SC或I
   * - SnpResp_UC
     - 0x1
     - 0b010
     - UC, UCE, SC或I
   * - SnpResp_UD
     - 0x1
     - 0b010
     - UD
   * - SnpResp_SD
     - 0x1
     - 0b011
     - SD

**5.4.4 允许的转发类型不带数据的监听响应**

| 响应 | RSP操作码 | Resp[2:0] | FwdState[2:0] | 被监听者处的缓存行状态 | 转发状态 | 说明 |
|------|-----------|-----------|---------------|------------------------|----------|------|
| SnpResp_I_Fwded_I | 0x9 | 0b000 | 0b000 | I | I |  |
| SnpResp_I_Fwded_SC | 0x9 | 0b000 | 0b001 | I | SC |  |
| SnpResp_I_Fwded_UC | 0x9 | 0b000 | 0b010 | I | UC |  |
| SnpResp_I_Fwded_UD_PD | 0x9 | 0b000 | 0b110 | I | UD | 更新内存的责任被传递 |
| SnpResp_I_Fwded_SD_PD | 0x9 | 0b000 | 0b111 | I | SD | 更新内存的责任被传递 |
| SnpResp_SC_Fwded_I | 0x9 | 0b001 | 0b000 | SC | I |  |
| SnpResp_SC_Fwded_SC | 0x9 | 0b001 | 0b001 | SC | SC |  |
| SnpResp_SC_Fwded_SD_PD | 0x9 | 0b001 | 0b111 | SC | SD | 更新内存的责任被传递 |
| SnpResp_UC_Fwded_I | 0x9 | 0b010 | 0b000 | UC或UD | I | 用于UC和UD的单个编码 |
| SnpResp_UD_Fwded_I | 0x9 | 0b010 | 0b000 | UC或UD | I | 用于UC和UD的单个编码 |
| SnpResp_SD_Fwded_I | 0x9 | 0b011 | 0b000 | SD | I |  |
| SnpResp_SD_Fwded_SC | 0x9 | 0b011 | 0b001 | SD | SC |  |

**允许的非转发类型带数据的监听响应：**

.. list-table:: 允许的非转发类型带数据的监听响应
   :widths: 35 15 15 20 30
   :header-rows: 1

   * - 响应
     - DAT操作码
     - Resp[2:0]
     - 缓存行状态
     - 说明
   * - SnpRespData_I
     - 0x1
     - 0b000
     - I
     -
   * - SnpRespData_UC
     - 0x1
     - 0b010
     - UC或UD
     - 用于UC和UD的单个编码
   * - SnpRespData_UD
     - 0x1
     - 0b010
     - UC或UD
     - 用于UC和UD的单个编码
   * - SnpRespData_SC
     - 0x1
     - 0b001
     - SC
     -
   * - SnpRespData_SD
     - 0x1
     - 0b011
     - SD
     -
   * - SnpRespData_I_PD
     - 0x1
     - 0b100
     - I
     - 更新内存的责任被传递给Home
   * - SnpRespData_UC_PD
     - 0x1
     - 0b110
     - UC
     - 更新内存的责任被传递给Home
   * - SnpRespData_SC_PD
     - 0x1
     - 0b101
     - SC
     - 更新内存的责任被传递给Home
   * - SnpRespDataPtl_I_PD
     - 0x5
     - 0b100
     - I
     - 部分数据。更新内存的责任被传递给Home
   * - SnpRespDataPtl_UD
     - 0x5
     - 0b010
     - UDP
     - 部分数据

**5.4.6 允许的转发类型带数据的监听响应**

.. list-table:: 允许的转发类型带数据的监听响应
   :widths: 40 15 15 15 15 15 40
   :header-rows: 1

   * - 响应
     - RSP操作码
     - Resp[2:0]
     - FwdState[2:0]
     - 缓存行状态
     - 转发状态
     - 说明
   * - SnpRespData_I_Fwded_SC
     - 0x6
     - 0b000
     - 0b001
     - I
     - SC
     -
   * - SnpRespData_I_Fwded_SD_PD
     - 0x6
     - 0b000
     - 0b111
     - I
     - SD
     - 更新内存的责任被传递给请求者
   * - SnpRespData_SC_Fwded_SC
     - 0x6
     - 0b001
     - 0b001
     - SC
     - SC
     -
   * - SnpRespData_SC_Fwded_SD_PD
     - 0x6
     - 0b001
     - 0b111
     - SC
     - SD
     - 更新内存的责任被传递给请求者
   * - SnpRespData_SD_Fwded_SC
     - 0x6
     - 0b011
     - 0b001
     - SD
     - SC
     -
   * - SnpRespData_I_PD_Fwded_I
     - 0x6
     - 0b100
     - 0b000
     - I
     - I
     - 更新内存的责任被传递给Home
   * - SnpRespData_I_PD_Fwded_SC
     - 0x6
     - 0b100
     - 0b001
     - I
     - SC
     - 更新内存的责任被传递给Home
   * - SnpRespData_SC_PD_Fwded_I
     - 0x6
     - 0b101
     - 0b000
     - SC
     - I
     - 更新内存的责任被传递给Home
   * - SnpRespData_SC_PD_Fwded_SC
     - 0x6
     - 0b101
     - 0b001
     - SC
     - SC
     - 更新内存的责任被传递给Home

**5.4.7 带Data Pull的监听响应**

在响应填充监听时，被监听者可以发送与监听响应组合的Read请求（SnpResp_X_Read），通过设置DataPull位。

**允许的带Data Pull的监听响应：**

**对于SnpUniqueStash：**
```
- SnpResp_I_Read
- SnpRespData_I_Read
- SnpRespData_I_PD_Read
- SnpRespDataPtl_I_PD_Read
```

**对于SnpMakeInvalidStash：**
```
- SnpResp_I_Read
```

**对于SnpStashUnique：**
```
- SnpResp_I_Read
- SnpResp_UC_Read
- SnpResp_SC_Read
- SnpResp_SD_Read
```

**对于SnpStashShared：**
```
- SnpResp_I_Read
- SnpResp_UC_Read
```

**5.4.8 错误处理**

**缓存行状态合法性：**
```
- 带数据的监听响应关联的缓存行状态必须是合法值
- 即使RespErr字段指示有数据错误（DERR）
- 带数据的监听响应不允许有NDERR
```

5 其他响应详解
----------------

**5.5.1 CompAck（完成确认）**

```
用途：请求者在接收到完成响应后发送
适用：
  - Read、Dataless、WriteNoSnp、WriteUnique和CopyBack写事务
特性：
  - RespErr字段适用且必须设置为0
  - 对于Immediate写事务，CompAck响应中的Resp值不适用且必须设置为0
  - 对于CopyBack写事务，CompAck响应中的Resp值适用
```

**CopyBack写事务的允许CompAck响应：**

.. list-table:: CopyBack写事务的允许CompAck响应
   :widths: 20 20 15 30 40 40
   :header-rows: 1

   * - 响应
     - RSP操作码
     - Resp[2:0]
     - 发送响应时的缓存行状态
     - Home处有缓存行隐藏副本时的操作
     - 说明
   * - CompAck_I
     - 0x2
     - 0b000
     - 不精确且必须被忽略
     - 不得尚未暴露
     - 指示CopyBack请求已被取消
   * - CompAck_UC
     - 0x2
     - 0b010
     - UC
     - 期望（但不要求）暴露，除非其他地方仍存在唯一副本
     -
   * - CompAck_SC
     - 0x2
     - 0b001
     - SC
     - 期望（但不要求）暴露
     -
   * - CompAck_UD_PD
     - 0x2
     - 0b110
     - UD
     - 期望（但不要求）暴露，除非其他地方仍存在唯一副本
     - 更新内存的责任被传递
   * - CompAck_SD_PD
     - 0x2
     - 0b111
     - SD
     - 期望（但不要求）暴露
     - 更新内存的责任被传递

**5.5.2 RetryAck（重试确认）**

```
用途：完成者如果由于缺乏适当资源而未接受请求时发送给请求者
适用：
  - 任何请求事务（PCrdReturn或PrefetchTgt除外）
特性：
  - RespErr字段适用且必须设置为0
  - Resp字段不适用且必须设置为0
```

**5.5.3 PCrdGrant（协议信贷授予）**

```
用途：授予协议信贷
特性：
  - 使用协议信贷的后续请求保证被目标接受
  - RespErr字段适用且必须设置为0
  - Resp字段不适用且必须设置为0
```

**5.5.4 ReadReceipt（读接收）**

```
用途：对于与来自同一请求者的其他有序请求有排序要求的请求发送
发送者：Subordinate节点
目的：指示它已接受读请求且不会发送RetryAck响应
适用：
  - ReadNoSnp、ReadNoSnpSep和ReadOnce*请求事务
特性：
  - RespErr字段适用且必须设置为0
  - Resp字段不适用且必须设置为0
```

**5.5.5 DBIDResp（数据缓冲区ID响应）**

```
用途：向请求者发出信号，表示资源可用于接受WriteData响应
特性：
  - DBIDResp响应还指示完成者提供某些事务排序保证
  - 适用于Write、组合写、DVMOp和原子请求事务
  - 响应允许从Home节点到请求节点和从Subordinate节点到Home节点和请求节点
  - RespErr字段适用且必须设置为0
  - Resp字段不适用且必须设置为0
```

**5.5.6 DBIDRespOrd（有序数据缓冲区ID响应）**

```
用途：向请求者发出信号，表示资源可用于接受WriteData响应
特性：
  - DBIDRespOrd响应还指示完成者提供某些事务排序保证
  - 适用于Write、组合写和原子请求事务
  - DBIDRespOrd不允许用于DVM事务
  - 响应仅允许从Home节点到请求节点
  - RespErr字段适用且必须设置为0
  - Resp字段不适用且必须设置为0
```

**5.5.7 Persist（持久化）**

```
用途：完成者为CleanSharedPersistSep事务发送，指示之前写入同一内存位置的任何数据已被持久化
特性：
  - RespErr字段适用
  - Resp字段不适用且必须设置为0
```

**5.5.8 StashDone（填充完成）**

```
用途：完成者为StashOnceSep发送，指示请求在完成者处的排序
特性：
  - RespErr字段适用
  - Resp字段不适用且必须设置为0
```

**5.5.9 TagMatch（标签匹配）**

```
用途：完成者为带有TagOp为Match的写事务发送，以指示Tag Match操作的完成
特性：
  - RespErr字段适用
  - Resp字段适用
```

-----------------------------------------------------------------------
缓存状态转换详解
-----------------------------------------------------------------------

1 静默缓存状态转换
--------------------

**6.1.1 静默转换概念**

```
定义：缓存可以由于内部事件而改变状态，无需通知系统其余部分
```

**6.1.2 合法的静默缓存状态转换**

**表B4.35：合法的静默缓存状态转换及如何使其非静默**

| RN-F操作 | 当前RN-F状态 | 下一RN-F状态 | 可用于使操作非静默的事务 |
|----------|--------------|-------------|-------------------------|
| 缓存驱逐 | UC | I | Evict、WriteEvictFull或WriteEvictOrEvict |
| 缓存驱逐 | UCE | I | Evict |
| 缓存驱逐 | SC | I | Evict或WriteEvictOrEvict |
| 本地共享 | UC | SC | - |
| 本地共享 | UD | SD | - |
| 缓存失效 | UD | I | Evict |
| 缓存失效 | UDP | I | Evict |

**RN-F操作的本地共享说明：**
```
描述：RN-F将唯一缓存行指定为共享，实际上忽略了缓存行对RN-F仍保持唯一的事实
示例：当RN-F包含多个内部代理且缓存行在它们之间变为共享时
```

**6.1.3 静默转换的规则**

**转换时机：**
```
1. 缓存驱逐和本地共享转换：
   - 可以在任何点发生
   - 由实现定义

2. 存储和缓存失效转换：
   - 只能作为故意操作的结果发生
   - 对于核心，由执行特定程序指令引起
```

**转换约束：**
```
- UC到UCE的缓存状态转换不被允许
- 静默转换序列也可以发生
- 导致缓存行处于UD、UDP或SC状态的任何静默转换可以进一步进行静默转换
```

**6.1.4 内部RN-F存储导致的合法静默缓存状态转换**

**表B4.36：内部RN-F存储导致的合法静默缓存状态转换**

| RN-F操作 | 当前RN-F状态 | 下一RN-F状态 | 说明 |
|----------|--------------|-------------|------|
| 存储的结果 | UC | UD | 完整或部分缓存行存储 |
| 存储的结果 | UCE | UDP | 部分缓存行存储 |
| 存储的结果 | UD | UD | 完整缓存行存储 |
| 存储的结果 | UDP | UD | 填充缓存行的存储 |

**说明：**
```
静默转换序列也可以发生
任何导致缓存行处于UD、UDP或SC状态的静默转换可以进一步进行静默转换
```

2 请求者的缓存状态转换
------------------------

**6.2.1 读请求事务**

**表B4.37：读请求事务的请求者缓存状态转换**

| 请求类型 | 初始预期状态 | 初始允许状态 | 最终状态 | Comp响应 | 分离响应 |
|----------|--------------|-------------|---------|----------|----------|
| ReadNoSnp | I | - | I | CompData_UC, CompData_I | RespSepData + DataSepResp_UC |
| ReadOnce | I | - | I | CompData_UC, CompData_I | RespSepData + DataSepResp_UC |
| ReadOnceCleanInvalid | I | - | I | CompData_UC, CompData_I | RespSepData + DataSepResp_UC |
| ReadOnceMakeInvalid | I | - | I | CompData_UD_PD, CompData_UC, CompData_I | RespSepData + DataSepResp_UC |
| ReadClean (TagOp=Transfer) | I | - | SC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadClean (TagOp=Transfer) | UC | - | UC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadClean (TagOp=Transfer) | UCE | - | UC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadClean (TagOp=Transfer) | UD, UDP | - | UD | CompData_SC | RespSepData + DataSepResp_SC |
| ReadClean (TagOp=Transfer) | SC | - | SC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadClean (TagOp=Transfer) | SD | - | SD | CompData_SC | RespSepData + DataSepResp_SC |
| ReadClean (TagOp!=Transfer) | I | - | SC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadClean (TagOp!=Transfer) | UC | - | UC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadClean (TagOp!=Transfer) | UCE | - | UC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadNotSharedDirty | I, UCE | - | SC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadNotSharedDirty | UC | - | UC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadNotSharedDirty | UD | - | UD | CompData_UD_PD | RespSepData + DataSepResp_UD_PD |
| ReadShared | I, UCE | - | SC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadShared | UC | - | UC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadShared | SD | - | SD | CompData_SD_PD | RespSepData + DataSepResp_UD_PD |
| ReadUnique | I, SC | UC, UCE | UC | CompData_UC | RespSepData + DataSepResp_UC |
| ReadUnique | UD | - | UD | CompData_UD_PD | RespSepData + DataSepResp_UD_PD |
| ReadUnique | SD | - | UD | CompData_UC | RespSepData + DataSepResp_UC |
| ReadPreferUnique (TagOp=Transfer) | I | - | SC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadPreferUnique (TagOp=Transfer) | UC | - | UC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadPreferUnique (TagOp=Transfer) | UD | - | UD | CompData_UD_PD | RespSepData + DataSepResp_UD_PD |
| ReadPreferUnique (TagOp=Transfer) | UD, UDP | - | UD | CompData_SC | RespSepData + DataSepResp_SC |
| ReadPreferUnique (TagOp=Transfer) | SC | - | SC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadPreferUnique (TagOp=Transfer) | SD | - | SD | CompData_SC | RespSepData + DataSepResp_SC |
| ReadPreferUnique (TagOp!=Transfer) | I | - | SC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadPreferUnique (TagOp!=Transfer) | UC | - | UC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadPreferUnique (TagOp!=Transfer) | UD | - | UD | CompData_UD_PD | RespSepData + DataSepResp_UD_PD |
| ReadPreferUnique (TagOp!=Transfer) | SC | - | SC | CompData_SC | RespSepData + DataSepResp_SC |
| ReadPreferUnique (TagOp!=Transfer) | SD | - | SD | CompData_SC | RespSepData + DataSepResp_SC |
| MakeReadUnique | - | - | - | 见表B4.40和B4.41 |  |

**注意事项：**
```
a. 对于ReadNotSharedDirty和ReadShared事务，初始状态为UCE的请求者不得在请求未完成时将缓存行升级到UDP或UD

b. 初始状态为UC的请求者接收到CompData_SC或DataSepResp_SC响应时必须保持在UC状态
   同样，使用监听过滤器跟踪请求者处缓存状态的Home不得根据对请求者的响应中的状态降级监听过滤器中的缓存行状态

c. 初始状态为UD的请求者接收到CompData_SC或DataSepResp_SC响应时必须保持在UD状态
   同样，使用监听过滤器跟踪请求者处缓存状态的Home不得根据对请求者的响应中的状态降级监听过滤器中的缓存行状态

d. 初始状态为SD的请求者接收到CompData_SC或DataSepResp_SC响应时必须保持在SD状态
   同样，使用监听过滤器跟踪请求者处缓存状态的Home不得根据对请求者的响应中的状态降级监听过滤器中的缓存行状态

e. 如果缓存状态为UD或SD，必须丢弃从内存接收的数据；如果缓存状态为UDP，则必须合并
   如果缓存状态为SC或UC，从内存接收的数据必须与缓存数据相同
```

**6.2.2 MakeReadUnique事务**

**6.2.2.1 允许的响应**

**表B4.38：MakeReadUnique事务中的允许响应**

| 接收的响应 | 缓存行状态 | 操作 |
|------------|-----------|------|
| Comp_UC | 可以是清洁或脏 | 请求者保留缓存行的副本。所有其他缓存副本已被失效 |
| Comp_UD_PD | 必须变为脏 | 请求者保留缓存行的副本。别处持有的脏副本已被失效 |
| CompData_UC | 清洁副本被给予请求者 | 请求者在事务进行期间丢失了缓存行。给出组合数据和完成响应 |
| CompData_UD_PD | 脏副本被给予请求者 | 请求者在事务进行期间丢失了缓存行。给出组合数据和完成响应 |
| RespSepData, DataSepResp_UC | 清洁副本被给予请求者 | 请求者在事务进行期间丢失了缓存行。给出分离数据和完成响应 |
| RespSepData, DataSepResp_UD_PD | 脏副本被给予请求者 | 请求者在事务进行期间丢失了缓存行。由Home给出分离数据和完成响应 |

**表B4.39：MakeReadUnique(Excl)事务中的额外允许响应**

| 接收的响应 | 全局独占检查 | 共享缓存副本 | 缓存行的副本 |
|------------|-------------|-------------|-------------|
| Comp_SC | 失败 | 存在于另一个缓存 | 由请求者保留 |
| CompData_SC | 失败 | 存在于另一个缓存 | 可能由请求者丢失 |
| RespSepData, DataSepResp_SC | 失败 | 存在于另一个缓存 | 可能由请求者丢失 |

**关键特性：**
```
1. 无数据响应（Comp_UD_PD）：
   - 指示请求者被传递脏缓存行责任
   - 当请求者持有共享清洁副本且另一个代理持有共享脏副本时可能发生
   - Home使共享脏副本失效（例如使用SnpMakeInvalid事务）
   - 随后将脏缓存行责任传递给发出MakeReadUnique事务的请求者

2. SC状态的响应：
   - 仅在对独占版本的MakeReadUnique的响应中允许
   - 当Home确定独占存储失败但监听过滤器或对请求者的SnpQuery监听响应指示请求者仍持有缓存行副本，而系统中存在另一个共享副本时，发送Comp_SC
```

**6.2.2.2 预期监听**

**Home使用监听使被监听者处的缓存行失效的用法：**

```
对于非独占MakeReadUnique，或通过独占检查的独占MakeReadUnique：

Home预期使用SnpCleanInvalid监听：
  - Home被允许使用SnpUnique而不是SnpCleanInvalid
  - 如果Home确定请求者已丢失缓存行的缓存副本，Home被允许使用SnpUniqueFwd
  - 如果Home尚未使发出MakeReadUnique的请求者失效，它也被允许使用SnpMakeInvalid
  - 来自SD副本的SnpResp_I响应可以用于隐式传递脏责任
```

**6.2.2.3 缓存行状态转换**

**最终状态确定因素：**
```
- MakeReadUnique事务后缓存行的最终状态取决于接收到事务响应之前缓存行的状态
- 这可能与发出事务时缓存行的状态不同
```

**缓存行保留规则：**
```
对于MakeReadUnique，请求者必须保留缓存行的副本，除非接收到失效监听
```

**失效监听类型：**
```
失效监听包括：
  - SnpUnique
  - SnpUniqueFwd
  - SnpCleanInvalid
  - SnpMakeInvalid
  - SnpUniqueStash
  - SnpMakeInvalidStash
```

**非失效监听处理：**
```
- 请求者被允许将SnpPreferUnique和SnpPreferUniqueFwd视为失效或非失效
- Home可以通过检查监听响应来确定被监听者如何处理这些监听
- 所有其他监听都是非失效的，请求者需要保留缓存行的副本
```

**精确监听过滤器的影响：**
```
如果Home没有精确的SF（监听过滤器），或SF不精确，且Home不能确定请求者在事务完成时仍有缓存行副本：
  - Home必须假设缓存行在请求者处丢失
  - 在响应中提供数据
```

**SD状态下的数据响应处理：**
```
请求者接收到带数据的响应时，如果仍保持行在SD状态：
  - 必须使用自己的缓存行副本，而不是响应返回的副本

说明：这意味着没有监听过滤器或监听过滤器不精确
在这种情况下，响应返回的数据可能是陈旧的

如果请求者知道没有监听过滤器：
  - 可以使用CleanUnique事务而不是MakeReadUnique
  - 以避免在缓存行未在任何其他代理处缓存时不必要的内存读取
```

**6.2.2.4 精确和不精确监听过滤器的处理**

**表B4.40：MakeReadUnique请求的请求者缓存状态转换（非Excl和Excl）**

.. list-table:: MakeReadUnique请求的请求者缓存状态转换（非Excl和Excl）
   :widths: 15 15 15 15 15 20 20 40
   :header-rows: 1

   * - 初始状态
     - 响应时的状态
     - Home发送失效监听
     - 最终状态
     - Comp响应
     - 精确监听过滤器
     - 不精确或不存在监听过滤器
     - 说明
   * - SD
     - SD
     - 否
     - UD
     - Comp_UC
     - Y
     - -
     -
   * - SD
     - SD
     - 否
     - UD
     - CompData_UC
     - -
     - Y
     - 返回的数据可能是陈旧的
   * - SD
     - SD
     - 否
     - UD
     - RespSepData, DataSepResp_UC
     - -
     - Y
     -
   * - SC, SD
     - SC
     - 否
     - UC
     - Comp_UC
     - Y
     - -
     -
   * - SC, SD
     - SC
     - 否
     - UC
     - CompData_UC
     - -
     - Y
     -
   * - SC, SD
     - SC
     - 否
     - UC
     - RespSepData, DataSepResp_UC
     - -
     - Y
     -
   * - UD
     - -
     - -
     - UD
     - Comp_UD_PD
     - Y
     - -
     -
   * - UD
     - -
     - -
     - UD
     - CompData_UD_PD
     - -
     - Y
     - 响应中的数据与请求者副本相同
   * - UD
     - -
     - -
     - UD
     - RespSepData, DataSepResp_UD_PD
     - -
     - Y
     -
   * - I
     - 是
     - -
     - UC
     - CompData_UC
     - Y
     - Y
     - 由于失效监听，行丢失
   * - I
     - 是
     - -
     - UC
     - RespSepData, DataSepResp_UC
     - Y
     - Y
     -
   * - I
     - 是
     - -
     - UD
     - CompData_UD_PD
     - Y
     - Y
     -
   * - I
     - 是
     - -
     - UD
     - RespSepData, DataSepResp_UD_PD
     - Y
     - Y
     -
   * - I, UC, UD
     - -
     - -
     - 不允许
     - -
     - -
     - -
     -

**表格键：**
```
Y = 允许
- = 不允许
```

**表B4.41：MakeReadUnique请求的请求者额外缓存状态转换（Excl）**

.. list-table:: MakeReadUnique请求的请求者额外缓存状态转换（Excl）
   :widths: 15 20 20 15 15 20 20
   :header-rows: 1

   * - 初始状态
     - 响应时的状态
     - Home发送失效监听时的状态
     - 最终状态
     - Comp响应
     - 精确监听过滤器
     - 不精确或不存在监听过滤器
   * - SC, SD
     - SC
     - 否
     - SC
     - Comp_SC
     - Y
     - -
   * - SC, SD
     - SC
     - 否
     - SC
     - CompData_SC
     - -
     - Y
   * - SC, SD
     - SC
     - 否
     - SC
     - RespSepData, DataSepResp_SC
     - -
     - Y
   * - I
     - 是
     - -
     - SC
     - CompData_SC
     - Y
     - Y
   * - I
     - 是
     - -
     - SC
     - RespSepData, DataSepResp_SC
     - Y
     - Y
   * - SD
     - SD
     - 否
     - SD
     - Comp_SC
     - Y
     - -
   * - SD
     - SD
     - 否
     - SD
     - CompData_SC
     - -
     - Y
   * - SD
     - SD
     - 否
     - SD
     - RespSepData, DataSepResp_SC
     - -
     - Y

**6.2.3 无数据请求事务**

**表B4.42：无数据请求事务的请求者缓存状态转换**

| 请求类型 | 初始预期状态 | 初始允许状态 | 最终状态 | Comp响应 |
|----------|--------------|-------------|---------|----------|
| CleanUnique | I | UC, UCE | UCE | Comp_UC |
| CleanUnique | - | SC | UC | Comp_UC |
| CleanUnique | - | SD | UD | Comp_UC |
| MakeUnique | I, SC, SD | UC, UCE | UD | Comp_UC |
| Evict | I | - | I | Comp_I |
| StashOnceUnique | I | - | I | Comp |
| StashOnceSepUnique | I | - | I | Comp + StashDone或CompStashDone |
| StashOnceShared | I | - | I | Comp |
| StashOnceSepShared | I | - | I | Comp + StashDone或CompStashDone |
| CleanShared, CleanSharedPersist | I, SC, UC | - | 无变化 | Comp_UC |
|  |  |  |  | Comp_SC |
|  |  |  |  | Comp_I |
| CleanSharedPersistSep | I, SC, UC | - | 无变化 | Comp_UC + Persist或CompPersist_UC |
|  |  |  |  | Comp_SC + Persist或CompPersist_SC |
|  |  |  |  | Comp_I + Persist或CompPersist_I |
| CleanInvalid | I | - | I | Comp_I |
| CleanInvalidPoPA | - | - | I | Comp_I |
| MakeInvalid | I | - | I | Comp_I |

**事务前的状态转换：**
```
在CleanInvalid、CleanInvalidPoPA、MakeInvalid或Evict事务之前，允许缓存状态为UC、UCE或SC
然而，要求在发出事务之前缓存状态转换到I状态
因此，表中显示I状态为唯一的初始状态
```

**6.2.4 写请求事务**

**表B4.43：写请求事务的请求者缓存状态转换**

| 请求类型 | 初始预期状态 | 初始允许状态 | 最终状态 | WriteData响应 | 组合或分离的完成和DBIDResp响应 |
|----------|--------------|-------------|---------|---------------|---------------------------|
| WriteNoSnpPtl | I | - | I | NonCopyBackWriteData | Comp或CompDBIDResp |
| WriteNoSnpFull | I | - | I | NonCopyBackWriteData | Comp或CompDBIDResp |
| WriteNoSnpDef | I | - | I | NonCopyBackWriteData | Comp或CompDBIDResp |
| WriteNoSnpZero | I | - | I | NonCopyBackWriteData | Comp或CompDBIDResp |
| WriteUniquePtl | I | - | I | NonCopyBackWriteData | Comp或CompDBIDResp |
| WriteUniqueFull | I | - | I | NonCopyBackWriteData | Comp或CompDBIDResp |
| WriteUniqueZero | I | - | I | NonCopyBackWriteData | Comp或CompDBIDResp |
| WriteUniquePtlStash | I | - | I | NonCopyBackWriteData | Comp或CompDBIDResp |
| WriteUniqueFullStash | I | - | I | NonCopyBackWriteData | Comp或CompDBIDResp |
| WriteBackPtl | - | - | I | CopyBackWriteData_UD_PD | CompDBIDResp |
| WriteBackFull | - | UD, SD | I | CopyBackWriteData_UD_PD或CopyBackWriteData_SD_PD | CompDBIDResp |
| WriteCleanFull | - | UD, SD | UC或SC | CopyBackWriteData_UD_PD或CopyBackWriteData_SD_PD | CompDBIDResp |
| WriteEvictFull | - | UC | I | CopyBackWriteData_UC | CompDBIDResp |
| WriteEvictOrEvict | - | UC或SC | I | CopyBackWriteData_UC或WriteDataCancel | CompDBIDResp |

**组合写请求：**
```
组合写请求未在表中列出
参见B4.2.4组合写请求
```

3 被监听者的缓存状态转换
--------------------------

**6.3.1 非转发和非填充监听事务**

**6.3.2 填充监听事务**

**6.3.3 转发监听事务**

**详细的状态转换规则和条件需要根据具体的监听请求类型和当前缓存状态进行分析。**

-------------------------------------------------------
危险条件处理
-------------------------------------------------------

1 危险条件概述
----------------

危险条件是指可能导致不一致或数据损坏的并发访问场景。CHI协议定义了多种危险条件及其处理机制。

**7.1.1 危险条件的类型**

1. **请求节点处的危险条件**
2. **互连（ICN(HN-F)）节点处的危险条件**

2 请求节点处的危险条件
------------------------

**7.2.1 危险条件场景**

```
1. 同时的读写访问
2. 缓存行状态转换期间的访问
3. 监听请求与本地操作的冲突
4. 原子操作的并发访问
```

**7.2.2 处理机制**

```
1. 监听请求的优先级
2. 缓存行状态锁定
3. 事务序列化
4. 重试机制
```

3 互连节点处的危险条件
--------------------------

**7.3.1 危险条件场景**

```
1. 多个请求同时到达
2. 监听响应的竞争条件
3. 数据传输的同步问题
4. 目录状态的一致性
```

**7.3.2 处理机制**

```
1. 请求排序
2. 监听过滤器的使用
3. 事务仲裁
4. 状态机管理
```

-------------------------------------------------------
一致性协议实际应用
-------------------------------------------------------

1 读操作的一致性流程
----------------------

**8.1.1 基本读操作流程**

```
步骤1：请求节点发送读请求到Home节点
步骤2：Home节点检查缓存行状态
步骤3：Home节点根据状态决定是否需要监听
步骤4：Home节点发送适当的监听请求（如果需要）
步骤5：被监听节点返回监听响应
步骤6：Home节点从内存或被监听节点获取数据
步骤7：Home节点向请求节点发送完成响应和数据
步骤8：请求节点更新缓存行状态
```

**8.1.2 不同读请求类型的具体流程**

**ReadShared流程：**
```
1. 请求节点发送ReadShared请求
2. Home检查缓存行状态
3. 如果有脏副本，发送SnpSharedFwd监听
4. 如果有清洁副本，发送SnpShared监听或不发送监听
5. 被监听节点返回数据（如果有）
6. Home向请求节点发送SC状态的数据
7. 请求节点缓存行转换为SC状态
```

**ReadUnique流程：**
```
1. 请求节点发送ReadUnique请求
2. Home检查缓存行状态
3. 发送SnpUniqueFwd监听以使其他副本失效
4. 被监听节点返回数据（如果有）
5. Home向请求节点发送UC或UD状态的数据
6. 请求节点缓存行转换为UC或UD状态
7. 请求者获得对缓存行的唯一所有权
```

2 写操作的一致性流程
----------------------

**8.2.1 立即写操作流程**

```
步骤1：请求节点发送写请求到Home节点
步骤2：Home节点检查是否需要监听
步骤3：Home节点发送监听请求（如果需要）
步骤4：被监听节点返回监听响应和数据（如果有）
步骤5：Home节点向请求节点发送DBIDResp
步骤6：请求节点发送WriteData
步骤7：Home节点合并数据并写入内存
步骤8：Home节点向请求节点发送完成响应
```

**8.2.2 CopyBack写操作流程**

```
步骤1：请求节点发送WriteBack请求
步骤2：Home节点向请求节点发送DBIDResp
步骤3：请求节点发送WriteData（包含缓存行状态信息）
步骤4：Home节点将数据写入内存
步骤5：Home节点向请求节点发送完成响应
步骤6：请求节点使缓存行失效
```

3 原子操作的一致性流程
------------------------

**8.3.1 原子操作执行流程**

```
步骤1：请求节点发送原子请求到Home节点
步骤2：Home节点发送SnpUnique监听使其他副本失效
步骤3：Home节点向Subordinate节点转发原子请求
步骤4：Subordinate节点执行原子操作
步骤5：Subordinate节点返回原始值（如果是Load类型）
步骤6：Home节点向请求节点发送完成响应
步骤7：请求节点接收结果
```

**8.3.2 原子操作的本地执行**

```
如果请求者有缓存行的唯一副本：
  - 可以在本地执行原子操作
  - 无需生成原子事务

如果请求者有共享副本：
  - 获取唯一所有权后本地执行
  - 或使本地副本失效后发送原子事务
```

4 缓存维护操作流程
--------------------

**8.4.1 CleanShared流程**

```
步骤1：请求节点发送CleanShared请求
步骤2：Home节点发送SnpCleanShared监听
步骤3：被监听节点如果持有脏副本，写回内存
步骤4：被监听节点转换为SC或I状态
步骤5：Home节点向请求节点发送完成响应
步骤6：所有缓存副本转换为非脏状态
```

**8.4.2 CleanInvalid流程**

```
步骤1：请求节点发送CleanInvalid请求
步骤2：Home节点发送SnpCleanInvalid监听
步骤3：被监听节点如果持有脏副本，写回内存
步骤4：被监听节点转换为I状态
步骤5：Home节点向请求节点发送完成响应
步骤6：所有缓存副本被失效
```

**8.4.3 持久化CMO流程**

```
步骤1：请求节点发送CleanSharedPersist请求
步骤2：Home节点发送SnpCleanShared监听
步骤3：被监听节点如果持有脏副本，写回到持久化点
步骤4：被监听节点转换为SC或I状态
步骤5：Home节点向请求节点发送完成响应
步骤6：Home节点向请求节点发送Persist响应
步骤7：数据被持久化到持久化点
```

-------------------------------------------------------
性能优化和设计考虑
-------------------------------------------------------

1 缓存一致性的性能优化
------------------------

**9.1.1 监听过滤器的使用**

```
优势：
  - 减少不必要的监听请求
  - 降低网络带宽消耗
  - 减少缓存干扰

实现方式：
  - 基于目录的监听过滤器
  - 基于位图的监听过滤器
  - 精确和不精确的监听过滤器
```

**9.1.2 数据转发机制**

```
转发监听：
  - 直接从被监听者转发数据到请求者
  - 减少延迟
  - 降低Home节点的负载

转发条件：
  - 被监听者有有效的数据副本
  - Home节点确定转发是安全的
  - 请求者需要数据
```

**9.1.3 缓存行状态的优化**

```
部分脏状态（UDP）：
  - 支持部分写入
  - 减少数据传输
  - 需要与内存数据合并

空缓存行（UCE）：
  - 提前获取写入权限
  - 节省系统带宽
  - 适用于写入密集型操作
```

**9.1.4 事务组合**

```
组合写和CMO：
  - 减少事务数量
  - 提高吞吐量
  - 简化排序要求

分离响应：
  - 支持流水线操作
  - 提高并发性
  - 需要额外的排序保证
```

2 一致性协议的设计考虑
-----------------------

**9.2.1 可扩展性**

```
1. 支持多种拓扑结构
   - Crossbar
   - Ring
   - Mesh
   - 混合拓扑

2. 支持多种节点类型
   - RN-F, RN-D, RN-I
   - HN-F, HN-I
   - SN-F, SN-I
   - MN

3. 支持多种缓存模型
   - MESI
   - MOESI
   - 自定义状态
```

**9.2.2 可靠性**

```
1. 错误检测和报告
   - 数据错误（DERR）
   - 非数据错误（NDERR）
   - 错误传播

2. 事务重试机制
   - 资源不足时重试
   - 协议信贷管理
   - 死锁避免

3. 数据完整性
   - 数据校验
   - 数据中毒
   - 数据一致性保证
```

**9.2.3 功耗管理**

```
1. 电源门控
   - 组件激活和停用
   - 时钟门控
   - Flit级时钟门控

2. 协议活动指示
   - TXSACTIVE信号
   - RXSACTIVE信号
   - LINKACTIVE信号

3. 低功耗信号
   - 链路层低功耗模式
   - 动态电压频率调整
```

**9.2.4 安全性**

```
1. TrustZone支持
   - 事务级别的安全隔离
   - 安全和非安全事务
   - 物理地址空间（PAS）

2. 领域管理扩展（RME）
   - 多领域支持
   - 领域隔离
   - 内存加密上下文

3. 内存标记扩展（MTE）
   - 内存标签检查
   - 标签一致性
   - 标签操作
```

-------------------------------------------------------
总结
-------------------------------------------------------

1 一致性协议的核心价值
-------------------------

AMBA CHI一致性协议提供了一套完整、高效、可扩展的缓存一致性解决方案：

**核心优势：**
```
1. 完整的一致性保证
   - 确保所有代理对内存的一致视图
   - 支持多核、多处理器系统
   - 维护内存访问的全局顺序

2. 高性能优化
   - 支持数据转发
   - 支持监听过滤器
   - 支持部分缓存行状态
   - 支持事务组合

3. 灵活的架构
   - 支持多种缓存模型
   - 支持多种拓扑结构
   - 支持多种节点类型
   - 可配置的参数和属性

4. 可靠性保证
   - 错误检测和报告
   - 事务重试机制
   - 数据完整性保证

5. 现代化特性
   - 支持原子操作
   - 支持缓存维护操作
   - 支持持久化
   - 支持内存标记
   - 支持领域管理
```

**10.2 关键设计原则**

```
1. 分层架构
   - 协议层：事务级通信
   - 网络层：数据包路由
   - 链路层：流控制

2. 写使无效机制
   - 确保写入时使其他副本失效
   - 维护缓存一致性

3. 状态机管理
   - 七态缓存模型
   - 明确的状态转换规则
   - 静默和显式转换

4. 协议信贷机制
   - 防止资源耗尽
   - 支持流控制
   - 提高吞吐量

5. 排序保证
   - 点串行化（PoS）
   - 点一致性（PoC）
   - 事务排序规则
```

**10.3 实际应用指导**

**系统设计考虑：**
```
1. 选择合适的拓扑结构
2. 配置监听过滤器
3. 选择缓存模型
4. 配置节点类型
5. 设置协议参数
```

**性能优化建议：**
```
1. 使用监听过滤器减少监听
2. 利用数据转发减少延迟
3. 使用事务组合提高吞吐量
4. 配置适当的缓存大小
5. 优化数据通路宽度
```

**可靠性保障：**
```
1. 实现错误检测机制
2. 配置重试策略
3. 监控协议状态
4. 实现恢复机制
5. 测试一致性保证
```

**10.4 未来发展方向**

```
1. 更高的一致性性能
2. 更低的功耗
3. 更强的安全性
4. 更好的可扩展性
5. 更智能的缓存管理
```

-------------------------------------------------------
参考文献
-------------------------------------------------------

1. AMBA® CHI Architecture Specification (ARM IHI 0050G)
2. AMBA® AXI Protocol Specification (ARM IHI 0022)
3. Arm® Architecture Reference Manual for A-profile Architecture (ARM DDI 0487)
4. Arm® Architecture Reference Manual Supplement, The Realm Management Extension (RME), for Armv9-A (ARM DDI 0615)

-------------------------------------------------------
文档版本信息
-------------------------------------------------------

文档版本：1.0
创建日期：2026年3月28日
基于规范：AMBA CHI Architecture Specification Issue G
作者：iFlow CLI
版权说明：本文档基于ARM AMBA CHI Architecture Specification进行分析和总结，原始规范的版权归Arm Limited或其关联公司所有。

================================================================================