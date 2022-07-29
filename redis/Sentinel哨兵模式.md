# Sentinel哨兵模式

1. **哨兵**是Redis的高可用性解决方案

2. **一个或多个哨兵实例组成的Sentinel系统可以监视任意多个主服务器以及主服务器属下的所有从服务器**

3. 启动并初始化Sentinel

   ```redis
   # linux命令
   : redis-sentinel /path/sentinel.conf
   : redis-server  /path/sentinel.conf --sentinel
   
   # 效果
   启动时 执行步骤
   1. 初始化服务器
   2. 将普通服务器使用的代码替换为Sentinel专用代码
   3. 初始化Sentinel状态
   4. 根据配置文件 初始化哨兵系统的监视主服务器列表
   5. 创建连向主服务器的网络连接
   ```

4. 初始化Sentinel服务器

   1. Sentinel服务器与Redis普通服务器执行工作不同 因此两者初始化过程并不完全相同

      ![image-20220729001303438](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220729001303438.png)

   2. **Sentinel不使用数据库 初始化Sentinel不会载入RDB和AOF文件**

      

   3. 使用Sentinel专用代码

      ```
      # 端口
      define REDIS_SENTINEL_PORT 26379
      
      # 使用sentinel.c/sentinelcmds作为服务器命令表
      PING
      SENTINEL
      INFO
      SUBSCRIBE
      UNSUBSCRIBE
      PSUBSUCRIBE
      PUNSUBSCRIBE
      ```

      

   4. 初始化Sentinel状态

      ```
      # Sentinel状态
      sturct sentinelState{
      	dict *masters;
      }
      
      # master属性
      # 键：被监视的主服务器的名字
      # 值：被监视的主服务器对应的
      # sentinelRedisInstance实例
      # sentinelRedisInstance：主/从/Sentinel服务器 
      
      ```

      - master字典的初始化是根据被载入的Sentinel配置文件来进行

        

   5. Sentinel创建连向主服务器的两条网络连接

      - 命令连接：专门用于向主服务器发送命令 并接收命令进行回复

      - 订阅连接：专门用于订阅主服务器的__sentinel:hello__频道

      - Redis的发布与订阅功能中 订阅后发送的消息不会保存在Redis服务器中 因此需要两个异步连接

        ![image-20220729003609707](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220729003609707.png)

        

5. 获取主服务器信息

   1. **Sentinel默认会以10s一次的频率通过命令连接向被监视的主服务器发送INFO命令，并通过分析INFO命令的恢复来获取主服务器当前的各种信息**

      1. 主要信息
         - 主服务器的运行ID：run_id
         - role域的服务器角色
         - 主服务器下的所有从服务器的IP地址及端口偏移量等
      2. 根据主服务器的run_id和role域记录的信息 Sentinel将对主服务器的实例结构进行更新
      3. 主服务器返回的从服务器的服务器信息将用于更新主服务器实例结构的slave字段
      4. **Sentinel**通过分析INFO命令得到从服务器信息时 会在slave字段中寻找是否有对应的从服务器实例结构
         1. 若有 则对该实例结构进行更新
         2. 否则 在slave字段中为这个从服务器创建一个新的实例结构

      ![image-20220729133231224](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220729133231224.png)

6. 获取从服务器信息

   1. **当Sentinel发现主服务器有新的从服务器出现时 Sentinel会为该从服务器创建实例结构 以及创建连接到从服务器的命令连接和订阅连接**

      ![image-20220729133738651](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220729133738651.png)

   2. **创建命令连接后 Sentinel在默认情况下 会以10s一次的频率通过命令连接向从服务器发送INFO命令**

      

7. 向主服务器和从服务器发送信息

   1. **默认情况下 Sentinel会以每2s一次的频率 通过命令连接向所有被监视的主服务器和从服务器发送固定格式的命令**

      ​		**该命令向服务器的____Sentinel:hello____频道发送信息**

      ```
      s_ 是Sentinel本身的IP、端口、运行ID、配置纪元（计数器）
      m_ 是Sentinel监视的主服务器信息/监视的从服务器正在复制的主服务器的信息
      ```

      

      ![image-20220729140958992](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220729140958992.png)

      

