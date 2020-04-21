相关代码
```
framework/native/cmds/servicemanager/
  - service_manager.c
  - binder.c
  
kernel/drivers/staging/android/binder.c
```

ServiceManager由init进程解析init.rc时启动，是Binder IPC通信过程中的守护进程，本身也是一个Binder服务，但并没有采用libbinder中的多线程模型来与Binder驱动通信，而是自行编写了binder.c直接和Binder驱动来通信，且只有一个循环binder_loop来进行读取和处理事务，好处是简单而高效。ServiceManager本身工作就是查询和注册服务。

## ServiceManager的启动

Service Manager的入口位于service_manager.c文件中的main函数：
```
int main(int argc, char** argv)
{
    struct binder_state *bs;
    union selinux_callback cb;
    char *driver;

    if (argc > 1) {
        driver = argv[1];
    } else {
        driver = "/dev/binder";
    }
    //打开binder驱动，申请128k的内存空间
    bs = binder_open(driver, 128*1024);
    if (!bs) {
#ifdef VENDORSERVICEMANAGER
        ALOGW("failed to open binder driver %s\n", driver);
        while (true) {
            sleep(UINT_MAX);
        }
#else
        ALOGE("failed to open binder driver %s\n", driver);
#endif
        return -1;
    }
    //成为binder机制的上下文管理者
    if (binder_become_context_manager(bs)) {
        ALOGE("cannot become context manager (%s)\n", strerror(errno));
        return -1;
    }

    cb.func_audit = audit_callback;
    selinux_set_callback(SELINUX_CB_AUDIT, cb);
    cb.func_log = selinux_log_callback;
    selinux_set_callback(SELINUX_CB_LOG, cb);

#ifdef VENDORSERVICEMANAGER
    sehandle = selinux_android_vendor_service_context_handle();
#else
    sehandle = selinux_android_service_context_handle();
#endif
    selinux_status_open(true);

    if (sehandle == NULL) {
        ALOGE("SELinux: Failed to acquire sehandle. Aborting.\n");
        abort();//无法获取sehandle
    }

    if (getcon(&service_manager_context) != 0) {
        ALOGE("SELinux: Failed to acquire service_manager context. Aborting.\n");
        abort();//无法获取service_manager上下文
    }

    /进入循环，处理client端发来的请求
    binder_loop(bs, svcmgr_handler);

    return 0;
}
```
main函数中主要做了以下工作：
* 打开Binder设备，做好初始化
* 设置为Binder上下文管理者
* 进入循环，处理事务

#### binder_open
/frameworks/native/cmds/servicemanager/binder.c
```
struct binder_state *binder_open(const char* driver, size_t mapsize)
{
    struct binder_state *bs;//记录SM中关于Binder的所有信息，如fd、map大小等
    struct binder_version vers;

    bs = malloc(sizeof(*bs));
    if (!bs) {
        errno = ENOMEM;
        return NULL;
    }

    bs->fd = open(driver, O_RDWR | O_CLOEXEC);//打开binder驱动
    ...
    bs->mapsize = mapsize;
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    if (bs->mapped == MAP_FAILED) {
        fprintf(stderr,"binder: cannot map device (%s)\n",
                strerror(errno));
        goto fail_map;
    }

    return bs;

fail_map:
    close(bs->fd);
fail_open:
    free(bs);
    return NULL;
}
```
使用open打开binder驱动，然后使用mmap，由mmap参数可知：
* 由Binder驱动决定被映射到进程空间的内存起始地址
* 映射区大小为128KB，只读
* 映射区的改变是私有的，不需要改变文件
* 从文件的起始地址开始映射

binder_become_context_manager只是简单地使用ioctl发送BINDER_SET_CONTEXT_MGR命令给Binder驱动

以上准备工作就绪，则在binder_loop中处理事务。
#### binder_loop
servicemanager/binder.c
```
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr;//执行BINDER_WRITE_READ命令所需地数据格式
    uint32_t readbuf[32];//一次读取容量

    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;

    readbuf[0] = BC_ENTER_LOOPER;//循环命令
    binder_write(bs, readbuf, sizeof(uint32_t));
```
`binder_write`通过ioctl系统调用把BC_ENTER_LOOPER指令发送到Binder驱动，就会调用到binder驱动的binder_ioctl函数
```
int binder_write(struct binder_state *bs, void *data, size_t len)
{
    struct binder_write_read bwr;
    int res;

    bwr.write_size = len;
    bwr.write_consumed = 0;
    bwr.write_buffer = (uintptr_t) data;
    bwr.read_size = 0;
    bwr.read_consumed = 0;
    bwr.read_buffer = 0;
    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
    if (res < 0) {
        fprintf(stderr,"binder_write: ioctl failed (%s)\n",
                strerror(errno));
    }
    return res;
}
```

