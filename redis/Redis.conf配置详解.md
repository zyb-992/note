# Redis.conf配置详解



1.  unit单位对大小写不敏感
   
   ![](D:\Program%20Files\电子书\go\md\image\2022-07-18-22-39-13-image.png)

2.  添加多个配置文件
   
   ![](D:\Program%20Files\电子书\go\md\image\2022-07-18-22-40-42-1658155224311.png)

3. 网络
   
   ```
   - bind 127.0.0.1 -::1
     
   - protected-mode yes
     
   - port 6379
   ```
   
   

4.  通用配置
   
   > General
   
   ```
   # 以守护进程方式进行
   daemonize yes
   
   
   # 若配置redis以后台方式运行 需要指定一个pid文件
   pidfile /var/run/redis_6379.pid
   
   # 日志
   # debug:用于生产调试信息
   # verbose:
   # notice: 生产环境
   # warning: 
   loglevel notice
   
   # 生产的日志文件
   logfile ""
   
   # 16个数据库
   database 16
   ```
   
   > 快照
   > 
   > 持久化 在规定时间内执行了多少次操作 就会持久化到.rdb文件 / .aof文件
   > 
   > **redis是内存数据库 如果没有持久化 那么数据断电就消失**
   
   ```
   # 如果900s内 如果至少有1个key进行修改 就进行持久化操作
   save 900 1
   
   # 如果300s内 如果至少有10个key进行了修改 就进行持久化操作
   save 300 10
   
   # 同理 以后可以自己设置该值
   save 60  10000
   
   
   # 若持久化出错 是否继续redis工作
   stop-writes-on-bgsave-error yes
   
   # 是否压缩rdb文件
   rdbcompression yes // 消耗cpu资源
   
   # 保存rdb文件的时候 是否进行错误校验
   rdbchecksum yes
   
   
   # rdb文件保存的目录
   dir ./ 
   
   ```
   
   > REPLICATION 复制  主从复制？
   
   > SECURITY 安全（设置密码等
   
   ```
   # 查询登录密码 默认为空
   config get requirepass 
   
   # 设置登陆密码 123456
   config set requirepass "123456"
   
   # 登录 
   auth 123456
   
   登陆成功！
   ```
   
   

5. 客户端
   
   ```
   # 设置连接redis服务器的最大数量
   maxclients 10000
   
   # redis的最大内存容量
   maxmemory <bytes>
   
   # 内存到达上限后的处理策略
   maxmemory-policy noeciction
   ```
   
6.  AOF配置（APPEND ONLY MODE）
   
   1. 默认不开启aof模式 默认使用的是rdb方式持久化
   
   2. 大部分所有情况下 RDB模式完全够用
   
   3. ```
      # 默认不开启aof
      appendonly no
      
      # aof持久化的文件名字
      appendfilename "appendonly.aof"
      
      
      # 每秒执行一次sync（同步） 可能会丢失这1s的数据
      appendfsync everysc
      
      # 每次修改都会sync 同步 消耗性能
      appendfsync always
      ```
      
   4. 
