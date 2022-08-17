# work
工作的一些问题和学习记录

>> 线上事故记录和优化按月记录
>> 学习记录放在最外面这个README里
>> 这个也就当日记看看，不会系统的讲一些东西

不定期开新坑+补充

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

- [redis](#redis)
    - [base-data-structure](#redis-data-structure)
        - [string](#redis-string)
        - [hashtable](#redis-hashtable)
        - [ziplist](#ziplist)
        - [quicklist](#quicklist)
        - [skiplist](#skiplist)
    - [redis线程](#redis-multi-thread)
        - [后台任务线程](#redis-bio)
        - [reactor](#redis-reactor)
    
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
            - 战力模块

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
    
    redis
    ====
    
    这个文档写的时候我的redis已经使用了半年，我们后端大部分逻辑是基于redis操作的，所以对一些基础命令已经十分熟悉了，对底层实现也有了解，知道什么情况下选用什么数据结构。这边不再记录这种比较基础的东西，当然如果有薄弱的模块，依然会在对应模块下面补充底层原理的研究。
    
    redis的学习基于两本书和一节课，《Redis开发与运维》，《Redis设计与实现》，第一本我拿来当作工具书，一些场景里的命令和基本原理，都是看这本书学习的，第二本书是我看源码分析的书，写doc的时候还在慢慢看这本书。这两本书都比较不错，但是redis版本有些老，后面极客时间上也学习了蒋德钧老师的《Redis核心技术与实战》，这门课的课程本身和评论区让我大受启发，本来doc里不准备写redis的，但学了这门课之后深感Redis的学习还是太浅。准备把课程中和评论区的一些实战经验，设计经验加以整理。
    
    redis-data-structure
    ====
    
    redis的基础数据结构
    
    redis-string
    =====
    
    string应该是redis里面比较简单的结构，只有SDS一种实现，编码上注意int，embstr，raw就行
    1. int不是无上限的，超出再incr会抛出异常，所以这个指令或者说redis里面所有xxxincr，xxxincrby都要注意这一点。当然出问题一般内网就直接看到了。另外redis的整数类型也有池子，但是内存淘汰策略开启了LRU之类的策略之后这个就没用了，因为指到常量对象上不好统计引用。
    2. embstr就是很短的字符串，老版本39位，新版本44位，优点是内存是一次申请完的连续内存，更紧凑，没有指针乱指，只要释放一次，且查找更快。业务里一般无法避免value太长，但是可以注意的是整个redis的string是一种实现，都适用这些策略。
    3. raw就是最一般的那种redis-str，没有什么优化，申请内存就是倍增到MB之后每次增加1MB，当然我们线上也没这么大的KEY。
    4. redis里面的字符串，维护了长度（保证二进制安全，避免strlen等优点），维护了编码（都要维护），所以一个string object本身就是有一个基础大小在这里的，如果短string特别多，就会严重浪费内存。这种情况可以考虑通过一些id的映射，让这些短string存在hashset之类的结构里（当然hashset的encoding要ziplist，比如线上hash zip entries设置512，那id/500之后500落到一个hashkey里，这500个小str就共享一个ziplist了）。
    
    redis-string-code
    ====
    
    1. 编译优化
    ```c
    struct __attribute__ ((__packed__)) sdshdr5 {
        unsigned char flags; /*SDS类型*/
        char buf[];
    };
    struct __attribute__ ((__packed__)) sdshdr8 {
        uint8_t len; /* 现有长度*/
        uint8_t alloc; /* 已经分配的空间 */
        unsigned char flags; /* 类型 */
        char buf[];
    };
    struct __attribute__ ((__packed__)) sdshdr16 {
        uint16_t len;
        uint16_t alloc;
        unsigned char flags;
        char buf[];
    };
    ```
    `__attribute__ ((__packed__))` 是紧凑分配内存的编译器优化项，c默认会按照8字节对齐的方式给结构体分配内存，向上分配最近的一个8的倍数。这样如果是特别短的字符串会浪费几个字节。
    然后他shshdr*，除了5之外都有2个uint*_t, unit*_t用来表示字符串长度和当前申请的内存，对不同长度的字符串用不同的unit记录这些数据，可以节约内存。
    
    2. string编码的部分，现在的object.c里面写的代码是
    ```c
    #define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
    robj *createStringObject(const char *ptr, size_t len) {
        if (len <= OBJ_ENCODING_EMBSTR_SIZE_LIMIT)
            return createEmbeddedStringObject(ptr,len);
        else
            return createRawStringObject(ptr,len);
    }
    ```
    这个我看的第一本书说是39，应该比较老了，git看是3.0就是44了,39算的比较粗糙，实际上是一次内存申请出来是64，减掉redisObject这个结构自带的16，再减sds自带的维护长度内存类型的3，再减一个'\0'，一共就是44.
    ```c
    typedef struct redisObject {
        unsigned type:4;
        unsigned encoding:4;
        unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */
        int refcount;
        void *ptr;
    } robj;
    ```
    
    这里是4+4+8=16，为什么最上面三个unsigned是4，原因是redis用了: ，也就是位域定义，来极致的节省内存，也就是这个unsigned有32位，type拿4位，encoding拿4位，剩下24位给lru。
    
    embtr申请内存是
    ```c
    robj *createEmbeddedStringObject(const char *ptr, size_t len) {
        robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr8)+len+1);
        struct sdshdr8 *sh = (void*)(o+1);
        ...
    }
    ```
    一次把robj的和sdshdr8的都申请出来了，所以说是申请一次，释放一次。
    
    3. 整数池的使用
    ```c
    robj *createStringObjectFromLongLongWithOptions(long long value, int valueobj) {
        robj *o;
    
        if (server.maxmemory == 0 ||
            !(server.maxmemory_policy & MAXMEMORY_FLAG_NO_SHARED_INTEGERS))
        {
            /* 这一行时说如果没有设置内存淘汰策略（swap），或者说策略不是某种策略（其实是LRU和LFU），就默认启用小整数对象池 */
            valueobj = 0;
        }
    
        // 小整数
        if (value >= 0 && value < OBJ_SHARED_INTEGERS && valueobj == 0) {
            incrRefCount(shared.integers[value]);
            o = shared.integers[value];
        } else {
            // longlong内encode才是int，否则就是str
            if (value >= LONG_MIN && value <= LONG_MAX) {
                o = createObject(OBJ_STRING, NULL);
                o->encoding = OBJ_ENCODING_INT;
                o->ptr = (void*)((long)value);
            } else {
                o = createObject(OBJ_STRING,sdsfromlonglong(value));
            }
        }
        return o;
    }
    
    /* 比较具体的值new一般是这个 */
    robj *createStringObjectFromLongLong(long long value) {
        return createStringObjectFromLongLongWithOptions(value,0);
    }
    
    /* 一般是key调用这个，方便统计引用次数 */
    robj *createStringObjectFromLongLongForValue(long long value) {
        return createStringObjectFromLongLongWithOptions(value,1);
    }
    ```
    
    内存淘汰的部分，可以看到LRU和LFU才会完全禁用小整数对象池
    ```c
    #define MAXMEMORY_FLAG_NO_SHARED_INTEGERS \
        (MAXMEMORY_FLAG_LRU|MAXMEMORY_FLAG_LFU)
    ```
    
    4. 一个最大的redis string是512M
    ```c
    createLongLongConfig("proto-max-bulk-len", NULL, DEBUG_CONFIG | MODIFIABLE_CONFIG, 1024*1024, LONG_MAX, server.proto_max_bulk_len, 512ll*1024*1024, MEMORY_CONFIG, NULL, NULL),
    ```
    
    5. 内存申请1MB以下倍增，以上每次加1MB
    ```c
    #define SDS_MAX_PREALLOC (1024*1024)
        sds _sdsMakeRoomFor(sds s, size_t addlen, int greedy) {
        ... 
        len = sdslen(s);
        sh = (char*)s-sdsHdrSize(oldtype);
        reqlen = newlen = (len+addlen);
        if (greedy == 1) {
            if (newlen < SDS_MAX_PREALLOC)
                newlen *= 2;
            else
                newlen += SDS_MAX_PREALLOC;
        }
    
        type = sdsReqType(newlen);
        ... 
        return s;
    }
    
    ```
    
    6. 注意redis的key都是sds
    ```c
    void dbAdd(redisDb *db, robj *key, robj *val) {
        sds copy = sdsdup(key->ptr);
        ...
    }

    ```
    
    Redis Server读取client的请求的时候，会先读如缓冲区，这个缓冲区也是SDS
    AOF缓冲区也是SDS
    
    string的部分先整理到这里
    
    redis-hashtable
    ====
    
    1. 4.0以后的hash算法是siphash
    2. hash的元素个数太多的时候会检查是否需要扩容,总是拓展成2^x
    3. rehash是渐进式的，全局表有个定时器，太久没访问一次删一百个哈希桶，自己定义的表每次操作清一个哈希桶
    4. rehash没完成的时候不能再rehash
    5. rehash判断的时候，如果能找到AOF重写或者RDB生成之类的子进程，哈希因子就会变成5，也就是严重冲突的时候依然会rehash
    6. 很多上层数据结构，和redis本身的kvdb，过期键之类的都是hash的实现，hash是最基本的数据结构之一
    
    redis-hashtable-code
    ====
    
    1. hash算法的选择
        redis 当前版本是siphash，但是算法细节没有过多了解。
        ```c
        /* The default hashing function uses SipHash implementation
         * in siphash.c. */
        
        uint64_t siphash(const uint8_t *in, const size_t inlen, const uint8_t *k);
        uint64_t siphash_nocase(const uint8_t *in, const size_t inlen, const uint8_t *k);
        
        uint64_t dictGenHashFunction(const void *key, size_t len) {
            return siphash(key,len,dict_hash_function_seed);
        }
        ```
    2. hash拓展逻辑：每次expand拓展到下一个2^x
    ```c
    static signed char _dictNextExp(unsigned long size)
    {
        unsigned char e = DICT_HT_INITIAL_EXP;
    
        // 不能超过longmax
        if (size >= LONG_MAX) return (8*sizeof(long)-1);
        while(1) {
            // 每次指数左移一位
            if (((unsigned long)1<<e) >= size)
                return e;
            e++;
        }
    }
    
    int _dictExpand(dict *d, unsigned long size, int* malloc_failed)
    {
    
        /* 在rehash或者rehash之后的size比要用的小就不合适 */
        if (dictIsRehashing(d) || d->ht_used[0] > size)
            return DICT_ERR;
    
        dictEntry **new_ht_table;
        unsigned long new_ht_used;
        signed char new_ht_size_exp = _dictNextExp(size);
    
        /* 把幂次拿出来当新的大小 */
        size_t newsize = 1ul<<new_ht_size_exp;
        if (newsize < size || newsize * sizeof(dictEntry*) < newsize)
            return DICT_ERR;
    
        /* rehash完是同样大小不行 */
        if (new_ht_size_exp == d->ht_size_exp[0]) return DICT_ERR;
    
        /*
            下面把hash[1]的空间malloc出来，给hash[1]赋值
        */
        ...
        return DICT_OK;
    }
    
    /* 外部都是调用这个，如果需要拓展则变为两倍 */
    static int _dictExpandIfNeeded(dict *d)
    {
        /* 还在rehash，那就不用拓展 */
        if (dictIsRehashing(d)) return DICT_OK;
    
        /* 空表 */
        if (DICTHT_SIZE(d->ht_size_exp[0]) == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);
    
        if (d->ht_used[0] >= DICTHT_SIZE(d->ht_size_exp[0]) &&
            
            // 这里是个全局设置的变量，AOF重写，RDB生成等过程，会把他设置成0，也就是不特别大的情况都不rehash
            (dict_can_resize ||
            // 比较哈希因子，如果特别大（这个数是5），那就只能强制rehash
             d->ht_used[0]/ DICTHT_SIZE(d->ht_size_exp[0]) > dict_force_resize_ratio) &&
             // 能不能申请下来这些内存
            dictTypeExpandAllowed(d))
        {
            return dictExpand(d, d->ht_used[0] + 1);
        }
        return DICT_OK;
    }
    ```
    
    往哈希表中增加/替换/增加或查找，对应了三个接口，`dictAdd`, `dictReplace`, `dictAddOrFind`，最终逻辑都是func -> `dictAddRow` -> `_dictKeyIndex` -> `dictExpandIfNeeded`, 这就是往一个hash里添加或者修改某个值的流程
     
    那来看一下`dictAddRow`
    ```c
    dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
    {
        long index;
        dictEntry *entry;
        int htidx;
    
        // 这里让rehash前进一步，也就是redis渐进rehash的过程
        if (dictIsRehashing(d)) _dictRehashStep(d);
    
        /* 相当于已经有了就return，这个函数只做增加，当然这个函数调用的时候会判断是否要rehash */
        if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
            return NULL;
    
        /* rehash的时候用1，要往新的里面加，不然用0 */
        htidx = dictIsRehashing(d) ? 1 : 0;
        ... /*申请一下内存*/
        
        /* 增加并设置，如果hash碰撞这边会通过next拉成链表 */
        entry->next = d->ht_table[htidx][index];
        d->ht_table[htidx][index] = entry;
        d->ht_used[htidx]++;
        dictSetKey(d, entry, key);
        return entry;
    }
    ```
    
    3. rehash过程
    刚才看到了`dictAddRow`的时候如果在rehash会`_dictRehashStep`, rehash一共有两个比较重要的函数，一个这个step，一个`dictRehash`
    
    ```c
    int dictRehash(dict *d, int n) {
        int empty_visits = n*10;
        //主循环，根据要拷贝的链表数量n，循环n次或者ht 0的内容全部被移动到ht 1
        while(n-- && d->ht[0].used != 0) {
            dictEntry *de, *nextde;
            while(d->ht_table[0][d->rehashidx] == NULL) {
                d->rehashidx++;
                // 这里说明拿了一个空的链表出来rehash，如果比要拷贝的十倍还多就会返回，避免拷贝太久，影响当前指令
                if (--empty_visits == 0) return 1;
            }
            de = d->ht_table[0][d->rehashidx];
            /*
                h[0]数据删除，增加到h[1]
            */
            while(de) {
                uint64_t h;
                nextde = de->next;
                h = dictHashKey(d, de->key) & DICTHT_SIZE_MASK(d->ht_size_exp[1]);
                de->next = d->ht_table[1][h];
                d->ht_table[1][h] = de;
                d->ht_used[0]--;
                d->ht_used[1]++;
                de = nextde;
            }
            /*
                原链表清空，rehash进度增加一次
            */
            d->ht_table[0][d->rehashidx] = NULL;
            d->rehashidx++;
        }
        //判断rehash是否已经完成了数据转移
        if (d->ht[0].used == 0) {
            //释放ht[0]内存空间
            zfree(d->ht[0].table);
            //让ht[0]指向ht[1]，因为不rehash是存到0的，这里相当于rehash结束了，所以要01互换
            d->ht[0] = d->ht[1];
            //清空ht[1]，重新设置大小
            _dictReset(&d->ht[1]);
            //rehash结束，设置一个flag
            d->rehashidx = -1;
            //返回0，表示rehash完成
            return 0;
        }
        //返回1，表示rehash没有完成
        return 1;
    }
    //这个oneStep就是移动一个链表到h[1]，所以每次请求至多移动一个链表
    static void _dictRehashStep(dict *d) {
        if (d->pauserehash == 0) dictRehash(d,1);
    }
    ```
    除了`dictAddRow`，还有一些函数调用的时候也会同时调用` _dictRehashStep`，一共对应了dict的大部分操作：增加(`dictAddRow`)，删除（`dictGenericDelete`），查找（`dictFind`），随机key(`dictGenRandomKey`), 随机部分key(`dictGenericSomeKey`)，接近于每次hash操作，都会让rehash前进一步（一个链表移走），同时empty那里保证了最多检查1*10=10张链表，保证了rehash不会拖慢这次请求太多。
    
    这里还注意到一个细节，SomeKey这种，拿N个key，他会Step N次，也就是尝试rehash N个哈希桶，这里说明redis在保证操作复杂度不变的情况下在尽量白嫖
    
    这些就是主动的渐进式rehash，除了这个之外，还有一个定时检查,`dictRehashMilliseconds`,会迁移全局hash表的数据，比如整个redis的db里的kv，整个db的过期kv等，也就是user定义的数据结构Hash Set等用到Hash的不会被这个函数影响，都是走的上面的Step函数去rehash
    
    ```c
    int dictRehashMilliseconds(dict *d, int ms) {
        if (d->pauserehash > 0) return 0;
    
        long long start = timeInMilliseconds();
        int rehashes = 0;
    
        while(dictRehash(d,100)) {
            rehashes += 100;
            if (timeInMilliseconds()-start > ms) break;
        }
        return rehashes;
    }
    ```
    
    ziplist
    ====

    redis 部分底层结构是用ziplist来节约空间的，hash元素较少，zset元素较少的时候，均会使用ziplist的底层结构，ziplist底层上是一整块连续的内存。每个元素都会保存上一个元素的大小（并且采用尽量短的数据类型保存），当前项的编码，以及当前项的具体数据，也就是
    ```c
    typedef struct zlentry {
        // xxlensize 代表记录xxlen用的大小
        unsigned int prevrawlensize;
        unsigned int prevrawlen;
        unsigned int lensize;
        unsigned int len;

        unsigned int headersize;     /* prevrawlensize + lensize. */
        unsigned char encoding;      /* ZIP_STR_* or ZIP_INT_* */
        unsigned char *p;
    } zlentry;
    ```
    整个ziplist是<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>
    ziplist有一些性能问题，主要看insert函数

    ```c
    unsigned char *__ziplistInsert(unsigned char *zl, unsigned char *p, unsigned char *s, unsigned int slen) {
        size_t curlen = intrev32ifbe(ZIPLIST_BYTES(zl)), reqlen, newlen;
        ... //取当前prevlen等值
    
        -- 根据是不是整数、整数类型/字符串实际大小分配空间
        if (zipTryEncoding(s,slen,&value,&encoding)) {
            reqlen = zipIntSize(encoding);
        } else {
            reqlen = slen;
        }
        // 把自己的prevlen也加进去
        reqlen += zipStorePrevEntryLength(NULL,prevlen);
        // 计算encoding大小
        reqlen += zipStoreEntryEncoding(NULL,encoding,slen);
    
        int forcelarge = 0;
        -- 插入位置的prevlen和实际的prevlen差, forcelarge说明没变，直接扩充，nextdiff不为0就会连锁更新
        nextdiff = (p[0] != ZIP_END) ? zipPrevLenByteDiff(p,reqlen) : 0;
        if (nextdiff == -4 && reqlen < 4) {
            forcelarge = 1;
            nextdiff = 0;
        }
    
        //分配这次的空间大小
        newlen = curlen+reqlen+nextdiff;
        zl = ziplistResize(zl,newlen);
    
        .. // 申请这次的空间

        // 检查是不是要连锁更新
        if (nextdiff != 0) {
            offset = p-zl;
            zl = __ziplistCascadeUpdate(zl,p+reqlen);
            p = zl+offset;
        }
    
        ... //保存值
        return zl;
    }
    ```

    连锁更新主要是后面节点的prelensize不足以保存现在的大小，发生的时候，需要给下一个更多的空间保存，给下一个更多的空间会导致下一个增大，就需要修改下下个的prevlen，如果也不足以表示，就会继续扩充，一直到最后。这种发生的概率并不大，应该只有所有数据都很临界的时候，插入了一个比临界大的，出现这种情况。但是ziplist本身的问题不止有极端情况下连锁更新导致的频繁内存操作，还有一个问题就是他的查询也是链式的。所以上层结构hash和zset会定一个使用ziplist的最大大小，避免太大导致效率大打折扣。
    
    针对这些问题，redis对ziplist提出了两种优化，quicklist和listpack，分别应用于list和stream等结构

    quicklist
    ====

    quicklist简单来说就是一堆ziplist连起来 z1<->z2<->z3<->...<->zn
    他设计出来主要是解决ziplist访问效率问题

    ```c
    typedef struct quicklistNode {
        struct quicklistNode *prev;     //前一个quicklistNode
        struct quicklistNode *next;     //后一个quicklistNode
        unsigned char *zl;              //quicklistNode指向的ziplist
        unsigned int sz;                //ziplist的字节大小
        unsigned int count : 16;        //ziplist中的元素个数
        unsigned int encoding : 2;   //编码格式，原生字节数组或压缩存储
        unsigned int container : 2;  //存储方式
        unsigned int recompress : 1; //数据是否被压缩
        unsigned int attempted_compress : 1; //数据能否被压缩
        unsigned int extra : 10; //预留的bit位
    } quicklistNode;

    typedef struct quicklist {
        quicklistNode *head;      //quicklist的链表头
        quicklistNode *tail;      //quicklist的链表尾
        unsigned long count;     //所有ziplist中的总元素个数
        unsigned long len;       //quicklistNodes的个数
        ...
    } quicklist;
    ```
    
    增加的时候先检查能不能插入当前ziplistn，不可以则新建一个节点
    数量不满足也会新建，具体逻辑在_quicklistNodeAllowInsert中
    ```c
    int quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
        ...
        if (likely(
                _quicklistNodeAllowInsert(quicklist->head, quicklist->fill, sz))) {
            quicklist->head->entry = lpPrepend(quicklist->head->entry, value, sz);
            quicklistNodeUpdateSz(quicklist->head);
        } else {
            quicklistNode *node = quicklistCreateNode();
            node->entry = lpPrepend(lpNew(0), value, sz);
            quicklistNodeUpdateSz(node);
            _quicklistInsertNodeBefore(quicklist, quicklist->head, node);
        }
        ...
    }
    ```
    判断能否插入的函数allowInsert，他会计算新插入元素后的大小（new_sz），这个大小等于 quicklistNode 的当前大小（node->sz）、插入元素的大小（sz），以及插入元素后 ziplist 的 prevlen 占用大小
    然后判断元素个数是不是满足要求,总大小是否满足要求

    ```c

    // 8196
    #define sizeMeetsSafetyLimit(sz) ((sz) <= SIZE_SAFETY_LIMIT)
    
    REDIS_STATIC int _quicklistNodeAllowInsert(const quicklistNode *node,
                                               const int fill, const size_t sz) {
        if (unlikely(!node))
            return 0;
        // 比较大的节点
        if (unlikely(QL_NODE_IS_PLAIN(node) || isLargeElement(sz)))
            return 0;
        // 检查大小
        if (likely(_quicklistNodeSizeMeetsOptimizationRequirement(new_sz, fill)))
            return 1;
        // 检查大小
        else if (!sizeMeetsSafetyLimit(new_sz))
            return 0;
        // 检查个数
        else if ((int)node->count < fill)
            return 1;
        else
            return 0;
    }
    ```

    另外这里看到redis虽然是内存友好，但是CPU方面也有做很多优化，比如用likely和unlikely进行分支预测



    
    listpack
    ====
    stream项目里基本没用，这个结构就没怎么仔细看了
    
    
    skiplist
    ====
    
    skiplist是用来实现有序集合的结构
    ```c
    typedef struct zskiplistNode {
        //Sorted Set中的元素
        sds ele;
        //元素权重值
        double score;
        //后向指针
        struct zskiplistNode *backward;
        //节点的level数组，保存每层上的前向指针和跨度
        struct zskiplistLevel {
            struct zskiplistNode *forward;
            unsigned long span;
        } level[];
    } zskiplistNode;
    
    typedef struct zskiplist {
        struct zskiplistNode *header, *tail;
        unsigned long length;
        int level;
    } zskiplist;
    ```
    
    他的基本原理和所有跳表一样，分层使用了随机某个点的层数的设计。
    ```c
    zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
        ...
        //查询部分，和跳表一样，同级倒着找，span是跨度，通过span计算出跳的rank
        for (i = zsl->level-1; i >= 0; i--) {
            /* store rank that is crossed to reach the insert position */
            rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
            while (x->level[i].forward &&
                    (x->level[i].forward->score < score ||
                        (x->level[i].forward->score == score &&
                        sdscmp(x->level[i].forward->ele,ele) < 0)))
            {
                rank[i] += x->level[i].span;
                x = x->level[i].forward;
            }
            update[i] = x;
        }
        // 随机生成新节点建几层索引，32以内，而且每层是0.25概率向上拓展
        level = zslRandomLevel();
        if (level > zsl->level) {
            ...//比当前level大，维护level层到当前层的span（通过上面的update信息）
        }
        x = zslCreateNode(level,score,ele);
        for (i = 0; i < level; i++) {
            // 维护每一层的span
        }
    
        // level比较低，上层span增加
        for (i = level; i < zsl->level; i++) {
            update[i]->level[i].span++;
        }
    
        // 元素个数增加
        ...
        return x;
    }
    ```

    redis-multi-thread
    ====
    
    redis经常被说是单线程模型，但其实这个说法并不正确。通过阅读redis的源码发现，redis其实只是在“处理客户端请求”这一块是单线程的。

    后台来看，redis很早就支持多线程了，通过bio，redis关闭文件，fsync aof日志，lazyfree 字典之类的对象使用的内存，都是通过创建一个bio任务，另起一个线程做的。这里是生产者消费者模型，生产者提交任务，消费者不停轮询这个任务队列
    
    这个主要是为了避免耗时的操作影响主线程，比如aof写入磁盘，如果在主线程中，那磁盘操作就成了性能瓶颈，肯定会严重影响redis的效率。


    前台来看，redis 在6.0之后（可惜我们项目并没有使用），从单Reactor单线程模式变成了单Reactor多线程模式，用来维护客户端的socket连接。

    redis-bio
    ====

    后台任务都是通过bio.c建立的，主要是通过`bioSubmitJob` 进行创建，通过`bioProcessBackgroundJobs` 进行消费

    bio init在redis server的`initServer` 之后，可以看到main中的顺序是先`initServer` 再 `initServerLast`，在这个last里调用了`bioInit` ，初始化了后台的任务队列

    bioinit:
    ```c
    void bioInit(void) {
        ...
        // 根据三种后台任务类型，初始化他的锁和任务列表
        for (j = 0; j < BIO_NUM_OPS; j++) {
            pthread_mutex_init(&bio_mutex[j],NULL);
            pthread_cond_init(&bio_newjob_cond[j],NULL);
            pthread_cond_init(&bio_step_cond[j],NULL);
            bio_jobs[j] = listCreate();
            bio_pending[j] = 0;
        }
    
        // 计算子线程栈空间大小
        ... 
    
        for (j = 0; j < BIO_NUM_OPS; j++) {
            void *arg = (void*)(unsigned long) j;
            //调用create创建子线程
            if (pthread_create(&thread,&attr,bioProcessBackgroundJobs,arg) != 0) {
                serverLog(LL_WARNING,"Fatal: Can't initialize Background Jobs.");
                exit(1);
            }
            bio_threads[j] = thread;
        }
    }
    ```

    主要就是做了几件准备工作，这里可以看到，他根据BIO_NUM_OPS，每种后台OP，创建一个任务链表，记录等待数量的数组。并且初始化每种任务的锁和两种条件变量。

    计算子线程大小这个是避免在一些系统下栈太小

    再调用pthread_create，创建子线程，并指定函数入口bioProcessBackgroundJobs，arg是他的编号，上面提到的三种后台OP类型

    这里执行完，就起了三个后台任务的消费者，每种消费者消费一种类型的后台任务


    bioProcessBackgroundJobs是消费函数，用来真正的处理任务
    ```c
        ...//给子线程起个名字之类的操作


        // 这里可以看到子线程可以去绑定单独的后台CPU的，当然这个要单独的宏定义USE_SETCPUAFFINITY去生效
        // 比如在redis.conf里增加bio_cpulist 1,3
        redisSetCpuAffinity(server.bio_cpulist);
        makeThreadKillable();

        // 先对线程加锁
        pthread_mutex_lock(&bio_mutex[type]);

        // 初始化信号屏蔽集合，只处理自己想要的信号
        sigemptyset(&sigset);
        sigaddset(&sigset, SIGALRM);
        if (pthread_sigmask(SIG_BLOCK, &sigset, NULL))
            serverLog(LL_WARNING,
                "Warning: can't mask SIGALRM in bio.c thread: %s", strerror(errno));

        while(1) {
            listNode *ln;

            // 这边注册一个条件变量,等待被后台任务，上面先锁定了，进来对时候总是先进入阻塞等待, 有newjob的时候会唤醒
            if (listLength(bio_jobs[type]) == 0) {
                pthread_cond_wait(&bio_newjob_cond[type],&bio_mutex[type]);
                continue;
            }

            // 拿出一个任务, 给线程解锁
            ln = listFirst(bio_jobs[type]);
            job = ln->value;
            pthread_mutex_unlock(&bio_mutex[type]);

            //每种任务类型的执行函数写在这里
            // 省略三个ifelse
            ...

            //释放任务本身
            zfree(job);

            // 锁上，给完成数量-1，列表node-1
            pthread_mutex_lock(&bio_mutex[type]);
            listDelNode(bio_jobs[type],ln);
            bio_pending[type]--;
            // step_cond唤醒
            pthread_cond_broadcast(&bio_step_cond[type]);
        }
    ```

    step_cond只是被设计出来，源码里还没找到在哪用过，应该是没有用过的。

    所以这个逻辑就变成了，线程cond_wait这个newjob类型的条件量，有的话拿出来根据job的类型处理一下

    这里是提交job的时候
    ```c
        void bioSubmitJob(int type, bio_job *job) {
            pthread_mutex_lock(&bio_mutex[type]);
            listAddNodeTail(bio_jobs[type],job);
            bio_pending[type]++;
            // 这里创建完job，唤醒newjob的cond_wait
            pthread_cond_signal(&bio_newjob_cond[type]);
            pthread_mutex_unlock(&bio_mutex[type]);
        }
    ```

    所有后台任务都是通过调用这个submitjob接口创建的，创建之后会唤醒对应type的后台线程，异步的处理上面说的三种工作

    redis后台任务线程用了大量的锁操作，对这块并不熟悉，看起来有点吃力。但是忽略锁的操作，就是比较简单的工作模式了。

    redis-reactor
    ====
    
    redis的reactor模式在6.0版本前后有所不同，我们项目是redis是5.x的版本，所以用不上6.0以后的io特性

    [reactor的教材放一下](https://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)

    6.0之前，redis是单Reactor单线程模式，也就是accept->read->handler->write

    6.0之后，redis是单Reactor多线程模式，也就是handler扔到了别的线程

    但本质上都还是io多路复用，看下redis里面的事件循环EventLoop

    注意只有io多路复用在6.0之后是多线程Reactor模式，io拿进输入区缓存之后是顺序执行的，还是只有一个线程在处理客户端请求

    不是说有若干个handler在处理客户端请求，只是写入缓冲区而已。

    - 事件的定义

    redis中有IO事件和时间事件两种基本事件，对应了aeFileEvent和aeTimeEvent，在ae.h中，看一下aeFileEvent

    ```c
    typedef struct aeFileEvent {
        int mask; /* one of AE_(READABLE|WRITABLE|BARRIER) */
        aeFileProc *rfileProc;
        aeFileProc *wfileProc;
        void *clientData;
    } aeFileEvent;
    ```

    mask是用来区分哪一种FileEvent， timeEvent则是使用id来进行区分

    这里看到mask是AE_(READABLE|WRITABLE|BARRIER)这三种事件，可读，可写，和反转读写事件（不知道这么翻译是否有失准确）

    可读，可写，是客户端连过来socket的状态，和正常socket的可读可写没有区别

    主要看看这个AE_BARRIER，这个类型的事件会先写再读，不带这个mask则是先读再写

    这个在networking.c（和客户端交互部分）有
    ```c
    if (server.aof_state == AOF_ON &&
        server.aof_fsync == AOF_FSYNC_ALWAYS)
    {
        ae_barrier = 1;
    }
    ```

    可以看到，AOF在写的时候，redis出于希望数据快速落盘的考虑，会启用这个标志，这样就是先写数据库，再写socket，这个后续会写一下事件循环的细节，这里继续介绍event结构

    aeFileProc类型的两个回调函数分别对应了读和写，在事件循环里拿到读事件调用读的回调，写调写的

    clientData很好理解，他指向客户端的私有数据

    - 事件循环主循环

    这部分代码在aeMain中，他长得就像这样

    ```c
    void aeMain(aeEventLoop *eventLoop) {
        eventLoop->stop = 0;
        while (!eventLoop->stop) {
            …
            aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
        }
    }
    ```

    调用的时候是

    ```c
    int main()
    {
        ...
        // 这边直接起一个aeMain，相当于main函数这里开一个死循环
        aeMain(server.el)
        // 执行到下面说明redis实例进入关闭状态
        aeDeleteEventLoop(server.el);
        return 0;
    }
    ```

    这个是main函数最下面几行，可以看到，aeMain是redis server全部准备就绪之后，直接起起来的，整个redis的核心也就是这个事件循环，它在主线程里面阻塞，不断的调用
    aeProcessEvent，处理事件循环中的事件

    这里已经调用aeMain进行事件循环了，准备工作在initServer中`server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);`,这里要分配内存空间，创建多路复用的实例,把多路复用实例的引用存到事件循环实例中。



    redis的主循环就是这个事件循环，接下来看看事件循环的每一步

    - 单步的事件循环

    ```c
    //flag用来判断执行哪种事件
    int aeProcessEvents(aeEventLoop *eventLoop, int flags)
    {
        int processed = 0, numevents;

        // 没有可以执行的
        if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;
    
        if (eventLoop->maxfd != -1 ||
            ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
            int j;
            struct timeval tv, *tvp;
    
            /*先执行before sleep，主线程先去等待多路复用api的返回之前先做一些其他工作，例如aof写入磁盘*/
            if (eventLoop->beforesleep != NULL && flags & AE_CALL_BEFORE_SLEEP)
                eventLoop->beforesleep(eventLoop);
    
            // 多路复用api拿到当前就绪了几个事件
            numevents = aeApiPoll(eventLoop, tvp);
    
            /*多路复用拿到可以执行的事件之后，要做一些准备工作*/
            if (eventLoop->aftersleep != NULL && flags & AE_CALL_AFTER_SLEEP)
                eventLoop->aftersleep(eventLoop);
    
            for (j = 0; j < numevents; j++) {
                int fd = eventLoop->fired[j].fd;
                aeFileEvent *fe = &eventLoop->events[fd];
                int mask = eventLoop->fired[j].mask;
                int fired = 0;
    
                // 这里是上面说的那种barrier类型，也就是AOF fsync开启的时候，命令进来会先落盘
                int invert = fe->mask & AE_BARRIER;
    
                // 下面根据是否barrier，调用rfileProc和wfileProc
                if (!invert && fe->mask & mask & AE_READABLE) {
                    fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                    fe = &eventLoop->events[fd];
                }
    
                if (fe->mask & mask & AE_WRITABLE) {
                    if (!fired || fe->wfileProc != fe->rfileProc) {
                        fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                        fired++;
                    }
                }
    
                if (invert) {
                    fe = &eventLoop->events[fd]; /* Refresh in case of resize. */
                    if ((fe->mask & mask & AE_READABLE) &&
                        (!fired || fe->wfileProc != fe->rfileProc))
                    {
                        fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                        fired++;
                    }
                }
                processed++;
            }
        }
        // 处理时间类型的事件, 下面再说
        if (flags & AE_TIME_EVENTS)
            processed += processTimeEvents(eventLoop);
    
        return processed;
    }
    ```

    整个流程大概是这样的，就是调用多路复用api，获取就绪的文件事件，执行，再处理TimeEvents。下面分别介绍一下两种事件的定义，注册，执行

    - IO事件

    创建: 主要是在事件循环和多路复用api中分别注册，并绑定对应的回调函数和数据

    ```c
    int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask, aeFileProc *proc, void *clientData){
        //根据文件描述符，尝试新建/取出一个io事件，并注册到多路复用api
        aeFileEvent *fe = &eventLoop->events[fd];
        if (aeApiAddEvent(eventLoop, fd, mask) == -1)
            return AE_ERR;
        // 根据mask类型，绑定回调函数
        fe->mask |= mask;
        if (mask & AE_READABLE) fe->rfileProc = proc;
        if (mask & AE_WRITABLE) fe->wfileProc = proc;
        // 设置数据, 更新事件循环里的最大文件描述符
        fe->clientData = clientData;
        if (fd > eventLoop->maxfd)
            eventLoop->maxfd = fd;
    }
    ```

    在redis initServer阶段，他会注册TCP连接的IO事件，并绑定对应回调函数为`acceptTcpHandler`

    `createSocketAcceptHandler(&server.ipfd, acceptTcpHandler) `

    ```c
    int createSocketAcceptHandler(socketFds *sfd, aeFileProc *accept_handler) {
        int j;
        for (j = 0; j < sfd->count; j++) {
            if (aeCreateFileEvent(server.el, sfd->fd[j], AE_READABLE, accept_handler,NULL) == AE_ERR) {
                // 创建失败回滚
                for (j = j-1; j >= 0; j--) aeDeleteFileEvent(server.el, sfd->fd[j], AE_READABLE);
                return C_ERR;
            }
        }
        return C_OK;
    }
    ```

    这样在起server的时候，就确定了处理TCP IO事件的回调函数，同样也会注册其他模块的回调函数。

    这个回调函数是绑定到redis的监听端口ipfd的，也就是客户端的连接请求都会走这个accectTcpHandler回调, 对应了Reactor模式的accepter

    ```c
    void acceptTcpHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
        ...

        while(max--) {
            //试图创建一个socket连接，返回这个连接的socket文件描述符
            cfd = anetTcpAccept(server.neterr, fd, cip, sizeof(cip), &cport);
            if (cfd == ANET_ERR) {
                return;
            }
            //先调用conn，建立连接
            //再把已连接socket传入accectCommonHandler
            acceptCommonHandler(connCreateAcceptedSocket(cfd),0,cip);
        }
    }

    static void acceptCommonHandler(connection *conn, int flags, char *ip) {
        ... 
        ...
        //检查能不能连接
        if (listLength(server.clients) + getClusterConnectionsCount()
            >= server.maxclients)
        {
            ...
            return;
        }
    
        //根据socket连接创建一个客户端
        if ((c = createClient(conn)) == NULL) {
            connClose(conn);
            return;
        }
    
        //最终确定接受这个客户端连接
        if (connAccept(conn, clientAcceptHandler) == C_ERR) {
            ...
            return;
        }
    }

    ```

    这里面有个重要的函数createClient，里面主要是
    ```c
        connEnableTcpNoDelay(conn);
        if (server.tcpkeepalive)
            connKeepAlive(conn,server.tcpkeepalive);
        connSetReadHandler(conn, readQueryFromClient);
        connSetPrivateData(conn, c);
    ```

    这里的connEnableTcpNoDelay(conn);允许了TCP小包发送
    connKeepAlive维护长连接
    connSetReadHandler相当于把读的回调绑定给了这个`readQueryFromClient`函数
    connSetPrivateData保存这个连接的数据

    所以看到了这里，IO读事件的回调，就是这个`readQueryFromClient`
    它主要是把指令读到缓冲区buffer中

    `readQueryFromClient` -> `processInputBufferAndReplicate` -> `processInputBuffer` -> `processCommand` -> `addApply`

    这个缓冲区，会被前面提到的beforeSleep通过调用`handleClientsWithPendingWrites`写回客户端，他一样会调用aeCreateFileEvent，但他创建的是一个可写事件

    相当于客户端过来可读，回去可写。可读可写都是基于redis机器而言的。

    这样一个请求进来的时候，完整的有
    建立连接 -> 调用accecpt函数，注册到事件循环并绑定事件循环的回调readQueryFromClient -----> 客户端数据来了调用回调函数写入缓冲区 -> 处理逻辑 -> 写到缓冲区，标记连接socket为可写


    - 时间事件

    这个内容少一些，类比上面，initServer的时候他一样注册了
    `aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL)`

    他就是每个周期都会执行的定时任务，里面主要是这几个

    ```c
    // 检查是不是关闭
    if (server.shutdown_asap) { if (prepareForShutdown(SHUTDOWN_NOFLAGS) == C_OK) exit(0); ... }
    // 客户端异步操作
    clientCron();
    // 后台操作，比如删除过期key之类的
    databaseCron();
    ```

    上面aeProcessEvents里面讲了每次循环都会检查时间列表有没有可以执行的时间事件，拿出来执行一下。

    注意这个时间事件都是在主线程执行的（因为是aeMain的循环里每一步都会调用）

    所以删除key(对应databasecron里面的activeExpireCycle)等操作是可能阻塞主线程的操作


    **************
    
    
    
    性能优化
    ====
    
    这一定会是长久的课题。我要对项目后端维护持续关注，并经常给自己打气。
    
    在做前端的时候，阿廖沙给我们讲了一节课，`Lua Profiler` 的使用，可以通过这个工具，分析游戏中Lua 和Mono等包括内存在内的一系列动态指标。`Profiler` 也可以，在项目中主要拿它来分析C#模块的资源加载和释放。这个课件还在Confluence的前端分享里面，大家有兴趣可以找找看，是非常不错的课程，我虽然当时听课的时候很小白，很多指标还没有具体概念，但听完课之后也明白了，性能分析要借助工具。
    
    做后端开始，我最先接触的问题是redis内存优化，当时后端一个业务redis要9G左右，单个实例内存占用过大，会给维护（AOF重写）等功能带来很多困扰，如果他超过上限，触发了swap或者内存淘汰机制，就更麻烦了。当时采用了离线工具[rdr-darwin](https://github.com/xueqiu/rdr) 把线上一个9G的redis实例 bgsave拿到我的mac上来进行内存分析，给一些冗余的数据增加过期逻辑或者执行脚本删除后，成功把每个实例内存控制在了5G以内。当时我非常开心，决定持续关注项目后端的性能。
    
    写这段话的时候已经做了半年后端，这段时间，我经常靠日志和火焰图工具来发现、定位一些后端的潜在问题，优化需要靠数据支撑，也需要靠技术支撑。目前的方法论一部分在工作内项目主程和其他很厉害的同事给我的指导和灵感，另一部分在于反复研究OpenResty作者的大作[动态追踪技术漫谈](https://blog.openresty.com.cn/cn/dynamic-tracing/)。有时候发现了问题可以独立完成优化，这让我能感觉到我的日常工作不止是每周一天两天就能完成的Jira单子，而是切实的对自己的提升。有时候不能独立完成优化（这在前面尤为常见，我大学的基础课程学的非常差，一度觉得自己在给项目丢人），那就要请教别人之后完成优化或者和别人一起完成优化，无论是哪一种，我都要仔细查阅文档，并仔细对比优化后的效果。成功的优化让我感到快乐，希望有一天我也能成为非常厉害的人。
    
