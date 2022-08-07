# work
工作的一些问题和学习记录

>> 线上事故记录和优化按月记录
>> 学习记录放在最外面这个README里
- [OpenResty](#OpenResty)
    - [LuaJit](#luajit)
        - [测试代码](#test)
        - [table优化](#table)
        - [string优化](#string)
        - [os.*](#os)

    - [OpenResty Lua](#openresty-lua)
        - [basic](#openresty-basic)
        - [cosocket](#cosocket)
        - [lua-resty-core](#lua-resty-core)
        - [cache](#cache)
    
    - [压力测试和火焰图](#压力测试和火焰图)
        - [wrk](#wrk)
        - [火焰图](#火焰图)
    
- [性能优化](#性能优化)
    

- 按月总结的bug,后端问题排查和优化
    - [2207](doc_2207)
        - [排查redis连接数过高](doc_2207#p1)
        - [排查nginx worker CPU占用100%](doc_2207#p2)
        - [http模块版本较低踩坑](doc_2207#p3)
    
    - [2208](doc_2208)
        - [天赋模块bug，战报反SQL注入转义bug，userData:NULL bug](doc_2208)
        - [后端性能优化, 登陆协议部分系统](doc_2208)
            - 佣兵
            - 活动道具
            - 公会信息
            - 竞技场信息

    OpenResty
    ====

    
    这周比较系统的学习了一下OpenResty（当然说掌握还差很多，但也收获不小），看看平时开发/测试的时候还是有一些坑，要进行规避
    
    luajit
    ====
    
    LuaJit ~= Lua5.1，用的时候还是要仔细选择高性能接口。Jit的原理基础的都懂，高深的暂时没有研究的想法。
    
    
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
    最后写一下我vimrc里面支持luajit的配置
    ```vim
    let g:syntastic_check_on_open = 1
    let g:syntastic_lua_checkers = ["luac", "luacheck"]
    let g:syntastic_lua_luacheck_args = "--no-unused-args --std luajit --globals ngx"
    ```
    
    openresty-lua
    ====
    
    这个模块用于对openresty lua的查漏补缺。
    
    openresty-basic
    ====
    
    OpenResty个人感觉下来对后端开发最大的便利就是同步非阻塞，不然后端读个redis实现起来可能相当复杂，后端每个redis操作相当于都是传命令+回调，先coroutine 装一下redis操作，里面luasocket连一下redis，协程阻塞读，读完返回resume执行回调。这样就太麻烦了，而且相当于没有好好利用nginx的事件循环机制去做这些事情。

    OpenResty在这一块已经完全做好了，在有网络IO的情况下他会自动把当前的lua runtime挂起来，然后把网络IO的回调注册到Nginx的事件循环里面，这个时候worker的CPU就可以去处理别的请求了。网络IO返回之后，这个lua runtime又会被唤醒（基于Nginx，其实是等待worker调度，这里也可以看出大量非网络IO的操作会影响其他协议的处理速度，两个都接到一个worker，都走到content_by_lua，一个开始大量运算，另一个就会被长时间挂起，引发更多问题）。这里还要说一下同步非阻塞说的必须是OpenResty提供的模块，nginx-xxx-module或者lus-resty-xxxx才是同步非阻塞的，因为只有用人家的接口人家才自动帮你做挂起，注册，唤醒这一系列操作。所以自己写的网络IO等操作（比如自己写了个模块调用luasocket请求），lua自带的函数（os.xxx, io.xxx），均不能被OpenResty调度，都是同步阻塞的，开发中要尽量减少使用，如果有大量的类似CPU运算，文件读取的操作，考虑扔给其他服务做。
    
    cosocket
    ====
    
    这个模块是现在OpenResty比较推荐使用的网络模块，lua-resty-*的大部分网络库比如redis mysql dns等都是基于这个实现的，也就是[ngx.socket.tcp](https://github.com/openresty/lua-nginx-module#tcpsockconnect)（其实也有[udp](https://github.com/openresty/lua-nginx-module#ngxsocketudp)版本），他可以看作lua-socket的非阻塞版。大部分生命周期都是可以使用的。
    
    理解是做什么的也很简单，协程套接字这个直译已经足够明确，做的是socket的工作，同时满足OpenResty同步非阻塞的基本要求，调用的时候会把当前请求挂起，并把网络事件的回调注册给Nginx

    仔细阅读这个的文档之后可以回答我上个月遇到的很多问题，比如connect的时候调用的是nginx配置的resolver，连接建立之后需要手动sslhandshake等。当然最重要的还是遇到问题知道来哪里查，以及网络基础要打好，这个也是后面学习的重点。
    
    lua-resty-core
    ====
    
    这个是OpenResty在某个版本后默认开启的模块，原理是改用了[luajit ffi](http://luajit.org/ext_ffi_api.html)去进行实现，而不是lua c function，这两者最主要的区别是lua ffi可以被Jit追踪优化，但是lua c function不行

    lua c function感觉主要缺陷是c和lua之间的返回值无法直接交换，比如返回值添加一个字符串需要c那边调用`lua_pushlstring(L, (char *) res, res.len)`把一个指定大小的字符串压到Lua的虚拟栈里。这个和之前做前端时候的tolua感觉挺像的，或者说跨语言交互原理上就是一样的。
    
    两者对比的优缺点还是很显然的，ffi是Jit提供的，很显然可以被Jit优化，而且写起来更简单，不用去栈里把返回值扣出来。但缺点是这个内存在某些情况下要你自己管理，如果是Lua C Function，那因为这个栈在Lua这边，所以GC一下没引用就没了。但是ffi的内存并不全是Lua管理的，也就是`ffi.new`返回的是cdata，这部分是LuaJit管理，`ffi.C.malloc`这样就是申请了一块C内存，需要`local p = ffi.gc(ffi.C.malloc(n), ffi.C.free)`，给他注册一个gc回调，p = nil的时候这个就被释放了。这样其实也有个好处，他可以突破OpenResty对Lua Vm 2G内存的限制
    
    然后去项目里看看，发现OpenResty的版本比较低，实际上也没在init阶段require，相当于我们项目还是lua-nginx-module 的实现，这个如果底层的ngx.xxx成为性能瓶颈，可以成为一个优化的点
    
    cache
    ====
    
    项目里用cache的地方不多，主要原因是大部分数据其实在redis里，mysql里面都是一些战报之类的数据，战报数据比较冷。
    
    用到的地方大概就是存gate信息的地方，存google cloud token的地方，这种是因为每个请求都要访问，属于过热的数据，就在cache里给他缓存了一层。当然也有在worker层面缓存的数据，主要是一些重构的表，一些redis脚本的sha1值之类的。但是这些没有用到cache api，而是直接
    ```lua
    local data_by_activity_id = nil
    
    function _M:get_datas_by_activity_id(activity_id)
        if data_by_activity_id == nil then
            -- init cache
            ...
        end
        return data_by_activity_id[activity_id]
    end
    ```
    这种就比较简陋，但也能用，也满足我们reload worker的时候清空的需求。
    
    cache的话OpenResty主要提供两种，`ngx.shared.dict` 和`resty.lrucache` ,主要区别是dict是跨worker共享的，lrucache是单worker的数据。这两个就不会被reload干掉，放在我们项目，shared dict有网关或者token的应用场景，lrucache暂时没有
    
    
    
    压力测试和火焰图
    ====
    
    wrk
    ====
    
    wrk是一款比较好用的压力测试工具，他相比较ab而言的最大优点是可以很轻松的多线程压测，单命令可以比较容易的产生跑满OpenResty的所有worker的压力（可以自定义压测线程数），并且可以自定义Lua脚本（也就是支持用Lua去模拟真实的请求）
    
    ```shell
    (base) ➜  ~wrk --help
    Usage: wrk <options> <url>                            
      Options:                                            
        -c, --connections <N>  Connections to keep open   
        -d, --duration    <T>  Duration of test           
        -t, --threads     <N>  Number of threads to use   
    
        -s, --script      <S>  Load Lua script file       
        -H, --header      <H>  Add header to request      
            --latency          Print latency statistics   
            --timeout     <T>  Socket/request timeout     
        -v, --version          Print version details      
    
      Numeric arguments may include a SI unit (1k, 1M, 1G)
      Time arguments may include a time unit (2s, 2m, 2h)

    ```
    
    c，d，t之类的都比较好理解，主要看这个-s，有几个操作
    ```lua
    -- 线程建立的时候有这个走这个
    function setup(thread)
    -- 运行时
    
    -- 单线程取命令行参数
    function init(args)
    -- 相邻请求延迟时间，更精细的控制-d
    function delay()
    -- 每次请求调用，可以生成body，官方建议是写一些简单的逻辑在里面，或者固定，不然就成了测试wrk本身的性能了
    function request()
    -- 解析结果，官方建议是不写，这样wrk就可以不解析header和body，节省时间
    function response(status, headers, body)
    
    -- 测试全部结束，可以自己处理一下结果
    function done(summary, latency, requests)
    
    
    -- 全局表
    wrk = {
        scheme  = "http",
        host    = "localhost",
        port    = 8080,
        method  = "POST",
        path    = "/",
        headers = {},
        body    = nil,
        thread  = <userdata>,
        format = function(method, path, headers, body)
        end,
        lookup = function(host, service)
        end,
        connect = function(addr)
        end,
    }
    
    -- 大概往我们战斗服发一下, 先打一场战斗，在Luaserver 把body存到文件battle
    local file = io.open('battle')
    local body = file:read('*a')
    file:close()
    wrk.method = "POST"
    wrk.body = body
    wrk.headers["Content-Type"] = "application/x-www-form-urlencoded"
    
    request = function()
        return wrk.format('POST', '/battle/v1')
    end
    ```
    上面就用wrk给我们战斗服发了一堆相同的战斗。实际应用中我们暂时用不到压力测试，一般都是火焰图看看是哪个函数，继而优化。

    我一般找到性能瓶颈之后，优化具体的接口我都是用[test](#test) 模块的办法自己写个command脚本，判断一下是否优化成功。wrk可以当成一个工具技能，先留着，相信早晚一天有用
    
    火焰图
    ====
    
    这个目前项目里有on-cpu的火焰图，一般是看那个。具体的图不方便放，上个月的文档里介绍了[openresty-systemtap](https://github.com/Wavator/openresty-systemtap-toolkit)这个东西，使用它定位了一个worker 100%跑满CPU的问题，但是实际应用中，这种死循环的情况并不多见，更多的时候是玩家感觉卡顿，要分析代码性能。
    
    这种时候就轮到火焰图出场了，用上面的工具结合[FlameGraph](https://github.com/brendangregg/FlameGraph)线上CPU高过一定值的时候会自动采样，并记录结果。
    ```shell
    TAPPATH=xxxxx
    FLAMEPATH=xxxxxxx
    
    pid,cpuload=#top+grep+awk print $1,$9+sort筛出一个最高的pid和cpuload
    #cpuload比较大就执行抓+存+推送的流程，我猜是这样写的，图省事可以全拿出来
    ${TAPPATH}/ngx-sample-lua-bt -p ${pid} --luajit20 -t 5 > /tmp/tmp.bt
    ${TAPPATH}/fix-lua-bt /tmp/tmp.bt > /tmp/tmp_1.bt
    ${FLAMEPATH}/stackcollapse-stap.pl /tmp/tmp_1.bt > /tmp/a.cbt
    ${FLAMEPATH}/flamegraph.pl /tmp/a.cbt > /tmp/a.svg
    
    #scp或者ftp之类的把这个a.svg换个名字发出去，放到网页上大家就都能看了
    ```
    
    性能优化
    ====
    
    这一定会是长久的课题。我要对项目后端维护持续关注，并经常给自己打气。
    
    在做前端的时候，阿廖沙给我们讲了一节课，`Lua Profiler` 的使用，可以通过这个工具，分析游戏中Lua 和Mono等包括内存在内的一系列动态指标。`Profiler` 也可以，在项目中主要拿它来分析C#模块的资源加载和释放。这个课件还在Confluence的前端分享里面，大家有兴趣可以找找看，是非常不错的课程，我虽然当时听课的时候很小白，很多指标还没有具体概念，但听完课之后也明白了，性能分析要借助工具。
    
    做后端开始，我最先接触的问题是redis内存优化，当时后端一个业务redis要9G左右，单个实例内存占用过大，会给维护（AOF重写）等功能带来很多困扰，如果他超过上限，触发了swap或者内存淘汰机制，就更麻烦了。当时采用了离线工具[rdr-darwin](https://github.com/xueqiu/rdr) 把线上一个9G的redis实例 bgsave拿到我的mac上来进行内存分析，给一些冗余的数据增加过期逻辑或者执行脚本删除后，成功把每个实例内存控制在了5G以内。当时我非常开心，决定持续关注项目后端的性能。
    
    写这段话的时候已经做了半年后端，这段时间，我经常靠日志和火焰图工具来发现、定位一些后端的潜在问题，优化需要靠数据支撑，也需要靠技术支撑。目前的方法论一部分在工作内项目主程和其他很厉害的同事给我的指导和灵感，另一部分在于反复研究OpenResty作者的大作[动态追踪技术漫谈](https://blog.openresty.com.cn/cn/dynamic-tracing/)。有时候发现了问题可以独立完成优化，这让我能感觉到我的日常工作不止是每周一天两天就能完成的Jira单子，而是切实的对自己的提升。有时候不能独立完成优化（这在前面尤为常见，我大学的基础课程学的非常差，一度觉得自己在给项目丢人），那就要请教别人之后完成优化或者和别人一起完成优化，无论是哪一种，我都要仔细查阅文档，并仔细对比优化后的效果。成功的优化让我感到快乐，希望有一天我也能成为非常厉害的人。
    
    