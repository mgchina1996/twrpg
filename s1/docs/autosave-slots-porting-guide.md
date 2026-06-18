# Warcraft III 本地 5 槽存档移植方案

这份文档记录一套可移植到其他地图的本地 5 槽存档逻辑。核心目标是：

- 玩家进图选择固定槽位。
- `-save` 时自动写入本地文件。
- 下次进图选择同一槽位后自动读取本地文件。
- 本地读取的数据通过同步接口发给所有玩家，再进入地图原有读档流程。

## 适用前提

地图需要满足：

- 已有可用的保存代码生成逻辑。
- 已有可用的读档逻辑，能把一段或多段存档代码加载进游戏。
- 地图运行环境支持本地文件写入读取，即 `PreloadGenStart`、`PreloadGenEnd`、`Preloader`。
- 地图运行环境支持数据同步 native：

```jass
native DzSyncData takes string prefix,string data returns nothing
native DzTriggerRegisterSyncData takes trigger trig,string prefix,boolean server returns nothing
native DzGetTriggerSyncPlayer takes nothing returns player
native DzGetTriggerSyncData takes nothing returns string
```

## 核心思想

Warcraft III 多人游戏里，本地文件只能由本地玩家读取。如果直接用本地读取结果改游戏状态，会不同步。

正确流程是：

1. 本地玩家读取自己的槽位文件。
2. 本地玩家把读取到的存档代码通过 `DzSyncData` 发出去。
3. 所有客户端收到同步数据。
4. 所有客户端把同步来的代码段交给原地图读档流程。
5. 原读档流程负责真正创建英雄、恢复装备、恢复属性等。

也就是说，这套逻辑不替代原读档，只解决“自动从本地文件拿到读档代码”。

## 文件写入方式

本地文件写入使用 `PreloadGen` 注入代码。

常见写法：

```jass
call PreloadGenClear()
call PreloadGenStart()
call Preload("\" )\n    call SetPlayerTechMaxAllowed(Player(14),0,"+I2S(count)+")\n    //")
call Preload("\" )\n    call SetPlayerName(Player(6),\""+chunk+"\")\n    //")
call PreloadGenEnd("MapName\\AutoSave\\slot1_code0.pld")
```

读取时：

```jass
call Preloader("MapName\\AutoSave\\slot1_code0.pld")
set count=GetPlayerTechMaxAllowed(Player(14),0)
set value=GetPlayerName(Player(6))
```

注意：

- `Preload` 字符串里的 `\n` 必须保留为 JASS 字符串内容。
- 通常用 `Player(6)` 到 `Player(15)` 的名字临时存字符串分片。
- 读完后要恢复这些玩家名，避免影响游戏显示。
- 读取不存在的文件时，要用哨兵值或范围判断防止误读旧值。

## 建议文件结构

固定 5 个槽位，每槽保存：

```text
MapName\\AutoSave\\slotN_count.pld
MapName\\AutoSave\\slotN_meta.pld
MapName\\AutoSave\\slotN_code0.pld
MapName\\AutoSave\\slotN_code1.pld
MapName\\AutoSave\\slotN_code2.pld
...
```

说明：

- `slotN_count.pld`：该槽位存档代码段数量。
- `slotN_meta.pld`：展示用信息，例如职业名、等级、保存时间。
- `slotN_codeX.pld`：实际存档代码段。

如果原地图保存代码只有一段，也可以只写 `slotN_code0.pld`。

## 推荐全局变量

```jass
integer array AS_ACTIVE_SLOT
string array AS_PLAY_NAME
hashtable AS_TABLE=InitHashtable()
string AS_SAVE_DIR="MapName\\AutoSave\\"
integer AS_CHUNK=30
```

用途：

- `AS_ACTIVE_SLOT[playerId]`：玩家当前选择的槽位。
- `AS_PLAY_NAME`：备份 PlayerName，读写后恢复。
- `AS_TABLE`：缓存同步收到的代码段。
- `AS_SAVE_DIR`：本地保存目录。
- `AS_CHUNK`：每个 PlayerName 分片长度，建议 30 左右。

## 功能函数拆分

建议拆成这些函数，方便移植：

```text
AutoSlot_SaveStr
AutoSlot_LoadStr
AutoSlot_SaveCurrent
AutoSlot_StartLoad
AutoSlot_OnSync
AutoSlot_BeginLoad
AutoSlot_OnSlot
AutoSlot_Init
```

职责如下：