在Binder Server进入循环前，将BC_ENTER_LOOPER通过binder_write发送给Binder驱动，告知Binder驱动这个情况。

```
    for (;;) {
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;

        //这里只设置了read，所以到binder_ioctl中只会执行binder_thread_read中的wait_event_freezable_exclusive，
        //会进入睡眠，等待客户端写入数据，然后再唤醒SM
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);

        if (res < 0) {
            ALOGE("binder_loop: ioctl failed (%s)\n", strerror(errno));
            break;
        }
        //从iocrl返回后，也就是从Binder驱动中接收到客户端发送的消息，然后开始解析读取的消息
        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        if (res == 0) {
            ALOGE("binder_loop: unexpected reply?!\n");
            break;
        }
        if (res < 0) {
            ALOGE("binder_loop: io error %d %s\n", res, strerror(errno));
            break;
        }
    }
}
```
SM进入循环，执行以下流程：
1. 从Binder驱动中读消息。通过ioctl发送BINDER_WRITE_READ，此命令可读可写，取决于bwr.read_size和bwr.write_size，这里write_size为0，read_size为sizeof(readbuf)，所以只从Binder驱动中读消息，在`binder_thread_read`中调用`wait_event_freezable_exclusive`进入睡眠，等待客户端写入数据后唤醒。
2. 唤醒后，在`binder_thread_read`中继续执行，读到数据后，使用binder_parse函数处理消息。
3. 发生错误则退出循环

#### binder_parse
```
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
    int r = 1;
    uintptr_t end = ptr + (uintptr_t) size;

    while (ptr < end) {
        uint32_t cmd = *(uint32_t *) ptr;//取出cmd
        ptr += sizeof(uint32_t);//ptr可能有多个命令，取得这个命令后增加指针大小，while循环的下一次就取得ptr的下一个命令
#if TRACE
        fprintf(stderr,"%s:\n", cmd_name(cmd));
#endif
```
计算到ptr的终点end，然后进入while循环（执行完命令）。
```
        switch(cmd) {
        case BR_NOOP:
            break;
        case BR_TRANSACTION_COMPLETE:
            break;
        case BR_INCREFS:
        case BR_ACQUIRE:
        case BR_RELEASE:
        case BR_DECREFS:
#if TRACE
            fprintf(stderr,"  %p, %p\n", (void *)ptr, (void *)(ptr + sizeof(void *)));
#endif
            ptr += sizeof(struct binder_ptr_cookie);
            break;
```
BR_NOOP和BR_TRANSACTION_COMPLETE以及操作引用计数的命令不需要特别处理。
```
        case BR_TRANSACTION: {//收到客户端的唤醒以及接收到数据后，处理BR_TRANSACTION
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            if ((end - ptr) < sizeof(*txn)) {
                ALOGE("parse: txn too small!\n");
                return -1;
            }
            binder_dump_txn(txn);
            if (func) { //这个fun就是传入的svcmgr_handler函数
                unsigned rdata[256/4];
                struct binder_io msg;//保存Binder驱动发来的消息
                struct binder_io reply;//保存回复给Binder驱动的消息
                int res;

                bio_init(&reply, rdata, sizeof(rdata), 4);//初始化reply
                bio_init_from_txn(&msg, txn);
                res = func(bs, txn, &msg, &reply);//具体处理消息
                if (txn->flags & TF_ONE_WAY) {
                    binder_free_buffer(bs, txn->data.ptr.buffer);
                } else {
                    binder_send_reply(bs, &reply, txn->data.ptr.buffer, res);//将reply发给Binder驱动
                }
            }
            ptr += sizeof(*txn);//处理完成，移动指针
            break;
        }
```
对BR_TRANSACTION命令的处理住要由func，也就是svcmgr_handler函数完成的，然后将结果返回给Binder驱动。

binder_io类型是Service Manager内部用于存储binder object的数据类型

