# 游戏登陆流程

### 概览

1. C#端`XYDApp.cs` 模块打开loading页面并启动`LuaManager`，从而启动lua虚拟机_luaState
2. `main.lua`模块被第一个加载，它主要是定义了游戏内的面向对象class方法，并启动`boot.lua`
3. boot中主要是根据`XYDDef.cs`中的pkgname和操作系统是ios/android来确定xyd.pkgUpdateURL字段（引导玩家更底包），并启动`UpdateController`
4. UpdateController主要是下载version文件，根据0x0003协议，version文件和本地保存的版本信息判断需要下载哪些资源或者是否更新底包等信息，确定满足启动条件之后，启动GUIScene和`Game.lua`
5. Game模块启动，会加载`xinyoudi.lua`，我们游戏大部分的单例类，全局表，全局函数都定义在xinyoudi中。加载完xinyoudi后，还会加载backend和游戏内所有的model（调用selfplayer:init()），之后启动`loadingController`
6. LoadingController主要是启动SDKManager，然后外网调用SDKManager的login，就是我们说的SDK登陆，内网或者测试服会跳过sdk登陆（UpdateController里跳的），打开内网账号登陆的LoginWindow
7. 外网情况下，收到SDK的login回调，会判断封号（注意这里是官网封号）等信息，然后把token，sid等信息通过0x0001通过HTTP request发给后端，0x0001可以从后端获取本服的封号信息（注意区别于官网封号，这个可以提示玩家换个服玩），账号注销信息，还有玩家的gate信息。玩家如果没有被封号或进行注销账号等操作，那么他就会根据玩家的gate信息与gate服务器建立TCP连接
8. 玩家和gate的TCP连接建立后，会发送0x0006协议，这里是后端检查token是否对应，来告诉gate玩家是否认证成功
9. 认证成功之后，发送0x0008协议和一些SDKChat信息获取的协议，初始化一些前端信息，0x0008回来之后调用`MainController`的`onGameStart`接口，这里主要是开一些最开始要开的窗口，然后开始backupdate。到这一步，玩家就进入挂机界面了，也就是玩家理解的登上账号了。


#### C#模块简介
APPScene上面可以找到XYDApp这个gameObject，然后上面挂载了XYDApp这个脚本，所以APPScene启动的时候XYDApp.cs这个脚本就会启动。他的Start函数里面主要包含了FPS的启动，XYDDef的启动，并调用StartApp，这个里面启动了UIManager（loading界面的启动），并调用startLua（这个函数启动LUAManager去启动lua虚拟机），这里不是我们今天的重点内容，只要理解lua层是XYDApp->LuaManager->lua_state->main.lua这个流程起的就可以

#### main，boot
这两个模块内容比较少，和概览里面的内容差不多（需要注意全局的面向对象方法class在main.lua里面），主要是Lua进来之后，通过XYDDef拿一些全局配置（包名，游戏语言等信息），然后启动UpdateController

#### UpdateController
UpdateController是我们游戏热更的关键，不能随便修改里面的代码。translation表，loading时候的提示先被加载，这个时候如果是在内网，那么会调用`startGame`，否则会调用`checkUpdate`，简单来说就是内网绕过了资源版本监测和SDK login这些环节，直接起了GUIScene和`Game.lua`模块。

继续看UpdateController的功能，下面介绍的都是ios或者android的情况，也就是走`checkUpdate()`的玩家

1. 第一步从PlayerPrefs里面拿到__version__这个字段，这里提一下，自己游戏内持久化数据的时候（比如做一些前端红点），尽量用xyd.db.misc而不是PlayerPrefs，并且定义字段名字一定要有区分度
2. 第二步checkResVersion函数，这里向后端发送**0x0003**，这里是向后端拿version信息，如果多次拿不到，会报**e:001**这个报错，方便定位问题。如果报e001，则大概率是连不上后端服务器（everlegionback），检查线上的这个问题，浏览器访问一下http://everlegionback.carolgames.com/api/v1 看看是不是连不上我们游戏的后端api/v1
3. checkResVersion从后端拿到版本信息，这里的版本信息包含了当前版本（serverversion），最小热更版本（minHotVersion），最小包版本（minPkgVersion），还有后端服务器的地址和端口（ios审核服是另一个，正常玩家还是连的上面的everlegionback.carolgames.com）。然后会根据系统和从后端拿到的版本信息，以及CDN服务器的地址，拿到一个version.json文件的地址。
4. 从前端PlayerPrefs里拿到当前版本信息，如果这里判断版本已经等于serverVersion，则进入游戏（直接看6）；如果小于minPkgVersion，则玩家需要更新底包，会引导玩家去商店更包；如果不等于serverVersion但大于minHotVersion，就把version信息塞进backupdateController里面，启动游戏让他偷偷玩，后端继续热更；如果小于minHotVersion，就会用刚才拿CDN和版本信息拼到的URL，下载一下version.json文件（如果下不下来，那就会打印**e:002**，这里如果报了，访问一下http://everlegionres.carolgames.com 看看能不能连上我们游戏的后端，这个和后面的**e:003**是一个排查方法，先问下是不是前端version文件没传，如果有个设备频繁报**e:002**或者**e:003**这种，是自己人就把它设备拿过来看看或者让他自己ping一下，如果是比较多玩家这样报。可以用一下前端的`xyd.SDKManager`的`GetPingRes`接口辅助ping一下后端，发一下日志。
5. version.json准备好之后，updateRes会把每条versionInfo（里面有version，md5等信息）对应的文件下下来，多次下不下来某一条会报**e:003**。全都下载完之后，标记更新状态为结束，并重启游戏
6. 重启游戏后，这次更新就完成了，ResManager会调用preloadAbAsync，前端加载GUIScene，lua加载Game模块

