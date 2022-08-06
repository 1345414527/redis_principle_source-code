### AOF

#### 介绍

AOF是redis持久化的一种方式，以日志的形式来记录每个写操作，将Redis执行过的所有写指令记录下来（读操作补不可记录），只许追加文件但不可以改写文件，保存的是appendonly.aof文件。

aof机制默认关闭，可以通过`appendonly = yes`参数开启aof机制，通过`appendfilename = myaoffile.aof`指定aof文件名称。

#### 写入AOF

当AOF持久化功能处于打开状态时，服务器在执行完一个写命令之后，会调用propagate(call()函数中)将被执行的写命令追加到服务器状态的aof_buf缓冲区的末尾。

```c
struct redisServer {
	// sds 是redis定义的char数组
	sds aof_buf; 
}
```

```c
void propagate(struct redisCommand *cmd, int dbid, robj **argv, int argc, int flags)
{
    if (server.aof_state != AOF_OFF && flags & PROPAGATE_AOF)
        // aof功能打开的前提下，把新的追加aof_buffer
        feedAppendOnlyFile(cmd,dbid,argv,argc);
    if (flags & PROPAGATE_REPL)
        replicationFeedSlaves(server.slaves,dbid,argv,argc);
}

```

然后在serverCron(server.c)定时任务末尾，都会调用flushAppendOnlyFile(0)函数(aof.c)，考虑是否需要将aof_buf缓冲区中的内容写入和保存到AOF 文件里面。

flushAppendonlyFile函数的行为由服务器配置的appendfsync选项的值来决定，各个不同值产生的行为如下：

```shell
#aof持久化策略的配置
#no表示不执行fsync， 仅仅调用write函数将缓冲区的数据写如操作系统内核缓冲区，至于内核缓冲区的数据什么时候写入磁盘，由操作系统决定，性能最快。
#always表示每次事件循环都执行fsync，以保证数据同步到磁盘。
#everysec表示每秒执行一次fsync，可能会导致丢失这1s数据。
appendfsync everysec
```



#### 恢复

因为AOF文件里面包含了重建数据库状态所需的所有写命令，所以服务器只要读入并重新执行一遍AOF文件里面保存的写命令，就可以还原服务器关闭之前的数据库状态。

步骤如下：

1. 创建一个不带网络连接的伪客户端( **fake client** )。因为Redis 的命令只能在客户端上下文中执行，而载人AOF文件时所使用的命令直接来源于AOF文件而不是网络连接，所以服务器使用了一个没有网络连接的伪客户端来执行AOF文件保存的写命令，伪客户端执行命令的效果和带网络连接的客户端执行命令的效果完全一样。
2. 从AOF文件中分析并读取出一条写命令。
3. 使用伪客户端执行被读出的写命令。
4. 一直执行步骤⒉和步骤3，直到AOF文件中的所有写命令都被处理完毕为止。



#### 重写

因为AOF持久化是通过保存被执行的写命令来记录数据库状态的，所以随着服务器运行时间的流逝，AOF文件中的内容会越来越多，文件的体积也会越来越大，如果不加以控制的话，体积过大的AOF文件很可能对Redis服务器、甚至整个宿主计算机造成影响，并且AOF 文件的体积越大，使用AOF 文件来进行数据还原所需的时间就越多。

为了解决AOF文件体积膨胀的问题，Redis提供了AOF文件重写（rewrite)功能。通过该功能，Redis 服务器可以创建--个新的AOF文件来替代现有的AOF 文件，新旧两个AOF文件所保存的数据库状态相同，但新AOF文件不会包含任何浪费空间的冗余命令，所以新AOF文件的体积通常会比旧AOF文件的体积要小得多。

aof触发的时机：

1. 用户调用BGREWRITEAOF命令；
2. aof日志大小超过预设的限额。

对于触发aof重写机制也可以通过配置文件来进行设置：

```shell
# aof自动重写配置。当目前aof文件大小超过上一次重写的aof文件大小的百分之多少进行重写，即当aof文件增长到一定大小的时候Redis能够调用bgrewriteaof对日志文件进行重写。当前AOF文件大小是上次日志重写得到AOF文件大小的二倍（设置为100）时，自动启动新的日志重写过程。
auto-aof-rewrite-percentage 100
# 设置允许重写的最小aof文件大小，避免了达到约定百分比但尺寸仍然很小的情况还要重写
auto-aof-rewrite-min-size 64mb
```

当aop重写时会引发重写和持久化追加同时发生的问题，可以通过`no-appendfsync-on-rewrite no`进行配置

```shell
# 在aof重写或者写入rdb文件的时候，会执行大量IO，此时对于everysec和always的aof模式来说，执行fsync会造成阻塞过长时间，no-appendfsync-on-rewrite字段设置为默认设置为no，是最安全的方式，不会丢失数据，但是要忍受阻塞的问题。如果对延迟要求很高的应用，这个字段可以设置为yes，设置为yes表示rewrite期间对新写操作不fsync,暂时存在内存中,不会造成阻塞的问题（因为没有磁盘竞争），等rewrite完成后再写入，这个时候redis会丢失数据。Linux的默认fsync策略是30秒。可能丢失30秒数据。因此，如果应用系统无法忍受延迟，而可以容忍少量的数据丢失，则设置为yes。如果应用系统无法忍受数据丢失，则设置为no。
no-appendfsync-on-rewrite no
```

aof_rewrite函数可以创建新的AOF文件，但是这个函数会进行大量的写入操作，所以调用这个函数的线程将被长时间的阻塞，因为Redis服务器使用单线程来处理命令请求；所以如果直接是服务器进程调用AOF_REWRITE函数的话，那么重写AOF期间，服务器将无法处理客户端发送来的命令请求；

Redis不希望AOF重写会造成服务器无法处理请求，所以Redis决定**将AOF重写程序放到子进程（后台）里执行**。这样处理的最大好处是：

- 子进程进行AOF重写期间，主进程可以继续处理命令请求；
- 子进程带有主进程的数据副本，使用子进程而不是线程，可以避免在锁的情况下，保证数据的安全性。

子进程在进行AOF重写期间，服务器进程还要继续处理命令请求，而新的命令可能对现有的数据进行修改，这会让当前数据库的数据和重写后的AOF文件中的数据不一致。

为了解决这种数据不一致的问题，Redis增加了一个**AOF重写缓存**，这个缓存在fork出子进程之后开始启用，Redis服务器主进程在执行完写命令之后，会同时将这个写命令追加到AOF缓冲区和AOF重写缓冲区。

当子进程完成对AOF文件重写之后，它会向父进程发送一个完成信号，父进程接到该完成信号之后，会调用一个信号处理函数，该函数完成以下工作：

- **将AOF重写缓存中的内容全部写入到新的AOF文件中**；这个时候新的AOF文件所保存的数据库状态和服务器当前的数据库状态一致；
- **对新的AOF文件进行改名，原子的覆盖原有的AOF文件**；完成新旧两个AOF文件的替换。

当这个信号处理函数执行完毕之后，主进程就可以继续像往常一样接收命令请求了。在整个AOF后台重写过程中，**只有最后的 “主进程写入命令到AOF缓存” 和 “对新的AOF文件进行改名，覆盖原有的AOF文件” **这两个步骤（信号处理函数执行期间）会造成主进程阻塞，在其他时候，AOF后台重写都不会对主进程造成阻塞，这将AOF重写对性能造成的影响降到最低。


伪代码：

