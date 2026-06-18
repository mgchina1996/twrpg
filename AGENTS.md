这是 Warcraft III 地图项目。

war3map.j 是地图脚本文件，只在需要查找 JASS 逻辑时使用。
war3map.w3a.json 是技能数据。
war3map.w3t.json 是物品数据。
war3map.w3u.json 是单位数据。

重要规则：
- 不要完整读取 war3map.j。
- war3map.j 很大，只能用 rg/grep 按关键词搜索。
- 优先分析 data/war3map.w3a.json、data/war3map.w3t.json、data/war3map.w3u.json。
- 每次只处理一个小任务，不要全项目扫描。

