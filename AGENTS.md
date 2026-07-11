这是 Warcraft III 地图项目。

war3map.j 是地图脚本文件，只在需要查找 JASS 逻辑时使用。
war3map.w3a.json 是技能数据。
war3map.w3t.json 是物品数据。
war3map.w3u.json 是单位数据。
war3map.w3h.json 是效果/装饰物数据。

重要规则：
- 不要完整读取 war3map.j。
- war3map.j 很大，只能用 rg/grep 按关键词搜索。
- war3map.w3u.json、war3map.w3t.json、war3map.w3h.json、war3map.w3a.json 只能查看分析，不要编辑；物编数据由用户在物编中修改。
- 优先分析 data/war3map.w3a.json、data/war3map.w3t.json、data/war3map.w3u.json、data/war3map.w3h.json。
- 每次只处理一个小任务，不要全项目扫描。
- JASS 函数必须定义在调用前；新增辅助函数时要放到首次调用位置之前。


每次修改都要写到s1\更新日志.txt，简洁的，按照顺序1,2,3...，相同的机制最好写到一条,文档类的更新不需要写到更新日志。

每次修改都移除native DzAPI_Map_GetPlayerUserName takes player whichPlayer returns string
然后添加以下函数用于测试
function DzAPI_Map_GetPlayerUserName takes player whichPlayer returns string
return "Heavy丶rain#5346"
endfunction
DzAPI_Map_GetPlayerUserName函数定义位置，统一放在native声明段之后。
