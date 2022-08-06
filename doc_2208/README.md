# 22年八月

- [线上事故总结](#线上事故总结)
    - [天赋模块](#天赋模块)

- [线上后端优化](#线上后端优化)
    - [活动道具清理](#活动道具清理)
    - [佣兵装备](#佣兵装备)
    - [佣兵登陆信息](#佣兵登陆信息)
    - [公会成员列表](#公会信息获取)

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


    线上后端优化
    ====
    
    活动道具清理
    ====
    
    我们活动结束后经常需要回收玩家的道具，发一些邮件，之前是对于每个not open的活动，回收活动里的道具。大概这样：
    ```lua
    
    for _, id in ipairs(activity_ids) do
        local act = mgr:get_act(player_id, id)
        if not act:is_open() then
            act:clear_items()
        end
    end
    
    ```
    
    随着活动表越配越多，这个方式出现了很大的问题，is_open的时候，一般是hmget一下开始结束时间和当前时间判断，clear的时候，再hmget几个活动的具体数据拿出来看看，我们一般hash存，拿出来split，split的代码还是string.find, string.sub的（这里我一直想换成ngx_re，但所有活动都在用这个，还得慎重考虑一下）。这里冗余操作就很多，一个是频繁判断is_open，要hmget，第二个是关闭的活动反复清理，还要hmget，这个get出来还得split
    
    大概思路是找个set把已经清的存一下，内网开活动的时候记一个count给他删掉
    
    ```lua
    local members = rd:smembers(act_clear_key)
    local len = #members
    local is_member = table.new(0, len)
    for i = 1, len do
        is_member[tonumber(members[i])] = true
    end

    local real_clears = {}
    for _, id in ipairs(activity_ids) do
        if not is_member[id] then
            local act = self:get_act(player_id, id)
            --[[
                这里没有批量获取is open，是因为不同活动is open判断不同
                这里统一判断会导致一些活动不清理
                成功清理的放到已经清理列表，这样尽量保证玩家列表都是玩家见过的活动
            ]]
            local f = act:clear_items()
            if f then
                table.insert(real_clears, id)
            end
        end
    end
    rd:sadd(act_clear_key, unpack(real_clears))
    ```
    就是每个玩家维护了一个已经清理过的set，这个后续还有优化空间，主要是针对内存和存取，但是我并不打算过早优化。第一是分key避免活动太多，目前的线上redis实例 intset的最大值是512，需要清理的活动在80左右，按照每周增加2个需要清理的活动，增加到512是超过6年的时间。且我只会放玩家看到过的活动，也就是玩了6年游戏的玩家的intset才会变成hashset，导致存储变大。我们游戏他真玩六年就让他大一点好了我觉得233333. 第二是改成sscan，也没必要，smembers不过On，拿1000以内完全不会对redis造成影响，他真1000个活动了，那个redis估计已经没人了（我们递增玩家id分配redis），第三是改为bitset，也没必要，主要是我怕策划配了个99999活动直接G了，很难对策划有要求说活动id不能太大，而且bit全量拿比较麻烦。优化不应该被过度设计，现在这样简单能用，火焰图体现明显，是合理的优化。
    
    佣兵装备
    ====
    
    前同事略摆，前端算个装备算不对，每次战斗都问后端请求佣兵具体信息。
    这个就让前端把后端代码抄一遍，不要每次都请求。
    
    佣兵登陆信息
    ====
    
    这个火焰图吓我一跳，一个系统顶七八个竞技场的消耗。
    这个主要是登陆的时候要拿公会+好友的佣兵具体信息（占了这个模块98%的CPU，总登陆的50%以上），完全没必要，登陆的时候不给这些信息，点进系统才给。
    
    公会信息获取
    ====
    
    公会成员信息这里比较卡，主要是个人信息多，相当于70个人走70个hmget，中间还有很多hget，大概200个左右redis操作，火焰图发现堆栈是get_info->redis:read_body, get_active->redis:read_body
    这种就是redis操作太多次了，pipeline可以优化时间，CPU应该可以相应减少，body总大小没变，相当于连接取出到唤醒的次数变少了，redis发一下把自己挂起，调度别人，这个过程是epoll做的，相当于还是有不小的CPU调度消耗，我们项目没有off cpu火焰图，但是他一定和这种openresty网络io相关。效果上确实下来了。