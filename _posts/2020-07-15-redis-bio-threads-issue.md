---
layout: blog
title: Redis의 Bio thread에서 오류 발생 시 정상 종료되지 않는 이슈
author: Sungho Kim
published: true
category: tech-review
excerpt: |
  Redis에서 사용하는 bio thread의 동작에 대해 소개하고, bio thread에서 오류 발생 시 정상적으로 프로세스가 종료되지 않는 이슈에 대해 소개합니다.
---

## Bio thread of Redis

Bio threads는 main thread에서 처리하는 기능 외 특정 목적을 가지고 지속적으로 job을 생성 및 수행하기 위한 thread를 관리하는 기능입니다.

``` console
 * The design is trivial, we have a structure representing a job to perform
 * and a different thread and job queue for every job type.
 * Every thread waits for new jobs in its queue, and process every job
 * sequentially.
 *
 * Jobs of the same type are guaranteed to be processed from the least
 * recently inserted to the most recently inserted (older jobs processed
 * first).
 *
 * Currently there is no way for the creator of the job to be notified about
 * the completion of the operation, this will only be added when/if needed.
```

현재 Redis에는 아래와 같이 3개의 bio thread를 사용하고 있으며, 목적에 맞게 추가해서 사용할 수 있습니다.

``` console
/* Background job opcodes */
#define BIO_CLOSE_FILE    0 /* Deferred close(2) syscall. */
#define BIO_AOF_FSYNC     1 /* Deferred AOF fsync. */
#define BIO_LAZY_FREE     2 /* Deferred objects freeing. */
#define BIO_NUM_OPS       3
```

각 thread들은 생성 후 지속적으로 'pthread_cond_wait'를 사용하여 수행해야 하는 job을 확인하여 처리를 하고, 수행해야 하는 job이 없으면 대기를 합니다.

``` c
void *bioProcessBackgroundJobs(void *arg) {
    ...
    ...
    pthread_mutex_lock(&bio_mutex[type]);
    ...
    ...

    while(1) {
        listNode *ln;

        /* The loop always starts with the lock hold. */
        if (listLength(bio_jobs[type]) == 0) {
            pthread_cond_wait(&bio_newjob_cond[type],&bio_mutex[type]);
            continue;
        }
        /* Pop the job from the queue. */
        ln = listFirst(bio_jobs[type]);
        job = ln->value;
        /* It is now possible to unlock the background system as we know have
         * a stand alone job structure to process.*/
        pthread_mutex_unlock(&bio_mutex[type]);

        /* Process the job accordingly to its type. */
        if (type == BIO_CLOSE_FILE) {
            close((long)job->arg1);
        } else if (type == BIO_AOF_FSYNC) {
            redis_fsync((long)job->arg1);
        } else if (type == BIO_LAZY_FREE) {
            /* What we free changes depending on what arguments are set:
             * arg1 -> free the object at pointer.
             * arg2 & arg3 -> free two dictionaries (a Redis DB).
             * only arg3 -> free the skiplist. */
            if (job->arg1)
                lazyfreeFreeObjectFromBioThread(job->arg1);
            else if (job->arg2 && job->arg3)
                lazyfreeFreeDatabaseFromBioThread(job->arg2,job->arg3);
            else if (job->arg3)
                lazyfreeFreeSlotsMapFromBioThread(job->arg3);
        } else {
            serverPanic("Wrong job type in bioProcessBackgroundJobs().");
        }
        ...
```

## Issue

이 bio thread 중 하나에 오류가 발생하여 특정 기능이 정상적으로 동작하지 못함에도 불구하고 redis-server process 자체는 종료되지 않아 failover가 정상 작동하지 않는 문제가 발생할 수 있습니다.

아래는 AOF_FSYNC 중 강제로 오류를 발생시키는 코드를 추가하여 확인한 것입니다.

``` c
void *bioProcessBackgroundJobs(void *arg) {
    ...
    ...
    pthread_mutex_lock(&bio_mutex[type]);
    ...
    ...

    while(1) {
        listNode *ln;

        /* The loop always starts with the lock hold. */
        if (listLength(bio_jobs[type]) == 0) {
            pthread_cond_wait(&bio_newjob_cond[type],&bio_mutex[type]);
            continue;
        }
        /* Pop the job from the queue. */
        ln = listFirst(bio_jobs[type]);
        job = ln->value;
        /* It is now possible to unlock the background system as we know have
         * a stand alone job structure to process.*/
        pthread_mutex_unlock(&bio_mutex[type]);

        /* Process the job accordingly to its type. */
        if (type == BIO_CLOSE_FILE) {
            close((long)job->arg1);
        } else if (type == BIO_AOF_FSYNC) {
            redis_fsync((long)job->arg1);
            asset(false);               // 강제로 오류 발생시킴.
        } else if (type == BIO_LAZY_FREE) {
        ...
```