```c
def AOF_REWRITE(tmp_tile_name):

f = create(tmp_tile_name)

    # 遍历所有数据库
    for db in redisServer.db:

# 如果数据库为空，那么跳过这个数据库
if db.is_empty(): continue

    # 写入 SELECT 命令，用于切换数据库
    f.write_command("SELECT " + db.number)

    # 遍历所有键
    for key in db:

# 如果键带有过期时间，并且已经过期，那么跳过这个键
if key.have_expire_time() and key.is_expired(): continue

    if key.type == String:

# 用 SET key value 命令来保存字符串键

value = get_value_from_string(key)

    f.write_command("SET " + key + value)

    elif key.type == List:

# 用 RPUSH key item1 item2 ... itemN 命令来保存列表键

item1, item2, ..., itemN = get_item_from_list(key)

    f.write_command("RPUSH " + key + item1 + item2 + ... + itemN)

    elif key.type == Set:

# 用 SADD key member1 member2 ... memberN 命令来保存集合键

member1, member2, ..., memberN = get_member_from_set(key)

    f.write_command("SADD " + key + member1 + member2 + ... + memberN)

    elif key.type == Hash:

# 用 HMSET key field1 value1 field2 value2 ... fieldN valueN 命令来保存哈希键

field1, value1, field2, value2, ..., fieldN, valueN =\
    get_field_and_value_from_hash(key)

    f.write_command("HMSET " + key + field1 + value1 + field2 + value2 +\
                    ... + fieldN + valueN)

    elif key.type == SortedSet:

# 用 ZADD key score1 member1 score2 member2 ... scoreN memberN
# 命令来保存有序集键

score1, member1, score2, member2, ..., scoreN, memberN = \
    get_score_and_member_from_sorted_set(key)

    f.write_command("ZADD " + key + score1 + member1 + score2 + member2 +\
                    ... + scoreN + memberN)

    else:

raise_type_error()

    # 如果键带有过期时间，那么用 EXPIREAT key time 命令来保存键的过期时间
    if key.have_expire_time():
f.write_command("EXPIREAT " + key + key.expire_time_in_unix_timestamp())

    # 关闭文件
    f.close()
```

实际为了避免执行命令时造成客户端输入缓冲区溢出，重写程序在处理list hash set zset时，会检查键所包含的元素的个数，如果元素的数量超过了redis.h/REDIS_AOF_REWRITE_ITEMS_PER_CMD常量的值，那么重写程序会使用多条命令来记录键的值，而不是单使用一条命令。该常量默认值是64– 即每条命令设置的元素的个数 是最多64个，使用多条命令重写实现集合键中元素数量超过64个的键；







#### 优势

1. 根据不同的策略，可以实现每秒，每一次修改操作的同步持久化，就算在最恶劣的情况下只会丢失不会超过两秒数据。

2. 当文件太大时，会触发重写机制，确保文件不会太大。
3. 文件可以简单的读懂

#### 劣势

1. aof文件的大小太大，就算有重写机制，但重写所造成的阻塞问题是不可避免的
2. aof文件恢复速度慢



#### 源码分析

##### 序列化

