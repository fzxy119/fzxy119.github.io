---
title: Redis高可用
date: 2021-09-22 21:11:51
tags:
- redis
categories:
- 技术
---

## Sentinel相关配置

bind 12.3.10.222

port 26377

sentinel monitor mymaster 12.3.10.222 6377 2

sentinel auth-pass mymaster cx

sentinel down-after-milliseconds mymaster 30000

sentinel parallel-syncs mymaster 3

sentinel failover-timeout mymaster 180000

loglevel debug    logfile "/usr/redis/log/sentinel-26377.log"     daemonize yes

启动方式

1. redis-sentinel /path/to/sentinel.conf

2. redis-server /path/to/sentinel.conf --sentinel



<!-- more -->



## Sentinel及服务状态感知能力

环境内信息共享，sentinel与主服务从服务之间建立命令链接及频道订阅发布链接，用以感知从服务器加入，新的sentinel加入

​	1. sentinel每10秒向主服务发送INFO命令收集信息，收集从服务信息，**发现新的服务加入**，并建立从服务数据结构，建立从服务命令及频道链接，

​	2. 建立好从服务链接后每10秒发送INFO命令收集从服务状态信息，并更新数据结构中从服务的状态信息

​	3. 向主服务器从服务器发布 \_\_sentinel\_\_:hello  频道  共享sentinel自身状态及主服务信息，所有sentinel发布信息到此频道

```
PUBLISH  __sentinel__:hello "s_ip s_port s_runid s_epoch m_name m_ip m_port m_epoch"
s_ip s_port s_runid s_epoch  发送消息的sentinel信息
m_name m_ip m_port m_epoch   发送消息的sentinel监测的master信息
```

​	4. 订阅主服务器从服务器 \_\_sentinel\_\_:hello  频道  感知其他sentinel信息及主服务信息，所有sentinel订阅此频道，并接受消息



## 状态检测：sentinel与所有服务包括主从服务sentinel服务保持命令链接

​	sentinel除了和主服务从服务建立命令链接和频道链接外，还与其他sentinel建立命令链接，用于状态监测 PING命令执行

###  		主观下线状态监测 PING

1. sentinel 和主从服务及其他sentinel建立连接后会以每秒一次的频率发送PING命令，并根据回复来判断实例是否下线，PING命令有固定的几个有效的回复，有效的回复有+PONG，-LOADING,-MASTERDOWN三种回复，除了三种回复的结果都是无效回复。

2. sentinel配置文件中 down-after-milliseconds 配置是sentinel判断主观下线所需的时间长度，在down-after-milliseconds毫秒内服务连续向sentinel返回无效回复，那么sentinel将实例结构中flags属性中打卡 SRI-S-DOWN标识，以此来表示实例进入主管下线状态。

3. down-after-milliseconds 配置不仅作用在判断主从服务下线检查，也作用在sentinel下线检查时长

   sentinel monitor mymaster host  port  quorum(客观下线所支持的投票数量)

   sentinel down-after-milliseconds mymaster  30000

### 		 客观下线状态监测 SENTINEL  is-master-down-by-addr 命令:

当sentinel判断主服务器为主观下线的时候，会向别的监视主服务的sentinel进行询问，看他们是否也认为主服务器已进入了下线状态（可已是主观也可以是客观），**当接收到足够数量的下线判断**（quorum）后，sentinel会将服务器判定为客观下线并进行故障转移

询问是否已下线请求：

```redis
发送：SENTINEL is-master-down-by-addr ip port current_epoch  runid
ip 被sentinel判断为主观下次的主服务ip
port 被sentinel判断为主观下次的主服务端口
current_epoch sentinel当前的配置纪元，用于sentinel领头选举
runid *或者是sentinel运行id

回复：down_state leader_runid leader_epoch
down_state 1 服务已下线 0 服务未下线
leader_runid  *或者是sentinel运行id，用于sentinel领头选举，询问阶段返回*
leader_epoch  配置纪元，用于sentinel领头选举 询问阶段返回0

```



##  选举领头sentinel