8. 接收来自主服务器和从服务器的频道信息

   1. **当Sentinel与一个主服务器或者从服务器建立起订阅连接后 Sentinel会通过订阅连接 向服务器发送命令**

      **Sentinel对该频道的订阅会一直持续到Sentinel与服务器的连接断开为止**

      ```
      SUBSCRIBE __sentinel__:hello
      ```

      

   2. **对于每个与Sentinel连接的服务器 Sentinel既通过命令连接向服务器的频道发送信息 又通过订阅连接从服务器的该频道接收信息**

      ![](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220729142424944.png)

   3. 更新主服务器实例结构的sentinel字典

      1. **sentinel字典是存储监视该主服务器的所有Sentinel结构**

         

      2. 字典的键是某个Sentinel的IP和PORT 字典的值是每个Sentinel对应的实例结构(SentinelRedisInstance)

         

      3. 当一个Sentinel(目的Sentinel)接收到其他Sentinel(源Sentinel)发送的信息时 目标Sentinel会根据信息提取出两方面参数

         1. 与源Sentinel有关的参数: IP PORT run_id 配置纪元

         2. 与主服务器有关的参数：源Sentinel正在监视的主服务器的各种信息

            

      4. 根据提取出的参数 目的Sentinel会在自己的Sentinel状态**(sentinelState)**中查找masters字典中的主服务器对应的实例结构 然后根据提取出的Sentinel参数 检查**主服务器实例结构的sentinels字典**中 源Sentinel结构是否存在

         1. 存在 则相应的对该结构进行更新
         2. 不存在 则说明源Sentinel是刚刚加入的对主服务器进行监控的一个哨兵 为此**目标Sentinel**会为**源Sentinel**创建一个新的实例结构 并将这个结构添加到**主服务器的sentinels字典中**

         ![image-20220729144307579](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220729144307579.png)

   4. 创建连向其他Sentinel的命令连接

      1. Sentinel与服务器之间有两条连接：命令连接与订阅连接
      2. Sentinel与Sentinel之间有两条连接：都是命令连接

      ![image-20220729144542440](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220729144542440.png)

      

9. **Sentinel检测服务器主观下线状态**

   1. **默认情况下 Sentinel会以每秒一次的频率向所有与它创建了命令连接的实例（主从服务器、Sentinel）发送Ping命令 通过返回的信息判断实例是否在线**

      1. 有效回复：+PONG / -LOADING / -MASTERDOWN

      2. 无效回复：除+PONG / -LOADING / -MASTERDOWN以外的回复 当收到无效回复时则主观认为该实例下线

         

   2. Sentinel配置文件中的down-after-millisenconds代表了Sentinel用来判断实例主观进入下线的时间长度：如果再Ping命令发送后的down-after-millisenconds时间内 实例连续向Sentinel发送无效回复 则该Sentinel将该实例主观判定为下线状态

      

10. **Sentinel检测服务器客观下线状态**

    1. **当Sentinel将一个主服务器判定为下线状态后 它还需要向其他监视该主服务器的Sentinel确认是否真的下线了**

       

    2. 发送SENTINEL is-master-down-by-addr命令 

       1. 询问其他Sentinel是否同意主服务器下线

          

    3. 接收SENTINEL is-master-down-by-addr命令 

       1. 当目的Sentinel接收到源Sentinel发送的这条命令后 需要对命令中的参数解析 然后根据其中给出的主服务器的IP和端口号 检查主服务器是否已下线 然后向源Sentinel返回一条包含三个参数的回复

       ![image-20220729145623974](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220729145623974.png)

    4. 接收SENTINEL is-master-down-by-addr命令的回复

       1. **根据接收其他Sentinel发来的回复 源Sentinel将统计同意将主服务器判定为下线状态的Sentinel数量 当达到配置指定的数量时 源Sentinel会将该主服务器实例结构的flag置为SRI_O_DOWN标识 表示主服务器进入客观下线状态**

          ![image-20220729145925590](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220729145925590.png)

       2. 注意：**不同的Sentinel判断客观下线的条件可能不同**

          

11. 选举领头Sentinel

    ![image-20220729150202231](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220729150202231.png)

    ![image-20220729150224375](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220729150224375.png)

    

12. **故障转移**

    1. **选出领头Sentinel后 领头将会对下线的主服务器进行故障转移操作**

       1. 在已下线的主服务器的所有从服务器中挑选出一个从服务器 并将其转化为主服务器
       2. 让已下线的主服务器属下的从服务器的主服务器更新为当前的主服务器
       3. 将已下线的主服务器设置为当前的主服务器的从服务器

    2. 选出新的主服务器

       1. 领头挑选出一个状态良好、数据完整的从服务器
       2. 向该服务器发送`SLAVEOF no one`命令 将这个服务器设置为主服务器
       3. 设置为新的主服务器的从服务器是如何挑选出来的？

       ![image-20220729150659958](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220729150659958.png)	

