---
layout:     post
title:      Redis-Cell 漏斗限流的使用
subtitle:   
date:       2021-04-14
author:     zhengguohuang
header-img: img/post-bg-rixi2.jpg
catalog: true
tags:
    - Redis
---

# Redis-Cell 漏斗限流的使用

1. 下载安装包https://github.com/brandur/redis-cell

2. 解压

   ```bash
   $ tar -zxf redis-cell-*.tar.gz
   $ cp libredis_cell.so /path/to/modules/
   ```

3. 进入 redis-cli，执行命令module load /aaa/bbb/libredis_cell.so
   这是临时的办法，重启redis就没了，或者$ redis-server --loadmodule /aaa/bbb/libredis_cell.so
   加到启动命令里。或者添加到配置文件

   如果这步失败，看日志文件中version `GLIBC_2.18' not found (required by /soft/redis/modules/libredis_cell.so)

   则需要升级GLIBC

   - 下载解压

   ```bash
   wget https://ftp.gnu.org/gnu/glibc/glibc-2.18.tar.gz
   tar -zxvf glibc-2.18.tar.gz
   ```

   - 编译安装

   ```bash
   cd glibc-2.18 && mkdir build
   cd build
   ../configure --prefix=/usr --disable-profile --enable-add-ons --with-headers=/usr/include --with-binutils=/usr/bin
   make && make install
   ```

   - 验证

   ```bash
   [root@VM_0_7_centos build]# ll /lib64/libc.so.6
   lrwxrwxrwx 1 root root 12 Mar 25 09:01 /lib64/libc.so.6 -> libc-2.18.so
   ```

   - 重启redis

4. 进入 redis-cli，执行命令，可以看看是否生效。

   ```bash
   127.0.0.1:6379> module load /soft/redis/modules/libredis_cell.so
   OK
   127.0.0.1:6379> command info CL.THROTTLE
   1) 1) "cl.throttle"
      2) (integer) -1
      3) 1) write
      4) (integer) 1
      5) (integer) 1
      6) (integer) 1
   127.0.0.1:6379> MODULE LIST
   1) 1) "name"
      2) "redis-cell"
      3) "ver"
      4) (integer) 1
   127.0.0.1:6379>
   ```

5. 使用

在高并发场景下有三把利器保护系统：缓存、降级、和限流。缓存的目的是提升系统的访问你速度和增大系统能处理的容量；降级是当服务出问题或影响到核心流程的性能则需要暂时屏蔽掉。而有些场景则需要限制并发请求量，如秒杀、抢购、发帖、评论、恶意爬虫等。

限流算法 常见的限流算法有：计数器，漏桶、令牌桶。

漏桶（Leaky Bucket）算法

思路：水（请求）先进入到漏桶里，漏桶一定的速度出水（接口有响应速率），当水流入速度过快时会直接溢出（访问速度超过接口响应速度），然后拒绝请求。

![congestion-control-1-25-638](https://gitee.com/zhengguohuang/img/raw/master/img/congestion-control-1-25-638.jpg)

Java实现的漏桶

```java
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class FunnelRateLimiter {
    static class Funnel {
        //漏斗的容量
        int capacity;
        //速率
        float leakingRate;
        //剩余容量
        int leftQuota;
        long leakingTs;

        public Funnel(int capacity, float leakingRate) {
            this.capacity = capacity;
            this.leakingRate = leakingRate;
            this.leftQuota = capacity;
            this.leakingTs = System.currentTimeMillis();
        }

        void makeSpace() {
            long nowTs = System.currentTimeMillis();
            long deltaTs = nowTs - leakingTs;
            int deltaQuota = (int) (deltaTs * leakingRate);
            // 间隔时间太长，整数数字过大溢出
            if (deltaQuota < 0) {
                this.leftQuota = capacity;
                this.leakingTs = nowTs;
                return;
            }
            // 腾出空间太小，最小单位是 1
            if (deltaQuota < 1) {
                return;
            }
            this.leftQuota += deltaQuota;
            this.leakingTs = nowTs;
            if (this.leftQuota > this.capacity) {
                this.leftQuota = this.capacity;
            }
        }

        boolean watering(int quota) {
            makeSpace();
            if (this.leftQuota >= quota) {
                this.leftQuota -= quota;
                return true;
            }
            return false;
        }
    }

    private Map<String, Funnel> funnels = new ConcurrentHashMap<>();

    public boolean isActionAllowed(String userId, String actionKey, int capacity, float leakingRate) {
        String key = String.format("%s:%s", userId, actionKey);
        Funnel funnel = funnels.get(key);
        if (funnel == null) {
            funnel = new Funnel(capacity, leakingRate);
            funnels.put(key, funnel);
        }
        // 需要 1 个 quota
        return funnel.watering(1);
    }
}