##### svcmgr_handler
```
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    struct svcinfo *si;
    uint16_t *s;
    size_t len;
    uint32_t handle;
    uint32_t strict_policy;
    int allow_isolated;
    uint32_t dumpsys_priority;

    //ALOGI("target=%p code=%d pid=%d uid=%d\n",
    //      (void*) txn->target.ptr, txn->code, txn->sender_pid, txn->sender_euid);

    if (txn->target.ptr != BINDER_SERVICE_MANAGER)//SM的句柄为0
        return -1;

    if (txn->code == PING_TRANSACTION)
        return 0;

    // Equivalent to Parcel::enforceInterface(), reading the RPC
    // header with the strict mode policy mask and the interface name.
    // Note that we ignore the strict_policy and don't propagate it
    // further (since we do no outbound RPCs anyway).
    strict_policy = bio_get_uint32(msg);
    s = bio_get_string16(msg, &len);
    if (s == NULL) {
        return -1;
    }
    ...

    switch(txn->code) {
    case SVC_MGR_GET_SERVICE:
    case SVC_MGR_CHECK_SERVICE:
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        //根据名称查找相应服务。SM中维护了一个全局的svclist变量，保存了全部Server的注册信息。返回server句柄
        handle = do_find_service(s, len, txn->sender_euid, txn->sender_pid);
        if (!handle)
            break;
        //将查到的server句柄存入参数reply中，reply.type为BINDER_TYPE_HANDLE
        bio_put_ref(reply, handle);
        return 0;

    case SVC_MGR_ADD_SERVICE:
        s = bio_get_string16(msg, &len);
        if (s == NULL) {
            return -1;
        }
        //客户端添加服务，会执行到Binder驱动中的bind_transaction的case BINDER_TYPE_BINDER，
        //其中的binder_get_ref_for_node就会为客户端分配好handle，Binder驱动传给SM后，在这里取出
        //（通过句柄可以在Binder驱动中当前进程对应的binder_proc->refs_by_desc中找到对应server的binder_ref，
        //从binder_ref可以找到对应的binder_node，binder_node中存储了cookie，就是server中的BpBinder的地址）
        handle = bio_get_ref(msg);
        allow_isolated = bio_get_uint32(msg) ? 1 : 0;
        dumpsys_priority = bio_get_uint32(msg);
        //注册指定服务
        if (do_add_service(bs, s, len, handle, txn->sender_euid, allow_isolated, dumpsys_priority,
                           txn->sender_pid))
            return -1;
        break;

    case SVC_MGR_LIST_SERVICES: {
        uint32_t n = bio_get_uint32(msg);
        uint32_t req_dumpsys_priority = bio_get_uint32(msg);

        if (!svc_can_list(txn->sender_pid, txn->sender_euid)) {
            ALOGE("list_service() uid=%d - PERMISSION DENIED\n",
                    txn->sender_euid);
            return -1;
        }
        si = svclist;
        // walk through the list of services n times skipping services that
        // do not support the requested priority
        while (si) {
            if (si->dumpsys_priority & req_dumpsys_priority) {
                if (n == 0) break;
                n--;
            }
            si = si->next;
        }
        if (si) {
            bio_put_string16(reply, si->name);
            return 0;
        }
        return -1;
    }
    default:
        ALOGE("unknown code %d\n", txn->code);
        return -1;
    }

    bio_put_uint32(reply, 0);
    return 0;
}
```
可以看到`svcmgr_handler`函数包含了`Service Manager`的关键：注册服务和查询服务。SM内部维护着一个`svclist`列表，用于存储所有Server相关信息，数据结果为`svcinfo`。

`svcmgr_handler`函数处理完后，`binder_parse`会将结果通过`binder_send_reply`来将执行结果回复给Binder驱动，进而传递给客户端。然后继续循环处理，直到ptr < end为false，此时表示SM从Binder驱动读到的这一次数据已处理完，然后回到 `binder_write`函数中的for循环，再次通过`ioctl`传递`BINDER_WRITE_READ`来读取新消息，如果没有就休眠等待。

##### binder_send_reply
```
void binder_send_reply(struct binder_state *bs,
                       struct binder_io *reply,
                       binder_uintptr_t buffer_to_free,
                       int status)
{
    struct {
        uint32_t cmd_free;
        binder_uintptr_t buffer;
        uint32_t cmd_reply;
        struct binder_transaction_data txn;
    } __attribute__((packed)) data;

    data.cmd_free = BC_FREE_BUFFER;
    data.buffer = buffer_to_free;
    data.cmd_reply = BC_REPLY;//传给Binder的命令类型，表示回复
    data.txn.target.ptr = 0;
    data.txn.cookie = 0;
    data.txn.code = 0;
    if (status) {
        ...status表示svcmgr_handler的错误值
    } else {//无异常情况
        data.txn.flags = 0;
        data.txn.data_size = reply->data - reply->data0;
        data.txn.offsets_size = ((char*) reply->offs) - ((char*) reply->offs0);
        data.txn.data.ptr.buffer = (uintptr_t)reply->data0;
        data.txn.data.ptr.offsets = (uintptr_t)reply->offs0;
    }
    binder_write(bs, &data, sizeof(data));//写入Binder驱动，内部将数据转换为bwr，只写入，然后调用ioctl
}
```
收到BR_TRANSACTION命令，开始处理，找到server的句柄值，然后在这里传给Binder驱动。Binder驱动中就可以


