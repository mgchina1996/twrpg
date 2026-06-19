# 本地 5 档存档 MVP 记录

## 当前状态

已在 `s1/data/war3map.j` 做了一个本地 5 档存档版本，测试进图可用。

当前版本采用原生 Warcraft III Dialog 两页选择存档。开局不再自动进入选英雄，先弹本地存档菜单；只有玩家选择新建角色时才进入原 `lo9li` 选英雄流程。

- `-slot`：打开本地存档列表。
- `-slot 1` 到 `-slot 5`：直接选择并读取对应存档，用作命令行读档入口。
- 点击存档 1 到 5：进入存档操作页。
- 操作页点击 `[1] 加载角色`：读取角色。
- 操作页点击 `[2] 使用该存档新建角色`：选择该存档作为后续保存位置，然后进入选英雄；不会立刻覆盖，只有后续 `-save` 才覆盖该存档。
- 操作页点击 `[0] 取消`：返回存档列表。
- `-save`：沿用原保存流程生成代码；如果当前已有本地存档编号，只写入本地存档，不再写旧的 `TWRPG\\角色名.txt` 明文保存文件。

## 使用流程

1. 开局 `l6Iui` 对每个有效玩家调用 `AutoSlot_ShowStartupMenu`。
2. 系统显示 5 个本地存档。
3. 玩家点击 `1` 到 `5` 选择存档。
4. 点击 `[1] 加载角色` 时，只走本地读档，不进入选英雄。
5. 点击 `[2] 使用该存档新建角色` 时，设置当前存档位并执行 `lo9li` 进入选英雄；此时不写文件。
6. 输入 `-save` 后，如果当前选择了本地存档，只写入 `TWRPG\\AutoSave\\slotN_*.pld`，不会再写旧的 `TWRPG\\角色名.txt`。

## 文件位置

本地存档文件写入：

```text
TWRPG\\AutoSave\\slotN_*.pld
```

当前保存内容拆成：

- `slotN_count.pld`：代码段数量。
- `slotN_meta.pld`：职业名、等级、代码段数量，当前 MVP 只写入，暂未展示。
- `slotN_code0.pld`、`slotN_code1.pld` 等：实际存档代码段。

## 代码入口

主改动文件：

```text
s1/data/war3map.j
```

新增全局：

- `AS_ACTIVE_SLOT`：每个玩家当前选择的存档。
- `AS_PLAY_NAME`：读写本地文件时临时占用玩家名，之后恢复。
- `AS_TABLE`：同步收到的存档代码临时缓存。
- `AS_SAVE_DIR`：本地保存目录。
- `AS_CHUNK`：写入 PlayerName 的字符串分片长度，目前为 30。
- `AS_STARTUP_MENU`：标记当前存档位菜单是否来自开局流程。
- `AS_PENDING_NEW`：延后执行的新建角色请求，用来避开 JASS 函数顺序限制。

新增 native：

- `DzSyncData`
- `DzTriggerRegisterSyncData`
- `DzGetTriggerSyncPlayer`
- `DzGetTriggerSyncData`

新增函数前缀：

```text
AutoSlot_*
```

关键函数：

- `AutoSlot_Init`：注册 `-slot` 命令、同步事件、开局提示。
- `AutoSlot_OnSlot`：处理 `-slot` 命令。
- `AutoSlot_StartLoad`：本地读取存档文件，并用 `DzSyncData` 同步。
- `AutoSlot_OnSync`：接收同步数据，缓存代码段。
- `AutoSlot_BeginLoad`：把缓存的代码段喂给原 `-load` 流程。
- `AutoSlot_SaveCurrent`：在原保存流程生成代码后写入当前存档位。
- `AutoSlot_SaveStr` / `AutoSlot_LoadStr`：本地文件写入和读取，移植自 `s1/war3map.j` 的 KLV 思路。
- `AutoSlot_Select`：命令和 Dialog 共用的存档选择入口。
- `AutoSlot_RequestNew`：记录新建角色请求，通过 `ExecuteFunc("AutoSlot_RunPendingNew")` 延后处理。
- `AutoSlot_RunPendingNew`：位于 `lo9li` 后面，真正设置存档并进入选英雄。
- `AutoSlot_OnDialog`：处理 Dialog 按钮点击。
- `AutoSlot_ShowSlotMenu`：显示第一页存档列表。
- `AutoSlot_ShowStartupMenu`：开局显示第一页存档列表，不自动进入选英雄。
- `AutoSlot_ShowActionMenu`：显示第二页存档操作。

## 接入点

保存接入：

```jass
call AutoSlot_SaveCurrent(p9io)
```

位置在原保存文本 `PreloadGenEnd("TWRPG"+"\\"+hM[ll0l[p9io]]+".txt")` 后。

当前实现中，`i9_0i` 增加了 `autoOnly` 判断：

```jass
local boolean autoOnly=AS_ACTIVE_SLOT[ol6O]>=1 and AS_ACTIVE_SLOT[ol6O]<=5 and FM[ll0l[p9io]]==1 and not llOl[p9io]
```

当 `autoOnly` 为 true 时，跳过旧的 `TWRPG\\角色名.txt` 写入，只调用 `AutoSlot_SaveCurrent(p9io)` 写本地存档。

初始化接入：

```jass
call ExecuteFunc("AutoSlot_Init")
```

位置在 `main` 末尾的 `ExecuteFunc` 列表。

开局接入：

```jass
call AutoSlot_ShowStartupMenu(Player(OuOo))
```

位置在 `l6Iui` 原本调用 `lo9li(Player(OuOo))` 的地方。现在 `lo9li` 只在新建角色时执行。

## 设计限制

当前版本有以下限制：

- 只保存职业存档，即原 `-save` 类型，暂不处理账号物品 `-asave`。
- 不做 ID 校验。
- 不做版本校验。
- 存档 metadata 当前只写入，不展示。
- 读档仍依赖原地图读档流程，MVP 只负责本地文件读取和同步代码段。
- 当前没有删除角色入口。

## 后续可扩展

下一步可以做：

- 读取 `slotN_meta.pld` 展示更漂亮的职业名和等级。
- 保存时显示当前写入存档。
- 支持覆盖二次确认。
- 如需要，再增加删除存档。
- 如需要，再扩展账号物品存档。

## 测试记录

已进图测试通过：

- 开局可输入 `-slot N`。
- 空会提示为空。
- 选择存档后 `-save` 可写入本地存档。
- 重开地图后输入同存档位可自动读取。

当前 Dialog 版本已测试：

- `-slot` 可打开存档菜单。
- 点击存档可打开操作页。
- `-slot N` 命令入口保留为直接读档，方便和 Dialog 点击读档互相对照。
- 当前改动后需要测试：开局是否先弹存档菜单、加载角色是否不进选英雄、新建角色是否才进入选英雄。
