## 服务端

### 命令请求的执行过程

1. 发送命令请求。Redis服务器的命令请求来自Redis客户端，当用户在客户端中键人一个命令请求时，客户端会将这个命令请求转换成协议格式，然后通过连接到服务器的套接字，将协议格式的命令请求发送给服务器。

2. 读取命令请求。当客户端与服务器之间的连接套接字因为客户端的写入而变得可读时，服务器将调用命令请求处理器来执行以下操作：①**读取套接字中协议格式的命令请求，并将其保存到客户端状态的输人缓冲区里面。**②**对输人缓冲区中的命令请求进行分析，提取出命令请求中包含的命令参数，以及命令参数的个数，然后分别将参数和参数个数保存到客户端状态的argv属性和argc属性里面。**③**调用命令执行器，执行客户端指定的命令。**

   (①和②是在networking.c# processInputBuffer中进行的，调用流程是aeProcessEvents-->aeApiPoll-->fe->rfileProc，rfileProc就是和每个fe绑定的readQueryFromClient，会调用processInputBufferAndReplicate-->processInputBuffer；③就是在processInputBuffer中调用了server.c#processCommand进行执行客户端的命令)

3. 命令执行-1-查找命令。**命令执行器要做的第一件事就是根据客户端状态的argv[0]参数，在命令表( server.commands)中查找参数所指定的命令，并将找到的命令保存到客户端状态的cmd属性里面。**(具体可以看客户端中的命令的实现函数-cmd)

4. 命令执行-2-执行预备操作。**在真正执行之前，还需要做一些预备操作，从而保证命令可以正确的、顺利地执行**。比如：①检查客户端状态的cmd指针是否指向NULL，如果是的话，那么说明用户输入的命令名字找不到相应的命令实现，服务器不再执行后续步骤，并向客户端返回一个错误；②根据客户端cmd 属性指向的rediscommand结构的arity属性，检查命令请求所给定的参数个数是否正确，当参数个数不正确时，不再执行后续步骤，直接向客户端返回一个错误。比如说，如果rediscommand结构的arity属性的值为-3，那么用户输入的命令参数个数必须大于等于3个才行；③检查客户端是否已经通过了身份验证，未通过身份验证的客户端只能执行AUTH命令，如果未通过身份验证的客户端试图执行除AUTH命令之外的其他命令，那么服务器将向客户端返回一个错误；④如果服务器打开了maxmemory 功能，那么在执行命令之前，先检查服务器的内存占用情况，并在有需要时进行内存回收，从而使得接下来的命令可以顺利执行。如果内存回收失败，那么不再执行后续步骤，向客户端返回一个错误等，具体的可以去看《redis设计与实现》.p181

5. 命令执行-3-调用命令的实现函数。在进行一系列的数据检查后，会调用call命令，进而调用c->cmd->proc(c)来执行函数，即**调用命令对应的命令处理函数**。**被调用的命令实现函数会执行指定的操作，并产生相应的命令回复，这些回复会被保存在客户端状态的输出缓冲区里面**( buf属性和reply属性)，之后**实现函数还会为客户端的套接字关联命令回复处理器**，这个处理器负责将命令回复返回给客户端。

6. 命令执行-4-执行后续工作。①如果服务器开启了慢查询日志功能，那么慢查询日志模块会检查是否需要为刚刚执行完的命令请求添加一条新的慢查询日志；②根据刚刚执行命令所耗费的时长，更新被执行命令的rediscommand结构的milliseconds属性，并将命令的rediscommand结构的calls计数器的值增一；③如果服务器开启了AOF持久化功能，那么AOF持久化模块会将刚刚执行的命令请求写入到AOF缓冲区里面(server.c#call)；④如果有其他从服务器正在复制当前这个服务器，那么服务器会将刚刚执行的命令传播给所有从服务器(networking.c#processInputBufferAndReplicate)。

7. 命令执行-5将命令恢复发送给客户端。命令实现函数会将命令回复保存到客户端的输出缓冲区里面，并放入clients_pending_write列表中，当处理时会直接发送给客户端，若还有树没发完，就为客户端的套接字关联命令回复处理器，当客户端套接字变为可写状态时，服务器就会执行命令回复处理器，将保存在客户端输出缓冲区中的命令回复发送给客户端。



[Redis 命令执行过程(下) - 程序员历小冰 - 博客园 (cnblogs.com)](https://www.cnblogs.com/remcarpediem/p/12038377.html)



### 命令全过程源码跟踪

```c
//1.每次进行循环
//ae.c#aeMain
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}

//2. 监听获取命令，并调用执行,具体的函数可以看event loop部分的介绍，aeApiPoll可以看File Event部分的介绍。
//ae.c#aeProcessEvents
int aeProcessEvents(aeEventLoop *eventLoop, int flags){
    ...
        numevents = aeApiPoll(eventLoop, tvp);
    ...
    if (!invert && fe->mask & mask & AE_READABLE) {
       fe->rfileProc(eventLoop,fd,fe->clientData,mask);
       fired++;
    }

    /* Fire the writable event. */
    if (fe->mask & mask & AE_WRITABLE) {
        if (!fired || fe->wfileProc != fe->rfileProc) {
            fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            fired++;
        }
    }

    /* If we have to invert the call, fire the readable event now
             * after the writable one. */
    if (invert && fe->mask & mask & AE_READABLE) {
        if (!fired || fe->wfileProc != fe->rfileProc) {
            fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            fired++;
        }
    }
    ...
}

//3.file event绑定的是readQueryFromClient，因此fe->rfileProc执行的就是这个函数
//最后会执行processInputBufferAndReplicate(c);
//networking.c#readQueryFromClient
void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    //设置当前服务的client，然后是设置这次从socket读取的数据的默认大小（REDIS_IOBUF_LEN为16KB）
    client *c = (client*) privdata;
    int nread, readlen;
    size_t qblen;
    UNUSED(el);
    UNUSED(mask);

    readlen = PROTO_IOBUF_LEN;
    /* If this is a multi bulk request, and we are processing a bulk reply
     * that is large enough, try to maximize the probability that the query
     * buffer contains exactly the SDS string representing the object, even
     * at the risk of requiring more read(2) calls. This way the function
     * processMultiBulkBuffer() can avoid copying buffers to create the
     * Redis Object representing the argument. */
    //这段代码重新设置读取数据的大小，避免频繁拷贝数据
    //如果当前请求是一个multi bulk类型的，并且要处理的bulk的大小大于REDIS_MBULK_BIG_ARG（32KB），则将读取数据大小设置为该bulk剩余数据的大小。
    if (c->reqtype == PROTO_REQ_MULTIBULK && c->multibulklen && c->bulklen != -1
        && c->bulklen >= PROTO_MBULK_BIG_ARG)
    {
        ssize_t remaining = (size_t)(c->bulklen+2)-sdslen(c->querybuf);

        /* Note that the 'remaining' variable may be zero in some edge case,
         * for example once we resume a blocked client after CLIENT PAUSE. */
        if (remaining > 0 && remaining < readlen) readlen = remaining;
    }

    //读取的请求内容会存储到redisClient->querybuf中
    //此处代码调整querybuf大小以便容纳这次read的数据
    qblen = sdslen(c->querybuf);
    if (c->querybuf_peak < qblen) c->querybuf_peak = qblen;
    //可能存在一次copy,如果buffer的空闲空间小于readlen，则buffer大小翻倍，并将数据拷贝到新buffer
    c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);
    //调用read系统调用，读取readlen大小的数据，并存储到querybuf中
    nread = read(fd, c->querybuf+qblen, readlen);
    //校验read的返回值，检测出错。如果read返回0，则客户端关闭连接，会释放掉该客户端。
    if (nread == -1) {
        //EAGAIN表示read不到数据
        if (errno == EAGAIN) {
            return;
        } else {
            serverLog(LL_VERBOSE, "Reading from client: %s",strerror(errno));
            freeClient(c);
            return;
        }
    } else if (nread == 0) {
        serverLog(LL_VERBOSE, "Client closed connection");
        freeClient(c);
        return;
    } else if (c->flags & CLIENT_MASTER) {
        /* Append the query buffer to the pending (not applied) buffer
         * of the master. We'll use this buffer later in order to have a
         * copy of the string applied by the last command executed. */
        c->pending_querybuf = sdscatlen(c->pending_querybuf,
                                        c->querybuf+qblen,nread);
    }

    sdsIncrLen(c->querybuf,nread);
    c->lastinteraction = server.unixtime;
    if (c->flags & CLIENT_MASTER) c->read_reploff += nread;
    server.stat_net_input_bytes += nread;
    //判断客户端的请求buffer是否超过配置的值server.client_max_querybuf_len（1GB），如果超过，会拒绝服务，并关闭该客户端。
    if (sdslen(c->querybuf) > server.client_max_querybuf_len) {
        sds ci = catClientInfoString(sdsempty(),c), bytes = sdsempty();

        bytes = sdscatrepr(bytes,c->querybuf,64);
        serverLog(LL_WARNING,"Closing client that reached max query buffer length: %s (qbuf initial bytes: %s)", ci, bytes);
        sdsfree(ci);
        sdsfree(bytes);
        freeClient(c);
        return;
    }

    /* Time to process the buffer. If the client is a master we need to
     * compute the difference between the applied offset before and after
     * processing the buffer, to understand how much of the replication stream
     * was actually applied to the master state: this quantity, and its
     * corresponding part of the replication stream, will be propagated to
     * the sub-slaves and to the replication backlog. */
    // 最后，会调用processInputBuffer函数解析请求。
    processInputBufferAndReplicate(c);
}

//4. 进行处理和同步到子节点
//networking.c#processInputBufferAndReplicate
void processInputBufferAndReplicate(client *c) {
    if (!(c->flags & CLIENT_MASTER)) {
        processInputBuffer(c);
    } else {
        size_t prev_offset = c->reploff;
        processInputBuffer(c);
        size_t applied = c->reploff - prev_offset;
        if (applied) {
            replicationFeedSlavesFromMasterStream(server.slaves,
                    c->pending_querybuf, applied);
            sdsrange(c->pending_querybuf,applied,-1);
        }
    }
}

//5. 进一步调用processCommand
//networking.c#processInputBuffer
void processInputBuffer(client *c) {
    server.current_client = c;

    /* Keep processing while there is something in the input buffer */
    //只要querybuf中还包含完整的命令就会一直处理
    while(c->qb_pos < sdslen(c->querybuf)) {
        /* Return if clients are paused. */
        if (!(c->flags & CLIENT_SLAVE) && clientsArePaused()) break;

        /* Immediately abort if the client is in the middle of something. */
        if (c->flags & CLIENT_BLOCKED) break;

        /* Don't process input from the master while there is a busy script
         * condition on the slave. We want just to accumulate the replication
         * stream (instead of replying -BUSY like we do with other clients) and
         * later resume the processing. */
        if (server.lua_timedout && c->flags & CLIENT_MASTER) break;

        /* CLIENT_CLOSE_AFTER_REPLY closes the connection once the reply is
         * written to the client. Make sure to not let the reply grow after
         * this flag has been set (i.e. don't process more commands).
         *
         * The same applies for clients we want to terminate ASAP. */
        if (c->flags & (CLIENT_CLOSE_AFTER_REPLY|CLIENT_CLOSE_ASAP)) break;

        //决定请求的类型
        if (!c->reqtype) {
            if (c->querybuf[c->qb_pos] == '*') {
                c->reqtype = PROTO_REQ_MULTIBULK;
            } else {
                c->reqtype = PROTO_REQ_INLINE;
            }
        }

        //根据请求的类型，分别调用processInlineBuffer和processMultibulkBuffer解析请求
        if (c->reqtype == PROTO_REQ_INLINE) {
            if (processInlineBuffer(c) != C_OK) break;
        } else if (c->reqtype == PROTO_REQ_MULTIBULK) {
            if (processMultibulkBuffer(c) != C_OK) break;
        } else {
            serverPanic("Unknown request type");
        }

        /* Multibulk processing could see a <= 0 length. */
        //解析完成后，接下来调用processCommand函数进行命令处理
        if (c->argc == 0) {
            resetClient(c);
        } else {
            /* Only reset the client when the command was executed. */
            if (processCommand(c) == C_OK) {
                if (c->flags & CLIENT_MASTER && !(c->flags & CLIENT_MULTI)) {
                    /* Update the applied replication offset of our master. */
                    c->reploff = c->read_reploff - sdslen(c->querybuf) + c->qb_pos;
                }

                /* Don't reset the client structure for clients blocked in a
                 * module blocking command, so that the reply callback will
                 * still be able to access the client argv and argc field.
                 * The client will be reset in unblockClientFromModule(). */
                if (!(c->flags & CLIENT_BLOCKED) || c->btype != BLOCKED_MODULE)
                    resetClient(c);
            }
            /* freeMemoryIfNeeded may flush slave output buffers. This may
             * result into a slave, that may be the active client, to be
             * freed. */
            if (server.current_client == NULL) break;
        }
    }

    /* Trim to pos */
    if (server.current_client != NULL && c->qb_pos) {
        sdsrange(c->querybuf,c->qb_pos,-1);
        c->qb_pos = 0;
    }

    server.current_client = NULL;
}

//6. 先进行预备操作，再调用call进行执行
//server.c#processCommand
int processCommand(client *c) {
    ...
    //根据argv[0]，查找command table，找到对应的命令
    c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
    ...
    //接下里进行一系列的校验，比如内存的清理
    if (server.maxmemory && !server.lua_timedout) {
        //如果设置了maxmemory 配置项为非 0 值,且Lua 脚本没有在超时运行则判断是否要进行内存的的清理，具体的清理根据内存淘汰策略会有所不同，具体看内存淘汰机制。
        int out_of_memory = freeMemoryIfNeededAndSafe() == C_ERR;
    ...
    if (c->flags & CLIENT_MULTI &&
        c->cmd->proc != execCommand && c->cmd->proc != discardCommand &&
        c->cmd->proc != multiCommand && c->cmd->proc != watchCommand)
    {
        queueMultiCommand(c);
        addReply(c,shared.queued);
    } else {
        call(c,CMD_CALL_FULL);
        c->woff = server.master_repl_offset;
        if (listLength(server.ready_keys))
            handleClientsBlockedOnKeys();
    }   
    return C_OK;
}

//7. 真正调用命令对应的函数，并将命令写入到AOF缓存
//server.c#call
void call(client *c, int flags) {
    ...
    //在执行前先将该命令的相关信息发送给各个监视器，监视器是用来监听服务器要处理的命令
    if (listLength(server.monitors) &&
        !server.loading &&
        !(c->cmd->flags & (CMD_SKIP_MONITOR|CMD_ADMIN)))
    {
        replicationFeedMonitors(c,server.monitors,c->db->id,c->argv,c->argc);
    }
    ...
    //dirty用于记录更新操作的次数，用于完成save配置(即用于RDB)
    dirty = server.dirty;
    updateCachedTime(0);
    start = server.ustime;
    //执行命令对应的处理函数
    c->cmd->proc(c);
    duration = ustime()-start;
    dirty = server.dirty-dirty;
    if (dirty < 0) dirty = 0;
    ...
    if (flags & CMD_CALL_PROPAGATE &&
        (c->flags & CLIENT_PREVENT_PROP) != CLIENT_PREVENT_PROP)
    {
        int propagate_flags = PROPAGATE_NONE;

        /* Check if the command operated changes in the data set. If so
         * set for replication / AOF propagation. */
        if (dirty) propagate_flags |= (PROPAGATE_AOF|PROPAGATE_REPL);

        /* If the client forced AOF / replication of the command, set
         * the flags regardless of the command effects on the data set. */
        if (c->flags & CLIENT_FORCE_REPL) propagate_flags |= PROPAGATE_REPL;
        if (c->flags & CLIENT_FORCE_AOF) propagate_flags |= PROPAGATE_AOF;

        /* However prevent AOF / replication propagation if the command
         * implementations called preventCommandPropagation() or similar,
         * or if we don't have the call() flags to do so. */
        if (c->flags & CLIENT_PREVENT_REPL_PROP ||
            !(flags & CMD_CALL_PROPAGATE_REPL))
                propagate_flags &= ~PROPAGATE_REPL;
        if (c->flags & CLIENT_PREVENT_AOF_PROP ||
            !(flags & CMD_CALL_PROPAGATE_AOF))
                propagate_flags &= ~PROPAGATE_AOF;

        /* Call propagate() only if at least one of AOF / replication
         * propagation is needed. Note that modules commands handle replication
         * in an explicit way, so we never replicate them automatically. */
        //调用propagate函数将写操作追加到aof_buf缓冲区和处理主从复制
        if (propagate_flags != PROPAGATE_NONE && !(c->cmd->flags & CMD_MODULE))
            propagate(c->cmd,c->db->id,c->argv,c->argc,propagate_flags);
    }
    ...
}

//8.以string类型的get为例
//t_string.c#getGenericCommand
int getGenericCommand(redisClient *c) {
    robj *o;

    //获取当前值的对象，如果为空，响应空
    if ((o = lookupKeyReadOrReply(c,c->argv[1],shared.nullbulk)) == NULL)
        return REDIS_OK;

    //如果值的对象不是string类型，则报错
    if (o->type != REDIS_STRING) {
        addReply(c,shared.wrongtypeerr);
        return REDIS_ERR;
    }
    //响应值，并返回Ok 
    else {
        addReplyBulk(c,o);
        return REDIS_OK;
    }
}

//9.处理返回数据
//networking.c#addReply
void addReply(client *c, robj *obj) {
    //判断是否需要返回数据，并且将当前 client 添加到等待写返回数据队列中(server.clients_pending_write)。
    if (prepareClientToWrite(c) != C_OK) return;

    if (sdsEncodedObject(obj)) {
        // 需要将响应内容添加到output buffer中。总体思路是，先尝试向固定buffer添加，添加失败的话，在尝试添加到响应链表
        if (_addReplyToBuffer(c,obj->ptr,sdslen(obj->ptr)) != C_OK)
            _addReplyStringToList(c,obj->ptr,sdslen(obj->ptr));
    } else if (obj->encoding == OBJ_ENCODING_INT) {
        char buf[32];
        size_t len = ll2string(buf,sizeof(buf),(long)obj->ptr);
        if (_addReplyToBuffer(c,buf,len) != C_OK)
            _addReplyStringToList(c,buf,len);
    } else {
        serverPanic("Wrong obj->encoding in addReply()");
    }
}


/*
* 10.判断是否需要返回数据
* networking.c#prepareClientToWrite
*prepareClientToWrite 首先根据客户端设置的标识进行一系列的判断，判断了当前 client是否需要返回数据：
*	Lua 脚本执行的 client 则需要返回值；
*	如果客户端发送来 REPLY OFF 或者 SKIP 命令，则不需要返回值；
*	如果是主从复制时的主实例 client，则不需要返回值；
*	当前是在 AOF loading 状态的假 client，则不需要返回值。
*	接着如果这个 client 还未处于延迟等待写入 (CLIENT_PENDING_WRITE)的状态，则将其设置为该状态，并将其加入到 *Redis 的等待写入返回值客户端队列中，也就是 clients_pending_write队列。
*/
int prepareClientToWrite(client *c) {
    // 如果是 lua client 则直接OK
    if (c->flags & (CLIENT_LUA|CLIENT_MODULE)) return C_OK;
    // 客户端发来过 REPLY OFF 或者 SKIP 命令，不需要发送返回值
    if (c->flags & (CLIENT_REPLY_OFF|CLIENT_REPLY_SKIP)) return C_ERR;
    // master 作为client 向 slave 发送命令，不需要接收返回值
    if ((c->flags & CLIENT_MASTER) &&
        !(c->flags & CLIENT_MASTER_FORCE_REPLY)) return C_ERR;
    // AOF loading 时的假client 不需要返回值
    if (c->fd <= 0) return C_ERR; 
   
    //如果当前客户端没有待写回数据，调用clientInstallWriteHandler函数
    if (!clientHasPendingReplies(c)) clientInstallWriteHandler(c);
    if (!clientHasPendingReplies(c) &&
        !(c->flags & CLIENT_PENDING_WRITE) &&
        (c->replstate == REPL_STATE_NONE ||
         (c->replstate == SLAVE_STATE_ONLINE && !c->repl_put_online_on_ack)))
    {
        
        c->flags |= CLIENT_PENDING_WRITE;
        listAddNodeHead(server.clients_pending_write,c);
    }
    // 表示已经在排队，进行返回数据
    return C_OK;
}

void clientInstallWriteHandler(client *c) {
    /*
     * 1. 客户端没有设置过 CLIENT_PENDING_WRITE 标识，即没有被推迟过执行写操作；
     * 2. 客户端所在实例没有进行主从复制，或者客户端所在实例是主从复制中的从节点，但全量复制的 RDB 文件已经传输完成，客户端可以接收请求。
     */
    if (!(c->flags & CLIENT_PENDING_WRITE) &&
        (c->replstate == REPL_STATE_NONE ||
         (c->replstate == SLAVE_STATE_ONLINE && !c->repl_put_online_on_ack)))
    {
        //将客户端的标识设置为待写回，即CLIENT_PENDING_WRITE
        c->flags |= CLIENT_PENDING_WRITE;
        // 将客户端加入clients_pending_write列表，下次事件周期会创建事件进行返回值写入
        listAddNodeHead(server.clients_pending_write,c);
    }
}
```

beforeSleep 函数会调用 handleClientsWithPendingWrites 函数来处理 clients_pending_write 列表。

```c
void aeMain(aeEventLoop *eventLoop) { // ae.c
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        /* 如果有需要在事件处理前执行的函数，那么执行它 */
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        /* 开始处理事件*/
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```

handleClientsWithPendingWrites 方法会遍历 clients_pending_write 列表，**对于每个 client 都会先调用 writeToClient 方法来尝试将返回数据从输出缓存区写入到 socekt中，如果还未写完，则只能调用 aeCreateFileEvent 方法来注册一个写数据事件处理器 sendReplyToClient，等待 Redis 事件机制的再次调用。**

这样的好处是对于返回数据较少的客户端，不需要麻烦的注册写数据事件，等待事件触发再写数据到 socket，而是在下一次事件循环周期就直接将数据写到 socket中，加快了数据返回的响应速度。

```c
// 直接将返回值写到client的输出缓冲区中，不需要进行系统调用，也不需要注册写事件处理器
//networking.c#handleClientsWithPendingWrites
int handleClientsWithPendingWrites(void) {
    listIter li;
    listNode *ln;
    // 获取系统延迟写队列的长度
    int processed = listLength(server.clients_pending_write);

    listRewind(server.clients_pending_write,&li);
    // 依次处理
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_WRITE;
        listDelNode(server.clients_pending_write,ln);

        // 将缓冲值写入client的socket中，如果写完，则跳过之后的操作。
        if (writeToClient(c->fd,c,0) == C_ERR) continue;

        // 还有数据未写入，只能注册写事件处理器了
        if (clientHasPendingReplies(c)) {
            int ae_flags = AE_WRITABLE;
            if (server.aof_state == AOF_ON &&
                server.aof_fsync == AOF_FSYNC_ALWAYS)
            {
                ae_flags |= AE_BARRIER;
            }
            // 注册写事件处理器 sendReplyToClient，等待执行
            if (aeCreateFileEvent(server.el, c->fd, ae_flags,
                                  sendReplyToClient, c) == AE_ERR)
            {
                freeClientAsync(c);
            }
        }
    }
    return processed;
}
```

而在下一次循环中，就会处理AE_WRITABLE事件，即调用sendReplyToClient进行处理。

```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags){
    ...
    if (!invert && fe->mask & mask & AE_READABLE) {
         fe->rfileProc(eventLoop,fd,fe->clientData,mask);
         fired++;
    }

    /* Fire the writable event. */
    if (fe->mask & mask & AE_WRITABLE) {
        if (!fired || fe->wfileProc != fe->rfileProc) {
            fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            fired++;
        }
    }

    /* If we have to invert the call, fire the readable event now
             * after the writable one. */
    if (invert && fe->mask & mask & AE_READABLE) {
        if (!fired || fe->wfileProc != fe->rfileProc) {
            fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            fired++;
        }
    }
    ...
}
```



### 初始化服务器

首先来看一下server.c中的main函数，里面由以下几个主要的部分：

```c
int main(int argc, char **argv) {
    ...
    initServerConfig();
    ...
    loadServerConfig(configfile,options);
    ...
    initServer();
    ...
    loadDataFromDisk();
    ...
    InitServerLast();
    ...
    aeMain(server.el);
}
```

**redis服务初始化分为五个阶段**：①初始化服务配置；②载入配置选项；③服务器初始化；④还原数据库状态；⑤服务器最终初始化；⑥启动event loop

初始化服务器的第一步就是创建一个struct redisServer类型的实例变量server作为服务器的状态。并为结构中的各个属性设置默认值。

#### 初始化服务配置

初始化server变量的工作由redis.c/initserverconfig函数完成，initserverconfig函数主要完成完成的主要工作：①设置服务器的运行ID；②设置服务器的默认运行频率；③设置服务器的默认配置文件路径；④设置服务器的运行架构；⑤设置服务器的默认端口号；⑥设置服务器的默认RDB持久化条件和AOF持久化条件；⑦初始化服务器的全局LRU时钟；⑧**创建命令表**。

创建命令表：调用populateCommandTable函数对redis的命令表初始化。全局变量redisCommandTable是redisCommand类型的数组，保存redis支持的所有命令。server.commands是一个dict，保存命令名到redisCommand的映射。populateCommandTable函数会遍历全局redisCommandTable表，把每条命令插入到server.commands中，根据每个命令的属性设置其flags。

initserverconfig函数设置的服务器状态属性基本都是一些整数、浮点数、或者字符串属性，除了命令表之外，initServerConfig函数没有创建服务器状态的其他数据结构，数据库、慢查询日志、Lua环境、共享对象这些数据结构在之后的initServer中才会被创建出来。


当initserverConfig函数执行完毕之后，服务器就可以进入初始化的第二个阶段一载人配置选项。

#### 载入配置选项

在初始化server变量之后，解析服务器启动时从命令行传入的参数，如果服务器启动时指定了配置文件，则这里就会开始载入用户给定的配置参数和配置文件redis.conf，并根据用户设定的配置，对server变量相关属性值进行更新。  

#### 服务初始化

 initServerConfig函数初始化时，程序只创建了命令表一个数据结构，除了这个命令表外，服务器状态还包括其它数据结构需要设置，因此initServer函数的工作如下：

(1) 设置信号处理函数：忽略SIGHUP和SIGPIPE两种信号，设置信号SIGTERM的信号处理函数为sigtermHandler，即进程在收到该信号时打印信息，终止服务器；

(2) 服务器的当前客户端（server.current_client）设置为NULL；

(3) 服务器的客户端链表（server.clients）初始化，这个链表记录了所有服务器相连的客户端状态结构；

(4) 服务器的待异步关闭的客户端链表（server.clients_to_close）初始化，这个链表记录了所有待异步关闭的客户端，一般在serverCron函数中关闭；

(5) 服务器的从服务器链表（server.slaves）初始化，这个链表保存了所有从服务器；

(6) 服务器的监视器链表（server.monitors）初始化，这个链表保存了所有监视器，即哨兵服务器；

(7) 服务器的未阻塞的客户端链表（server.unblocked_clients）初始化，这个链表保存了下一个循环前未阻塞的客户端；

(8) 服务器的等待回复的客户端链表（server.clients_waiting_acks）初始化，这个链表保存了等待回复的客户端列表，即阻塞在“WAIT”命令上的客户端，“WAIT”命令表示主从服务器同步复制，客户端的命令中如果带有“WAIT”，用户指定至少多少个replication成功以及超时时间；

(9) 服务器的客户端中是否有等待slave回复的客户端（server.get_ack_from_slaves）初始化为0，如果为1，表示有客户端正在等待从服务器slave的回复；

(10) 调用函数createSharedObjects（）创建共享对象，通过复用来减少内存碎片：

```c
struct sharedObjectsStruct {
    robj *crlf, *ok, *err, *emptybulk, *czero, *cone, *cnegone, *pong, *space,
    *colon, *nullbulk, *nullmultibulk, *queued,
    *emptymultibulk, *wrongtypeerr, *nokeyerr, *syntaxerr, *sameobjecterr,
    *outofrangeerr, *noscripterr, *loadingerr, *slowscripterr, *bgsaveerr,
    *masterdownerr, *roslaveerr, *execaborterr, *noautherr, *noreplicaserr,
    *busykeyerr, *oomerr, *plus, *messagebulk, *pmessagebulk, *subscribebulk,
    *unsubscribebulk, *psubscribebulk, *punsubscribebulk, *del, *rpop, *lpop,
    *lpush, *emptyscan, *minstring, *maxstring,
    *select[REDIS_SHARED_SELECT_CMDS],
    *integers[REDIS_SHARED_INTEGERS],
    *mbulkhdr[REDIS_SHARED_BULKHDR_LEN], /* "*<value>\r\n" */
        *bulkhdr[REDIS_SHARED_BULKHDR_LEN];  /* "$<value>\r\n" */
};
```

(11) 调用函数adjustOpenFilesLimit（）根据服务器最大客户端数量（server.maxclients）调整一个进程最大可以打开的文件个数

(12) 调用函数aeCreateEventLoop（）初始化服务器的事件循环结构体（server.el）(注意serverCron和acceptTcpHandler)：

```c
server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);
```

函数中需要对结构体进行初始化，包括：

- 初始化文件事件结构体数组：eventLoop->events
- 初始化已就绪文件事件结构体数组：eventLoop->fired
- 设置已追踪的最大描述符大小为server.maxclients + REDIS_EVENTLOOP_FDSET_INCR
- 初始化时间时间结构eventLoop->timeEventHead、eventLoop->timeEventNextId等
- 调用函数aeApiCreate（）创建epoll句柄，并初始化eventLoop->apidata
- 调用函数 listenToPort（）创建套接字，并且打开监听端口

(13) 根据数据库的数量（server.dbnum）给服务器的数据库数组（server.db）分配内存，这个数组包含了服务器的所有数据库，并且初始化数组中每个元素的所有字典项：

```c
typedef struct redisDb {
    dict *dict;                // 数据库键空间，保存着数据库中的所有键值对
    dict *expires;             // 键的过期时间，字典的键为键，字典的值为过期事件 UNIX 时间戳
    dict *blocking_keys;        // 正处于阻塞状态的键
    dict *ready_keys;          // 可以解除阻塞的键
    dict *watched_keys;         // 正在被 WATCH 命令监视的键
    struct evictionPoolEntry *eviction_pool;    /* Eviction pool of keys */
    int id;                  // 数据库号码
    long long avg_ttl;        // 数据库的键的平均 TTL ，统计信息
} redisDb;
```

(14) 初始化server.pubsub_channels和server.pubsub_patterns：用于保存频道订阅信息和模式订阅信息

(15) 初始化server.lua：保存用于执行Lua脚本的Lua环境

(16) 初始化server.slowlog：用于保存慢查询日志

(17) 调用函数aeCreateTimeEvent（）注册时间事件，将函数serverCron（）注册给时间事件处理器，并且设定1ms后执行这个函数，并且将注册的时间时间放在事件处理器的时间事件链表的表头节点.

```c
if (aeCreateTimeEvent(server.el, 1, serverCron, NULL, NULL) == AE_ERR) {
    serverPanic("Can't create event loop timers.");
    exit(1);
}
```

(18) 调用函数aeCreateFileEvent（）为监听套接字关联连接应答处理器(acceptTcpHandler)，这个函数做的事儿为：

- 将监听的套接字加入到epoll的句柄中，并且将事件类型mask设置为AE_READABLE；
- 取出套接字在事件处理器中对应的文件事件aeFileEvent，并且初始化文件事件的事件类型mask设置AE_READABLE，文件事件的读事件处理器和写事件处理器设置为acceptTcpHandler（）。

```c
for (j = 0; j < server.ipfd_count; j++) {
    if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
                          acceptTcpHandler,NULL) == AE_ERR)
    {
        serverPanic(
            "Unrecoverable error creating server.ipfd file event.");
    }
}
if (server.sofd > 0 && aeCreateFileEvent(server.el,server.sofd,AE_READABLE,
                                         acceptUnixHandler,NULL) == AE_ERR) serverPanic("Unrecoverable error creating server.sofd file event.");
```

(19) 如果AOF持久化功能打开，那么打开现有的AOF文件，如果AOF文件不存在，那么创建并打开一个新的AOF文件，为AOF写入做好准备。

(20) 如果是32位的架构，则设置服务器的最大内存（server.maxmemory）为3GB，内存淘汰策略为REDIS_MAXMEMORY_NO_EVICTION，如果是64位架构，则没这个限制

(21) 初始化Lua脚本系统：scriptingInit()

(22) 初始化慢查询功能：slowlogInit()

(23) 初始化服务器的后台异步I/O模块，为将来的I/O操作做好准备：bioInit()，这里的BIO模块其实是创建一个线程池，线程数量大小为2（REDIS_BIO_NUM_OPS），每个线程会处理一种类型的后台处理任务，分别是关闭文件任务和调用fdatasync，将AOF文件缓冲区的内容写入到磁盘 。线程池中需要使用到两个互斥量，分别用于对两个任务队列进行加锁，需要使用到两个条件变量。



#### 还原数据库状态

完成对server变量的初始化之后，服务器需要载入RDB文件或者AOF文件，并根据文件记录的内容来还原服务器的数据库状态。

1. 如果开启了AOF持久化功能，那么会优先使用AOF文件来恢复数据库，调用函数为：loadAppendOnlyFile（）。
2. 如果没有开启AOF持久化功能，就会使用RDB文件来恢复数据库，调用函数为：rdbLoad（）。

```c
void loadDataFromDisk(void) {
    long long start = ustime();
    //如果开启了aof，调用loadAppendOnlyFile进行加载
    if (server.aof_state == AOF_ON) {
        if (loadAppendOnlyFile(server.aof_filename) == C_OK)
            serverLog(LL_NOTICE,"DB loaded from append only file: %.3f seconds",(float)(ustime()-start)/1000000);
    } else {
        //否则，调用rdbLoad进行加载
        rdbSaveInfo rsi = RDB_SAVE_INFO_INIT;
        if (rdbLoad(server.rdb_filename,&rsi) == C_OK) {
            serverLog(LL_NOTICE,"DB loaded from disk: %.3f seconds",
                      (float)(ustime()-start)/1000000);

            /* Restore the replication ID / offset from the RDB file. */
            if ((server.masterhost ||
                 (server.cluster_enabled &&
                  nodeIsSlave(server.cluster->myself))) &&
                rsi.repl_id_is_set &&
                rsi.repl_offset != -1 &&
                /* Note that older implementations may save a repl_stream_db
                 * of -1 inside the RDB file in a wrong way, see more
                 * information in function rdbPopulateSaveInfo. */
                rsi.repl_stream_db != -1)
            {
                memcpy(server.replid,rsi.repl_id,sizeof(server.replid));
                server.master_repl_offset = rsi.repl_offset;
                /* If we are a slave, create a cached master from this
                 * information, in order to allow partial resynchronizations
                 * with masters. */
                replicationCacheMasterUsingMyself();
                selectDb(server.cached_master,rsi.repl_stream_db);
            }
        } else if (errno != ENOENT) {
            serverLog(LL_WARNING,"Fatal error loading the DB: %s. Exiting.",strerror(errno));
            exit(1);
        }
    }
}
```

#### 服务器最终初始化

服务器初始化过程最后会调用InitServerLast函数，这个函数就两个作用：①调用bioInit()初始化三个常驻BIO线程 ；②设置服务器内存使用量

```c
void InitServerLast() {
    //初始化三个常驻BIO线程
    bioInit();
    //设置服务器内存使用量
    server.initial_memory_usage = zmalloc_used_memory();
}
```

bioInit 函数是在bio.c文件中实现的，它的主要作用就是初始化BIO线程使用的数据结构，以及调用 pthread_create 函数创建三个常驻BIO线程：

- BIO_CLOSE_FILE：关闭文件任务；
- BIO_AOF_FSYNC：AOF新增数据刷盘任务；
- BIO_LAZY_FREE：惰性删除任务。