- 当sentinel监测到主服务下线会进行leader选举，并由选举出的sentinel负责进行故障转移，选举的命令是通过SENTINEL is-master-down-by-addr 进行发送的，并根据返回结果进行收集，当超过半数的sentinel服务投票决定，那台sentinel将成为leader。
- 当客观quorum达到后sentinel会向别的sentinel（目标）发送SENTINEL is-master-down-by-addr ip port current_epoch  runid 命令，并携带current_epoch  ，runid 
- 目标sentinel接收到is-master-down-by-addr  请求，如果当前sentinel未进行过投票，则返回发送自己的epoch  ，和请求投票的sentinel的runid表示投票，如果已经投过票，就拒绝请求
- 源sentinel接受到回复后，会比对leader_epoch 纪元是否和自己的相等，如果相等判断leader_runid是否是自己的runid如果是，表明投票给自己。
- 每个sentinel在每个纪元epoch内只能进行一次的投票，并且只能投票一次，根据请求到达的先后顺序，投票遵循先到先得，leader申请的sentinel根据的leader_runid，leader_epoch结果统计投票信息，并在过半后成为leader。
- 每个sentinel都有机会成为leader，每次lead选举无论成功或者失败epoch纪元都会增1
- 如果给定时限内没有一个sentinel被选举为leader，那么sentinel将在一段时间后再次进行选举，知道选举出leader为止。



## 实现故障转移

在已下线的主服务器的从服务器里挑选一个从服务，并将其转换为主服务

1. ​     选择从服务规则（主服务中slaves列表）
2. ​      删除已经下线的从服务
3. ​	  删除最近5秒内没有回复过leader的从服务
4. ​      删除与主服务断开连接超过 down-after-milliseconds * 10 毫秒的从服务
5. ​     根据从服务的优先级排序，选择优先级最高的从服务如果有多个，将按照复制偏移量最大的优先，如果优先级复制偏移量都一样，就按照runid进行排序 宣传runid最小的为主服务
6.  	向从服务发送 SLAVEOF no one 命令，调整从服务未主服务

将已下线的主服务下的所有从服务，改为复制挑选出的主服务

- ​	向未被选择的从服务发送 SLAVEOF new_master new_master_por 进行复制新的主服务命令，并调整配置文件

将已下线的主服务器设置为新选择的主服务的从服务，当就的主服务重新上线是，他会成为新的主服务的从服务

- 调整配置文件

<!--所有被调整过的服务，redis.conf 复制服务配置将被调整-->

sentinel.c 关于sentinel的状态的数据结构