GDB로 오류 발생 후 상황을 보면 아래와 같습니다.

문제가 된 AOF_FSYNC thread는 종료가 되었지만, main thread와 다른 thread는 정상적으로 동작을 할 수 없음에도 불구하고 계속 살아있으면서 대기하고 있는 것을 알 수 있습니다.

이러한 이유로 정상적으로 failover가 수행되지 않고 있습니다.

``` bash 
(gdb) info threads
  Id   Target Id         Frame
  12   Thread 0x7f3dfe1ff700 (LWP 134497) "rocksdb:bg0" 0x00007f3e061e8a35 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
  11   Thread 0x7f3dfd9fe700 (LWP 134498) "rocksdb:bg1" 0x00007f3e061e8a35 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
  10   Thread 0x7f3dfd1fd700 (LWP 134501) "rocksdb:bg2" 0x00007f3e061e8a35 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
  9    Thread 0x7f3dfc5ff700 (LWP 134504) "rocksdb:bg3" 0x00007f3e061e8a35 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
  8    Thread 0x7f3dfbdfe700 (LWP 134505) "rocksdb:bg4" 0x00007f3e061e8a35 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
  7    Thread 0x7f3dfb1ff700 (LWP 134506) "rocksdb:bg5" 0x00007f3e061e8a35 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
  6    Thread 0x7f3df9fff700 (LWP 134507) "rocksdb:bg0" 0x00007f3e061e8a35 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
  5    Thread 0x7f3df97fe700 (LWP 134508) "rocksdb:bg1" 0x00007f3e061e8a35 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
  4    Thread 0x7f3df8dfd700 (LWP 134509) "rocksdb:bg2" 0x00007f3e061e8a35 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
  3    Thread 0x7f3deb71b700 (LWP 134516) "redis-server" 0x00007f3e061e8a35 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
  2    Thread 0x7f3deaf1a700 (LWP 134517) "redis-server" 0x00007f3e061e8a35 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
* 1    Thread 0x7f3e07b569c0 (LWP 134406) "redis-server" 0x00007f3e05f0deb3 in epoll_wait () from /lib64/libc.so.6

(gdb) t 1
[Switching to thread 1 (Thread 0x7f3e07b569c0 (LWP 134406))]
#0  0x00007f3e05f0deb3 in epoll_wait () from /lib64/libc.so.6
(gdb) bt
#0  0x00007f3e05f0deb3 in epoll_wait () from /lib64/libc.so.6
#1  0x000000000043c56e in aeApiPoll (tvp=<optimized out>, eventLoop=<optimized out>) at ae.c:403
#2  aeProcessEvents (eventLoop=eventLoop@entry=0x7f3e05a2bc40, flags=flags@entry=3) at ae.c:412
#3  0x000000000043c96b in aeMain (eventLoop=0x7f3e05a2bc40) at ae.c:487
#4  0x0000000000430a71 in main (argc=<optimized out>, argv=0x7ffd6d0c4fa8) at redis.c:5700

(gdb) t 2
[Switching to thread 2 (Thread 0x7f3deaf1a700 (LWP 134517))]
#0  0x00007f3e061e8a35 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
(gdb) bt
#0  0x00007f3e061e8a35 in pthread_cond_wait@@GLIBC_2.3.2 () from /lib64/libpthread.so.0
#1  0x0000000000486ec5 in bioProcessBackgroundJobs (arg=<optimized out>) at bio.c:236
#2  0x00007f3e061e4ea5 in start_thread () from /lib64/libpthread.so.0
#3  0x00007f3e05f0d8dd in clone () from /lib64/libc.so.6

(gdb) p bio_mutex[0]
$1 = {__data = {__lock = 2, __count = 0, __owner = 134514, __nusers = 1, __kind = 0, __spins = 0, __elision = 0, __list = {__prev = 0x0, __next = 0x0}},
  __size = "\002\000\000\000\000\000\000\000r\r\002\000\001", '\000' <repeats 26 times>, __align = 2}
(gdb) p bio_mutex[1]
$2 = {__data = {__lock = 0, __count = 0, __owner = 0, __nusers = 0, __kind = 0, __spins = 0, __elision = 0, __list = {__prev = 0x0, __next = 0x0}}, __size = '\000' <repeats 39 times>,
  __align = 0}
```

## Root cause

