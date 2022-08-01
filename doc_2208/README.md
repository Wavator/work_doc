# 22年八月

- [线上事故总结](#线上事故总结)
    - [天赋模块](#天赋模块)

- [OpenResty](#OpenResty)
    - [LuaJit](#luajit)
        - [测试代码](#test)
        - [table优化](#table)
        - [string优化](#string)
        - [os.*](#os)

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



    OpenResty
    ====

    
    这周比较系统的学习了一下OpenResty（当然说掌握还差很多，但也收获不小），看看平时开发/测试的时候还是有一些坑，要进行规避
    
    luajit
    ====
    
    LuaJit ~= Lua5.1，用的时候还是要仔细选择高性能接口。Jit的原理基础的都懂，高深的暂时没有研究的想法。
    
    另外看github有个开源工具[lj-releng](https://github.com/openresty/openresty-devel-utils/blob/master/lj-releng),可以扫Jit2.1语法，现在内网vim是Lua5.1语法，table.new是红色的，试试在内网给他进行一个装。
    
    test
    ====
    
    OpenResty的Lua是LuaJit2.1，测试的时候有点不一样，根据其作者的建议，如果我们要测试一段代码(比如这个func)，那他应该要长这样：
    ```lua
    local function test(func, ...)
        -- 让LuaJit觉得这段代码过热从而翻译成字节码
        for i = 1, 10000 do
            func(...)
        end
        ngx.update_time()
        local t = ngx.now()
        func(...)
        ngx.update_time()
        print(ngx.now() - t, 's')
    end
    local function test_a()
    ....
    end
    test(test_a)
    ```
    下面第一个模块table的时候会比对一下不循环很多次的时候和循环很多次的时候的区别
    
    table
    ====
    
    新操作还是不多的，concat之类的操作项目里会经常用。以前知道table.new，但一直用的比较少，这次来实际对比一下有什么区别
    测试code大概是
    ```lua
    local sz = 1000000

    local function test_a(real)
        local a = table.new(sz, 0)
        for i = 1, sz do
            a[sz] = sz - i
        end
    end
    -- 0.0049998760223389s
    local function test_b(real)
        local a = {}
        for i = 1, sz do
            a[i] = sz - i
        end
    end

    -- 0.027999877929688s
    ```
    性能差距还是有点大的，sz去掉一个0（这个sz跑insert根本跑不完，O(t*n^2)，去掉一个0我都跑了特别久），测试一下`table.insert`，比第二个还慢几十倍，原因是每次取#是On的操作，虽然LuaJit官网说可以优化append类型的insert，但是还是很慢，估计#操作并不好优化，不然应该是慢2倍而不是几十倍了。另外不循环很多次前两个时间上没有区别。
    实际项目中，对于已知大小的table/hashtable，应当采用`table.new`，`table.insert`虽然代码可读性很高，但是还是减少使用，冷热代码均是如此，原因是因为对于没被Jit优化的代码他是真的O(n^2)，优化过之后也有几十倍的性能差距
    
    table.clear和table池估计实际用不到，不测了，等table new成为性能瓶颈可以回来看看
    
    
    string
    ====
    
    - 字符串拼接
        比较基础，项目里我也用的比较多，`table.concat`优化，原理层面也比较简单，就是Lua的字符串理解成一个final类型的量，每次Jit会去string池检查有没有这个final的量，有就把新字符串指向他，没有就新建一个扔进去。Gc的时候没有引用的就删掉了，所以顺序拼接"a", "b", "c"m "d"内存里会有"a", "b", "c", "d", "ab", "abc", "abcd", concat可以把中间的省掉，会省很多内存（这里看上去小是因为string每个只有一个字母，长了就明显了。这个项目里str的基类拼接没用，我改了一下。
    
    - string.find,这个我看最新的[luajit NYI](http://wiki.luajit.org/NYI)里面,他说Only plain string searches (no patterns). 也就是你往里面扔一坨正则不行，不但不行，他的意思其实是要把find的第四个参数传true才行,他本来是`string.find(s, pattern [, init [, plain]] )`，要显式声明第四个true。这个我看项目里str库的代码传了，之前的大佬太猛了。但是后面一些str操作就没有了，如果哪天项目测试压力小一点，考虑给项目里面换成ngx.re，因为用的地方太多了，换的话就也不太好换，但先记住这个find很蠢
    - string.byte，据说这个是神器，string.char(string.byte("abc", 1, 2)) 比string.sub的写法少生成很多中间string，让我来试试，测试用1000长度的字符串随机两个pos，分别用string.sub(posa, posb), string.char(string.byte("abc", posa, posb))比对运行时间
        ```lua
        local sz = 1000

        local str
        
        local a = table.new(sz, 0)
        for i = 1, sz do
            a[i] = math.random(9)
        end
        
        str = table.concat(a)
        
        -- 0s
        local function test_a()
            local t = table.new(1000, 0)
            for i = 1, 1000 do
                local pa = math.random(sz - 100)
                local pb = pa + math.random(sz - pa)
                t[i] = string.sub(str, pa, pb)
            end
        end
        
        -- 0.0019998550415039s
        local function test_b()
            local t = table.new(1000, 0)
            for i = 1, 1000 do
                local pa = math.random(sz - 100)
                local pb = pa + math.random(sz - pa)
                t[i] = string.char(string.byte(str, pa, pb))
            end
        end
        ```
        测试并没有得到我想要的结果，byte这样会更慢，应该是char+byte是两步操作，且sub和char，byte都不在NYI里面，所以单独效率上没有区别，那再看看后面这种是否比sub更节约内存，top了一下分别运行，感觉没有，内存是一样的。
        说明string.sub是可以用的操作，OpenResty最佳实践的作者可能版本老一些，或者我测试的姿势不对.
        
    
    os
    ====
    
    Lua的os，io库都是同步阻塞操作，而OpenResty的理念是同步非阻塞，他有一套自己的shell工具`resty.shell`，用于执行shell语句，大概长这样：
    ```lua
    local resty_shell = require 'resty.shell'
    local ok, res, err, reason, status = resty_shell.run([[
        echo lunlungaygay
    ]])
    if ok then
        ngx.say(res)
    end
    ```
    io同样有一套，如果以前没有需要重新编译nginx
    ```lua
    local ngx_io = require 'ngx.io'
    local path = 'xxx'
    local file, err_1 = ngx_io.open(path, 'rb')
    local data, err_2 = file:read('*a')
    file:close()
    ```
    项目里用到shell操作的也就取时间，os.date啥的，比较轻量，还没有成为性能瓶颈，就先不优化了，将来有哪些大型shell操作，再考虑用这个解决。
    
    另外课上有句话说的我很认同，io和CPU消耗不会消失，他只是去了另一个地方，所以当有这种大型的磁盘读写，CPU计算的时候，可以适当交给别的服务做。这一点项目里的体现就是lua-battle，他把大量的CPU计算用resty.http扔给了另一台机器做，resty.http是基于cosocket的http模块，发过去之后worker会把当前请求yield掉，主要提供服务的机器CPU就能处理别的请求了，等战斗的机器处理完结果，这个战斗的请求又会被resume回来，这样就不会有一个worker一直用CPU，阻塞了其他请求的情况。
    
    lua部分的坑先开到这里