```c
/* Main state. */
struct sentinelState {
    char myid[CONFIG_RUN_ID_SIZE+1]; /* This sentinel ID. */
    uint64_t current_epoch;         /* Current epoch. */
    
    /* 报错所有被这个sentinel监视的服务 结构是sentinelRedisInstances */
    dict *masters;      /* Dictionary of master sentinelRedisInstances.
                           Key is the instance name, value is the
                           sentinelRedisInstance structure pointer. */
    
    
    int tilt;           /* Are we in TILT mode? */
    int running_scripts;    /* Number of scripts in execution right now. */
    mstime_t tilt_start_time;       /* When TITL started. */
    mstime_t previous_time;         /* Last time we ran the time handler. */
    list *scripts_queue;            /* Queue of user scripts to execute. */
    char *announce_ip;  /* IP addr that is gossiped to other sentinels if
                           not NULL. */
    int announce_port;  /* Port that is gossiped to other sentinels if
                           non zero. */
    unsigned long simfailure_flags; /* Failures simulation. */
} sentinel;

typedef struct sentinelRedisInstance {
    int flags;      /* See SRI_... defines */
    char *name;     /* Master name from the point of view of this sentinel. */
    char *runid;    /* Run ID of this instance, or unique ID if is a Sentinel.*/
    uint64_t config_epoch;  /* Configuration epoch. */
    sentinelAddr *addr; /* Master host. */
    instanceLink *link; /* Link to the instance, may be shared for Sentinels. */
    mstime_t last_pub_time;   /* Last time we sent hello via Pub/Sub. */
    mstime_t last_hello_time; /* Only used if SRI_SENTINEL is set. Last time
                                 we received a hello from this Sentinel
                                 via Pub/Sub. */
    mstime_t last_master_down_reply_time; /* Time of last reply to
                                             SENTINEL is-master-down command. */
    mstime_t s_down_since_time; /* Subjectively down since time. */
    mstime_t o_down_since_time; /* Objectively down since time. */
    
/* 配置 sentinel down-after-milliseconds mymaster 30000 */
    mstime_t down_after_period; /* Consider it down after that period. */
    
    
    mstime_t info_refresh;  /* Time at which we received INFO output from it. */

    /* Role and the first time we observed it.
     * This is useful in order to delay replacing what the instance reports
     * with our own configuration. We need to always wait some time in order
     * to give a chance to the leader to report the new configuration before
     * we do silly things. */
    int role_reported;
    mstime_t role_reported_time;
    mstime_t slave_conf_change_time; /* Last time slave master addr changed. */

/* Master specific. */
    dict *sentinels;    /* Other sentinels monitoring the same master. */
    dict *slaves;       /* Slaves for this master instance. */
    unsigned int quorum;/* Number of sentinels that need to agree on failure. */
    
/* 配置 sentinel parallel-syncs mymaster 3 */
    int parallel_syncs; /* How many slaves to reconfigure at same time. */
    
    
    char *auth_pass;    /* Password to use for AUTH against master & slaves. */

    /* Slave specific. */
    mstime_t master_link_down_time; /* Slave replication link down time. */
    int slave_priority; /* Slave priority according to its INFO output. */
    mstime_t slave_reconf_sent_time; /* Time at which we sent SLAVE OF <new> */
    struct sentinelRedisInstance *master; /* Master instance if it's slave. */
    char *slave_master_host;    /* Master host as reported by INFO */
    int slave_master_port;      /* Master port as reported by INFO */
    int slave_master_link_status; /* Master link status as reported by INFO */
    unsigned long long slave_repl_offset; /* Slave replication offset. */
    /* Failover */
    char *leader;       /* If this is a master instance, this is the runid of
                           the Sentinel that should perform the failover. If
                           this is a Sentinel, this is the runid of the Sentinel
                           that this Sentinel voted as leader. */
    uint64_t leader_epoch; /* Epoch of the 'leader' field. */
    uint64_t failover_epoch; /* Epoch of the currently started failover. */
    int failover_state; /* See SENTINEL_FAILOVER_STATE_* defines. */
    mstime_t failover_state_change_time;
    mstime_t failover_start_time;   /* Last failover attempt start time. */
    
/*刷新故障迁移状态的最大时限 sentinel failover-timeout mymaster 180000 */
    mstime_t failover_timeout;      /* Max time to refresh failover state. */
    
    mstime_t failover_delay_logged; /* For what failover_start_time value we
                                       logged the failover delay. */
    struct sentinelRedisInstance *promoted_slave; /* Promoted slave instance. */
    /* Scripts executed to notify admin or reconfigure clients: when they
     * are set to NULL no script is executed. */
    char *notification_script;
    char *client_reconfig_script;
    sds info; /* cached INFO output */
} sentinelRedisInstance;


/**************************************************************** sentinel可执行的命令*/
struct redisCommand sentinelcmds[] = {
    {"ping",pingCommand,1,"",0,NULL,0,0,0,0,0},
    {"sentinel",sentinelCommand,-2,"",0,NULL,0,0,0,0,0},
    {"subscribe",subscribeCommand,-2,"",0,NULL,0,0,0,0,0},
    {"unsubscribe",unsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
    {"psubscribe",psubscribeCommand,-2,"",0,NULL,0,0,0,0,0},
    {"punsubscribe",punsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
    {"publish",sentinelPublishCommand,3,"",0,NULL,0,0,0,0,0},
    {"info",sentinelInfoCommand,-1,"",0,NULL,0,0,0,0,0},
    {"role",sentinelRoleCommand,1,"l",0,NULL,0,0,0,0,0},
    {"client",clientCommand,-2,"rs",0,NULL,0,0,0,0,0},
    {"shutdown",shutdownCommand,-1,"",0,NULL,0,0,0,0,0}
};

```

 

# Lua脚本

RedisServer中嵌入了Lua环境，Redis客户端可以使用lua脚本在服务器上执行多个redis命令。

server在启动后初始了lua环境，lua环境初始化中载入了多个lua函数库,并创建redis全局表格（其中就包含了redis.call(),redis.pcall），并且创建给lua环境脚本执行辅助的工具，命令执行伪客户端和脚本字典（脚本字典是实现SCRIPT EXISTS和脚本复制功能根本），

LUA脚本命令执行步骤： (EVAL)

​	定义脚本函数( f_SHA1CODE)载入lua环境

​	保存脚本到脚本字典lua_scripts（key:SHA1CODE,value:script）

​	执行脚本函数(redis命令会通过，伪客户端来向server发送指令来执行，然后结果返回给伪客户端，伪客户端返回结果到lua环境)

LUA环境脚本管理：

​	SCRIPT FLUSH:清空lua环境脚本函数，并销毁并重新构建一个lua环境

​	SCRIPT EXISTS：判断给定的SHA1值对应的脚本是否存在于lua环境

​	SCRIPT LOAD：加载一个lua脚本并，创建脚本函数并加载到lua环境，并将脚本加入到脚本字典

​	SCRIPT KILL：lua-time-limit 超时使用





