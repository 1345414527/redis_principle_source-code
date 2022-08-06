## Redis的发布订阅(一般不用)

### 订阅频道

Redis 发布订阅(pub/sub)是一种消息通信模式：发送者(pub)发送消息，订阅者(sub)接收消息。

Redis客户端可以订阅任意数量的频道。

下图展示了频道 channel1 ， 以及订阅这个频道的三个客户端 —— client2 、 client5 和 client1 之间的关系：

<center><img src="https://img-blog.csdnimg.cn/20190925161943181.png" /></center>

当有新消息通过 PUBLISH 命令发送给频道 channel1 时， 这个消息就会被发送给订阅它的三个客户端：

<center><img src="https://img-blog.csdnimg.cn/20190925161952995.png" /></center>

Redis将所有频道的订阅关系都保存在服务器状态的pubsub_channels字典里面，这个字典的键是某个被订阅的频道，而键的值则是一个链表，链表里面记录了所有订阅这个频道的客户端：

```c
struct redisserver {
     ...
     //保存所有频道的订阅关系
     dict pubsub_channels:
     ...
};
```



### 订阅模式

除了订阅频道之外。客户端还可以通过执行**PSUBSCRIBE**命令订阅一个或多个模式，从而成为这些模式的订阅者:每当有其他客户端向某个频道发送消息时，消息不仅会被发送给这个频道的所有订阅者，它还会被发送给所有与这个额道相匹配的模式的订阅者。

服务器也将所有模式的订阅关系都保存在服务器状态的pubsub_patterns属性里面:

```c
struct rediaserver {
  	  ...
      //保存所有模式订阅关系
      list *pubsubpatterns;
  	  ...
};
```

pubsub_patterns属性是一个链表，链表中的每个节点都包含着一个pubsubPattern结构：

```c
typedef struct pubsubPattern {
    client *client;  //订阅模式的客户端
    robj *pattern;  //被订阅的模式
} pubsubPattern;
```

当一个Redis 客户端执行PUBLISH \<channel> \<message>命令将消息message发送给频道channel的时候,服务器需要执行以下两个动作：

1. 将消息message发送给channel频道的所有订阅者；
2. 如果有一个或多个模式pattern与频道channel 相匹配，那么将消息message发送给pattern模式的订阅者。因为服务器状态中的pubsub_patterns链表记录了所有模式的订阅关系，所以为了将消息发送给所有与channel频道相匹配的模式的订阅者，PUBLISH命令要做的就是遍历整个pubsub_patterns链表，查找那些与channel频道相匹配的模式，并将消息发送给订阅了这些模式的客户端。



### 命令

```c
subscribe channel [channel…]：订阅一个或多个频道的信息
psubscribe pattern [pattern…]：订阅一个或多个符合规定模式的频道
publish channel message ：将信息发送到指定频道
unsubscribe [channel[channel…]]：退订频道
punsubscribe [pattern[pattern…]]：退订所有给定模式的频道
```



### 发布命令的实现

发布命令在 Redis 的实现中对应的是 publish。在前面我们就详细说过，Redis server 在初始化时，会初始化一个命令表 **redisCommandTable**，表中就记录了 Redis 支持的各种命令，以及对应的实现函数。

```c
struct redisCommand redisCommandTable[] = {
    …
    {"publish",publishCommand,3,"pltF",0,NULL,0,0,0,0,0},
    …
}
```

这张命令表是在 server.c 文件中定义的，当你需要了解 Redis 某个命令的具体实现函数时，一个快捷的方式就是在这张表中查找对应命令，然后就能定位到该命令的实现函数了。我们同样可以用这个方法来定位 publish 命令，这样就可以看到它对应的实现函数是 pubsub.c#publishCommand，如下所示：

```c
/* PUBLISH <channel> <message> */
void publishCommand(client *c) {
    // 调用pubsubPublishMessage发布消息
    int receivers = pubsubPublishMessage(c->argv[1],c->argv[2]);
    // 如果Redis启用了cluster，那么在集群中发送publish命令
    if (server.cluster_enabled)
        clusterPropagatePublish(c->argv[1],c->argv[2]);
    else
        forceCommandPropagation(c,PROPAGATE_REPL);
    // 返回接收消息的订阅者数量
    addReplyLongLong(c,receivers);
}
```

pubsub.c#pubsubPublishMessage:

```c
/* Publish a message */
//参数分别是要发布消息的频道，以及要发布的具体消息。
int pubsubPublishMessage(robj *channel, robj *message) {
    int receivers = 0;
    dictEntry *de;
    dictIterator *di;
    listNode *ln;
    listIter li;

    /* Send to clients listening for that channel */
    // 发送给监听该频道的客户端
    // 查找频道是否存在
    de = dictFind(server.pubsub_channels,channel);
    if (de) {
        // 频道存在
        list *list = dictGetVal(de);
        listNode *ln;
        listIter li;

        listRewind(list,&li);
        // 遍历频道对应的订阅者，向订阅者发送要发布的消息
        while ((ln = listNext(&li)) != NULL) {
            client *c = ln->value;
            addReplyPubsubMessage(c,channel,message);
            receivers++;
        }
    }
    /* Send to clients listening to matching channels */
    // 发送给监听的与该频道匹配的模式的客户端
    di = dictGetIterator(server.pubsub_patterns);
    if (di) {
        channel = getDecodedObject(channel);
        while((de = dictNext(di)) != NULL) {
            robj *pattern = dictGetKey(de);
            list *clients = dictGetVal(de);
            if (!stringmatchlen((char*)pattern->ptr,
                                sdslen(pattern->ptr),
                                (char*)channel->ptr,
                                sdslen(channel->ptr),0)) continue;

            listRewind(clients,&li);
            while ((ln = listNext(&li)) != NULL) {
                client *c = listNodeValue(ln);
                addReplyPubsubPatMessage(c,pattern,channel,message);
                receivers++;
            }
        }
        decrRefCount(channel);
        dictReleaseIterator(di);
    }
    return receivers;
}
```



### 接收命令的实现

和查找发布命令的方法一样，我们可以在 redisCommandTable 表中，找到订阅命令 subscribe 对应的实现函数是 pubsub.c#subscribeCommand：

```c
/* SUBSCRIBE channel [channel ...] */
void subscribeCommand(client *c) {
    int j;
    if ((c->flags & CLIENT_DENY_BLOCKING) && !(c->flags & CLIENT_MULTI)) {
        /**
         * A client that has CLIENT_DENY_BLOCKING flag on
         * expect a reply per command and so can not execute subscribe.
         *
         * Notice that we have a special treatment for multi because of
         * backword compatibility
         */
        // 具有 CLIENT_DENY_BLOCKING 标志的客户端期望对每个命令进行回复，因此无法执行订阅。
        // 请注意，由于向后兼容，我们对 multi 进行了特殊处理
        addReplyError(c, "SUBSCRIBE isn't allowed for a DENY BLOCKING client");
        return;
    }

    for (j = 1; j < c->argc; j++)
        pubsubSubscribeChannel(c,c->argv[j]);
    c->flags |= CLIENT_PUBSUB;
}
```

从代码中，你可以看到，subscribeCommand 函数的参数是 client 类型的变量，而它会**根据 client 的 argc 成员变量执行一个循环，并把 client 的每个 argv 成员变量传给 pubsubSubscribeChannel 函数执行**。每一个argv就是每一个要订阅的频道。

主要是pubsub.c#pubsubSubscribeChannel 函数来完成订阅操作：

```c
/* Subscribe a client to a channel. Returns 1 if the operation succeeded, or
 * 0 if the client was already subscribed to that channel. */
int pubsubSubscribeChannel(client *c, robj *channel) {
    dictEntry *de;
    list *clients = NULL;
    int retval = 0;

    /* Add the channel to the client -> channels hash table */
    // 在pubsub_channels哈希表中查找频道
    if (dictAdd(c->pubsub_channels,channel,NULL) == DICT_OK) {
        retval = 1;
        incrRefCount(channel);
        /* Add the client to the channel -> list of clients hash table */
        de = dictFind(server.pubsub_channels,channel);
        // 如果频道不存在
        if (de == NULL) {
            // 创建订阅者对应的列表
            clients = listCreate();
            // 新插入频道对应的哈希项
            dictAdd(server.pubsub_channels,channel,clients);
            incrRefCount(channel);
        } else {
            // 频道已存在，获取订阅者列表
            clients = dictGetVal(de);
        }
        // 将订阅者加入到订阅者列表
        listAddNodeTail(clients,c);
    }
    /* Notify the client */
    // 通知客户端，给订阅者返回成功订阅的频道数量
    addReplyPubsubSubscribed(c,channel);
    return retval;
}
```

主要分为三步：

- 首先，它把要订阅的频道加入到 server 记录的 pubsub_channels 中。如果这个频道是新创建的，那么它会在 pubsub_channels 哈希表中新建一个哈希项，代表新创建的这个频道，并且会创建一个列表，用来保存这个频道对应的订阅者；如果频道已经在 pubsub_channels 哈希表中存在了，那么pubsubSubscribeChannel 函数就直接获取该频道对应的订阅者列表；
- 然后，pubsubSubscribeChannel 函数把执行 subscribe 命令的订阅者，加入到订阅者列表中；
- 最后，pubsubSubscribeChannel 函数会把成功订阅的频道个数返回给订阅者。



### redis中的应用

在redis中，发布订阅的一个典型应用就是用在sentinel（哨兵）中，sentinel中，sentinel会利用sentinelEvent对不同的频道发布消息，而且也会利用hello频道处理和其它sentinel节点的关系。具体的可以看sentinel章节部分。