- `AutoSlot_SaveStr`：把字符串写入本地 `.pld` 文件。
- `AutoSlot_LoadStr`：从本地 `.pld` 文件读取字符串。
- `AutoSlot_SaveCurrent`：从原保存流程中取出存档代码段，写入当前槽位。
- `AutoSlot_StartLoad`：本地读取当前槽位，并逐段 `DzSyncData`。
- `AutoSlot_OnSync`：接收同步数据，按玩家和槽位缓存代码段。
- `AutoSlot_BeginLoad`：所有代码段收到后，调用原地图读档流程。
- `AutoSlot_OnSlot`：处理 `-slot 1` 到 `-slot 5`。
- `AutoSlot_Init`：注册同步事件、聊天命令、开局提示或弹框。

## 接入原保存流程

在原保存成功、代码段已经生成之后，调用：

```jass
call AutoSlot_SaveCurrent(playerIdOrSaveProcessId)
```

这个函数内部要做三件事：

1. 判断玩家是否选择了 1 到 5 的槽位。
2. 从原保存系统拿到代码段数量和每段代码。
3. 写入 `slotN_count.pld` 和 `slotN_codeX.pld`。

不同地图要替换的部分就是“从原保存系统拿代码段”。

示例伪代码：

```jass
set count=OriginalSave_GetCodeCount(pid)
call AutoSlot_SaveStr(Player(pid),"slot"+I2S(slot)+"_count",I2S(count))

set i=0
loop
    exitwhen i>=count
    call AutoSlot_SaveStr(Player(pid),"slot"+I2S(slot)+"_code"+I2S(i),OriginalSave_GetCode(pid,i))
    set i=i+1
endloop
```

## 接入原读档流程

收到同步数据后，不要自己恢复英雄。应该调用原地图已有的读档入口。

示例伪代码：

```jass
call OriginalLoad_Start(pid)

set i=0
loop
    exitwhen i>=count
    call OriginalLoad_InputCode(pid,LoadStr(AS_TABLE,parent,i),true)
    set i=i+1
endloop
```

不同地图要替换的部分就是：

- `OriginalLoad_Start`
- `OriginalLoad_InputCode`

如果原地图读档就是聊天 `-load xxxx`，也可以把同步到的代码交给同一套解析函数。

## 同步数据格式

推荐简单格式：

```text
pid|slot|index|count:code
```

含义：

- `pid`：玩家 ID。
- `slot`：槽位 1 到 5。
- `index`：当前第几段代码，从 0 开始。
- `count`：总段数。
- `code`：实际存档代码。

收到后：

1. 解析 `pid`、`slot`、`index`、`count`。
2. 用 `pid*10+slot` 作为 hashtable parent key。
3. `index` 作为 child key 保存代码段。
4. 收齐 `count` 段后开始读档。

## 命令版 MVP

最小版本可以先不用 Dialog：

```text
-slot 1
-slot 2
-slot 3
-slot 4
-slot 5
```

处理逻辑：

```jass
set AS_ACTIVE_SLOT[pid]=slot
call AutoSlot_StartLoad(pid,slot)
```

保存时：

```jass
if AS_ACTIVE_SLOT[pid]>=1 and AS_ACTIVE_SLOT[pid]<=5 then
    call AutoSlot_SaveCurrent(pid)
endif
```

## 弹框版扩展

后续可以把 `-slot N` 改成开局 Dialog：

- 按钮 1：槽位 1，显示 `职业 等级`。
- 按钮 2：槽位 2。
- 按钮 3：槽位 3。
- 按钮 4：槽位 4。
- 按钮 5：槽位 5。

Dialog 显示内容来自 `slotN_meta.pld`。

选择按钮后仍然调用：

```jass
set AS_ACTIVE_SLOT[pid]=slot
call AutoSlot_StartLoad(pid,slot)
```

底层逻辑不用变。

## 常见坑

### 不能本地读完直接改游戏状态

本地文件读取只在本地玩家机器发生，直接创建英雄或改属性会不同步。

必须用 `DzSyncData` 把结果同步出去。

### 不要用 JASS 关键字当变量名

例如：

```jass
local string code
```

`code` 是 JASS 类型关键字，会导致编译错误。

可以改成：

```jass
local string saveCode
```

### 读文件前后要处理 PlayerName

用 `SetPlayerName` 临时存字符串后，要恢复玩家名。

### 空文件要防误读

读取不存在的 `.pld` 时，`GetPlayerTechMaxAllowed` 可能保留旧值。

建议：

```jass
set count=GetPlayerTechMaxAllowed(Player(14),0)
call SetPlayerTechMaxAllowed(Player(14),0,2147483645)
if count<0 or count>10 then
    set count=0
endif
```

### 每段字符串不要太长

PlayerName 承载能力有限，建议每段 30 字符左右。

如果存档代码很长，先拆成多段文件。

## 当前项目落地参考

当前地图的落地版本在：

```text
s1/data/war3map.j
```

项目记录在：

```text
s1/docs/autosave-slots.md
```

