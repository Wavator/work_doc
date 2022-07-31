# 22年八月

- [线上事故总结](#线上事故总结)
    - [天赋模块](#天赋模块)


线上事故总结
====



天赋模块
====
- 这个月上线了天赋系统，但是测试覆盖面在我看来不尽人意，一些战斗模式下战报100%报错，战力计算模块100%报错。后端无法避免第二个问题（只能说依赖的其他模块扔进ngx.timer异步处理且不报警）。第一个问题排查下来是sql防注入的问题。
    在原本的后端代码中，战报存储有两种写法：
    ```lua
    local battle_report = self:get_report()
    local ok, json_report = pcall(cjson.encode, battle_report)
    if not ok then
        ngx.ctx.error_msg = 'report encode err'
        return false
    end
    
    -- 第一种
    local sql = [[
        insert into report_db (report_id, player_id, report, time)
        values
        (%d, %d, %s, %d)
    ]]
    sql = string.format(sql, battle_report.report_id, self.player_id, ngx.quote_sql_str(json_report), ngx.time())
    ngx.ctx.mysql:query(sql)
    
    -- 第二种
    local sql = [[
        insert into report_db (report_id, player_id, report, time)
        values
        (%d, %d, '%s', %d)
    ]]
    sql = string.format(sql, battle_report.report_id, self.player_id, json_report, ngx.time())
    ngx.ctx.mysql:quary(sql)
    ```
    
    拿战报的时候(简单举一种例子)
    ```lua
    local sql = [[
        select * from report_db where player_id = %d limit 10
    ]]
    sql = string.format(sql, self.player_id)
    local res = ngx.ctx.mysql:query(sql)
    if not res or not next(res) then
        return {}
    end
    
    local results = {}
    for i = 1, #res do
        local json_report = res[i].report
        local report = cjson.decode(json_report)
        table.insert(results, report)
    end
    return results
    ```
    
    发现在第二种存法下，拿的时候decode有些情况会报错
    原因是我们本来战斗有一个额外参数数组，数组里存了json str，就大概类似于
    {arr: ["{"val":1}"]}
    乍一看也挺对的，仔细一看啥都不对，cjson decode的时候会就近匹配一个引号，也就是"{"这一段，导致他认为这是个字符串，所以他觉得val的v应该是逗号或者右中括号（关闭json arr或者下一个str），这样就出问题了。这几个用了2的模式的代码在项目中稳定运行了比较久了，这个额外的json str是这周版本加进去的，报错了一时很难确认是存的有问题还是传的有问题（前端给后端一样传的json str），后面其他模式打了一下感觉战报没有问题，取的代码一样那只能是存的问题了，把第二种改成第一种，问题解决。
    
    解决了问题之后我复盘了一下，这个第二种情况，其实没有防止sql注入，只是不加引号不让往SQL里insert，因为分析器会报错，不加引号和上面report的类型对不上。所以第二种情况做法上就是不安全的. 不但不安全，'\'不经过正确转义还会被吃掉
    写了一段测试代码
    ```lua
    local cjson = require 'cjson'
    local val_str = cjson.encode({val = 1})

    local arr = {
        x = {
            val_str
        }
    }
    local report = cjson.encode(arr)

    local sql = [[
        insert into report_db (activity_id, stage_id, player_id, content, created_time)
        values
        (%d, %d, %d, '%s', %d)
    ]]
    sql = string.format(sql, 100, 1, 1000000001, report, ngx.time())
    print(sql)

    local ok, err =ngx.ctx.mysql:query(sql)
    if not ok then
        print(err)
    end
    print('before ' .. report)

    sql = [[
        select * from report_db where player_id = 1000000001 limit 1
    ]]

    local res, err = ngx.ctx.mysql:query(sql)
    print('after ' .. res[1].content)
    ```
    运行结果
    ```shell
    before {"x":["{\"val\":1}"]}
    after {"x":["{"val":1}"]}
    ```
    可以看到，这个str不被正确转义的话，json encode之后再被encode用来标记"的\会被干掉，原因是因为ngx的mysql认为'report'是被转过的，他存进去的时候会同时去掉''和第一层\，导致第一层的转义消失了，所以出现了上面的两个encode之后decode失败的问题。
- ngx.null 的问题，天赋可以借给佣兵使用，佣兵有比较多的pipeline操作，也就是很多类似于
    ```lua
    rd:init_pipeline()
    for _, partner_id in ipairs(partner_ids) do
        local key = get_key(self.player_id, partner_id)
        rd:hmget(key, ...)
    end
    local rd_res = rd:commit_pipeline()
    ```
    他这个有很多，其中一处我忘了判断 == ngx.null，塞到了redis里，然后拿的时候我加了== ngx.null，这个时候已经失效了，因为他塞进redis的时候就成了'userdata: NULL'，和ngx.null并不相等，额外增加一层字符串的判断，问题解决
    
    这个我觉得我的问题是改的不完全，漏改了一些地方。
    原始代码为什么一个文件里拿partner info要在六个地方写hmget而不封装一层，我一时也无力吐槽。踩了雷肯定就是自己的问题，以后注意吧。

- 家具效果在一些战斗模块中完全失效
    这个就是bug，没什么技术含量，原理是一张config的表传参数的时候会被覆盖，以后要尽量用
    ```lua
        x.config = x.config or {}
        x.config.val = xxx
    ```
    
    而不是直接`x.config = {val = xxx}`
    当然这个跟config的设计理解有关，我理解的是config是具体模式的配置，比如剩余血量，能量，布阵的位置等等。数据应该不在这张表维护，比如装备，等级，品质这些都是在基本的info里维护的，家具作为英雄的数据，应该放在info里。但是战斗模块开发好之后跟我说要在config里，我直接改动了上游接口，也就是
    ```lua
    function _M:get_info()
        local info = self:get_info()
        info.config = info.config or {}
        -- 这里就比较丑，他本来在info里，但是战斗这样要求也没办法
        info.config.furniture = info.furniture
        return info
    end
    ```
    然后下游其他战斗模式调用的时候，很多模块（包括以前不是我开发的模块），对config的理解都是具体某一场的配置，所以很多都是`x.config = {val = xxx}` 这种写法，用于初始化自己模块的战斗数据，这样就把我传的config覆盖了。
    
    总结就是写完再仔细看看，当时内网发现了这个问题，所以测了的战斗模块改掉了，但是有些模块上了家具似乎完全没有测试。我觉得没有预警到巅峰竞技场的config有类似问题是我的问题，但是没测这个就和时间太紧有关系吧。
    
    为了提升周末的生活质量，写的时候还是要谨慎小心。


    