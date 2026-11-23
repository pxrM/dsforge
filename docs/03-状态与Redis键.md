# 状态与 Redis 键

记录日期：2026-11-23。源码基线：`b0332a8a8bee9572d673acf1923d1042fe74c41d`。
键名来自常量和读写函数；表中的状态链路用于理解预期生命周期，不代表代码已保证所有转换原子且合法。

## 源码事实：键与类型

主要依据：[team.go](../dsserver/gameserver/team/team.go) 的常量、`InitScript`；[keys.go](../dsserver/store/dsastore/keys.go)；[store.go](../dsserver/store/dsastore/store.go) 的 `WriteRealmSkeleton`、`CreateDSMeta`、`RegisterAgent`。

表中的 `<id>`、`<uid>` 为占位符；花括号为真实键名的一部分。

| 键名 | 类型 | 内容与用途 |
| --- | --- | --- |
| `{team}Zset` | ZSet | 匹配队伍索引；分数主要用于开始/超时扫描 |
| `{team}_<id>` | Hash | teamID、realmCfgID、state、timeout、forceStart 等 |
| `{team}_<id>_uidList` | List | 队伍成员 UserID |
| `{team}u:<uid>` | String | 玩家 → 队伍键 |
| `{realm}_<id>` | Hash | Realm 状态、DSID、地址、更新时间等 |
| `{realm}_<id>_uidList` | List | 对局成员 UserID |
| `{realm}u:<uid>` | String | 玩家 → RealmID |
| `{realm}2ds:<id>` | String | RealmID → DSID |
| `{realm}Index` | Set | 用于巡检的 RealmID 索引 |
| `{dsc}ds:<id>` | Hash | DSID、RealmID、AgentID、状态、地址、更新时间 |
| `{dsc}dsIndex` | Set | DSID 索引 |
| `{dsc}agent:<id>` | Hash | 地址、容量、dsCount、心跳、模板就绪状态等 |
| `{dsc}agents` | ZSet | Agent 选机索引，分数按 dsCount 更新 |

通知频道为 `realm:ready:<realmID>` 和 `realm:fail:<realmID>`。频道不是可查询历史消息的普通状态键；接收逻辑见 watch.go `subscribeLoop`。

### 状态定义

| 对象 | 定义 | 预期主流程 |
| --- | --- | --- |
| Team | 0 matching、1 closed | matching → closed → 删除队伍 |
| Realm | creating、ready、ingame、fail、ended | creating → ready → ingame → ended；异常进入 fail |
| Redis DS | creating、ready、playing、ended、dead | creating → ready → playing → ended；异常进入 dead |
| Agent | online、offline | 注册/心跳维持 online，巡检超时标记 offline |
| DSA 本地 DS | DS_CreatIng、DS_CreatOK | 本地内存管理，与 Redis DS 状态不是同一组常量 |

状态定义见 keys.go、team.go、[dsmgr.go](../dsserver/dsaserver/dsa/dsmgr.go)。
`alloc` 是 Realm Hash 中的分配占位字段，不是对局业务状态。
`templateReady` 是 Agent 的可调度条件之一，也不是 online/offline 的替代状态。

### 状态写入和清理函数

| 函数 | 责任 |
| --- | --- |
| `TryClaimRealmAlloc` | Lua 抢占 Realm 分配权并返回是否已创建/创建中 |
| `ReserveOptimalAgent` | Lua 选 Agent 并增加容量预占计数 |
| `CreateDSMeta` | 分两组写入 `{dsc}` 元数据和 `{realm}` 绑定 |
| `MarkRealmReady` | 写就绪状态与地址，清 alloc，随后发布通知 |
| `MarkRealmFail` / `MarkRealmEnded` | 标记失败或结束 |
| `DeleteDSMeta` | 解绑并删除 DS 元数据；实际删除成功后释放容量槽 |
| `CleanupRealm` | 删除对局 Hash、成员 List、用户映射、DS 映射和索引 |
| `SweepRealmFragments` | 回收超时的残片、终态或无有效 DS 的 Realm |

以上函数均位于 store/dsastore/store.go。Pipeline 是批量命令机制，不能仅因使用 Pipeline 就推断它具有事务原子性。

### 默认超时

依据：[assets/config.json](../dsserver/assets/config.json)、[config.go](../dsserver/config/config.go) 的 `applyDefaults`。实际运行值还需核对加载路径及环境变量覆盖。

| 配置 | 默认值 |
| --- | --- |
| 匹配等待 | 10 秒 |
| 玩家断连宽限 | 60 秒 |
| Agent 心跳间隔 / 超时 | 3 秒 / 15 秒 |
| DS 创建超时 | 60 秒 |
| ready 后空闲超时 | 120 秒 |
| playing 更新超时 | 3600 秒 |
| DSC 巡检间隔 | 30 秒 |

## 自己的理解（待核对）

- 用户映射便于登录恢复；索引便于后台巡检；实例元数据便于故障定位和回收。
- 相同 hash tag 让相关键落在同一槽，但也将这一类数据集中到单个槽。
- `{team}` 与 `{realm}` 不同，转换队伍和写入对局之间需要处理失败窗口。
- 实例 ready、玩家收到通知、玩家真正进入游戏是三个需要分别验证的事件。

## 待验证问题

- [ ] 逐步记录创建前、creating、ready、ended 和清理后的键值。
- [ ] 重复回收同一个 DS 后，Agent dsCount 是否只减少一次？
- [ ] 清理旧 Realm 是否可能误删玩家新 Realm 的映射？
- [ ] 并发结束与迟到 READY 是否可能使终态被覆盖？
- [ ] 核对哪些键有 TTL、哪些依赖显式清理，记录异常退出后的残留。

## 键值观察模板

- 日期、环境、源码提交：
- 关联对象 ID：
- 操作前的状态与计数：
- 触发动作、读取命令与结果：
- 操作后的状态与计数：
- 是否满足“无重复实例、无残留映射、容量和端口正确释放”：
