## 后端记录

#### 查看某个redis连接数过高

- 问题起因： 查询某个问题的时候在lua-server查看netstat，发现连接redis某个公用端口：6380的连接中CLOSE_WAIT过高![](assets/16589158799398.jpg)

- 排查经过：
    1. 怀疑redis没有keepalive，这个仔细看了一下项目的源码，发现每次`do_command` 之后都会把连接放回池子，排除这种可能
    2. 怀疑有些慢指令，后面命令在排队，`slowlog get 20`看了一下，发现没有最近一段时间的慢指令
    3. 查看redis具体在干嘛，谨慎的把两个公用redis（6379，6380）开了几秒monitor，发现6380日志大小是6379的七倍，那看来应该首先从指令数上就不对，6380这个redis我们用的并不多。接下来基于monitor的日志排查
        
        - awk print一下命令前缀，发现hget，get，set，zrank，zrevrange这种比较高，联系项目实际情况，hget, zrank, zrevrange这几个指令是boss系统的指令，对boss这个模块优化![](assets/16589156408945.jpg)
        - 优化后发现其实还是很高，vim进去看看这些get set都在干啥，发现好像是上锁操作，看看自己项目的上锁代码确实没有keepalive并且占用了公用redis端口，问题这个时候已经查到了![](assets/16589157649199.jpg)
        
    
- 解决方法：
    新开一个端口用于上锁，同时请求结束的时候lock的redis统一keepalive，连接数过高的情况解决