```

核心逻辑就是makeSpace，在每次灌水前调用以触发漏水，给漏斗腾出空间。 funnels我们可以利用Redis中的hash结构来存储对应字段，灌水时将字段取出进行逻辑运算后再存入hash结构中即可完成一次行为频度的检测。但这有个问题就是整个过程的原子性无法保证，意味着要用锁来控制，但如果加锁失败，就要重试或者放弃，这回导致性能下降和影响用户体验，同时代码复杂度也升高了，此时Redis提供了一个插件，Redis-Cell出现了。

Redis4.0提供的一个限流模块Redis-Cell，提供漏斗算法，并提供原子限流指令。

```bash
CL.THROTTLE <key> <max_burst> <count per period> <period> [<quantity>]
```

key是限流的标志，可能是：

- 用户唯一标识符
- 原始IP地址
- 静态字符串（比如global）来限制进入系统的行为

```
CL.THROTTLE user123 15 30 60 1
               ▲     ▲  ▲  ▲ ▲
               |     |  |  | └───── apply 1 token (default if omitted)
               |     |  └──┴─────── 30 tokens / 60 seconds，即每两秒钟滴下来一滴水
               |     └───────────── 15 max_burst，0 ~ 15共16滴水
               └─────────────────── key "user123"
```

该指令意思为，允许用户user123请求的频率为每60s最多30次，漏斗初始容量为16（因为是从0开始计数，到15为16个），默认每个行为占据的空间为1（可选参数）。

返回值

```bash
127.0.0.1:6379> CL.THROTTLE test 5 5 60 1
1) (integer) 0			# 0表示允许，1表示拒绝
2) (integer) 6			# 漏斗容量capacity，可容纳的最大水滴数
3) (integer) 5			# 漏斗剩余空间left_quota
4) (integer) -1			# 如果拒绝了，需要多少秒后再试
5) (integer) 12			# 多少秒后，漏斗完全空
```

Java代码实现

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisCluster;
import redis.clients.jedis.commands.JedisCommands;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

public class RedisCellDistributedRateLimiter {
    /** jedis命令客户端*/
    protected JedisCommands jedis;

    /** 自定义限流key*/
    protected String key;

    /** 时间窗口 单位秒*/
    protected int period;

    /**最大令牌容量*/
    protected int maxCapacity;

    public RedisCellDistributedRateLimiter(JedisCommands jedis, String key, int period, int maxCapacity) {
        this.jedis = jedis;
        this.key = key;
        this.period = period;
        this.maxCapacity = maxCapacity;
    }

    /**
     * 尝试申请资源
     * @param quote 目标资源数
     * @return 是否申请成功
     */
    public boolean tryAcquire(int quote) throws IOException {
        List<String> keys = new ArrayList<>();
        keys.add(key);
        List<String> argvs = new ArrayList<>();
        argvs.add(String.valueOf(maxCapacity));
        argvs.add(String.valueOf(maxCapacity));
        argvs.add(String.valueOf(period));
        argvs.add(String.valueOf(quote));
        Object result = null;
        if (jedis instanceof Jedis) {
            result = ((Jedis) this.jedis).eval(LUA_SCRIPT, keys, argvs);
        } else if (jedis instanceof JedisCluster) {
            result = ((JedisCluster) this.jedis).eval(LUA_SCRIPT, keys, argvs);
        } else {
            throw new RuntimeException("redis instance is error") ;
        }
        List<Long> res =  (List<Long>)result;
        System.out.println("通过拒绝：\t" + res.get(0));
        System.out.println("漏斗容量：\t" + res.get(1));
        System.out.println("剩余空间：\t" + res.get(2));
        System.out.println("重试时间：\t" + res.get(3));
        System.out.println("漏空时间：\t" + res.get(4));

        return ((List<Long>)result).get(0) == 0;
    }

    public static final String LUA_SCRIPT = "local key = KEYS[1]\n"+
            "local init_burst = tonumber(ARGV[1])\n"+
            "local max_burst = tonumber(ARGV[2])\n"+
            "local period = tonumber(ARGV[3])\n"+
            "local quota = ARGV[4]\n"+
            "return redis.call('CL.THROTTLE',key,init_burst,max_burst,period,quota)";
    public static void main(String[] args) throws Exception{
        //构造Jedis集群
        Jedis jedisCommands = new Jedis("ip", 6379);
        jedisCommands.auth("密码");
        String key = "test";
        int period = 60;
        int maxCapacity = 5;
        //构造限流器
        RedisCellDistributedRateLimiter limiter =
                new RedisCellDistributedRateLimiter(jedisCommands, key, period, maxCapacity);
        for (;;){
            Thread.sleep(1000);

            if (limiter.tryAcquire(1)){
                System.out.println(("---------通过限流---------"));
            }else {
                System.out.println(("---------无法通过---------"));
            }
        }
    }
}
```





参考

- https://zhuanlan.zhihu.com/p/97321026
- https://github.com/brandur/redis-cell
- https://blog.csdn.net/rlk512974883/article/details/103928569
- https://www.jianshu.com/p/41781605ed29
- https://www.jianshu.com/p/7815852d6c95
- https://zhuanlan.zhihu.com/p/306717681
- https://blog.csdn.net/zhouwenjun0820/article/details/107471324