#### Game
Game模块主要就是做三件事
1. 初始化config，xinyoudi.lua，全局变量，全局函数，单例类等，基本都在xinyoudi里，所以这一步之后，xyd.xxxx基本就都可用了
2. 初始化所有的models，这一步之后前端就完成了models的准备
3. 启动loadingController

#### LoadingController
这里分两种情况，第一种就是内网，在刚才的UpdateController中提到，内网不会做下载资源这一步，所以直接进了loadingController，这里内网是开一个绿色的能输入端口的窗口。第二种是SDK login，然后收到回调之后判断是否被官网封号（封号但没有封号信息**e:004**，没见过这种情况，所以不总结怎么排查）

两种登陆，在非SDKLogin点击登陆，或者SDKLogin的回调和官网封号流程走完之后，会向后端发送**0x001**，注意这里是HTTP请求，并不经过gate，也不需要proto协议等。这里HTTP请求收不到回调会报**e:005**，报这个错误是api/v1的后端连不上或者后端`get_access_token`报错，这里可能是token校验不通过或者连到了不该连上的服务器，后面这种情况不多，token校验不通过可能是玩家短时间请求了几次（上次平台检查问题说的），这里后面考虑给玩家部分请求加个锁，因为玩家HTTP请求不是200的情况会重发，但可能不是200的情况是超时了之类的情况，token最好不要反复获取。
如果0x0001协议正常返回，那么会返回gate信息，封号信息，注销账号信息等。如果有登陆之前的校验信息，一般是放在0x0001的返回里。

登陆协议返回后，判断玩家是不是在本服被封了，如果被封了，有提示他可以换个服玩。如果没异常情况，就根据0x0001返回的gate信息（ip+端口），跟gate进行TCP连接（backend:connect()）
在socket建立TCP连接之后(如果连不上gate会报**e:006**)，会发**0x0006**协议，这个协议会发给后端去校验token，并**告知gate校验成功**。这个过程有两种错误情况：校验失败则是**e:007**一般不会发生，如果发生了，可以检查是不是前端在校验之前又发了0001，导致后端player对应的token被取代，而**e:008**则表示跟gate发消息的时候超时了，e:006,e:008这种要根据错误信息提供的端口排查gate本身有没有问题，比如负载过高或者进程关闭

这一步完成之后玩家就连上gate并校验成功了，然后就会发**0x0008**和其他一些登陆时候请求的协议（聊天信息等），在0x0008返回之后，MainController就会起起来了。

#### MainController
0x0008->selfPlayer分发GAME_START, mainController->onGameStart
这里onGameStart说明登陆的时候的协议已经基本处理完了，这时前端会开一些必要的窗口，比如flyItemWindow，stageWindow（挂机战斗界面）。整个登陆流程就彻底完成了


## 错误码汇总

错误码 | 原因 | 排查
--- | --- | ---
e:001| 连不上游戏后端api/v1|浏览器访问http://everlegionback.carolgames.com/api/v1
e:002|下不到version文件|大概率是前端打完包没传version.json文件，先问下前端打包有没有问题，没问题再ping everlegionres.carolgames.com
e:003|某条versionInfo对应的资源下不到| ping everlegionres.carolgames.com ，检查一下网络问题
e:004|玩家封号了，但没有封号信息|联系运营排查封号的时候是不是没选理由
e:005|0x0001HTTP请求失败|看下是不是后端是不是因为平台token校验不过报很多LOGIN FAILED，问下平台同事原因
e:006|连gate超时|根据报错打的gate host+port，看看是哪个gate进行排查
e:007|gate校验不通过|没见过，看看是不是0x001返回后又发了0x001
e:008|跟gate发消息超时|根据报错打的gate host+port，看看是哪个gate进行排查


