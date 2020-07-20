---
title: Backup Request
layout: page
menubar: administration_menu
---

# 背景
在当前的Pegasus实现中，由于向secondary读取会导致不一致的情况发生，所以目前Pegasus仅仅支持对primary副本的读取。但是在某些情况下（例如：负载均衡、热点写入等）经常会导致primary不稳定。所以我们希望在primary不稳定时能够读取secondary，通过牺牲部分强一致性来降低读请求的长尾并提高系统的可用性。backup request便是用来实现此功能的。

# 设计实现

backup reqeust的实现原理比较简单：对于读操作（目前写操作不支持backup request），当client向primary发送请求后，如果经过一段时间延时（通常是p999）其response仍然没有返回，则随机选择一台secondary并向其发送backup request。最后获取最快返回来的response进行处理。

这里发送secondary请求的延时我们建议选择p999，因为backup request操作是用来实现消除长尾的，并不是提升集群性能的。如果将该值设置过低，则会由于backup request的请求量过大而导致集群压力增大（假设选择p50作为其延时，这样便会有50%的请求向secondary发送请求，系统负载便会增大50%）。

# 如何使用
在Pegasus java client v2.0.0中，我们增加了一个接口，通过该接口可以打开某个表的backup reqeust功能。其实现如下：
```java
public PegasusTableInterface openTable(String tableName, int backupRequestDelayMs) throws PException;
```

相比于老版本的`openTable`接口，我们增加了一个`backupRequestDelayMs`参数。这个参数便是上文所指的时延，即：向primary发送请求，如果过了`backupRequestDelayMs`毫秒response仍没有返回，则向secondary发送backup request。需要注意的是，`backupRequestDelayMs <= 0`代表禁用backup reqeust功能。

另外在老版本的`openTable`接口中，backup request功能默认是关闭的。

# 性能测试

下面表格里展示了某场景下是否打开backup request的性能对比，这里我们选取了未打开backup request时读请求的p999时间作为backup request的delay时间。数据显示，打开backup request之后get请求的p9999时延降低到了未打开时的15%左右。

|  test case   | enable backup request  |  read p9999  |  
| ---- | ---- | ---- |
| 3-clients 15-threads | no | 988671 |
| 3-clients 15-threads | yes| 153599 |

