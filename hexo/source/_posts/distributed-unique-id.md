---
title: 常见分布式唯一ID生成策略总结
date: 2018-07-22 18:30:20
tags: 分布式系统
---
最近在基于Google Dapper论文搭建分布式跟踪系统，traceid作为一次用户请求的标识，需要在分布式系统中使用唯一ID，这里对常见的生成策略进行简单总结。

<!--more-->
全局唯一的ID在很多分布式系统中都会遇到，比如本文引言中所述的分布式跟踪系统，使用traceid来标记一次完整的服务调用，一般在系统的边界（接入层）生成，并向调用链上的后继服务传递，实现对整个调用链的跟踪。
借此契机，对常见的分布式唯一ID生成策略进行总结。
### 使用数据库的auto_increment生成
#### 优点
- 使用数据库现有的功能，比较简单
- 具有唯一性和递增性
- id之间的步长是固定且可自定义的

#### 缺点
- 可用性和扩展性较差。因为数据库常见架构是一主多从+读写分离，主库的写性能决定了ID的生成性能上限，并且难以扩展。

#### 改进方案
对于可用性较差的问题，可以通过“冗余主库”、“数据水平切分”等方法来改善。如下图所示，由1个写库变成3个写库，每个写库设置不同的auto\_increment初始值，以及相同的步长，以保证每个数据库生成的ID各不相同。
![](http://km.oa.com/files/photos/pictures//20180902//1535881731_49.png)
对于写压力大的问题，可以使用批量生成的方式来改善。如下图所示，数据库中只存储当前ID的最大值。假设ID生成服务每次批量获取5个ID，服务访问数据库，将当前ID的最大值设置为4，这样不需要每次访问数据库，ID生成服务就能依次派发0,1,2,3,4这些ID了。
这种方式下，ID生成服务存在单点问题，我们可以通过“主备”的方式来实现容灾和水平扩展，不过又会引发一致性问题。
![](http://km.oa.com/files/photos/pictures//20180902//1535881973_11.png)

### 使用Redis来生成
使用数据库生成ID的性能较低，可以尝试使用Redis来改善。这主要源于Redis是单线程的，可以通过其原子操作INCR和INCRBY来获取全局唯一的ID。
#### 优点
- 灵活方便，性能优于数据库
- 数字ID天然排序，对需要排序的场景有帮助

#### 缺点
- 需要引入Redis组件，需要评估系统的复杂度

### uuid
上述数据库，ID生成服务或是Redis的方法，业务方都需要进行一次远程调用，这对于时延敏感的业务影响较大。uuid是一种常见的本地生成ID的方法。
uuid是指一台机器在同一时间中生成的数字在所有机器中是唯一的，其主要由以下三部分组成：
- 当前日期和时间
- 时钟序列
- 全局唯一的IEEE机器识别号

标准的uuid格式为xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx(8-4-4-4-12)，包含32个16进制数字，以连字号分为五段。
#### 优点
- 性能高，本地生成，没有网络消耗
- 扩展性好，基本可以认为没有性能上限

#### 缺点
- 不易存储，uuid为16字节128位，通常以36长度的字符串表示，有些场景不适用。----常见的优化方案为“转化为两个uint64整数存储”或者“折半存储”（折半后不能保证唯一性）
- 信息不安全：基于MAC地址生成uuid的算法可能会造成MAC地址泄露
- 无法保证递增

### 取当前毫秒/微妙数
针对uuid无法保证趋势递增，以及作为字符串ID检索效率低的缺点，有人提出了取当前毫秒/微妙数的方案。
#### 优点
- 本地生成ID
- 生成ID递增
- 生成的ID是整数，建立索引后查询效率高

#### 缺点
并发量超过1000/1000000个ID，会发生冲突。

### Twitter提出的Snowflake算法
Snowflake是twitter提出的一种分布式ID生成算法，被广泛应用于分布式系统中，其核心思想是一个long型的id由以下四部分组成：
- 1bit作为标识符 --- 最高位是符号位，始终是0，代表正数
- 41bit作为毫秒数 --- 41位的长度可以使用69年
- 10bit作为机器编号（5bit是数据中心，5bit是机器id） --- 10位的长度可部署1024个节点
- 12bit作为毫秒内序列号 --- 12位的长度支持单个节点每毫秒产生4096个ID号

如下图所示，算法单节点每秒理论上最多可以产生1000*(2^12)个ID。
![](http://km.oa.com/files/photos/pictures//20180902//1535885058_21.png)

#### Snowflake的C++实现
在实际系统中采用C++语言实现了Snowflake算法，类的定义为：
```cpp
#ifndef SNOW_FLAKE_H
#define SNOW_FLAKE_H

class Snowflake
{
public:
    Snowflake();
    ~Snowflake();

    void setEpoch(uint64_t epoch);
    void setMachine(int machine);
    int64_t generate();

private:
    uint64_t getTime();

    uint64_t epoch_;
    int machine_;
    int sequence_;
};

#endif
```
具体实现代码如下：
```cpp
#include "Snowflake.h"

#include <sys/time.h>

Snowflake::Snowflake()
{
    this->epoch_ = 0;
    this->time_ = 0;
    this->machine_ = 0;
    this->sequence_ = 0;
}

Snowflake::~Snowflake()
{
}

//设置毫秒起始点
void Snowflake::setEpoch(uint64_t epoch) {
    this->epoch_ = epoch;
}

void Snowflake::setMachine(int machine) {
    this->machine_ = machine;
}

//id生成函数
int64_t Snowflake::generate() {
    int64_t value = 0;
    //和开始时间戳的差值
    uint64_t time = getTime() - this->epoch_;

    //41bit,毫秒时间
    value |= time << 22;

    //10bit,机器编号
    value |= this->machine_ & 0x3FF << 12;

    //12bit,毫秒内序列号
    value |= this->sequence_++ & 0xFFF;
   
    if (this->sequnce_ == 0x1000) {
    	this->sequnce_ = 0;
    }

    return value;
}

//获取当前毫秒数
uint64_t Snowflake::getTime() {
    struct timeval tv;
    gettimeofday(&tv, NULL);
   
    return tv.tv_usec / 1000 + tv.tv_sec * 1000;
}

```

