# 本地 5 槽存档 MVP 记录

## 当前状态

已在 `s1/data/war3map.j` 做了一个本地 5 槽存档版本，测试进图可用。

当前版本采用原生 Warcraft III Dialog 两页选择槽位。

- `-slot`：打开本地槽位列表。
- `-slot 1` 到 `-slot 5`：直接选择并读取对应槽位，用作命令行读档入口。
- 点击槽位 1 到 5：进入槽位操作页。
- 操作页点击 `[1] 加载角色`：读取角色。
- 操作页点击 `[3] 删除角色`：当前暂未实现。
- 操作页点击 `[0] 取消`：返回槽位列表。
- `-save`：沿用原保存流程，同时把当前职业存档写入已选择的本地槽位。

## 使用流程

1. 玩家输入 `-slot`。
2. 系统显示 5 个本地槽位。
3. 玩家点击 `1` 到 `5` 选择槽位。
4. 空槽会提示为空，暂时不做操作。
5. 已有槽位会显示操作页。
6. 在操作页点击 `[1] 加载角色` 加载角色。
7. 输入 `-save` 后，原来的保存文本照常生成，同时自动写入当前选择的本地槽位。

## 文件位置

本地槽位文件写入：

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

- `AS_ACTIVE_SLOT`：每个玩家当前选择的槽位。
- `AS_PLAY_NAME`：读写本地文件时临时占用玩家名，之后恢复。
- `AS_TABLE`：同步收到的槽位代码临时缓存。
- `AS_SAVE_DIR`：本地保存目录。
- `AS_CHUNK`：写入 PlayerName 的字符串分片长度，目前为 30。

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
- `AutoSlot_StartLoad`：本地读取槽位文件，并用 `DzSyncData` 同步。
- `AutoSlot_OnSync`：接收同步数据，缓存代码段。
- `AutoSlot_BeginLoad`：把缓存的代码段喂给原 `-load` 流程。
- `AutoSlot_SaveCurrent`：在原保存完成后写入当前槽位。
- `AutoSlot_SaveStr` / `AutoSlot_LoadStr`：本地文件写入和读取，移植自 `s1/war3map.j` 的 KLV 思路。
- `AutoSlot_Select`：命令和 Dialog 共用的槽位选择入口。
- `AutoSlot_OnDialog`：处理 Dialog 按钮点击。
- `AutoSlot_ShowSlotMenu`：显示第一页槽位列表。
- `AutoSlot_ShowActionMenu`：显示第二页槽位操作。

## 接入点

保存接入：

```jass
call AutoSlot_SaveCurrent(p9io)
```

位置在原保存文本 `PreloadGenEnd("TWRPG"+"\\"+hM[ll0l[p9io]]+".txt")` 后。

初始化接入：

```jass
call ExecuteFunc("AutoSlot_Init")
```

位置在 `main` 末尾的 `ExecuteFunc` 列表。

## 设计限制

当前版本有以下限制：

- 只保存职业存档，即原 `-save` 类型，暂不处理账号物品 `-asave`。
- 不做 ID 校验。
- 不做版本校验。
- 槽位 metadata 当前只写入，不展示。
- 读档仍依赖原地图读档流程，MVP 只负责本地文件读取和同步代码段。

## 后续可扩展

下一步可以做：

- 读取 `slotN_meta.pld` 展示更漂亮的职业名和等级。
- 保存时显示当前写入槽位。
- 支持删除/覆盖确认。
- 如需要，再扩展账号物品槽位。

## 测试记录

已进图测试通过：

- 开局可输入 `-slot N`。
- 空槽会提示为空。
- 选择槽位后 `-save` 可写入本地槽位。
- 重开地图后输入同槽位可自动读取。

当前 Dialog 版本已测试：

- `-slot` 可打开槽位菜单。
- 点击槽位可打开操作页。
- `-slot N` 命令入口保留为直接读档，方便和 Dialog 点击读档互相对照。