在“写入AOF中”部分，我们已经知道了redis每次执行完写操作后(server.c#call)，会调用propagate函数将写操作追加到aof_buf缓冲区和处理主从复制：

```c
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
```

server.c#propagate：

```c
void propagate(struct redisCommand *cmd, int dbid, robj **argv, int argc, int flags)
{
    if (server.aof_state != AOF_OFF && flags & PROPAGATE_AOF)
        // aof功能打开的前提下，把新的追加aof_buffer
        feedAppendOnlyFile(cmd,dbid,argv,argc);
    if (flags & PROPAGATE_REPL)
        replicationFeedSlaves(server.slaves,dbid,argv,argc);
}
```

 在redis开启aof，并且该命令需要记录aof时，会调用aof.c#feedAppendOnlyFile函数用于生成并写入aof。下面看一下这个函数：

```c
void feedAppendOnlyFile(struct redisCommand *cmd, int dictid, robj **argv, int argc) {
    sds buf = sdsempty();
    robj *tmpargv[3];

    /* The DB this command was targeting is not the same as the last command
     * we appended. To issue a SELECT command is needed. */
    //当前操作的db与aof对应的db不同时，需要一个切换db的命令
    //在全局server结构中得aof_selected_db记录当前aof对应的数据库，如果当前命令操作的数据库与之不同的话，首先需要切换数据库。
    if (dictid != server.aof_selected_db) {
        char seldb[64];

        snprintf(seldb,sizeof(seldb),"%d",dictid);
        buf = sdscatprintf(buf,"*2\r\n$6\r\nSELECT\r\n$%lu\r\n%s\r\n",
                           (unsigned long)strlen(seldb),seldb);
        server.aof_selected_db = dictid;
    }

    //将命令序列化，并保存到buf
    if (cmd->proc == expireCommand || cmd->proc == pexpireCommand ||
        cmd->proc == expireatCommand) {
        /* Translate EXPIRE/PEXPIRE/EXPIREAT into PEXPIREAT */
        buf = catAppendOnlyExpireAtCommand(buf,cmd,argv[1],argv[2]);
    } else if (cmd->proc == setexCommand || cmd->proc == psetexCommand) {
        /* Translate SETEX/PSETEX to SET and PEXPIREAT */
        tmpargv[0] = createStringObject("SET",3);
        tmpargv[1] = argv[1];
        tmpargv[2] = argv[3];
        buf = catAppendOnlyGenericCommand(buf,3,tmpargv);
        decrRefCount(tmpargv[0]);
        buf = catAppendOnlyExpireAtCommand(buf,cmd,argv[1],argv[2]);
    } else if (cmd->proc == setCommand && argc > 3) {
        int i;
        robj *exarg = NULL, *pxarg = NULL;
        /* Translate SET [EX seconds][PX milliseconds] to SET and PEXPIREAT */
        buf = catAppendOnlyGenericCommand(buf,3,argv);
        for (i = 3; i < argc; i ++) {
            if (!strcasecmp(argv[i]->ptr, "ex")) exarg = argv[i+1];
            if (!strcasecmp(argv[i]->ptr, "px")) pxarg = argv[i+1];
        }
        serverAssert(!(exarg && pxarg));
        if (exarg)
            buf = catAppendOnlyExpireAtCommand(buf,server.expireCommand,argv[1],
                                               exarg);
        if (pxarg)
            buf = catAppendOnlyExpireAtCommand(buf,server.pexpireCommand,argv[1],
                                               pxarg);
    } else {
        /* All the other commands don't need translation or need the
         * same translation already operated in the command vector
         * for the replication itself. */
        buf = catAppendOnlyGenericCommand(buf,argc,argv);
    }

    /* Append to the AOF buffer. This will be flushed on disk just before
     * of re-entering the event loop, so before the client will get a
     * positive reply about the operation performed. */
    //判断是否开启aof，如果开启则将aof追加到aof buffer。此处，应该可以提前判断，避免关闭aof时的aof的序列化开销。
    if (server.aof_state == AOF_ON)
        server.aof_buf = sdscatlen(server.aof_buf,buf,sdslen(buf));

    /* If a background append only file rewriting is in progress we want to
     * accumulate the differences between the child DB and the current one
     * in a buffer, so that when the child process will do its work we
     * can append the differences to the new append only file. */
    // 如果开启了aof rewrite进程，将命令也添加到aof rewrite buf中,等rewrite完之后，再将rewrite buf的数据追加到文件中
    // aof_child_pid记录aof rewrite进程的pid，如果rewrite正在进行，这个值不为-1;
    // 如果当前正在进行aof rewrite，则将命令的aof追加到aof rewrite buffer，待rewrite结束后进行replay。
    if (server.aof_child_pid != -1)
        aofRewriteBufferAppend((unsigned char*)buf,sdslen(buf));

    sdsfree(buf);
}
```

这里注意一个问题，在feedAppendOnlyFile的源码中，同一份aof数据既保存到了aof buffer中，又保存到了aof rewrite buffer中，是不是显得很冗余？当然不是，我们需要知道的一点是：重写是由fork出来的子进程来完成的，在前面RDB的COW中我们介绍了，当前这个子进程并不是和主进程共享数据的，子进程拥有自己的一份副本数据，当前这个副本数据并不包含fork之后的新加的命令，因此需要在重写完成后的后序处理中再写入aof rewrite buffer中的数据。这样子，新的aof文件和旧的aof文件就是完全相同的了。



##### 刷盘

具体的刷盘时机是靠aof持久化策略的配置(appendfsync)来决定的，在每一次循环前的beforeSleep中进行一次刷盘：

```c
void beforeSleep(struct aeEventLoop *eventLoop) {
    ...
    /* Write the AOF buffer on disk */
    //在beforeSleep中，是为了在给客户端发送响应内容前进行，保证返回给客户端的内容都是写过aof的。
    // 同时，也保证一轮事件循环，对于多个客户端的请求处理只写一次aof，提升性能（当然，这样做的缺点就是不能保证数据的一致性）。
    flushAppendOnlyFile(0);
	...
}
```

在每次执行serverCron时也会对AOF刷盘进行判断：

```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
   ...
   /* AOF postponed flush: Try at every cron cycle if the slow fsync
     * completed. */
    //在aof_flush_postponed_start不为0时调用，即存在延迟flush的情况。
    // 主要是保证fsync完成之后，可以快速的进入下一次flush。
    // 尽量保证fsync策略是everysec时，每秒都可以进行fsync，同时缩短两次fsync的间隔，减少影响。
    if (server.aof_flush_postponed_start) flushAppendOnlyFile(0);

    /* AOF write errors: in this case we have a buffer to flush as well and
     * clear the AOF error in case of success to make the DB writable again,
     * however to try every second is enough in case of 'hz' is set to
     * an higher frequency. */
    //保证aof出错时，尽快执行下一次flush，以便从错误恢复。
    run_with_period(1000) {
        if (server.aof_last_write_status == C_ERR)
            flushAppendOnlyFile(0);
    }
   ...
}
```

看具体的刷盘函数：

```c
/*
 * force表示是否是强迫进行flush:
 * 如果是serverCron中或是beforeSleep中调用则为0，不是强迫；
 * 如果是redis关闭前调用则为1，强迫进行刷盘
 */
void flushAppendOnlyFile(int force) {
    ssize_t nwritten;
    int sync_in_progress = 0;
    mstime_t latency;

    // 检查aof buffer是否为空，空的话直接返回，没必要进行flush。
    if (sdslen(server.aof_buf) == 0) {
        /* Check if we need to do fsync even the aof buffer is empty,
         * because previously in AOF_FSYNC_EVERYSEC mode, fsync is
         * called only when aof buffer is not empty, so if users
         * stop write commands before fsync called in one second,
         * the data in page cache cannot be flushed in time. */
        if (server.aof_fsync == AOF_FSYNC_EVERYSEC &&
            server.aof_fsync_offset != server.aof_current_size &&
            server.unixtime > server.aof_last_fsync &&
            !(sync_in_progress = aofFsyncInProgress())) {
            goto try_fsync;
        } else {
            return;
        }
    }

    /*
     *  fsync是阻塞操作，避免影响主线程的事件循环，fsync操作由后台线程完成
     */

    //如果设置的fsync策略是everysec，获取是否有后台线程正在进行fsync
    if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
        sync_in_progress = aofFsyncInProgress();

    if (server.aof_fsync == AOF_FSYNC_EVERYSEC && !force) {
        /* With this append fsync policy we do background fsyncing.
         * If the fsync is still in progress we can try to delay
         * the write for a couple of seconds. */
        if (sync_in_progress) {
            //记录上次延迟flush的时间戳，如果等于0，说明没有延迟
            if (server.aof_flush_postponed_start == 0) {
                /* No previous write postponing, remember that we are
                 * postponing the flush and return. */
                server.aof_flush_postponed_start = server.unixtime;
                return;
                //如果当前时间戳server.unixtime与延迟flush的时间戳间隔小于2s，那么没有违反everysec策略，不进行flush，直接返回
            } else if (server.unixtime - server.aof_flush_postponed_start < 2) {
                /* We were already waiting for fsync to finish, but for less
                 * than two seconds this is still ok. Postpone again. */
                return;
            }
            /* Otherwise fall trough, and go write since we can't wait
             * over two seconds. */
            server.aof_delayed_fsync++;
            serverLog(LL_NOTICE,"Asynchronous AOF fsync is taking too long (disk is busy?). Writing the AOF buffer without waiting for fsync to complete, this may slow down Redis.");
        }
    }
    /* We want to perform a single write. This should be guaranteed atomic
     * at least if the filesystem we are writing is a real physical one.
     * While this will save us against the server being killed I don't think
     * there is much to do about the whole server stopping for power problems
     * or alike */
    /*
     * 如果没有当前没有进行fsync，或者当前时间戳server.unixtime与延迟flush的时间戳间隔大于2s，
     * 就会跳过这段代码，进行flush操作。
     */

    latencyStartMonitor(latency);
    //调用write函数将aof_buf的数据写入文件的内核缓冲区，此处只是写入page cache，还需要fsync
    nwritten = aofWrite(server.aof_fd,server.aof_buf,sdslen(server.aof_buf));
    latencyEndMonitor(latency);
    /* We want to capture different events for delayed writes:
     * when the delay happens with a pending fsync, or with a saving child
     * active, and when the above two conditions are missing.
     * We also use an additional event name to save all samples which is
     * useful for graphing / monitoring purposes. */
    if (sync_in_progress) {
        latencyAddSampleIfNeeded("aof-write-pending-fsync",latency);
    } else if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) {
        latencyAddSampleIfNeeded("aof-write-active-child",latency);
    } else {
        latencyAddSampleIfNeeded("aof-write-alone",latency);
    }
    latencyAddSampleIfNeeded("aof-write",latency);

    /* We performed the write so reset the postponed flush sentinel to zero. */
    // 重置aof_flush_postponed_start，因为接下来会进行flush。
    server.aof_flush_postponed_start = 0;

    //判断aof buf中的数据是否全部写入
    if (nwritten != (ssize_t)sdslen(server.aof_buf)) {
        //错误分支
        static time_t last_write_error_log = 0;
        int can_log = 0;

        /* Limit logging rate to 1 line per AOF_WRITE_LOG_ERROR_RATE seconds. */
        // 限制记录错误日志的频率
        if ((server.unixtime - last_write_error_log) > AOF_WRITE_LOG_ERROR_RATE) {
            can_log = 1;
            last_write_error_log = server.unixtime;
        }

        /* Log the AOF write error and record the error code. */
        //write返回值是-1，说明调用错误，只记录日志。
        if (nwritten == -1) {
            if (can_log) {
                serverLog(LL_WARNING,"Error writing to the AOF file: %s",
                          strerror(errno));
                server.aof_last_write_errno = errno;
            }
        } else {
            //接下来是处理部分写的情况
            if (can_log) {
                serverLog(LL_WARNING,"Short write while writing to "
                          "the AOF file: (nwritten=%lld, "
                          "expected=%lld)",
                          (long long)nwritten,
                          (long long)sdslen(server.aof_buf));
            }

            // aof_current_size记录当前正确写入的aof的长度,当前write只写入部分数据，此处保证完整性，将写入的部分数据删掉
            //如果ftruncate成功，会设置nwritten为-1;
            //如果失败的话，后面会将aof_current_size增加部分写的数据长度，同时将aof_buf中截取已写入部分。
            if (ftruncate(server.aof_fd, server.aof_current_size) == -1) {
                if (can_log) {
                    serverLog(LL_WARNING, "Could not remove short write "
                              "from the append-only file.  Redis may refuse "
                              "to load the AOF the next time it starts.  "
                              "ftruncate: %s", strerror(errno));
                }
            } else {
                /* If the ftruncate() succeeded we can set nwritten to
                 * -1 since there is no longer partial data into the AOF. */
                nwritten = -1;
            }
            server.aof_last_write_errno = ENOSPC;
        }

        /* Handle the AOF write error. */
        //fsync策略是always，write失败后，不能恢复，直接退出
        if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
            /* We can't recover when the fsync policy is ALWAYS since the
             * reply for the client is already in the output buffers, and we
             * have the contract with the user that on acknowledged write data
             * is synced on disk. */
            serverLog(LL_WARNING,"Can't recover from AOF write error when the AOF fsync policy is 'always'. Exiting...");
            exit(1);
        } else {
            /* Recover from failed write leaving data into the buffer. However
             * set an error to stop accepting writes as long as the error
             * condition is not cleared. */
            //修改最后一次的写入状态
            server.aof_last_write_status = C_ERR;

            /* Trim the sds buffer if there was a partial write, and there
             * was no way to undo it with ftruncate(2). */
            if (nwritten > 0) {
                //在ftruncate失败时，修改aof_current_size，并把已经写入内核缓冲区的数据从buffer中清空
                //剩余的数据下次再尝试写入和刷盘
                server.aof_current_size += nwritten;
                sdsrange(server.aof_buf,nwritten,-1);
            }
            return; /* We'll try again on the next call... */
        }
    } else {
        //正确分支
        /* Successful write(2). If AOF was in error state, restore the
         * OK state and log the event. */
        // 如果fsync策略不是always，在write出错时，会有server.aof_last_write_status记录错误状态。
        // 如果后续的write操作正常，此处只是打印日志，表示错误恢复正常。
        if (server.aof_last_write_status == C_ERR) {
            serverLog(LL_WARNING,
                      "AOF write error looks solved, Redis can write again.");
            server.aof_last_write_status = C_OK;
        }
    }
    //修改aof文件的大小
    server.aof_current_size += nwritten;

    /* Re-use AOF buffer when it is small enough. The maximum comes from the
     * arena size of 4k minus some overhead (but is otherwise arbitrary). */
    // aof_buf已成功写入文件，可以清空。
    // 为避免频繁分配、释放内存，此处保证在buf小于4K时，会一直重用该buf。如果大于4K，就会释放旧的buf，分配新的。
    if ((sdslen(server.aof_buf)+sdsavail(server.aof_buf)) < 4000) {
        sdsclear(server.aof_buf);
    } else {
        sdsfree(server.aof_buf);
        server.aof_buf = sdsempty();
    }

try_fsync:
    // 如果配置了no-appendfsync-on-rewrite，即在有aof rewrite或者是rdb save的子进程时不进行fsync，
    // 主要是避免对磁盘产生过大压力，这里会直接返回，不进行fsync。
     if (server.aof_fsync == AOF_FSYNC_ALWAYS) {
        /* redis_fsync is defined as fdatasync() for Linux in order to avoid
         * flushing metadata. */
        latencyStartMonitor(latency);
        //调用fsync进行刷盘
        redis_fsync(server.aof_fd); /* Let's try to get this data on the disk */
        latencyEndMonitor(latency);
        latencyAddSampleIfNeeded("aof-fsync-always",latency);
        server.aof_fsync_offset = server.aof_current_size;
        server.aof_last_fsync = server.unixtime;
    } else if ((server.aof_fsync == AOF_FSYNC_EVERYSEC &&
                server.unixtime > server.aof_last_fsync)) {
        //因为unixtime是以秒为单位的，因此就可以判断出是不是超过了一秒
        //如果此时没有在fsync
        if (!sync_in_progress) {
            //提交一个aof刷盘的后台任务，由指定类型的BIO后台线程进行执行。类似于fsync，也是一个子线程进行执行，但并不一定立即执行，因为当前任务可能会等待
            aof_background_fsync(server.aof_fd);
            server.aof_fsync_offset = server.aof_current_size;
        }
        server.aof_last_fsync = server.unixtime;
    }
}
```

可以看一下aof_background_fsync函数：

```c
void aof_background_fsync(int fd) {
    //创建一个AOF Fsync后台任务
    bioCreateBackgroundJob(BIO_AOF_FSYNC,(void*)(long)fd,NULL,NULL);
}
```

BIO后台任务相关的内容可以看后面的“线程问题”章节。



##### 载入

 AOF加载的流程要简单些，在启动后，读取AOF，然后将每条命令进行replay即可。下面看一下具体代码。在redis.c的main函数中，完成初始化后，会调用loadDataFromDisk()完成数据的加载：

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

其中调用loadAppendOnlyFile进行加载：

```c
int loadAppendOnlyFile(char *filename) {
    //声明一个伪客户端
    struct client *fakeClient;
    //接下来打开aof文件并检查其大小
    FILE *fp = fopen(filename,"r");
    struct redis_stat sb;
    int old_aof_state = server.aof_state;
    long loops = 0;
    off_t valid_up_to = 0; /* Offset of latest well-formed command loaded. */
    off_t valid_before_multi = 0; /* Offset before MULTI command loaded. */

    if (fp == NULL) {
        serverLog(LL_WARNING,"Fatal error: can't open the append log file for reading: %s",strerror(errno));
        exit(1);
    }

    /* Handle a zero-length AOF file as a special case. An empty AOF file
     * is a valid AOF because an empty server with AOF enabled will create
     * a zero length file at startup, that will remain like that if no write
     * operation is received. */
    if (fp && redis_fstat(fileno(fp),&sb) != -1 && sb.st_size == 0) {
        server.aof_current_size = 0;
        server.aof_fsync_offset = server.aof_current_size;
        fclose(fp);
        return C_ERR;
    }

    /* Temporarily disable AOF, to prevent EXEC from feeding a MULTI
     * to the same file we're about to read. */
    //要短暂的关闭AOF，防止事务向我们要读取的文件执行命令
    server.aof_state = AOF_OFF;

    //创建伪客户端，fd=-1
    fakeClient = createFakeClient();
    //startLoading会设置状态信息:
    //  1. redisServer.loading置为1，表示当前正处于数据加载阶段。此时有客户端访问时，会根据loading状态返回“数据正在加载...”;
    //  2. 将当前时间赋值给redisServer.loading_start_time，用以统计数据加载时间;
    //  3. 将aof文件大小赋值给redisServer.loading_total_bytes，用以统计加载进度。
    startLoading(fp);

    /* Check if this AOF file has an RDB preamble. In that case we need to
     * load the RDB file and later continue loading the AOF tail. */
    //检查AOF文件是否有一个RDB格式的前缀。在这种情况下，我们需要先加载RDB文件，然后再加载后面的AOF文件
    char sig[5]; /* "REDIS" */
    if (fread(sig,1,5,fp) != 5 || memcmp(sig,"REDIS",5) != 0) {
        /* No RDB preamble, seek back at 0 offset. */
        if (fseek(fp,0,SEEK_SET) == -1) goto readerr;
    } else {
        /* RDB preamble. Pass loading the RDB functions. */
        rio rdb;

        serverLog(LL_NOTICE,"Reading RDB preamble from AOF file...");
        if (fseek(fp,0,SEEK_SET) == -1) goto readerr;
        rioInitWithFile(&rdb,fp);
        if (rdbLoadRio(&rdb,NULL,1) != C_OK) {
            serverLog(LL_WARNING,"Error reading the RDB preamble of the AOF file, AOF loading aborted");
            goto readerr;
        } else {
            serverLog(LL_NOTICE,"Reading the remaining AOF tail...");
        }
    }

    /* Read the actual AOF file, in REPL format, command by command. */
    //接下来是一个while循环，不断的读取命令并执行
    while(1) {
        int argc, j;
        unsigned long len;
        robj **argv;
        char buf[128];
        sds argsds;
        struct redisCommand *cmd;

        /* Serve the clients from time to time */
        // loops记录循环次数，在每执行1000次循环时，会更新一下加载进度。
        if (!(loops++ % 1000)) {
            loadingProgress(ftello(fp));
            //由于加载过程一般比较长，所以此处会调用processEventsWhileBlocked函数，处理文件io事件，避免客户端一直阻塞。
            // 这个函数可以完成，客户端连接的建立，同时响应请求（数据正在加载，不完整，所以响应的内容都是返回错误，并提示“数据正在加载...”）。
            processEventsWhileBlocked();
        }


        /*
         * 接下来根据inline和multibulk协议的格式进行分别的加载！！！这一部分在客户端的multibulk有讲解
         */

        //读一行，遇到\n
        if (fgets(buf,sizeof(buf),fp) == NULL) {
            //读到eof，加载完毕
            if (feof(fp))
                break;
            else
                goto readerr;
        }
        //处理'*MULTI_BULK_LEN\r\n'
        if (buf[0] != '*') goto fmterr;
        if (buf[1] == '\0') goto readerr;
        argc = atoi(buf+1);
        if (argc < 1) goto fmterr;

        /* Load the next command in the AOF as our fake client
         * argv. */
        argv = zmalloc(sizeof(robj*)*argc);
        fakeClient->argc = argc;
        fakeClient->argv = argv;

        // 读取multi bulk的长度，接下来是一个for循环，一次读取每个bulk。
        for (j = 0; j < argc; j++) {
            /* Parse the argument len. */
            //处理'$BULK_LEN\r\n'
            char *readres = fgets(buf,sizeof(buf),fp);
            if (readres == NULL || buf[0] != '$') {
                fakeClient->argc = j; /* Free up to j-1. */
                freeFakeClientArgv(fakeClient);
                if (readres == NULL)
                    goto readerr;
                else
                    goto fmterr;
            }
            len = strtol(buf+1,NULL,10);

            /* Read it into a string object. */
            //分配响应大小的buffer
            argsds = sdsnewlen(SDS_NOINIT,len);
            //二进制读取len大小的buffer
            if (len && fread(argsds,len,1,fp) == 0) {
                sdsfree(argsds);
                fakeClient->argc = j; /* Free up to j-1. */
                freeFakeClientArgv(fakeClient);
                goto readerr;
            }
            argv[j] = createObject(OBJ_STRING,argsds);

            /* Discard CRLF. */
            //跳过\r\n
            if (fread(buf,2,1,fp) == 0) {
                fakeClient->argc = j+1; /* Free up to j. */
                freeFakeClientArgv(fakeClient);
                goto readerr;
            }
        }

        /* Command lookup */
        //解析出完整命令后，需要执行该命令，首先根据命令名，查找对应的command结构，最后回调命令处理函数。
        cmd = lookupCommand(argv[0]->ptr);
        if (!cmd) {
            serverLog(LL_WARNING,
                      "Unknown command '%s' reading the append only file",
                      (char*)argv[0]->ptr);
            exit(1);
        }

        if (cmd == server.multiCommand) valid_before_multi = valid_up_to;

        /* Run the command in the context of a fake client */
        fakeClient->cmd = cmd;
        if (fakeClient->flags & CLIENT_MULTI &&
            fakeClient->cmd->proc != execCommand)
        {
            queueMultiCommand(fakeClient);
        } else {
            cmd->proc(fakeClient);
        }

        /* The fake client should not have a reply */
        // fake client对应的socket fd为负数，准备响应的函数prepareClientToWrite会据此作判断，不返回响应内容
        //此处进行校验，因为fake client不可能有响应内容
        serverAssert(fakeClient->bufpos == 0 &&
                     listLength(fakeClient->reply) == 0);

        /* The fake client should never get blocked */
        serverAssert((fakeClient->flags & CLIENT_BLOCKED) == 0);

        /* Clean up. Command code may have changed argv/argc so we use the
         * argv/argc of the client instead of the local variables. */
        //最后清理fake client，以便下一个命令的执行。
        // valid_up_to记录当前正确解析的日志长度，在数据不完整（提前读到eof）并且设置aof_load_truncated时，会将aof文件截断到valid_up_to字节。
        freeFakeClientArgv(fakeClient);
        fakeClient->cmd = NULL;
        if (server.aof_load_truncated) valid_up_to = ftello(fp);
    }
```



##### 重写

> 先提一句，我觉得重写的源码和rdbSaveBackground的相关源码有很多相似之处，可以对比看一看。

 首先看一下，BGREWRITEAOF命令对应的处理函数：

```c
void bgrewriteaofCommand(redisClient *c) {
    //of_child_pid指示进行aof rewrite进程的pid，rdb_child_pid指示进行rdb dump的进程pid。
    if (server.aof_child_pid != -1) {
        //如果当前正在进行aof rewrite，则返回客户端错误；
        addReplyError(c,"Background append only file rewriting already in progress");
    } else if (server.rdb_child_pid != -1) {
        //如果当前正在进行rdb dump，为了避免对磁盘造成压力，将aof_rewrite_scheduled置为1，随后在没有进行aof rewrite和rdb dump时，再开启rewrite；
        server.aof_rewrite_scheduled = 1;
        addReplyStatus(c,"Background append only file rewriting scheduled");
    } else if (rewriteAppendOnlyFileBackground() == REDIS_OK) {
        //如果当前没有aof rewrite和rdb dump在进行，则调用rewriteAppendOnlyFileBackground进行aof rewrite。
        addReplyStatus(c,"Background append only file rewriting started");
    } else {
        //异常情况，直接返回错误。
        addReply(c,shared.err);
    }
}
```

下面，看一下serverCron中是如何触发aof rewrite的：

> 注意：为了方便看，我这里将rdb和aof的相关源码全部贴出来了。

```c
int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
    ...
        
    /*
     * AOF重写在serverCron中第一个触发点：
     * 需要确认当前没有aof rewrite和rdb dump在进行，并且设置了aof_rewrite_scheduled(在bgrewriteaofCommand里设置的)，
     * 调用rewirteAppendOnlyFileBackground进行aof rewrite。
     */
    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1 &&
        server.aof_rewrite_scheduled)
    {
        rewriteAppendOnlyFileBackground();
    }

    // 检测bgsave、aof重写是否在执行过程中，或者是否有子线程
    if (server.rdb_child_pid != -1 || server.aof_child_pid != -1 ||
        ldbPendingChildren())
    {
        int statloc;
        pid_t pid;

        //等待所有的子进程
        if ((pid = wait3(&statloc,WNOHANG,NULL)) != 0) {
            //取得子进程exit()返回的结束代码
            int exitcode = WEXITSTATUS(statloc);
            int bysignal = 0;

            //如果子进程是因为信号而结束则此宏值为真
            if (WIFSIGNALED(statloc)) bysignal = WTERMSIG(statloc);

            if (pid == -1) {
                //如果此时没有子线程
                serverLog(LL_WARNING,"wait3() returned an error: %s. "
                    "rdb_child_pid = %d, aof_child_pid = %d",
                    strerror(errno),
                    (int) server.rdb_child_pid,
                    (int) server.aof_child_pid);
            } else if (pid == server.rdb_child_pid) {
                //如果已经完成了bgsave,会调用backgroundSaveDoneHandler函数做最后处理
                backgroundSaveDoneHandler(exitcode,bysignal);
                if (!bysignal && exitcode == 0) receiveChildInfo();
            } else if (pid == server.aof_child_pid) {
                //如果已经完成了aof重写，会调用backgroundRewriteDoneHandler函数做最后处理
                backgroundRewriteDoneHandler(exitcode,bysignal);
                if (!bysignal && exitcode == 0) receiveChildInfo();
            } else {
                if (!ldbRemoveChild(pid)) {
                    serverLog(LL_WARNING,
                        "Warning, detected child with unmatched pid: %ld",
                        (long)pid);
                }
            }
            updateDictResizePolicy();
            closeChildInfoPipe();
        }
    } else {
        /*
         * 如果此时没有子线程在进行save或rewrite，则判断是否要save或rewrite
         */

        //先判断是否要进行save
        /*
         * RDB的持久化在serverCron中第一个触发点，根据条件判断定期触发：
         */
        for (j = 0; j < server.saveparamslen; j++) {
            //每一个saveparam表示一个配置文件中的save
            struct saveparam *sp = server.saveparams+j;

            /* Save if we reached the given amount of changes,
             * the given amount of seconds, and if the latest bgsave was
             * successful or if, in case of an error, at least
             * CONFIG_BGSAVE_RETRY_DELAY seconds already elapsed. */
            //判断修改次数和时间
            if (server.dirty >= sp->changes &&
                server.unixtime-server.lastsave > sp->seconds &&
                (server.unixtime-server.lastbgsave_try >
                 CONFIG_BGSAVE_RETRY_DELAY ||
                 server.lastbgsave_status == C_OK))
            {
                serverLog(LL_NOTICE,"%d changes in %d seconds. Saving...",
                    sp->changes, (int)sp->seconds);
                rdbSaveInfo rsi, *rsiptr;
                rsiptr = rdbPopulateSaveInfo(&rsi);
                //调用rdbSaveBackground进行save
                rdbSaveBackground(server.rdb_filename,rsiptr);
                break;
            }
        }

        /*
         * AOF重写在serverCron中第二个触发点：
         * 判断aof文件的大小超过预定的百分比，
         * 当aof文件超过了预定的最小值，并且超过了上一次aof文件的一定百分比，则会触发aof rewrite。
         */
        if (server.aof_state == AOF_ON &&
            server.rdb_child_pid == -1 &&
            server.aof_child_pid == -1 &&
            server.aof_rewrite_perc &&
            server.aof_current_size > server.aof_rewrite_min_size)
        {
            long long base = server.aof_rewrite_base_size ?
                server.aof_rewrite_base_size : 1;
            long long growth = (server.aof_current_size*100/base) - 100;
            if (growth >= server.aof_rewrite_perc) {
                serverLog(LL_NOTICE,"Starting automatic rewriting of AOF on %lld%% growth",growth);
                rewriteAppendOnlyFileBackground();
            }
        }
    }


    /* AOF postponed flush: Try at every cron cycle if the slow fsync
     * completed. */
    //进行AOF buffer的刷盘:
    // 在aof_flush_postponed_start不为0时调用，即存在延迟flush的情况。
    // 主要是保证fsync完成之后，可以快速的进入下一次flush。
    // 尽量保证fsync策略是everysec时，每秒都可以进行fsync，同时缩短两次fsync的间隔，减少影响。
    if (server.aof_flush_postponed_start) flushAppendOnlyFile(0);

    /* AOF write errors: in this case we have a buffer to flush as well and
     * clear the AOF error in case of success to make the DB writable again,
     * however to try every second is enough in case of 'hz' is set to
     * an higher frequency. */
    //进行AOF buffer的刷盘:保证aof出错时，尽快执行下一次flush，以便从错误恢复。
    run_with_period(1000) {
        if (server.aof_last_write_status == C_ERR)
            flushAppendOnlyFile(0);
    }
    
    ...
        
    /*
     * RDB的持久化在serverCron中第二个触发点，需要确认当前没有aof rewrite和rdb dump在进行，并且设置了rdb_bgsave_scheduled：
     * 如果上次触发bgsave时已经有进程在执行aof的重写了(rdbSaveBackground中)，就会标记rdb_bgsave_scheduled=1，
     * 然后放到serverCron，然后在serverCron的最后在进行判断是否能够执行
     */
    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1 &&
        server.rdb_bgsave_scheduled &&
        (server.unixtime-server.lastbgsave_try > CONFIG_BGSAVE_RETRY_DELAY ||
         server.lastbgsave_status == C_OK))
    {
        rdbSaveInfo rsi, *rsiptr;
        rsiptr = rdbPopulateSaveInfo(&rsi);
        if (rdbSaveBackground(server.rdb_filename,rsiptr) == C_OK)
            server.rdb_bgsave_scheduled = 0;
    }
    
    ...
}
```

接下来看看重写的具体逻辑吧，也就是rewriteAppendOnlyFileBackground， rewrite的大致流程是：创建子进程，获取当前快照，同时将之后的命令记录到aof_rewrite_buf中，子进程遍历db生成aof临时文件，然后退出；父进程wait子进程，待结束后，将aof_rewrite_buf中的数据追加到该aof文件中，最后重命名该临时文件为正式的aof文件。

```c
int rewriteAppendOnlyFileBackground(void) {
    pid_t childpid;
    long long start;

    //如果已经有aof重写子线程或rdb持久化子线程就直接返回
    if (server.aof_child_pid != -1 || server.rdb_child_pid != -1) return C_ERR;
    if (aofCreatePipes() != C_OK) return C_ERR;
    openChildInfoPipe();
    //获取当前时间，用于统计fork耗时
    start = ustime();
    //调用fork，进入子进程的流程
    if ((childpid = fork()) == 0) {
        char tmpfile[256];

        /* Child */
        //子进程首先关闭监听socket，避免接收客户端连接
        closeClildUnusedResourceAfterFork();
        //设置进程的title
        redisSetProcTitle("redis-aof-rewrite");
        //生成临时aof文件名
        snprintf(tmpfile,256,"temp-rewriteaof-bg-%d.aof", (int) getpid());
        //调用rewriteAppendOnlyFile进行rewrite
        if (rewriteAppendOnlyFile(tmpfile) == C_OK) {
            //如果rewrite成功，统计copy-on-write的脏页并记录日志，然后以退出码0退出进程
            size_t private_dirty = zmalloc_get_private_dirty(-1);

            if (private_dirty) {
                serverLog(LL_NOTICE,
                          "AOF rewrite: %zu MB of memory used by copy-on-write",
                          private_dirty/(1024*1024));
            }

            server.child_info_data.cow_size = private_dirty;
            sendChildInfo(CHILD_INFO_TYPE_AOF);
            exitFromChild(0);
        } else {
            //如果rewrite失败，则退出进程并返回1作为退出码
            exitFromChild(1);
        }
    } else {
        /* Parent */
        // 父进程更新redisServer记录一些信息，例如：fork进程消耗的时间stat_fork_time,
        server.stat_fork_time = ustime()-start;
        // 更新redisServer记录fork速率：每秒多少G；zmalloc_used_memory()的单位是字节，所以通过除以(1024*1024*1024),得到GB；由于记录的fork_time即fork时间是微妙，所以*1000000，得到每秒钟fork多少GB的速度；
        server.stat_fork_rate = (double) zmalloc_used_memory() * 1000000 / server.stat_fork_time / (1024*1024*1024); /* GB per second. */
        latencyAddSampleIfNeeded("fork",server.stat_fork_time/1000);

        // 如果fork子进程出错，即childpid为-1
        if (childpid == -1) {
            closeChildInfoPipe();
            serverLog(LL_WARNING,
                      "Can't rewrite append only file in background: fork: %s",
                      strerror(errno));
            aofClosePipes();
            return C_ERR;
        }
        serverLog(LL_NOTICE,
                  "Background append only file rewriting started by pid %d",childpid);
        //如果fork子进程成功
        //对aof_rewrite_scheduled清零，记录rewrite开始时间以及aof_child_pid（redis通过这个属性判断是否有aof rewrite在进行）
        server.aof_rewrite_scheduled = 0;
        server.aof_rewrite_time_start = time(NULL);
        server.aof_child_pid = childpid;
        //调用updateDictResizePolicy调整db的key space的rehash策略
        //由于创建了子进程，避免copy-on-write复制大量内存页，这里会禁止dict的rehash
        updateDictResizePolicy();
        /* We set appendseldb to -1 in order to force the next call to the
         * feedAppendOnlyFile() to issue a SELECT command, so the differences
         * accumulated by the parent into server.aof_rewrite_buf will start
         * with a SELECT statement and it will be safe to merge. */
        //将aof_selected_db置为-1，目的是，下一条aof会首先生成一条select db的日志，同时会写到aof_rewrite_buf中，
        //这样就可以将aof_rewrite_buf正常的追加到rewrite之后的文件。（序列化中的FeedAppendOnlyFile）
        server.aof_selected_db = -1;
        replicationScriptCacheFlush();
        return C_OK;
    }
    return C_OK; /* unreached */
}
```

看一下具体的rewrite过程，即rewriteAppendOnlyFile，大体上，就是遍历所有key，进行序列化，然后记录到aof文件中。

```c
int rewriteAppendOnlyFile(char *filename) {
    rio aof;
    FILE *fp;
    char tmpfile[256];
    char byte;

    /* Note that we have to use a different temp name here compared to the
     * one used by rewriteAppendOnlyFileBackground() function. */
    //生成临时文件名并创建该文件
    snprintf(tmpfile,256,"temp-rewriteaof-%d.aof", (int) getpid());
    fp = fopen(tmpfile,"w");
    if (!fp) {
        serverLog(LL_WARNING, "Opening the temp file for AOF rewrite in rewriteAppendOnlyFile(): %s", strerror(errno));
        return C_ERR;
    }

    // rio就是面向流的I/O接口，底层可以有不同实现，目前提供了文件和内存buffer的实现。这里对rio进行初始化。
    server.aof_child_diff = sdsempty();
    //如果配置了server.aof_rewrite_incremental_fsync，则在写aof时会增量地进行fsync，
    //这里配置的是每写入32M就sync一次。避免集中sync导致磁盘跑满。
    rioInitWithFile(&aof,fp);

    if (server.aof_rewrite_incremental_fsync)
        rioSetAutoSync(&aof,REDIS_AUTOSYNC_BYTES);

    if (server.aof_use_rdb_preamble) {
        int error;
        if (rdbSaveRio(&aof,&error,RDB_SAVE_AOF_PREAMBLE,NULL) == C_ERR) {
            errno = error;
            goto werr;
        }
    } else {
        //借助rio进行所有K-V的重写
        if (rewriteAppendOnlyFileRio(&aof) == C_ERR) goto werr;
    }

    /* Do an initial slow fsync here while the parent is still sending
     * data, in order to make the next final fsync faster. */
    if (fflush(fp) == EOF) goto werr;
    if (fsync(fileno(fp)) == -1) goto werr;

    /* Read again a few times to get more data from the parent.
     * We can't read forever (the server may receive data from clients
     * faster than it is able to send data to the child), so we try to read
     * some more data in a loop as soon as there is a good chance more data
     * will come. If it looks like we are wasting time, we abort (this
     * happens after 20 ms without new data). */
    int nodata = 0;
    mstime_t start = mstime();
    while(mstime()-start < 1000 && nodata < 20) {
        if (aeWait(server.aof_pipe_read_data_from_parent, AE_READABLE, 1) <= 0)
        {
            nodata++;
            continue;
        }
        nodata = 0; /* Start counting from zero, we stop on N *contiguous*
                       timeouts. */
        aofReadDiffFromParent();
    }

    /* Ask the master to stop sending diffs. */
    if (write(server.aof_pipe_write_ack_to_parent,"!",1) != 1) goto werr;
    if (anetNonBlock(NULL,server.aof_pipe_read_ack_from_parent) != ANET_OK)
        goto werr;
    /* We read the ACK from the server using a 10 seconds timeout. Normally
     * it should reply ASAP, but just in case we lose its reply, we are sure
     * the child will eventually get terminated. */
    if (syncRead(server.aof_pipe_read_ack_from_parent,&byte,1,5000) != 1 ||
        byte != '!') goto werr;
    serverLog(LL_NOTICE,"Parent agreed to stop sending diffs. Finalizing AOF...");

    /* Read the final diff if any. */
    aofReadDiffFromParent();

    /* Write the received diff to the file. */
    serverLog(LL_NOTICE,
              "Concatenating %.2f MB of AOF diff received from parent.",
              (double) sdslen(server.aof_child_diff) / (1024*1024));
    if (rioWrite(&aof,server.aof_child_diff,sdslen(server.aof_child_diff)) == 0)
        goto werr;

    /* Make sure data will not remain on the OS's output buffers */
    //调用fflush将输出缓冲区刷新到page cache(内核缓冲区)，然后调用fsync将cache中的内容写盘，最后关闭文件。
    if (fflush(fp) == EOF) goto werr;
    if (fsync(fileno(fp)) == -1) goto werr;
    if (fclose(fp) == EOF) goto werr;

    /* Use RENAME to make sure the DB file is changed atomically only
     * if the generate DB file is ok. */
    //将临时文件重命名(其实还是一个临时文件名，两个格式有所不同，最终的命名在后序处理中backgroundRewriteDoneHandler中)，确保生成的aof文件完全ok，避免出现aof不完整的情况
    if (rename(tmpfile,filename) == -1) {
        serverLog(LL_WARNING,"Error moving temp append only file on the final destination: %s", strerror(errno));
        unlink(tmpfile);
        return C_ERR;
    }
    serverLog(LL_NOTICE,"SYNC append only file rewrite performed");
    return C_OK;

werr:
    serverLog(LL_WARNING,"Write error writing append only file on disk: %s", strerror(errno));
    fclose(fp);
    unlink(tmpfile);
    return C_ERR;
}
```

借助RIO对所有的K-V对进行重写：

```c
int rewriteAppendOnlyFileRio(rio *aof) {
    dictIterator *di = NULL;
    dictEntry *de;
    size_t processed = 0;
    int j;

    //接下来是一个循环，用于遍历redis的每个db，对其进行rewirte
    for (j = 0; j < server.dbnum; j++) {
        //首先，生成对应db的select命令，然后查看如果db为空的话，就跳过，rewrite下一个db
        char selectcmd[] = "*2\r\n$6\r\nSELECT\r\n";
        redisDb *db = server.db+j;
        dict *d = db->dict;
        if (dictSize(d) == 0) continue;
        //然后获取该db的迭代器
        di = dictGetSafeIterator(d);

        /* SELECT the new DB */
        //将select db的命令写入文件
        if (rioWrite(aof,selectcmd,sizeof(selectcmd)-1) == 0) goto werr;
        if (rioWriteBulkLongLong(aof,j) == 0) goto werr;

        //遍历db的每一个key，生成相应的命令
        while((de = dictNext(di)) != NULL) {
            sds keystr;
            robj key, *o;
            long long expiretime;

            keystr = dictGetKey(de);
            o = dictGetVal(de);
            initStaticStringObject(key,keystr);

            expiretime = getExpire(db,&key);

            //redis3.0中做了一个判断 if (expiretime != -1 && expiretime < now) continue;
            //也就是重写会跳过已经过期的键，这里没有


            // 接下来，根据对象的类型，序列化成相应的命令。并将命令写入aof文件中
            if (o->type == OBJ_STRING) {
                /* Emit a SET command */
                char cmd[]="*3\r\n$3\r\nSET\r\n";
                if (rioWrite(aof,cmd,sizeof(cmd)-1) == 0) goto werr;
                /* Key and value */
                if (rioWriteBulkObject(aof,&key) == 0) goto werr;
                if (rioWriteBulkObject(aof,o) == 0) goto werr;
            } else if (o->type == OBJ_LIST) {
                if (rewriteListObject(aof,&key,o) == 0) goto werr;
            } else if (o->type == OBJ_SET) {
                if (rewriteSetObject(aof,&key,o) == 0) goto werr;
            } else if (o->type == OBJ_ZSET) {
                if (rewriteSortedSetObject(aof,&key,o) == 0) goto werr;
            } else if (o->type == OBJ_HASH) {
                if (rewriteHashObject(aof,&key,o) == 0) goto werr;
            } else if (o->type == OBJ_STREAM) {
                if (rewriteStreamObject(aof,&key,o) == 0) goto werr;
            } else if (o->type == OBJ_MODULE) {
                if (rewriteModuleObject(aof,&key,o) == 0) goto werr;
            } else {
                serverPanic("Unknown object type");
            }
            // 如果有超时时间，同样序列化成命令记录到aof文件
            if (expiretime != -1) {
                char cmd[]="*3\r\n$9\r\nPEXPIREAT\r\n";
                if (rioWrite(aof,cmd,sizeof(cmd)-1) == 0) goto werr;
                if (rioWriteBulkObject(aof,&key) == 0) goto werr;
                if (rioWriteBulkLongLong(aof,expiretime) == 0) goto werr;
            }
            /* Read some diff from the parent process from time to time. */
            if (aof->processed_bytes > processed+AOF_READ_DIFF_INTERVAL_BYTES) {
                processed = aof->processed_bytes;
                aofReadDiffFromParent();
            }
        }
        dictReleaseIterator(di);
        di = NULL;
    }
    return C_OK;

werr:
    if (di) dictReleaseIterator(di);

```



##### 后序处理/重写追加

多进程编程中，子进程退出后，父进程需要对其进行清理，否则子进程会编程僵尸进程。同样是在serverCron函数中，主进程完成对rewrite进程的清理。

下面这段代码应该不陌生，前面也多次出现了，在前面的RDB的serverCron处理中已经对RDB的后序处理backgroundSaveDoneHandler函数进行了简单说明，这里就接下来看看AOF的后序处理backgroundRewriteDoneHandler(aof.c)函数。

```c
// 检测bgsave、aof重写是否在执行过程中，或者是否有子线程
if (server.rdb_child_pid != -1 || server.aof_child_pid != -1 ||
        ldbPendingChildren())
    {
        int statloc;
        pid_t pid;

        //等待所有的子进程
        if ((pid = wait3(&statloc,WNOHANG,NULL)) != 0) {
            //取得子进程exit()返回的结束代码
            int exitcode = WEXITSTATUS(statloc);
            int bysignal = 0;

            //如果子进程是因为信号而结束则此宏值为真
            if (WIFSIGNALED(statloc)) bysignal = WTERMSIG(statloc);

            if (pid == -1) {
                //如果此时没有子线程
                serverLog(LL_WARNING,"wait3() returned an error: %s. "
                    "rdb_child_pid = %d, aof_child_pid = %d",
                    strerror(errno),
                    (int) server.rdb_child_pid,
                    (int) server.aof_child_pid);
            } else if (pid == server.rdb_child_pid) {
                //如果已经完成了bgsave,会调用backgroundSaveDoneHandler函数做最后处理
                backgroundSaveDoneHandler(exitcode,bysignal);
                if (!bysignal && exitcode == 0) receiveChildInfo();
            } else if (pid == server.aof_child_pid) {
                //如果已经完成了aof重写，会调用backgroundRewriteDoneHandler函数做最后处理
                backgroundRewriteDoneHandler(exitcode,bysignal);
                if (!bysignal && exitcode == 0) receiveChildInfo();
            } else {
                if (!ldbRemoveChild(pid)) {
                    serverLog(LL_WARNING,
                        "Warning, detected child with unmatched pid: %ld",
                        (long)pid);
                }
            }
            updateDictResizePolicy();
            closeChildInfoPipe();
        }
    } else {
    ...
```

在看具体的后序处理函数之前，这里先讲一个概念：

在linux中，删除文件时，会判断打开这个文件的所有进程是否都已经关闭，如果还有一个进程没有关闭，那么这个文件的空间将不会释放。只有所有打开这个文件的进程都关闭以后，这个文件的空间才会释放。因此在接下来的这个函数中，因为我们在initServer中或者前一个backgroundRewriteDoneHandler中已经打开了旧的aof文件，因此就算将新的临时aof文件重命名，表面上旧的aof文件已经删除了，但实际上，因为我们仍然保留着旧的aof文件描述符，这个文件还是打开着的，然而这个旧aof文件后序的情况其实对主程序来说没有任何影响，因此可以使用BIO后台任务异步的进行关闭，而减轻我们主线程的阻塞情况。

```c
void backgroundRewriteDoneHandler(int exitcode, int bysignal) {
    // 如果正常退出的情况下，就是没有被信号kill，并且退出码等于0
    if (!bysignal && exitcode == 0) {
        int newfd, oldfd;
        char tmpfile[256];
        long long now = ustime();
        mstime_t latency;

        //首先是记录日志
        serverLog(LL_NOTICE,
                  "Background AOF rewrite terminated with success");

        /* Flush the differences accumulated by the parent to the
         * rewritten AOF. */
        //然后打开临时写入的rewrite文件
        latencyStartMonitor(latency);
        snprintf(tmpfile,256,"temp-rewriteaof-bg-%d.aof",
                 (int)server.aof_child_pid);
        newfd = open(tmpfile,O_WRONLY|O_APPEND);
        if (newfd == -1) {
            serverLog(LL_WARNING,
                      "Unable to open the temporary AOF produced by the child: %s", strerror(errno));
            goto cleanup;
        }

        //将rewrite buf追加到文件
        if (aofRewriteBufferWrite(newfd) == -1) {
            serverLog(LL_WARNING,
                      "Error trying to flush the parent diff to the rewritten AOF: %s", strerror(errno));
            close(newfd);
            goto cleanup;
        }
        latencyEndMonitor(latency);
        latencyAddSampleIfNeeded("aof-rewrite-diff-write",latency);

        serverLog(LL_NOTICE,
                  "Residual parent diff successfully flushed to the rewritten AOF (%.2f MB)", (double) aofRewriteBufferSize() / (1024*1024));

        if (server.aof_fd == -1) {
            /* AOF disabled */

            /* Don't care if this fails: oldfd will be -1 and we handle that.
             * One notable case of -1 return is if the old file does
             * not exist. */
            oldfd = open(server.aof_filename,O_RDONLY|O_NONBLOCK);
        } else {
            /* AOF enabled */
            oldfd = -1; /* We'll set this to the current AOF filedes later. */
        }

        /* Rename the temporary file. This will not unlink the target file if
         * it exists, because we reference it with "oldfd". */
        latencyStartMonitor(latency);
        //将临时文件重命名为最终的aof文件,因为我们保留了旧文件的文件描述符，因此这个文件并没有关闭
        if (rename(tmpfile,server.aof_filename) == -1) {
            serverLog(LL_WARNING,
                      "Error trying to rename the temporary AOF file %s into %s: %s",
                      tmpfile,
                      server.aof_filename,
                      strerror(errno));
            close(newfd);
            if (oldfd != -1) close(oldfd);
            goto cleanup;
        }
        latencyEndMonitor(latency);
        latencyAddSampleIfNeeded("aof-rename",latency);

        if (server.aof_fd == -1) {
            /* AOF disabled, we don't need to set the AOF file descriptor
             * to this new file, so we can close it. */
            close(newfd);
        } else {
            //将server中旧的文件描述符替换成新的文件描述符
            oldfd = server.aof_fd;
            server.aof_fd = newfd;
            //进行一次fsync刷盘，因为前面又将rewrite buf中的数据写入文件了
            if (server.aof_fsync == AOF_FSYNC_ALWAYS)
                redis_fsync(newfd);
            else if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
                //创建一个aof刷盘的后台任务，由指定类型的BIO后台线程进行执行，类似于fsync，也是一个子线程进行执行，但并不一定立即执行，因为当前任务可能会等待
                aof_background_fsync(newfd);
            server.aof_selected_db = -1; /* Make sure SELECT is re-issued */
            aofUpdateCurrentSize();
            server.aof_rewrite_base_size = server.aof_current_size;
            server.aof_fsync_offset = server.aof_current_size;

            /* Clear regular AOF buffer since its contents was just written to
             * the new AOF from the background rewrite buffer. */
            sdsfree(server.aof_buf);
            server.aof_buf = sdsempty();
        }

        server.aof_lastbgrewrite_status = C_OK;

        serverLog(LL_NOTICE, "Background AOF rewrite finished successfully");
        /* Change state from WAIT_REWRITE to ON if needed */
        //更新状态，这里我看好像整个流程没有修改状态为AOF_WAIT_REWRITE，
        //只有在主从同步之后调用restartAOFAfterSYNC，从而调用startAppendOnly函数才进行了更新
        if (server.aof_state == AOF_WAIT_REWRITE)
            server.aof_state = AOF_ON;

        /* Asynchronously close the overwritten AOF. */
        //异步关闭之前的aof文件。创建一个关闭文件的后台任务，由指定类型的BIO后台线程进行执行。
        if (oldfd != -1) bioCreateBackgroundJob(BIO_CLOSE_FILE,(void*)(long)oldfd,NULL,NULL);

        serverLog(LL_VERBOSE,
                  "Background AOF rewrite signal handler took %lldus", ustime()-now);
    } else if (!bysignal && exitcode != 0) {
        //如果rewrite子进程异常退出，由信号kill或者退出码非0，则只是记录 日志。
        server.aof_lastbgrewrite_status = C_ERR;

        serverLog(LL_WARNING,
                  "Background AOF rewrite terminated with error");
    } else {
        /* SIGUSR1 is whitelisted, so we have a way to kill a child without
         * tirggering an error condition. */
        if (bysignal != SIGUSR1)
            server.aof_lastbgrewrite_status = C_ERR;

        serverLog(LL_WARNING,
                  "Background AOF rewrite terminated by signal %d", bysignal);
    }

    cleanup:
    aofClosePipes();
    aofRewriteBufferReset();
    aofRemoveTempFile(server.aof_child_pid);
    server.aof_child_pid = -1;
    server.aof_rewrite_time_last = time(NULL)-server.aof_rewrite_time_start;
    server.aof_rewrite_time_start = -1;
    /* Schedule a new rewrite if we are waiting for it to switch the AOF ON. */
    if (server.aof_state == AOF_WAIT_REWRITE)
        server.aof_rewrite_scheduled = 1;
}
```






