## 线程问题

###  Redis 单线程模型

**为什么说redis能够快速执行**

> (1) 绝大部分请求是纯粹的内存操作（非常快速）

> (2) 采用单线程,避免了不必要的上下文切换和竞争条件

> (3) 非阻塞IO - IO多路复用，Redis采用epoll做为I/O多路复用技术的实现，再加上Redis自身的事件处理模型将epoll中的连接，读写，关闭都转换为了时间，不在I/O上浪费过多的时间。

**Redis 基于 Reactor 模式来设计开发了自己的一套高效的事件处理模型** （Netty 的线程模型也基于 Reactor 模式，Reactor 模式不愧是高性能 IO 的基石），这套事件处理模型对应的是 Redis 中的文件事件处理器（file event handler）。

而说Redis是单线程，**主要是指文件事件处理器是单线程方式运行，也可以说是由一个主线程进行网络 IO 和键值对读写**。

**既然是单线程，那怎么监听大量的客户端连接呢？**

Redis 通过**IO 多路复用程序** 来监听来自客户端的大量连接（或者说是监听多个 socket），它会将感兴趣的事件及类型(读、写）注册到内核中并监听每个事件是否发生。

这样的好处非常明显： **I/O 多路复用技术的使用让 Redis 不需要额外创建多余的线程来监听客户端的大量连接，降低了资源的消耗**（和 NIO 中的 `Selector` 组件很像）。

接下来细说一下。

#### 为什么使用单线程设计

在学习操作系统的时候，又或者是平时学习时，我们都知道：**使用多线程，可以增加系统吞吐率，或是可以增加系统扩展性**。的确，对于一个多线程的系统来说，在有合理的资源分配的情况下，可以增加系统中处理请求操作的资源实体，进而提升系统能够同时处理的请求数，即吞吐率。下面的左图是我们采用多线程时所期待的结果。但是，请你注意，通常情况下，在我们采用多线程后，如果没有良好的系统设计，实际得到的结果，其实是右图所展示的那样。我们刚开始增加线程数时，系统吞吐率会增加，但是，再进一步增加线程时，系统吞吐率就增长迟缓了，有时甚至还会出现下降的情况。

<center><img src="assets/image-20220728184422791.png" width="80%"></center>

为什么会出现这种情况呢？一个关键的瓶颈在于，系统中通常会存在被多线程同时访问的共享资源，比如一个共享的数据结构。当有多个线程要修改这个共享资源时，为了保证共享资源的正确性，就需要有额外的机制进行保证，而这个额外的机制，就会带来额外的开销。

共享资源的安全访问一直是多线程开发中的一个难点问题，如果没有精细的设计，比如说，只是简单地采用一个粗粒度互斥锁，就会出现不理想的结果：即使增加了线程，大部分线程也在等待获取访问共享资源的互斥锁，并行变串行，系统吞吐率并没有随着线程的增加而增加。而且，采用多线程开发一般会引入同步原语来保护共享资源的并发访问，这也会降低系统代码的易调试性和可维护性。为了避免这些问题，Redis 直接采用了单线程模式。

#### 为什么redis的单线程很快？

通常来说，单线程的处理能力要比多线程差很多，但是 Redis 却能使用单线程模型达到每秒数十万级别的处理能力，这是为什么呢？其实，这是 Redis 多方面设计选择的一个综合结果。

- 一方面，**Redis 的大部分操作在内存上完成**，再加上它采用了高效的数据结构，例如哈希表和跳表，这是它实现高性能的一个重要原因；
- 另一方面，就是 **Redis 采用了多路复用机制**，使其在网络 IO 操作中能并发处理大量的客户端请求，实现高吞吐率。接下来，我们就重点学习下多路复用机制。



### redis是单线程吗？

这里可以直接说结论：**redis不是单线程的，而是一个主线程进行网络 IO 和键值对读写，另外有三个常驻BIO线程分别处理：关闭文件、异步AOF刷盘、懒惰释放。**除了这四个线程，在执行bgsave(rdbSaveBackground)和BGREWRITEAOF(rewriteAppendOnlyFileBackground)都会**fork**出一个线程进行相应的操作。特别在redis6.0中，默认还会创建128个IO线程进行读写。

其中三个常驻BIO线程是在server进行初始化的时候就创建了，具体的函数就是initServerLast和bio.h下的函数。

#### 初始化BIO线程

在main函数中，服务器初始化过程最后会调用InitServerLast函数，这个函数就两个作用：①调用bioInit()初始化三个常驻BIO线程 ；②设置服务器内存使用量

```c
void InitServerLast() {
    //初始化三个常驻BIO线程
    bioInit();
    //设置服务器内存使用量
    server.initial_memory_usage = zmalloc_used_memory();
}
```

bioInit 函数是在bio.c文件中实现的，它的主要作用就是初始化BIO线程使用的数据结构，以及调用 pthread_create 函数创建多个常驻BIO线程。

先看下相关的宏和数据结构：

```c
/* Background job opcodes */
#define BIO_CLOSE_FILE    0 /* Deferred close(2) syscall. */
#define BIO_AOF_FSYNC     1 /* Deferred AOF fsync. */
#define BIO_LAZY_FREE     2 /* Deferred objects freeing. */
#define BIO_NUM_OPS       3  //BIO线程数量，也就是任务类型数量
```

在 bio.h 文件中，你还可以看到四个宏定义，**BIO_NUM_OPS**是BIO线程数量，也就是后台BIO任务类型数量，而 BIO_CLOSE_FILE、BIO_AOF_FSYNC 和 BIO_LAZY_FREE，它们分别表示三种后台任务的操作码，这些操作码可以用来标识不同的任务：

- BIO_CLOSE_FILE：关闭文件任务；
- BIO_AOF_FSYNC：AOF新增数据刷盘任务；
- BIO_LAZY_FREE：惰性删除任务。

```c
// 保存线程的数组
static pthread_t bio_threads[BIO_NUM_OPS];
// 保存互斥锁的数组,针对bio_jobs的锁
static pthread_mutex_t bio_mutex[BIO_NUM_OPS];
// 保存条件变量的两个数组
//用于BIO线程等待任务：每创建一个任务(bioCreateBackgroundJob)则唤醒线程；在bioProcessBackgroundJobs中如果没有任务则等待
static pthread_cond_t bio_newjob_cond[BIO_NUM_OPS];
//用于其它线程等待BIO线程：当其他线程等待某任务执行完成，可以调用bioWaitStepOfType进行等待，在bioProcessBackgroundJobs中最后会取消阻塞在
static pthread_cond_t bio_step_cond[BIO_NUM_OPS];
//以后台线程方式运行的任务列表，每个数组元素是一个list
static list *bio_jobs[BIO_NUM_OPS];
//用来表示每种任务中，处于等待状态的任务个数。将该数组每个元素初始化为 0，其实就是表示初始时，每种任务都没有待处理的具体任务。
static unsigned long long bio_pending[BIO_NUM_OPS];

struct bio_job {
    time_t time; //任务创建时间
    void *arg1, *arg2, *arg3;  //任务参数，如果不止三个参数，可以通过指针指向一个结构体或类似
};
```

然后来看下bio.c#bioInit函数，主要就是干了三件事情：

1）初始化相关数据结构。bioInit 函数首先会初始化互斥锁数组和条件变量数组。然后，该函数会调用 listCreate 函数，给 bio_jobs 这个数组的每个元素创建一个列表，同时给 bio_pending 数组的每个元素赋值为 0。

2）设置线程属性和栈大小。bioInit 函数会调用 pthread_attr_init 函数，初始化线程属性变量 attr，然后调用 pthread_attr_getstacksize 函数，获取线程的栈大小这一属性的当前值，并根据当前栈大小和 REDIS_THREAD_STACK_SIZE 宏定义的大小（默认值为 4MB），来计算最终的栈大小属性值。紧接着，bioInit 函数会调用 pthread_attr_setstacksize 函数，来设置栈大小这一属性值。

3）为每种后台任务创建一个线程。调用pthread_create 函数来创建线程。

```c
int  pthread_create(pthread_t *tidp, const  pthread_attr_t *attr,
( void *)(*start_routine)( void *), void  *arg);
```

可以看到，pthread_create 函数一共有 4 个参数，分别是：

- \*tidp，指向线程数据结构pthread_t 的指针；
- \*attr，指向线程属性结构pthread_attr_t 的指针；
- \*start_routine，线程所要运行的函数的起始地址，也是指向函数的指针；
- \*arg，传给运行函数的参数。

```c
void bioInit(void) {
    pthread_attr_t attr;
    pthread_t thread;
    size_t stacksize;
    int j;

    /* Initialization of state vars and objects */
    for (j = 0; j < BIO_NUM_OPS; j++) {
        //首先初始化互斥锁数组和条件变量数组
        pthread_mutex_init(&bio_mutex[j],NULL);
        pthread_cond_init(&bio_newjob_cond[j],NULL);
        pthread_cond_init(&bio_step_cond[j],NULL);
        //调用 listCreate 函数，给 bio_jobs 这个数组的每个元素创建一个列表
        bio_jobs[j] = listCreate();
        //将 bio_pending 数组的每个元素赋值为 0
        bio_pending[j] = 0;
    }

    //初始化线程属性
    pthread_attr_init(&attr);
    //获取线程的栈大小这一属性的当前值，并根据当前栈大小和 REDIS_THREAD_STACK_SIZE 宏定义的大小（默认值为 4MB），来计算最终的栈大小属性值。
    pthread_attr_getstacksize(&attr,&stacksize);
    if (!stacksize) stacksize = 1; //针对Solaris系统做处理
    while (stacksize < REDIS_THREAD_STACK_SIZE) stacksize *= 2;
    //设置栈大小这一属性值
    pthread_attr_setstacksize(&attr, stacksize);

    //依次为每种后台任务创建一个线程。循环的次数是由 BIO_NUM_OPS 宏定义决定的，也就是 3 次。
    //相应的，bioInit 函数就会调用 3 次 pthread_create 函数，并创建 3 个线程。
    for (j = 0; j < BIO_NUM_OPS; j++) {
        void *arg = (void*)(unsigned long) j;
        //bioInit 函数让这 3 个线程执行的函数都是 bioProcessBackgroundJobs。
        //在这三次线程的创建过程中，传给这个函数的参数分别是 0、1、2，因为三种后台任务类型 BIO_CLOSE_FILE、BIO_AOF_FSYNC 和 BIO_LAZY_FREE 对应的操作码，它们的取值分别为 0、1、2。
        if (pthread_create(&thread,&attr,bioProcessBackgroundJobs,arg) != 0) {
            serverLog(LL_WARNING,"Fatal: Can't initialize Background Jobs.");
            exit(1);
        }
        bio_threads[j] = thread;
    }
}
```

这里是创建了 3 个线程，因为一共有三种后台任务，而每一个线程处理一种任务，处理函数就是bioProcessBackgroundJobs ，该函数接收的参数分别是 0、1、2 ，也就是对应的三种后台任务类型 BIO_CLOSE_FILE、BIO_AOF_FSYNC 和 BIO_LAZY_FREE 对应的操作码。



#### BIO线程处理函数

上面已经介绍了，每一个BIO后台线程对应的处理函数就是bioProcessBackgroundJobs，根据初始化时传入的参数不同，具体的操作会有些许不同。

主要的逻辑就是一个死循环，在这个循环中，线程首先会从bio_jobs中获取当前处理类型的任务，如果没有任务，则进入相应的条件变量中阻塞，直到有新的该类型任务到达，则被唤醒，在下一个循环中获取该类型任务集合中的第一个任务，并根据类型进行不同的处理：

- 任务类型是 BIO_CLOSE_FILE，则调用 close 函数；
- 任务类型是 BIO_AOF_FSYNC，则调用 redis_fsync 函数；
- 任务类型是 BIO_LAZY_FREE，则再根据参数个数等情况，分别调用 lazyfreeFreeObjectFromBioThread、lazyfreeFreeDatabaseFromBioThread 和 lazyfreeFreeSlotsMapFromBioThread 这三个函数。

具体代码如下：

```c
void *bioProcessBackgroundJobs(void *arg) {
    struct bio_job *job;
    //获取当前函数要处理的任务类型
    //三种后台任务类型 BIO_CLOSE_FILE、BIO_AOF_FSYNC 和 BIO_LAZY_FREE 对应的操作码，它们的取值分别为 0、1、2。
    unsigned long type = (unsigned long) arg;
    sigset_t sigset;

    /* Check that the type is within the right interval. */
    if (type >= BIO_NUM_OPS) {
        serverLog(LL_WARNING,
                  "Warning: bio thread started with wrong type %lu",type);
        return NULL;
    }

    /* Make the thread killable at any time, so that bioKillThreads()
     * can work reliably. */
    pthread_setcancelstate(PTHREAD_CANCEL_ENABLE, NULL);
    pthread_setcanceltype(PTHREAD_CANCEL_ASYNCHRONOUS, NULL);

    //获取锁
    pthread_mutex_lock(&bio_mutex[type]);
    /* Block SIGALRM so we are sure that only the main thread will
     * receive the watchdog signal. */
    sigemptyset(&sigset);
    sigaddset(&sigset, SIGALRM);
    if (pthread_sigmask(SIG_BLOCK, &sigset, NULL))
        serverLog(LL_WARNING,
                  "Warning: can't mask SIGALRM in bio.c thread: %s", strerror(errno));

    while(1) {
        listNode *ln;

        //如果没有当前线程要处理的任务，则根据对应的条件变量进行等待
        if (listLength(bio_jobs[type]) == 0) {
            //在pthread_cond_wait之前必须获取该共享数据的互斥锁，线程挂起时释放锁，并在满足条件离开条件变量时重新加锁
            pthread_cond_wait(&bio_newjob_cond[type],&bio_mutex[type]);
            //被唤醒了，说明有任务了，进入下一次循环
            continue;
        }
        //获取当前类型的任务集合中第一个任务
        ln = listFirst(bio_jobs[type]);
        job = ln->value;
        /* It is now possible to unlock the background system as we know have
         * a stand alone job structure to process.*/
        //释放锁
        pthread_mutex_unlock(&bio_mutex[type]);

        // 判断当前处理的后台任务类型是哪一种
        if (type == BIO_CLOSE_FILE) {
            //如果是关闭文件任务，那就调用close函数
            close((long)job->arg1);
        } else if (type == BIO_AOF_FSYNC) {
            //如果是AOF同步写任务，那就调用redis_fsync函数(其实就是fsync)
            redis_fsync((long)job->arg1);
        } else if (type == BIO_LAZY_FREE) {
            /*
             * 如果是惰性删除任务，那根据任务的参数分别调用不同的惰性删除函数执行，分别调用 lazyfreeFreeObjectFromBioThread、lazyfreeFreeDatabaseFromBioThread 和 lazyfreeFreeSlotsMapFromBioThread 这三个函数。
             */
            if (job->arg1)
                lazyfreeFreeObjectFromBioThread(job->arg1);
            else if (job->arg2 && job->arg3)
                lazyfreeFreeDatabaseFromBioThread(job->arg2,job->arg3);
            else if (job->arg3)
                lazyfreeFreeSlotsMapFromBioThread(job->arg3);
        } else {
            serverPanic("Wrong job type in bioProcessBackgroundJobs().");
        }
        zfree(job);

        /* Lock again before reiterating the loop, if there are no longer
         * jobs to process we'll block again in pthread_cond_wait(). */
        pthread_mutex_lock(&bio_mutex[type]);
        // 任务执行完成后，调用listDelNode在任务队列中删除该任务
        listDelNode(bio_jobs[type],ln);
        // 将对应的等待任务个数减一
        bio_pending[type]--;

        // 取消阻塞在 bioWaitStepOfType() 上阻塞线程（如果有）
        pthread_cond_broadcast(&bio_step_cond[type]);
    }
}
```



#### 创建后台任务

后台任务是由bio.c#bioCreateBackgroundJob函数进行创建的：

```c
//type表示后台任务类型，以及该任务的三个参数
void bioCreateBackgroundJob(int type, void *arg1, void *arg2, void *arg3) {
    struct bio_job *job = zmalloc(sizeof(*job));

    job->time = time(NULL);
    job->arg1 = arg1;
    job->arg2 = arg2;
    job->arg3 = arg3;
    //加个互斥锁
    pthread_mutex_lock(&bio_mutex[type]);
    //将任务加到bio_jobs数组的对应任务列表中
    listAddNodeTail(bio_jobs[type],job);
    // 将对应任务列表上等待处理的任务个数加1
    bio_pending[type]++;
    //唤醒等待线程
    pthread_cond_signal(&bio_newjob_cond[type]);
    pthread_mutex_unlock(&bio_mutex[type]);
}
```

注意每创建一个任务，都要对该任务的集合进行加锁和释放锁，并且唤醒条件变量中的等待线程，这样等待线程就能及时的获取该任务并进行处理。

我们来看三个相关例子吧，分别对应三种类型的BIO：

1）**AOF的刷盘**。我们知道，每一条命令，都会记录到aof buf中，当刷盘的时候根据不同的策略来进行处理的，首先是先讲aof buf中的所有数据写入到aof file中，此时aof file还是在内核缓冲区中，必须进行一次fsync操作才能完成刷盘。而这一步always和everysec策略又有所不同：

```c
try_fsync:
    // 如果配置了no-appendfsync-on-rewrite，即在有aof rewrite或者是rdb save的子进程时不进行fsync，
    // 主要是避免对磁盘产生过大压力，执行fsync会造成阻塞过长时间
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
            //创建一个aof刷盘的后台任务，由指定类型的BIO后台线程进行执行，类似于fsync，也是一个子线程进行执行，但并不一定立即执行，因为当前任务可能会等待
            aof_background_fsync(server.aof_fd);
            server.aof_fsync_offset = server.aof_current_size;
        }
        server.aof_last_fsync = server.unixtime;
    }
```

always策略是直接以主线程进行fsync，而everysec则是通过aof_background_fsync提交一个aof刷盘的后台任务：

```c
void aof_background_fsync(int fd) {
    //创建一个AOF Fsync后台任务
    bioCreateBackgroundJob(BIO_AOF_FSYNC,(void*)(long)fd,NULL,NULL);
}
```

2）**AOF重写的后序处理**。后序处理中有两处都创建了后台任务，分别是AOF刷盘任务和关闭文件任务。

```c
//进行一次fsync刷盘，因为前面又将rewrite buf中的数据写入文件了
if (server.aof_fsync == AOF_FSYNC_ALWAYS)
    redis_fsync(newfd);
else if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
    //创建一个aof刷盘的后台任务，由指定类型的BIO后台线程进行执行，类似于fsync，也是一个子线程进行执行，但并不一定立即执行，因为当前任务可能会等待
    aof_background_fsync(newfd);
```

刷盘任务是因为在重写的时候，客户端请求的新命令会保存到rewrite buf中，然后在backgroundRewriteDoneHandler函数中进行写入到新的AOF文件中，这最后要进行一次刷盘。

```c
//异步关闭旧的AOF文件。创建一个关闭文件的后台任务，由指定类型的BIO后台线程进行执行。
if (oldfd != -1) bioCreateBackgroundJob(BIO_CLOSE_FILE,(void*)(long)oldfd,NULL,NULL);
```

关闭文件任务是因为server已经使用新的AOF文件了，旧的文件描述符指向的旧AOF文件就要进行关闭了，然而这个旧aof文件后序的情况其实对主程序来说没有任何影响，因此可以使用BIO后台任务异步的进行关闭，而减轻我们主线程的阻塞情况。（注意一点情况，在操作系统中，删除文件时，会判断打开这个文件的所有进程是否都已经关闭，如果还有一个进程没有关闭，那么这个文件的空间将不会释放，因此新的AOF文件重命名覆盖旧的AOF文件时，其实并没有真正删除旧的AOF文件，通过文件描述符还是可以获取的）

3）**内存淘汰策略中选取出一个最佳淘汰key进行删除**。当根据不同的淘汰策略选举出一个最佳淘汰key后，就要进行删除，其中一步就是判断是否借助BIO线程进行惰性删除：

```c
// 是否开启lazyfree机制
// lazyfree的原理就是在删除对象时只是进行逻辑删除，然后把对象丢给后台，让后台线程去执行真正的destruct，避免由于对象体积过大而造成阻塞。
if (server.lazyfree_lazy_eviction)
    dbAsyncDelete(db,keyobj);
else
    dbSyncDelete(db,keyobj);
```

而在lazyfree.c#dbAsyncDelete中有一步就是创建BIO的惰性删除任务：

```c
if (free_effort > LAZYFREE_THRESHOLD && val->refcount == 1) {
    atomicIncr(lazyfree_objects,1);
    bioCreateBackgroundJob(BIO_LAZY_FREE,val,NULL,NULL);
    dictSetVal(db->dict,de,NULL);
}
```



> 上述涉及到的AOF、BIO源码可以去看对应章节处，可以更好的进行理解。



### 多IO线程

实际上，在redis6.0中，Redis 在执行模型中还进一步使用了多线程来处理IO任务，这样设计的目的，就是为了**充分利用当前服务器的多核特性，使用多核运行多线程，让多线程帮助加速数据读取、命令解析以及数据写回的速度，提升 Redis 整体性能**。

虽然，Redis6.0 引入了多线程，但是 Redis 的多线程只是在网络数据的读写这类耗时操作上使用了， 执行命令仍然是单线程顺序执行。因此，你也不需要担心线程安全问题。

Redis6.0 的多线程默认是禁用的，只使用主线程。如需开启需要修改 redis 配置文件 `redis.conf` ：

``` bash
io-threads-do-reads yes
```

开启多线程后，还需要设置线程数，否则是不生效的。同样需要修改 redis 配置文件 `redis.conf` :

``` bash
io-threads 4 #官网建议4核的机器建议设置为2或3个线程，8核的建议设置为6个线程
```



#### 多 IO 线程的初始化

我在上篇文章给你介绍过，Redis 中的三个后台线程，是 server 在初始化过程的最后，调用 InitSeverLast 函数，而 InitServerLast 函数再进一步调用 bioInit 函数来完成的。如果我们在 Redis 6.0 中查看 InitServerLast 函数，会发现和 Redis 5.0 相比，该函数在调完 bioInit 函数后，又调用了 initThreadedIO 函数。而 initThreadedIO 函数正是用来初始化多 IO 线程的，这部分的代码调用如下所示：

```c
void InitServerLast() {
    //初始化三个常驻BIO线程
    bioInit();
    //调用initThreadedIO函数初始化IO线程
    initThreadedIO();
    //设置后台jemalloc线程，进行内存管理
    set_jemalloc_bg_thread(server.jemalloc_bg_thread);
    //设置服务器内存使用量
    server.initial_memory_usage = zmalloc_used_memory();
}
```

所以下面，我们就来看下 initThreadedIO 函数的主要执行流程，这个函数是在**networking.c**文件中实现的。

先看下相关的宏和数据结构：

```c
#define IO_THREADS_MAX_NUM 128
#define IO_THREADS_OP_READ 0
#define IO_THREADS_OP_WRITE 1

// 记录线程描述符的数组
pthread_t io_threads[IO_THREADS_MAX_NUM];
// 记录线程互斥锁
pthread_mutex_t io_threads_mutex[IO_THREADS_MAX_NUM];
// 记录线程待处理的客户端个数
_Atomic unsigned long io_threads_pending[IO_THREADS_MAX_NUM];
// IO操作是读还是写
int io_threads_op;      
// 记录线程对应处理的客户端，每一个数组元素是一个list
list *io_threads_list[IO_THREADS_MAX_NUM];
```

这里注意个全局变量**io_threads_op**，这个变量是用来切换IO线程的读写操作，初看到这个变量的时候，我在疑惑，为什么仅仅设置一个变量，而不是数组？后来看了源码，才明白了我的两个误点，这里注意一下：

1. **所有的IO线程都是统一执行读或写事件,不会有部分线程执行读操作，部分执行写操作**，在handleClientsWithPendingReadsUsingThreads中执行读操作；在handleClientsWithPendingWritesUsingThreads中执行写操作。
2. **所有的IO线程虽然在后台运行，但真正的执行操作并不是完全异步的，IO线程可以并行的执行，但是在IO线程并行的处理IO操作的时候主线程不能继续往下执行，而是等待所有的IO线程执行完毕**。这点可以看 "分配客户端读/写推迟操作" 小节。

然后再来看initThreadedIO函数，这个函数就干了三件事情：

1）设置 IO 线程的激活标志。这个激活标志保存在 redisServer 结构体类型的全局变量 server 当中，对应 redisServer 结构体的成员变量 **io_threads_active**。initThreadedIO 函数会把 io_threads_active 初始化为 0，表示 IO 线程还没有被激活。

2）会对设置的 IO 线程数量进行判断。这个数量就是保存在全局变量 server 的成员变量 **io_threads_num** 中的。那么在这里，IO 线程的数量判断会有三种结果。

- 如果 IO 线程数量为 1，就表示只有 1 个主 IO 线程，initThreadedIO 函数就直接返回了。此时，Redis server 的 IO 线程和 Redis 6.0 之前的版本是相同的；
- 如果 IO 线程数量大于宏定义 IO_THREADS_MAX_NUM（默认值为 128），那么 initThreadedIO 函数会报错，并退出整个程序；
- 如果 IO 线程数量大于 1，并且小于宏定义 **IO_THREADS_MAX_NUM**，那么，initThreadedIO 函数会执行一个循环流程，该流程的循环次数就是设置的 IO 线程数量。

3）循环初始化相关数据结构，并调用 **pthread_create** 函数创建相应数量的线程，这个函数在BIO后台线程中讲过了。对于 initThreadedIO 函数来说，它创建的线程要运行的函数是 **IOThreadMain**，参数是当前创建线程的编号。不过要注意的是，这个编号是从 1 开始的，编号为 0 的线程其实是运行 Redis server 主流程的主 IO 线程。

```c
void initThreadedIO(void) {
    //设置 IO 线程的激活标志,0表示 IO 线程还没有被激活
    server.io_threads_active = 0;

    // 如果用户选择了单个线程，则不要产生任何线程：我们将直接从主线程处理 IO。
    //此时就和redis6.0之前一样了
    if (server.io_threads_num == 1) return;

    //如果 IO 线程数量大于宏定义 IO_THREADS_MAX_NUM（默认值为 128），那么 initThreadedIO 函数会报错，并退出整个程序。
    if (server.io_threads_num > IO_THREADS_MAX_NUM) {
        serverLog(LL_WARNING,"Fatal: too many I/O threads configured. "
                  "The maximum number is %d.", IO_THREADS_MAX_NUM);
        exit(1);
    }

    //如果 IO 线程数量大于 1，并且小于宏定义 IO_THREADS_MAX_NUM，那么，initThreadedIO 函数会执行一个循环流程，该流程的循环次数就是设置的 IO 线程数量。
    for (int i = 0; i < server.io_threads_num; i++) {
        //调用 listCreate 函数，给 io_threads_list 这个数组的每个元素创建一个列表
        io_threads_list[i] = listCreate();
        if (i == 0) continue; //线程0是主线程，要从1开始创建

        /* Things we do only for the additional threads. */
        pthread_t tid;
        //初始化互斥锁数组
        pthread_mutex_init(&io_threads_mutex[i],NULL);
        //将 io_threads_pending 数组的每个元素赋值为 0
        io_threads_pending[i] = 0;
        //获取当前要创建的线程的互斥锁
        pthread_mutex_lock(&io_threads_mutex[i]);
        //让每个线程执行的函数都是IOThreadMain
        //参数是当前创建线程的编号,不过要注意的是，这个编号是从 1 开始的，编号为 0 的线程其实是运行 Redis server 主流程的主 IO 线程。
        if (pthread_create(&tid,NULL,IOThreadMain,(void*)(long)i) != 0) {
            serverLog(LL_WARNING,"Fatal: Can't initialize IO thread.");
            exit(1);
        }
        //初始化io_threads数组，设置值为线程标识
        io_threads[i] = tid;
    }
}
```



#### IO线程处理函数

每一个IO线程的处理函数都是 IOThreadMain ，也是在 networking.c 文件中定义的，它的主要执行逻辑是一个 while(1) 循环。在这个循环中，IOThreadMain 函数会把 io_threads_list 数组中，每个 IO 线程对应的待处理客户端列表读取出来，并依次取出要处理的客户端，然后根据线程要执行的操作标记进行不同的处理：

- io_threads_op 的值为宏定义 **IO_THREADS_OP_WRITE**：这表明该 IO 线程要做的是写操作，线程会调用 writeToClient 函数将数据写回客户端；
- io_threads_op 的值为宏定义 **IO_THREADS_OP_READ**：这表明该 IO 线程要做的是读操作，线程会调用 readQueryFromClient 函数从客户端读取数据。

```c
void *IOThreadMain(void *myid) {
    /* The ID is the thread number (from 0 to server.iothreads_num-1), and is
     * used by the thread to just manipulate a single sub-array of clients. */
    long id = (unsigned long)myid;
    char thdname[16];

    snprintf(thdname, sizeof(thdname), "io_thd_%ld", id);
    redis_set_thread_title(thdname);
    redisSetCpuAffinity(server.server_cpulist);
    makeThreadKillable();

    while(1) {
        //自旋，循环1000000次判断是否有待处理的客户端
        for (int j = 0; j < 1000000; j++) {
            if (io_threads_pending[id] != 0) break;
        }

        if (io_threads_pending[id] == 0) {
            //在延迟写的客户端分配函数handleClientsWithPendingWritesUsingThreads中:
            //调用startThreadedIO会释放所有互斥锁，这里可以获取继续执行
            //调用stopThreadedIOIfNeeded会获取所有互斥锁，这里会阻塞
            pthread_mutex_lock(&io_threads_mutex[id]);
            pthread_mutex_unlock(&io_threads_mutex[id]);
            continue;
        }

        serverAssert(io_threads_pending[id] != 0);

        if (tio_debug) printf("[%ld] %d to handle\n", id, (int)listLength(io_threads_list[id]));

        /* Process: note that the main thread will never touch our list
         * before we drop the pending count to 0. */
        listIter li;
        listNode *ln;
        // 获取IO线程要处理的客户端列表
        listRewind(io_threads_list[id],&li);
        while((ln = listNext(&li))) {
            // 从客户端列表中获取一个客户端
            client *c = listNodeValue(ln);
            if (io_threads_op == IO_THREADS_OP_WRITE) {
                // 如果线程操作是写操作，则调用writeToClient将数据写回客户端
                writeToClient(c,0);
            } else if (io_threads_op == IO_THREADS_OP_READ) {
                // 如果线程操作是读操作，则调用readQueryFromClient从客户端读取数据
                readQueryFromClient(c->conn);
            } else {
                serverPanic("io_threads_op value is unknown");
            }
        }
        // 处理完所有客户端后，清空该线程的客户端列表
        listEmpty(io_threads_list[id]);
        // 将该线程的待处理任务数量设置为0
        io_threads_pending[id] = 0;

        if (tio_debug) printf("[%ld] Done\n", id);
    }
}
```



#### 获取/释放互斥锁

**获取/释放互斥锁是为了方便主线程动态，灵活调整IO线程而设计的**，**当clients数量较少的时候可以方便直接停止IO线程。停止IO线程的阈值是，当等待写的client客户端数量小于IO线程数量的两倍，就会停止IO线程避免多线程带来不必要的开销**。

```c
int stopThreadedIOIfNeeded(void) {
    int pending = listLength(server.clients_pending_write);

    /* Return ASAP if IO threads are disabled (single threaded mode). */
    if (server.io_threads_num == 1) return 1;

    //如果待写客户端数量是否小于 IO 线程数量的 2 倍,则关闭IO线程
    if (pending < (server.io_threads_num*2)) {
        if (server.io_threads_active) stopThreadedIO();
        return 1;
    } else {
        return 0;
    }
}

void startThreadedIO(void) {
    if (tio_debug) { printf("S"); fflush(stdout); }
    if (tio_debug) printf("--- STARTING THREADED IO ---\n");
    serverAssert(server.io_threads_active == 0);
    //释放所有互斥锁
    for (int j = 1; j < server.io_threads_num; j++)
        pthread_mutex_unlock(&io_threads_mutex[j]);
    //设置激活标志为1
    server.io_threads_active = 1;
}

void stopThreadedIO(void) {
    /* We may have still clients with pending reads when this function
     * is called: handle them before stopping the threads. */
    handleClientsWithPendingReadsUsingThreads();
    if (tio_debug) { printf("E"); fflush(stdout); }
    if (tio_debug) printf("--- STOPPING THREADED IO [R%d] [W%d] ---\n",
                          (int) listLength(server.clients_pending_read),
                          (int) listLength(server.clients_pending_write));
    serverAssert(server.io_threads_active == 1);
    //获取所有互斥锁
    for (int j = 1; j < server.io_threads_num; j++)
        pthread_mutex_lock(&io_threads_mutex[j]);
    //设置激活标志位0
    server.io_threads_active = 0;
}
```

stopThreadedIO，startThreadedIO 和 stopThreadedIOIfNeeded这三个函数中有体现，其中在stopThreadedIOIfNeeded中会判断当前待写出客户端数量是否大于2倍IO线程数量，如果不是则会调用stopThreadedIO函数通过io_threads_mutex的方式停止所有IO线程（主线程除外，因为index是从1开始的）并且将io_threads_active设置为0，并且后续调用stopThreadedIOIfNeeded函数会返回0，在handleClientsWithPendingWritesUsingThreads函数中会直接调用handleClientsWithPendingWrites来使用单线程进行写出。

在handleClientsWithPendingWritesUsingThreads中一开始就会进行判断，流程： handleClientsWithPendingWritesUsingThreads-> stopThreadedIOIfNeeded -> stopThreadedIO -> 设置io_threads_active为0，并lock住IO线程 -> 直接返回1 -> handleClientsWithPendingWrites进行单线程处理

当待写出client的数量上来的时候，stopThreadedIOIfNeeded函数中判断，待写出client数量大于2倍IO线程数量，返回0，然后调用startThreadedIO激活IO线程。

> 具体的可以联系 “分配客户端写推迟操作” 小节。





#### 添加客户端推迟操作

**看到这里可能也会产生一些疑问，IO 线程要处理的客户端是如何添加到 io_threads_list 数组中的呢？**

简单点来说就是：

server 变量中有两个 List 类型的成员变量：**clients_pending_write** 和 **clients_pending_read**，它们分别记录了待写回数据的客户端和待读取数据的客户端。

server 在接收到客户端请求和给客户端返回数据的过程中，会根据一定条件，推迟客户端的读写操作，并分别把待读写的客户端保存到这两个列表中。而在每次server的循环中会调用一次beforeSleep函数，从而将列表中的客户端添加到 io_threads_list 数组中，交给 IO 线程进行处理。

因此，接下来将从两方面来讲解，即：如何添加客户端推迟操作、如何分配客户端推迟操作。

##### 添加客户端读推迟操作

Redis server 在和一个客户端建立连接后，就会开始监听这个客户端上的可读事件(AE_READABLE file event)，而处理可读事件的处理函数是 **readQueryFromClient**。来看下在redis6.0中的该函数：

```c
void readQueryFromClient(connection *conn) {
    //从连接数据结构中获取客户端
    client *c = connGetPrivateData(conn);
    int nread, readlen;
    size_t qblen;

    /* Check if we want to read from the client later when exiting from
     * the event loop. This is the case if threaded I/O is enabled. */
    // 判断是否推迟从客户端读取数据
    if (postponeClientRead(c)) return;

    /* Update total number of reads on server */
    atomicIncr(server.stat_total_reads_processed, 1);

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

    /* There is more data in the client input buffer, continue parsing it
     * in case to check if there is a full command to execute. */
    processInputBuffer(c);
}
```

其实该函数和redis5.0大体相同，不同的大体就两点：

- 开头调用postponeClientRead判断是否推迟从客户端读取数据；
- 结尾调用processInputBuffer直接进行解析输入的数据，而不是调用processInputBufferAndReplicate如果是master服务器还要进行一次复制。

现在，我们就来看下 **postponeClientRead** 函数的执行逻辑。这个函数会根据四个条件判断能否推迟从客户端读取数据：

```c
int postponeClientRead(client *c) {
    if (server.io_threads_active &&
        server.io_threads_do_reads &&
        !clientsArePaused() &&
        !ProcessingEventsWhileBlocked &&
        !(c->flags & (CLIENT_MASTER|CLIENT_SLAVE|CLIENT_PENDING_READ)))
    {
        c->flags |= CLIENT_PENDING_READ; //设置推迟标志
        listAddNodeHead(server.clients_pending_read,c);  //加入clients_pending_read
        return 1;
    } else {
        return 0;
    }
}
```

1. **全局变量 server 的 io_threads_active 值为 1**。这表示多 IO 线程已经激活。我刚才说过，这个变量值在 initThreadedIO 函数中是会被初始化为 0 的，也就是说，多 IO 线程初始化后，默认还没有激活（我一会儿还会给你介绍这个变量值何时被设置为 1）；
2. **全局变量 server 的 io_threads_do_read 值为 1**。这表示多 IO 线程可以用于处理延后执行的客户端读操作。这个变量值是在 Redis 配置文件 redis.conf 中，通过配置项 io-threads-do-reads 设置的，默认值为 no，也就是说，多 IO 线程机制默认并不会用于客户端读操作。所以，如果你想用多 IO 线程处理客户端读操作，就需要把 io-threads-do-reads 配置项设为 yes；
3. **ProcessingEventsWhileBlocked 变量值为 0**。这表示 processEventsWhileBlokced 函数没有在执行。ProcessingEventsWhileBlocked 是一个全局变量，它会在 processEventsWhileBlokced 函数执行时被设置为 1，在 processEventsWhileBlokced 函数执行完成时被设置为 0。而 processEventsWhileBlokced 函数是在networking.c文件中实现的。当 Redis 在读取 RDB 文件或是 AOF 文件时，这个函数会被调用，用来处理事件驱动框架捕获到的事件。这样就避免了因读取 RDB 或 AOF 文件造成 Redis 阻塞，而无法及时处理事件的情况。所以，当 processEventsWhileBlokced 函数执行处理客户端可读事件时，这些客户端读操作是不会被推迟执行的；
4. **客户端现有标识不能有 CLIENT_MASTER、CLIENT_SLAVE 和 CLIENT_PENDING_READ**。其中，CLIENT_MASTER 和 CLIENT_SLAVE 标识分别表示客户端是用于主从复制的客户端，也就是说，这些客户端不会推迟读操作。CLIENT_PENDING_READ 本身就表示一个客户端已经被设置为推迟读操作了，所以，对于已带有 CLIENT_PENDING_READ 标识的客户端，postponeClientRead 函数就不会再推迟它的读操作了。总之，只有前面这四个条件都满足了，postponeClientRead 函数才会推迟当前客户端的读操作。具体来说，postponeClientRead 函数会给该客户端设置 CLIENT_PENDING_REA 标识，并调用 listAddNodeHead 函数，把这个客户端添加到全局变量 server 的 clients_pending_read 列表中。



##### 添加客户端写推迟操作

Redis 在执行了客户端命令，要给客户端返回结果时，会调用 addReply 函数判断是否要返回数据并将待返回结果写入客户端输出缓冲区：

```c
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
```

在函数一开始的时候，就会调用prepareClientToWrite函数判断是否需要返回数据：

```c
int prepareClientToWrite(client *c) {
    // 如果是 lua client 则直接OK
    if (c->flags & (CLIENT_LUA|CLIENT_MODULE)) return C_OK;

    //如果客户端设置了CLIENT_CLOSE_ASAP标识，不需要发送返回值
    if (c->flags & CLIENT_CLOSE_ASAP) return C_ERR;

     // 客户端发来过 REPLY OFF 或者 SKIP 命令，不需要发送返回值
    if (c->flags & (CLIENT_REPLY_OFF|CLIENT_REPLY_SKIP)) return C_ERR;

    // master 作为client 向 slave 发送命令，不需要接收返回值
    if ((c->flags & CLIENT_MASTER) &&
        !(c->flags & CLIENT_MASTER_FORCE_REPLY)) return C_ERR;

    // AOF loading 时的假client 不需要返回值
    if (!c->conn) return C_ERR; 

 
    //如果当前客户端没有待写回数据，调用clientInstallWriteHandler函数
    if (!clientHasPendingReplies(c) && !(c->flags & CLIENT_PENDING_READ))
        clientInstallWriteHandler(c);

    return C_OK;
}

```

这个函数会根据客户端设置的标识进行一系列的判断。其中，该函数会调用 clientHasPendingReplies 函数，判断当前客户端是否还有留存在输出缓冲区中的数据等待写回。

如果没有的话，那么，prepareClientToWrite 就会调用 clientInstallWriteHandler 函数，再进一步判断能否推迟该客户端写操作：

```c
void clientInstallWriteHandler(client *c) {
    /* Schedule the client to write the output buffers to the socket only
     * if not already done and, for slaves, if the slave can actually receive
     * writes at this stage. */
    /*
     * 1. 客户端没有设置过 CLIENT_PENDING_WRITE 标识，即没有被推迟过执行写操作；
     * 2. 客户端所在实例没有进行主从复制，或者客户端所在实例是主从复制中的从节点，但全量复制的 RDB 文件已经传输完成，客户端可以接收请求。
     */
    if (!(c->flags & CLIENT_PENDING_WRITE) &&
        (c->replstate == REPL_STATE_NONE ||
         (c->replstate == SLAVE_STATE_ONLINE && !c->repl_put_online_on_ack)))
    {
        /* Here instead of installing the write handler, we just flag the
         * client and put it into a list of clients that have something
         * to write to the socket. This way before re-entering the event
         * loop, we can try to directly write to the client sockets avoiding
         * a system call. We'll only really install the write handler if
         * we'll not be able to write the whole reply at once. */
        // 将客户端的标识设置为待写回，即CLIENT_PENDING_WRITE
        c->flags |= CLIENT_PENDING_WRITE;
        // 将客户端加入clients_pending_write列表
        listAddNodeHead(server.clients_pending_write,c);
    }
}
```

一般满足两个条件，clientInstallWriteHandler 函数就会把客户端标识设置为 CLIENT_PENDING_WRITE，表示推迟该客户端的写操作。同时，clientInstallWriteHandler 函数会把这个客户端添加到全局变量 server 的待写回客户端列表中，也就是 clients_pending_write 列表中。

> **其实上面的操作和redis5.0的操作是完全相同的，不同的地方在于redis5.0中是将clients_pending_write列表中的等待写操作在下一次循环时创建file event再进行返回给客户端，而redis6.0中是借助IO线程进行后台的返回。**





#### 分配客户端推迟操作

不过，当 Redis 使用 clients_pending_read 和 clients_pending_write 两个列表，保存了推迟执行的客户端后，这些客户端又是如何分配给多 IO 线程执行的呢？这就和下面两个函数相关了：

- **handleClientsWithPendingReadsUsingThreads** 函数：该函数主要负责将 clients_pending_read 列表中的客户端分配给 IO 线程进行处理；
- **handleClientsWithPendingWritesUsingThreads** 函数：该函数主要负责将 clients_pending_write 列表中的客户端分配给 IO 线程进行处理。

在 Redis 6.0 版本的代码中，事件驱动框架同样是调用 aeMain 函数来执行事件循环流程，该循环流程会调用 aeProcessEvents 函数处理各种事件。而**在 aeProcessEvents 函数实际调用 aeApiPoll 函数捕获 IO 事件之前，beforeSleep 函数会被调用**。

> 这里和redis5.0有些许不同，在redis5.0中，beforeSleep函数是在aeProcessEvents 调用之前进行调用的，而在redis6.0中是放在了计算最快发生的定时器事件的发生时间之后、调用 aeApiPoll 函数捕获 IO 事件之前。不同之处在于定时器时间的发生时间的变化，进而导致aeApiPoll 函数捕获 IO 事件 能阻塞的时间发生变化。

在beforeSleep函数中就会调用**handleClientsWithPendingReadsUsingThreads**和**handleClientsWithPendingWritesUsingThreads** 这两个函数：

```c
void beforeSleep(struct aeEventLoop *eventLoop) {
    ...
    handleClientsWithPendingReadsUsingThreads();
    ...
    handleClientsWithPendingWritesUsingThreads();
    ...
}
```



##### 分配客户端读推迟操作

**handleClientsWithPendingReadsUsingThreads** 函数主要负责将 clients_pending_read 列表中的客户端分配给 IO 线程进行处理。

这里注意一个很容易忽视的问题，**每一个IO线程做的事情仅仅是读取客户端的所有数据存储到queryBuf中，然后解析第一条命令。这里第一条命令并不会执行，并且后序的命令也不会进行解析！！**因为如果并发的执行命令，就会产生线程安全问题，而这明显不符合redis的设计原则！！！而控制函数的执行是靠CLIENT_PENDING_READ和CLIENT_PENDING_COMMAND这两个标识来实现的。

```c
int handleClientsWithPendingReadsUsingThreads(void) {
    //1.先进行一些判断：
    //根据全局变量 server 的 io_threads_active 成员变量，判定 IO 线程是否激活，
    // 并根据 server 的 io_threads_do_reads 成员变量，判定用户是否设置了 Redis 可以用 IO 线程处理待读客户端
    if (!server.io_threads_active || !server.io_threads_do_reads) return 0;
    //获取 clients_pending_read 列表的长度，这代表了要处理的待读客户端个数
    int processed = listLength(server.clients_pending_read);
    if (processed == 0) return 0;

    if (tio_debug) printf("%d TOTAL READ pending clients\n", processed);

    /* Distribute the clients across N different lists. */
    listIter li;
    listNode *ln;
    listRewind(server.clients_pending_read,&li);
    int item_id = 0;
    //2. 遍历每一个客户端，这里采用了一个类似RoundRobin的策略，就是所有客户端依次轮询分配给IO线程
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id],c);
        item_id++;
    }

    /* Give the start condition to the waiting threads, by setting the
     * start condition atomic var. */
    //将 IO 线程的操作标识设置为读操作
    io_threads_op = IO_THREADS_OP_READ;
    //设置每一个IO线程待处理的客户端数量
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]);
        io_threads_pending[j] = count;
    }

    /* Also use the main thread to process a slice of clients. */
    // 3. 使用主线程来处理一部分客户端
    //取出数组0元素中的待处理客户端，并调用readQueryFromClient进行处理
    //注意这里主线程仅仅只是读取客户端的所有数据到querybuf中，并解析出每一个客户端的第一个命令，而且并不能执行，在后续串行的进行执行
    listRewind(io_threads_list[0],&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        readQueryFromClient(c->conn);
    }
    // 处理完后，清空0号列表,也就是清空主线程待处理的客户端
    listEmpty(io_threads_list[0]);

    /* Wait for all the other threads to end their work. */
    //4. 等待所有 IO 线程完成待读客户端的处理
    //注意每一个IO线程仅仅只是读取客户端的所有数据到querybuf中，并解析出每一个客户端的第一个命令，而且并不能执行，在后续串行的进行执行。
    //命令并行的执行就会出现并发问题，而这显然不符合redis的设计理念
    while(1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += io_threads_pending[j];
        if (pending == 0) break;
    }
    if (tio_debug) printf("I/O READ All threads finshed\n");

    /* Run the list of clients again to process the new buffers. */
    // 5. 再次遍历一遍 clients_pending_read 列表，这是一个串行的执行
    // 这里主要是对每一个客户端解析出来的第一条命令进行执行，并继续解析第一条命令后的命令和执行
    while(listLength(server.clients_pending_read)) {
        //依次取出其中的客户端
        ln = listFirst(server.clients_pending_read);
        client *c = listNodeValue(ln);
        //取消到当前客户端的CLIENT_PENDING_READ标识，不然执行完第一条命令后在processInputBuffer中不能解析后序的命令并执行
        c->flags &= ~CLIENT_PENDING_READ;
        listDelNode(server.clients_pending_read,ln);
        /* Clients can become paused while executing the queued commands,
         * so we need to check in between each command. If a pause was
         * executed, we still remove the command and it will get picked up
         * later when clients are unpaused and we re-queue all clients. */
        // 客户端在执行排队的命令时可能会暂停，因此我们需要在每个命令之间进行检查。
        // 如果执行了暂停，我们仍然会删除该命令，并且稍后会在客户端取消暂停并重新排队时在将该客户端拾取。
        if (clientsArePaused()) continue;

        // 执行当前客户端解析出来的第一条命令
        if (processPendingCommandsAndResetClient(c) == C_ERR) {
            /* If the client is no longer valid, we avoid
             * processing the client later. So we just go
             * to the next. */
            continue;
        }

        //当前客户端可能出了一条命令外还有其它命令，此时客户端没有CLIENT_PENDING_READ和CLIENT_PENDING_COMMAND标识,
        //因此可以在一个循环中执行剩下的所有命令
        processInputBuffer(c);

        /* We may have pending replies if a thread readQueryFromClient() produced
         * replies and did not install a write handler (it can't).
         */
        // 如果线程 readQueryFromClient() 产生了回复并且没有去进行写处理（它不能），可以进行推迟
        if (!(c->flags & CLIENT_PENDING_WRITE) && clientHasPendingReplies(c))
            clientInstallWriteHandler(c);
    }

    /* Update processed count on server */
    server.stat_io_reads_processed += processed;

    return processed;
}
```

在handleClientsWithPendingReadsUsingThreads中一共就干了五件事情：

1）进行一些判断。判定 IO 线程是否激活，并且根据 server 的 io_threads_do_reads 成员变量，判定用户是否设置了 Redis 可以用 IO 线程处理待读客户端，只有在 IO 线程激活，并且 IO 线程可以用于处理待读客户端时，handleClientsWithPendingReadsUsingThreads 函数才会继续执行，否则该函数就直接结束返回了；获取 clients_pending_read列表的长度，这代表了要处理的待读客户端个数，如果没有待处理的客户端则执行返回。

2）分配客户端，并进行相应变量设置。遍历每一个客户端，采用了一个类似RoundRobin的策略，就是将所有客户端依次轮询分配给IO线程；将 IO 线程的操作标识io_threads_op设置为读操作，也就是IO_THREADS_OP_READ，然后，遍历 io_threads_list 数组中的每个元素列表长度，等待每个线程处理的客户端数量，赋值给 io_threads_pending 数组。

3）使用主线程来处理一部分客户端。取出数组0元素中的待处理客户端，并调用readQueryFromClient进行处理，注意这里主线程仅仅只是读取客户端的所有数据到querybuf中，并解析出每一个客户端的第一个命令，而且并不能执行，在后续串行的进行执行。

4）等待所有 IO 线程完成待读客户端的处理。注意每一个IO线程仅仅只是读取客户端的所有数据到querybuf中，并解析出每一个客户端的第一个命令，而且并不能执行，在后续串行的进行执行。命令并行的执行就会出现并发问题，而这显然不符合redis的设计理念。

5）再次遍历一遍 clients_pending_read 列表，对每一个客户端解析出来的第一条命令进行执行，并继续解析第一条命令后的命令并执行。这是一个串行操作。



上诉的代码如果不看更深一层的源代码可能会很懵逼，这里继续进行一个讲解，一定要牢记**CLIENT_PENDING_READ**和**CLIENT_PENDING_COMMAND**这两个标识！！！

1）首先，我们在postponeClientRead中知道一旦一个客户端的读任务要被推迟，那么一定会设置**CLIENT_PENDING_READ**标识；

2）在handleClientsWithPendingReadsUsingThreads中，不论是主线程还是IO线程都是调用readQueryFromClient函数进行操作，这个函数大体只干两件时间：①将客户端的数据全部读取到querybuf中；②调用processInputBuffer解析命令并执行；

3）重点就在processInputBuffer函数上。简单看下源码;

```c
void processInputBuffer(client *c) {
    /* Keep processing while there is something in the input buffer */
    while(c->qb_pos < sdslen(c->querybuf)) {
        //如果客户端暂停了则返回
        if (!(c->flags & CLIENT_SLAVE) && 
            !(c->flags & CLIENT_PENDING_READ) && 
            clientsArePaused()) break;

        /* Immediately abort if the client is in the middle of something. */
        //如果客户端正在处理某事，则立即中止
        if (c->flags & CLIENT_BLOCKED) break;

        /* Don't process more buffers from clients that have already pending
         * commands to execute in c->argv. */
        //如果客户端设置了CLIENT_PENDING_COMMAND,则说明此时已经有待命令要执行了，则立即中止
        if (c->flags & CLIENT_PENDING_COMMAND) break;

        ...  省略解析操作  ...

        //如果此时是推迟的客户端读操作，则不能简单的执行，要设置CLIENT_PENDING_COMMAND，推迟执行
        if (c->flags & CLIENT_PENDING_READ) {
            c->flags |= CLIENT_PENDING_COMMAND;
            break;
        }

        /* We are finally ready to execute the command. */
        if (processCommandAndResetClient(c) == C_ERR) {
            /* If the client is no longer valid, we avoid exiting this
                 * loop and trimming the client buffer later. So we return
                 * ASAP in that case. */
            return;
        }
    }
}
```

上诉列出来的就是关键部分，也是redis6.0和redis5.0中该函数不同的部分，而这不同就是针对多IO线程进行的修改。

当第一次进行该函数时，客户端并没有**CLIENT_PENDING_COMMAND**标识，会继续执行下去，当解析完第一条命令后、准备执行前，会判断当前客户端是否具体推迟写标识，即**CLIENT_PENDING_READ**，如果没有，则执行命令并进行下一次循环；如果有，则给客户端加上**CLIENT_PENDING_COMMAND**标识，表示推迟执行命令，并结束当前循环。

这里简单总结一下：

1. 如果客户端推迟了读操作，则在IO线程执行操作的时候必带有**CLIENT_PENDING_READ**和**CLIENT_PENDING_COMMAND**标识（只要有**CLIENT_PENDING_READ**标识就会带上**CLIENT_PENDING_COMMAND**标识），这样只能解析第一条命令；
2. 如果客户端没有推迟读操作，或者是在handleClientsWithPendingReadsUsingThreads中主线程执行完推迟客户端的第一条命令后调用processInputBuffer进行后续命令的解析和执行，则此刻客户端不带有**CLIENT_PENDING_READ**和**CLIENT_PENDING_COMMAND**标识，可以一直循环直到所有的命令执行完毕。

4）而主线程在执行后续命令的解析和执行的时候，会依次删除客户端的**CLIENT_PENDING_READ**和**CLIENT_PENDING_COMMAND**标识。

```c
while(listLength(server.clients_pending_read)) {
    //依次取出其中的客户端
    ln = listFirst(server.clients_pending_read);
    client *c = listNodeValue(ln);
    //取消到当前客户端的CLIENT_PENDING_READ标识，不然执行完第一条命令后在processInputBuffer中不能解析后序的命令并执行
    c->flags &= ~CLIENT_PENDING_READ;
    listDelNode(server.clients_pending_read,ln);
```

遍历每个客户端的时候，会删除客户端的CLIENT_PENDING_READ标识，没有这个标识，后续的processInputBuffer操作就不会给客户端加上**CLIENT_PENDING_COMMAND**标识。

```c
int processPendingCommandsAndResetClient(client *c) {
    //如果有 CLIENT_PENDING_COMMAND 标识，表明该客户端中的命令已经被某一个 IO 线程解析过，已经可以被执行了
    if (c->flags & CLIENT_PENDING_COMMAND) {
        //先取消掉当前客户端的CLIENT_PENDING_COMMAND标识，不然执行完当前命令后不能解析后序的命令并执行
        c->flags &= ~CLIENT_PENDING_COMMAND;
        if (processCommandAndResetClient(c) == C_ERR) {
            return C_ERR;
        }
    }
    return C_OK;
}
```

执行当前客户端的第一条命令之前，会取消客户端的**CLIENT_PENDING_COMMAND**标识。

而在取消了上诉的两个标识后，processInputBuffer就会一直循环解析和执行客户端的所有命令。





##### 分配客户端写推迟操作

和待读客户端的分配处理类似，待写客户端分配处理是由 **handleClientsWithPendingWritesUsingThreads** 函数来完成的，主要就是将 clients_pending_write 列表中的客户端分配给 IO 线程进行处理。该函数也是在 **beforeSleep** 函数中被调用的。

```c
int handleClientsWithPendingWritesUsingThreads(void) {
    int processed = listLength(server.clients_pending_write);
    if (processed == 0) return 0; /* Return ASAP if there are no clients. */

    /* If I/O threads are disabled or we have few clients to serve, don't
     * use I/O threads, but thejboring synchronous code. */
    //1. 如果当前只有一个线程或者可以停止使用IO线程(即待写客户端数量是否小于 IO 线程数量的 2 倍)则调用handleClientsWithPendingWrites函数进行处理，
    //就和redis5.0中相同的处理方式一样：1. 将缓冲值写入client的socket中，如果写完，则跳过之后的操作；2. 还有数据未写入，注册写事件，后序进行处理
    if (server.io_threads_num == 1 || stopThreadedIOIfNeeded()) {
        return handleClientsWithPendingWrites();
    }

    /* Start threads if needed. */
    //判断 IO 线程是否已激活。如果没有激活，它就会调用 startThreadedIO 函数，
    //把全局变量 server 的 io_threads_active 成员变量值设置为 1，表示 IO 线程已激活
    if (!server.io_threads_active) startThreadedIO();

    if (tio_debug) printf("%d TOTAL WRITE pending clients\n", processed);

    /* Distribute the clients across N different lists. */
    //2. 把待写客户端，按照轮询方式分配给 IO 线程，添加到 io_threads_list 数组各元素中
    // 并取消客户端的CLIENT_PENDING_WRITE标志，
    listIter li;
    listNode *ln;
    listRewind(server.clients_pending_write,&li);
    int item_id = 0;
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_WRITE;

        /* Remove clients from the list of pending writes since
         * they are going to be closed ASAP. */
        if (c->flags & CLIENT_CLOSE_ASAP) {
            listDelNode(server.clients_pending_write, ln);
            continue;
        }

        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id],c);
        item_id++;
    }

    /* Give the start condition to the waiting threads, by setting the
     * start condition atomic var. */
    //将 IO 线程的操作标识设置为写操作
    io_threads_op = IO_THREADS_OP_WRITE;
    //设置每一个IO线程待处理的客户端数量
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]);
        io_threads_pending[j] = count;
    }

    /* Also use the main thread to process a slice of clients. */
    //3. 使用主线程来处理一部分客户端
    listRewind(io_threads_list[0],&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        writeToClient(c,0);
    }
    listEmpty(io_threads_list[0]);

    /* Wait for all the other threads to end their work. */
    //4. 等待所有 IO 线程完成待读客户端的处理
    while(1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += io_threads_pending[j];
        if (pending == 0) break;
    }
    if (tio_debug) printf("I/O WRITE All threads finshed\n");

    /* Run the list of clients again to install the write handler where
     * needed. */
    // 5. 再次遍历一遍 clients_pending_read 列表
    listRewind(server.clients_pending_write,&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);

        /* Install the write handler if there are pending writes in some
         * of the clients. */
        //如果有客户端还有留存在缓冲区中的数据，那么，就调用 connSetWriteHandler 函数注册可写事件，而这个可写事件对应的回调函数是 sendReplyToClient 函数。
        if (clientHasPendingReplies(c) &&
            connSetWriteHandler(c->conn, sendReplyToClient) == AE_ERR)
        {
            freeClientAsync(c);
        }
    }
    listEmpty(server.clients_pending_write);

    /* Update processed count on server */
    server.stat_io_writes_processed += processed;

    return processed;
}
```

类似，在handleClientsWithPendingWritesUsingThreads中一共就干了五件事情(主要是第一步和第二步的不同)：

1）进行一些判断。如果当前只有一个线程或者可以停止使用IO线程(即**待写客户端数量是否小于 IO 线程数量的 2 倍**)则调用handleClientsWithPendingWrites函数进行处理，这样做的目的，主要是为了在待写客户端数量不多时，**避免采用多线程，从而节省 CPU 开销**；判断IO 线程是否已激活，如果没有激活，它就会调用 startThreadedIO 函数进行激活。

2）分配客户端，并进行相应变量设置。遍历每一个客户端，采用了一个类似RoundRobin的策略，就是将所有客户端依次轮询分配给IO线程，**并取消客户端的CLIENT_PENDING_WRITE标志**；将 IO 线程的操作标识io_threads_op设置为读操作，也就是IO_THREADS_OP_WRITE，然后，遍历 io_threads_list 数组中的每个元素列表长度，等待每个线程处理的客户端数量，赋值给 io_threads_pending 数组。

3）使用主线程来处理一部分客户端。取出数组0元素中的待处理客户端，并调用writeToClient将客户端缓冲区数据写给客户端。

4）等待所有 IO 线程完成待读客户端的处理。IO线程也是调用writeToClient进行处理。

5）再次遍历一遍 clients_pending_read 列表，如果有客户端还有留存在缓冲区中的数据，那么，就调用 connSetWriteHandler 函数注册可写事件，而这个可写事件对应的回调函数是 **sendReplyToClient** 函数：

```c
void sendReplyToClient(connection *conn) {
    client *c = connGetPrivateData(conn);
    writeToClient(c,1);
}
```

其实还是调用 **writeToClient** 函数，把客户端缓冲区中的数据写回。

这里，你需要注意的是，connSetWriteHandler 函数最终会映射为 connSocketSetWriteHandler 函数，而 connSocketSetWriteHandler 函数是在connection.c文件中实现的。connSocketSetWriteHandler 函数会调用 aeCreateFileEvent 函数创建 AE_WRITABLE 事件，这就是刚才介绍的可写事件的注册。


#### 小节

总结来说，Redis 6.0 先是在初始化过程中，根据用户设置的 IO 线程数量，创建对应数量的 IO 线程。当 Redis server 初始化完成后正常运行时，它会在 readQueryFromClient 函数中通过调用 postponeClientRead 函数来决定是否推迟客户端读操作。同时，Redis server 会在 addReply 函数中通过调用 prepareClientToWrite 函数，来决定是否推迟客户端写操作。而待读写的客户端会被分别加入到 clients_pending_read 和 clients_pending_write 两个列表中。这样，每当 Redis server 要进入事件循环流程前，都会在 beforeSleep 函数中分别调用 handleClientsWithPendingReadsUsingThreads 函数和 handleClientsWithPendingWritesUsingThreads 函数，将待读写客户端以轮询方式分配给 IO 线程，加入到 IO 线程的待处理客户端列表 io_threads_list 中。

而 IO 线程一旦运行后，本身会一直检测 io_threads_list 中的客户端，如果有待读写客户端，IO 线程就会调用 readQueryFromClient 或 writeToClient 函数来进行处理。最后，我也想再提醒你一下，多 IO 线程本身并不会执行命令，它们只是利用多核并行地读取数据和解析命令，或是将 server 数据写回（下节课我还会结合分布式锁的原子性保证，来给你介绍这一部分的源码实现。）。所以，Redis 执行命令的线程还是主 IO 线程。这一点对于你理解多 IO 线程机制很重要，可以避免你误解 Redis 有多线程同时执行命令。这样一来，我们原来针对 Redis 单个主 IO 线程做的优化仍然有效，比如避免 bigkey、避免阻塞操作等。

1. Redis 6.0 之前，处理客户端请求是单线程，这种模型的缺点是，只能用到「单核」CPU。如果并发量很高，那么在读写客户端数据时，容易引发性能瓶颈，所以 Redis 6.0 引入了多 IO 线程解决这个问题；
2. **所有的IO线程都是统一执行读或写事件,不会有部分线程执行读操作，部分执行写操作**，在handleClientsWithPendingReadsUsingThreads中执行读操作；在handleClientsWithPendingWritesUsingThreads中执行写操作。
3. **所有的IO线程虽然在后台运行，但真正的执行操作并不是完全异步的，IO线程可以并行的执行，但是在IO线程并行的处理IO操作的时候主线程不能继续往下执行，而是等待所有的IO线程执行完毕**。这点可以看 "分配客户端读/写推迟操作" 小节。
4. 配置文件开启 io-threads N 后，Redis Server 启动时，会启动 N - 1 个 IO 线程（主线程也算一个 IO 线程），这些 IO 线程执行的逻辑是 networking.c 的 IOThreadMain 函数。但默认只开启多线程「写」client socket，如果要开启多线程「读」，还需配置 io-threads-do-reads = yes；
5. Redis 在读取客户端请求时，判断如果开启了 IO 多线程，则把这个 client 放到 clients_pending_read 链表中（postponeClientRead 函数），之后主线程在处理每次事件循环之前，把链表数据轮询放到 IO 线程的链表（io_threads_list）中；
6. 同样地，在写回响应时，是把 client 放到 clients_pending_write 中（prepareClientToWrite 函数），执行事件循环之前把数据轮询放到 IO 线程的链表（io_threads_list）中；
7. 主线程把 client 分发到 IO 线程时，自己也会读写客户端 socket（主线程也要分担一部分读写操作），之后「等待」所有 IO 线程完成读写，再由主线程「串行」执行后续逻辑；
8. 每个 IO 线程，不停地从 io_threads_list 链表中取出 client，并根据指定类型读、写 client socket；
9. IO 线程在处理读、写 client 时有些许差异，如果 write_client_pedding < io_threads * 2，则直接由「主线程」负责写，不再交给 IO 线程处理，从而节省 CPU 消耗；
10. Redis 官方建议，服务器最少 4 核 CPU 才建议开启 IO 多线程，4 核 CPU 建议开 2-3 个 IO 线程，8 核 CPU 开 6 个 IO 线程，超过 8 个线程性能提升不大；
11. Redis 官方表示，开启多 IO 线程后，性能可提升 1 倍。当然，如果 Redis 性能足够用，没必要开 IO 线程