오류 발생 시 sigsegvHandler()에서 process 종료를 수행하는데, 이 과정 중에 memtest_test_linux_anonymous_maps()를 통해 process 종료 전 memory crash 여부를 확인하는 부분이 있습니다.

이 memory crash 여부를 확인할 때 false alarm을 방지하기 위해 bio thread들을 정리하는 bioKillTrheads()를 먼저 호출합니다.

``` c
void sigsegvHandler(int sig, siginfo_t *info, void *secret) {
...

#if defined(HAVE_PROC_MAPS)
    /* Test memory */
    serverLogRaw(LL_WARNING|LL_RAW, "\n------ FAST MEMORY TEST ------\n");
    bioKillThreads();
    if (memtest_test_linux_anonymous_maps()) {
        serverLogRaw(LL_WARNING|LL_RAW,
            "!!! MEMORY ERROR DETECTED! Check your memory ASAP !!!\n");
    } else {
        serverLogRaw(LL_WARNING|LL_RAW,
            "Fast memory test PASSED, however your memory can still be broken. Please run a memory test for several hours if possible.\n");
    }
#endif
...

    serverLogRaw(LL_WARNING|LL_RAW,
"\n=== REDIS BUG REPORT END. Make sure to include from START to END. ===\n\n"
"       Please report the crash by opening an issue on github:\n\n"
"           http://github.com/antirez/redis/issues\n\n"
"  Suspect RAM error? Use redis-server --test-memory to verify it.\n\n"
);

    /* free(messages); Don't call free() with possibly corrupted memory. */
    if (server.daemonize && server.supervised == 0) unlink(server.pidfile);

    /* Make sure we exit with the right signal at the end. So for instance
     * the core will be dumped if enabled. */
    sigemptyset (&act.sa_mask);
    act.sa_flags = SA_NODEFER | SA_ONSTACK | SA_RESETHAND;
    act.sa_handler = SIG_DFL;
    sigaction (sig, &act, NULL);
    kill(getpid(),sig);
}
```

아래 코드를 보면 순차적으로 bio thread들을 종료시키는데, main thread에서 호출한 경우에는 정상적으로 모두 종료시키고 'kill(getpid(), sig)'를 호출하여 프로세스를 종료시키지만, 

bio thread들 중 하나에서 sigsegvHandler()를 타고 들어온 경우에 이 bioKillThreads()를 호출하면 self thread도 종료시키게 됩니다.

참고로 pthread_cancel()을 사용하면 종료를 요청하게 하여 cancelation point까지는 동작을 수행하게 되는데, 이후 이어서 pthread_join()을 호출함으로써 바로 cancelation point에 도달하고 바로 종료를 하게 된 것입니다.

``` c
/* Kill the running bio threads in an unclean way. This function should be
 * used only when it's critical to stop the threads for some reason.
 * Currently Redis does this only on crash (for instance on SIGSEGV) in order
 * to perform a fast memory check without other threads messing with memory. */
void bioKillThreads(void) {
    int err, j;

    for (j = 0; j < BIO_NUM_OPS; j++) {
        if (bio_threads[j] && pthread_cancel(bio_threads[j]) == 0) {
            if ((err = pthread_join(bio_threads[j],NULL)) != 0) {
                serverLog(LL_WARNING,
                    "Bio thread for job type #%d can be joined: %s",
                        j, strerror(err));
            } else {
                serverLog(LL_WARNING,
                    "Bio thread for job type #%d terminated",j);
            }
        }
    }
}
```

이로 인해 해당 thread는 종료되어 프로세스를 종료시키지는 못하게 된 것입니다.

## Solution

따라서 아래와 같이 main thread가 아닌 경우, self thread는 종료시키지 않도록 수정하여 정상적으로 프로세스가 죽고, failover가 수행되도록 하였습니다.

``` c
/* Kill the running bio threads in an unclean way. This function should be
 * used only when it's critical to stop the threads for some reason.
 * Currently Redis does this only on crash (for instance on SIGSEGV) in order
 * to perform a fast memory check without other threads messing with memory. */
void bioKillThreads(void) {
    int err, j;

    uint64_t tid = pthread_self();
    for (j = 0; j < REDIS_BIO_NUM_OPS; j++) {
        if (bio_threads[j] && tid != bio_threads[j] && pthread_cancel(bio_threads[j]) == 0) {
            if ((err = pthread_join(bio_threads[j],NULL)) != 0) {
                redisLog(REDIS_WARNING,
                    "Bio thread for job type #%d can be joined: %s",
                        j, strerror(err));
            } else {
                redisLog(REDIS_WARNING,
                    "Bio thread for job type #%d terminated",j);
            }
        }
    }
}
```

