---
title: 网络编程一些常见问题总结
date: 2017-08-04 17:19:14
categories: 技术开发
tags: [网络编程]
---
## 1 设置网络socket非阻塞：
``` cpp
u_long has = 1;
ioctl(m_sock, FIONBIO , &has);
```
这个函数很有可能返回success，却并没有设置成功。
windows对此有优化，对于linux版本应采用fcntl设置。

总结如下：
``` cpp
int
make_socket_nonblocking(sockfd fd)
{
#ifdef WIN32
    {
        u_long nonblocking = 1;
        if (ioctlsocket(fd, FIONBIO, &nonblocking) == SOCKET_ERROR) {
            cout << "fcntl failed, fd is : " << fd; 
            
            return -1;
        }
    }
#else
    {
        int flags;
        if ((flags = fcntl(fd, F_GETFL, NULL)) < 0) {
            cout << "fcntl failed, fd is : " << fd;
            return -1;
        }
        if (fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1) {
            cout << "fcntl failed, fd is : " << fd;
            return -1;
        }
    }
#endif
    return 0;
}
```
<!--more-->
## 2 windows环境下查看错误
``` cpp
使用WSAGetLastError函数需要配置
 
lib,"ws2_32.lib"
```
## 3 EPOLLET这个宏是最小int

EPOLLET这个宏的数值为-2147483648， 是能表示的最小int值。

## 4 make: 警告：检测到时钟错误。您的创建可能是不完整的。

可以通过ls -l查看具体的是哪个文件的时间错了，就可以对症下药了，直接 " touch 对应文件 " 就可以解决这个问题。

或者读者可以用 " touch * " 来更新整个项目文件的时间,这也可以解决问题。

## 5 select fd_set 对于不同平台实现是不同的

在windows平台实现
``` cpp
typedef struct fd_set {
        u_int fd_count;               /* how many are SET? */
        SOCKET  fd_array[FD_SETSIZE];   /* an array of SOCKETs */
} fd_set;
```

很明了，一个计数的fd_count，另一个就是SOCKET数组。其中，FD_SETSIZE是可以设置的。整个fd_set的过程实际上就是将对应的
fd_count作为数组下标，数组元素存储的是对应socket fd。比如说当前读事件集合readset的fd_count 为7，当要监控socket fd为5 的读事件到来时，那么readset这个集合中下标为8的数组元素为5，fd_count  = 8以此类推。当调用select时，会返回对应读，写集合所有的描述符数组，并且重置内部的fd_count数量，然后分别调用读写函数即可。

下面是fd_set在linux下的实现：
``` cpp
typedef struct
  {
    /* XPG4.2 requires this member name.  Otherwise avoid the name
       from the global namespace.  */
#ifdef __USE_XOPEN
    __fd_mask fds_bits[__FD_SETSIZE / __NFDBITS];
# define __FDS_BITS(set) ((set)->fds_bits)
#else 
    __fd_mask __fds_bits[__FD_SETSIZE / __NFDBITS];
# define __FDS_BITS(set) ((set)->__fds_bits)
#endif
  } fd_set;
```
根据UNIX网络编程对fd_set的介绍，fd_set是个整数数组，用每个bit位来表示fd的。比如，一个32位整数，则数组第一个整数表示0-31的fd，以此类推，第二个整数表示32-63
查看linux的FD_SET、FD_CLR是用汇编实现的。根据说明可以知道，就是给bit置位。
fd_set在不同平台实现的机制不一样，select第一个参数在linux环境下表示最大描述符数+1。windows无意义。