`BR_TRANSACTION`命令的处理流程分析完后，下面是`binder_parse`函数中的`BR_REPLY`：
```
        case BR_REPLY: {
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            if ((end - ptr) < sizeof(*txn)) {
                ALOGE("parse: reply too small!\n");
                return -1;
            }
            binder_dump_txn(txn);
            if (bio) {
                bio_init_from_txn(bio, txn);
                bio = 0;
            } else {
                /* todo FREE BUFFER */
            }
            ptr += sizeof(*txn);
            r = 0;
            break;
        }
        case BR_DEAD_BINDER: {
            struct binder_death *death = (struct binder_death *)(uintptr_t) *(binder_uintptr_t *)ptr;
            ptr += sizeof(binder_uintptr_t);
            death->func(bs, death->ptr);
            break;
        }
        case BR_FAILED_REPLY:
            r = -1;
            break;
        case BR_DEAD_REPLY:
            r = -1;
            break;
        default:
            ALOGE("parse: OOPS %d\n", cmd);
            return -1;
        }
    }

    return r;
}
```
`BR_REPLY`中并没有实质的动作，因为SM主要是作为服务端，所以关键在于`BR_TRANSACTION`。


do_add_service函数 注册服务
```
int do_add_service(struct binder_state *bs, const uint16_t *s, size_t len, uint32_t handle,
                   uid_t uid, int allow_isolated, uint32_t dumpsys_priority, pid_t spid) {
    struct svcinfo *si;

    //ALOGI("add_service('%s',%x,%s) uid=%d\n", str8(s, len), handle,
    //        allow_isolated ? "allow_isolated" : "!allow_isolated", uid);

    if (!handle || (len == 0) || (len > 127))
        return -1;
    //检查是否有注册service的权限
    if (!svc_can_register(s, len, spid, uid)) {
        ALOGE("add_service('%s',%x) uid=%d - PERMISSION DENIED\n",
             str8(s, len), handle, uid);
        return -1;
    }
    //根据服务名在链表svclist上查找服务是否已经注册
    si = find_svc(s, len);
    if (si) {
        if (si->handle) {
            ALOGE("add_service('%s',%x) uid=%d - ALREADY REGISTERED, OVERRIDE\n",
                 str8(s, len), handle, uid);
            svcinfo_death(bs, si);//如果已注册，则释放
        }
        si->handle = handle;
    } else {
        //如果服务没有注册，就分配一个结构svcinfo并添加到链表svclist中
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        if (!si) {//内存不足，无法分配足够内存
            ALOGE("add_service('%s',%x) uid=%d - OUT OF MEMORY\n",
                 str8(s, len), handle, uid);
            return -1;
        }
        si->handle = handle;
        si->len = len;
        //内存拷贝服务名称
        memcpy(si->name, s, (len + 1) * sizeof(uint16_t));
        si->name[len] = '\0';
        //设置death监听函数
        si->death.func = (void*) svcinfo_death;
        si->death.ptr = si;
        si->allow_isolated = allow_isolated;
        si->dumpsys_priority = dumpsys_priority;
        si->next = svclist;//保存到列表
        svclist = si;
    }
    //增加binder的应用计数
    binder_acquire(bs, handle);
    //注册服务退出的监听
    binder_link_to_death(bs, handle, &si->death);
    return 0;
}
```

do_find_service 查询服务

```
uint32_t do_find_service(const uint16_t *s, size_t len, uid_t uid, pid_t spid){
    //查询相应的服务
    struct svcinfo *si = find_svc(s, len);

    if (!si || !si->handle) {
        return 0;
    }

    if (!si->allow_isolated) {
        // If this service doesn't allow access from isolated processes,
        // then check the uid to see if it is isolated.
        uid_t appid = uid % AID_USER;
        if (appid >= AID_ISOLATED_START && appid <= AID_ISOLATED_END) {
            return 0;
        }
    }

    if (!svc_can_find(s, len, spid, uid)) {
        return 0;
    }

    return si->handle;
}
```