下面是我根据libevent早期版本实现的一套select模型：
``` cpp
#include "modelmanager.h"


#ifdef WIN32
#include "netmodeldef.h"
#define XFREE(ptr) do { if (ptr) free(ptr); } while (0)


struct win_fd_set {
    u_int fd_count;
    SOCKET fd_array[1];
};

struct win32op {
    unsigned num_fds_in_fd_sets;
    int resize_out_sets;
    struct win_fd_set *readset_in;
    struct win_fd_set *writeset_in;
    struct win_fd_set *readset_out;
    struct win_fd_set *writeset_out;
    struct win_fd_set *exset_out;
    unsigned signals_are_broken : 1;
};

static void *win32_init(void *);
static int win32_add(void *, sockfd, short old, short events, void *_idx);
static int win32_del(void *, sockfd, short old, short events, void *_idx);
static int win32_dispatch(void *base, struct timeval *);
static void win32_dealloc(void *);

struct ModelOp win32ops = {
    "win32",
    win32_init,
    win32_add,
    win32_del,
    win32_dispatch,
    win32_dealloc,
};

#define FD_SET_ALLOC_SIZE(n) ((sizeof(struct win_fd_set) + ((n)-1)*sizeof(SOCKET)))

static int
grow_fd_sets(struct win32op *op, unsigned new_num_fds)
{
    size_t size;

    if( !(new_num_fds >= op->readset_in->fd_count &&
        new_num_fds >= op->writeset_in->fd_count) )
        return -1;
    if( !(new_num_fds >= 1) )
        return -1;

    size = FD_SET_ALLOC_SIZE(new_num_fds);
    if (!(op->readset_in = (struct win_fd_set *)realloc(op->readset_in, size)))
        return (-1);
    if (!(op->writeset_in = (struct win_fd_set *)realloc(op->writeset_in, size)))
        return (-1);
    op->resize_out_sets = 1;
    op->num_fds_in_fd_sets = new_num_fds;
    return (0);
}

static int
do_fd_set(struct win32op *op, struct SocketIndex *ent, SOCKET s, int read)
{
    struct win_fd_set *set = read ? op->readset_in : op->writeset_in;
    
    if (read) {
        if (ent->read_pos_plus1 > 0)
            return (0);
    } else {
        if (ent->write_pos_plus1 > 0)
            return (0);
    }

    if (set->fd_count == op->num_fds_in_fd_sets) {
        if (grow_fd_sets(op, op->num_fds_in_fd_sets*2))
            return (-1);
        // set pointer will have changed and needs reiniting!
        set = read ? op->readset_in : op->writeset_in;
    }
    set->fd_array[set->fd_count] = s;
    if (read)
        ent->read_pos_plus1 = set->fd_count+1;
    else
        ent->write_pos_plus1 = set->fd_count+1;
    return (set->fd_count++);
}

static int
do_fd_clear(void *base,
struct win32op *op, struct SocketIndex *ent, int read)
{
    ModelManager* pDispatcher = (ModelManager*)base;

    int i;
    struct win_fd_set *set = read ? op->readset_in : op->writeset_in;
    if (read) {
        i = ent->read_pos_plus1 - 1;
        ent->read_pos_plus1 = 0;
    } else {
        i = ent->write_pos_plus1 - 1;
        ent->write_pos_plus1 = 0;
    }
    if (i < 0)
        return (0);
    if (--set->fd_count != (unsigned)i) {
        struct SocketIndex *ent2;
        SOCKET s2;
        s2 = set->fd_array[i] = set->fd_array[set->fd_count];

        ent2 = pDispatcher->getSocketIndex( s2 );

        if (!ent2) // This indicates a bug.
            return (0);
        if (read)
            ent2->read_pos_plus1 = i+1;
        else
            ent2->write_pos_plus1 = i+1;
    }
    return (0);
}

#define NEVENT 32
void *
win32_init(void *base)
{
    struct win32op *winop;
    size_t size;
    if (!(winop = (struct win32op*)malloc( sizeof(struct win32op))))
        return NULL;
    winop->num_fds_in_fd_sets = NEVENT;
    size = FD_SET_ALLOC_SIZE(NEVENT);
    if (!(winop->readset_in = (struct win_fd_set *)malloc(size)))
        goto err;
    if (!(winop->writeset_in = (struct win_fd_set *)malloc(size)))
        goto err;
    if (!(winop->readset_out = (struct win_fd_set *)malloc(size)))
        goto err;
    if (!(winop->writeset_out = (struct win_fd_set *)malloc(size)))
        goto err;
    if (!(winop->exset_out = (struct win_fd_set *)malloc(size)))
        goto err;
    winop->readset_in->fd_count = winop->writeset_in->fd_count = 0;
    winop->readset_out->fd_count = winop->writeset_out->fd_count
        = winop->exset_out->fd_count = 0;

    winop->resize_out_sets = 0;

    return (winop);
err:
    XFREE(winop->readset_in);
    XFREE(winop->writeset_in);
    XFREE(winop->readset_out);
    XFREE(winop->writeset_out);
    XFREE(winop->exset_out);
    XFREE(winop);
    return (NULL);
}

int
win32_add(void *base, SOCKET fd,
          short old, short events, void *_idx)
{
    ModelManager* pDispatcher = (ModelManager*)base;
    struct win32op *winop = (struct win32op *)pDispatcher->getModelData();
    struct SocketIndex *idx = (struct SocketIndex *)_idx;

    if (!(events & (EV_READ|EV_WRITE)))
        return (0);

    //event_debug(("%s: adding event for %d", __func__, (int)fd));
    if (events & EV_READ) {
        if (do_fd_set(winop, idx, fd, 1)<0)
            return (-1);
    }
    if (events & EV_WRITE) {
        if (do_fd_set(winop, idx, fd, 0)<0)
            return (-1);
    }
    return (0);
}

int
win32_del(void *base, SOCKET fd, short old, short events,
          void *_idx)
{
    ModelManager* pDispatcher = (ModelManager*)base;
    struct win32op *winop = (struct win32op *)pDispatcher->getModelData();
    struct SocketIndex *idx = (struct SocketIndex *)_idx;

    //event_debug(("%s: Removing event for "EV_SOCK_FMT,__func__, EV_SOCK_ARG(fd)));
    if ( (old & EV_READ) && !(events & EV_READ) )
        do_fd_clear(base, winop, idx, 1);
    if ( (old & EV_WRITE) && !(events & EV_WRITE) )
        do_fd_clear(base, winop, idx, 0);

    return 0;
}

static void
fd_set_copy(struct win_fd_set *out, const struct win_fd_set *in)
{
    out->fd_count = in->fd_count;
    memcpy(out->fd_array, in->fd_array, in->fd_count * (sizeof(SOCKET)));
}

/*
static void dump_fd_set(struct win_fd_set *s)
{
unsigned int i;
printf("[ ");
for(i=0;i<s->fd_count;++i)
printf("%d ",(int)s->fd_array[i]);
printf("]\n");
}
*/

int
win32_dispatch(void *base, struct timeval *tv)
{
    ModelManager* pDispatcher = (ModelManager*)base;
    struct win32op *winop = (struct win32op *)pDispatcher->getModelData();
    int res = 0;
    unsigned j, i;
    int fd_count;
    SOCKET s;

    if (winop->resize_out_sets) {
        size_t size = FD_SET_ALLOC_SIZE(winop->num_fds_in_fd_sets);
        if (!(winop->readset_out = (struct win_fd_set *)realloc(winop->readset_out, size)))
            return (-1);
        if (!(winop->exset_out = (struct win_fd_set *)realloc(winop->exset_out, size)))
            return (-1);
        if (!(winop->writeset_out = (struct win_fd_set *)realloc(winop->writeset_out, size)))
            return (-1);
        winop->resize_out_sets = 0;
    }

    fd_set_copy(winop->readset_out, winop->readset_in);
    fd_set_copy(winop->exset_out, winop->writeset_in);
    fd_set_copy(winop->writeset_out, winop->writeset_in);

    fd_count =
        (winop->readset_out->fd_count > winop->writeset_out->fd_count) ?
        winop->readset_out->fd_count : winop->writeset_out->fd_count;

    if (!fd_count) {
        Sleep(tv->tv_usec/1000);
        return (0);
    }


    res = select(fd_count,
        (struct fd_set*)winop->readset_out,
        (struct fd_set*)winop->writeset_out,
        (struct fd_set*)winop->exset_out, tv);


    //event_debug(("%s: select returned %d", __func__, res));

    if (res <= 0) {
        if( res == -1 )
        {
            printf("error:%d\n", getErrno() );
        }
        return res;
    }

    if (winop->readset_out->fd_count) {
        i = rand() % winop->readset_out->fd_count;
        for (j=0; j<winop->readset_out->fd_count; ++j) {
            if (++i >= winop->readset_out->fd_count)
                i = 0;
            s = winop->readset_out->fd_array[i];
            pDispatcher->insertActiveList( s, EV_READ);
        }
    }
    if (winop->exset_out->fd_count) {
        i = rand() % winop->exset_out->fd_count;
        for (j=0; j<winop->exset_out->fd_count; ++j) {
            if (++i >= winop->exset_out->fd_count)
                i = 0;
            s = winop->exset_out->fd_array[i];
            pDispatcher->insertActiveList( s, EV_WRITE);
        }
    }
    if (winop->writeset_out->fd_count) {
        SOCKET s;
        i = rand() % winop->writeset_out->fd_count;
        for (j=0; j<winop->writeset_out->fd_count; ++j) {
            if (++i >= winop->writeset_out->fd_count)
                i = 0;
            s = winop->writeset_out->fd_array[i];
            pDispatcher->insertActiveList( s, EV_WRITE);
        }
    }
    return (0);
}

void
win32_dealloc(void *base)
{
    ModelManager* pDispatcher = (ModelManager*)base;
    struct win32op *winop = (struct win32op *)pDispatcher->getModelData();

    if (winop->readset_in)
        free(winop->readset_in);
    if (winop->writeset_in)
        free(winop->writeset_in);
    if (winop->readset_out)
        free(winop->readset_out);
    if (winop->writeset_out)
        free(winop->writeset_out);
    if (winop->exset_out)
        free(winop->exset_out);


    memset(winop, 0, sizeof(winop));
    free(winop);
}

#endif
```