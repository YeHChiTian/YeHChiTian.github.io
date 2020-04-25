---
title: libuv多线程中使用uv_accept
date: 2020-03-27 00:01:33
categories: 
- libuv
tags: libuv uv_accept multithread loop 
---

![theme](./libuv多线程中使用uv-accept/theme.jpg)

# 前言

在libuv中，我想在线程A绑定端口，只负责监听，uv_accept接收新连接，将接收到的client,  放到线程B中使用另外一个loop。然而，发现accept一个client, 并没有办法将client放到另外的loop中。 于是开始寻找解决办法， 本文使用uv_write2间接实现了想要的效果。

# 一、问题描述

在github和stackoverflow社区上，2016年时候，已经有人遇到几乎完全相同的问题。

[<font color="#0000FF">uv_accept in one thread and uv_read_start from other thread</font>](https://github.com/libuv/libuv/issues/961)

[<font color="#0000FF">about accept multiple loops</font>](https://github.com/libuv/libuv/issues/932)

[<font color="#0000FF">How to use uv_accept in multithread environment?</font>](https://stackoverflow.com/questions/29608662/how-to-use-uv-accept-in-multithread-environment)

libuv官方开发人员指出，没有直接的办法实现这个需求， 但是可以间接实现这个要求。

>There is no way to accept a client on another loop. There are a number of things you can do:
>
>- create the listener handle, then create X new loops, send the handle to them using uv_write2 and listen in all of them
>- if you're on Linux (> 3.9 IIRC), create X new loops and bind a handle to the same IP and port using SO_REUSEPORT
>- have a single loop accept all incoming connections and send them to other loops using uv_write2.

看上去，只有第三个可以间接实现这个需求。使用一个loop监听accept所有链接，将收到的链接client，通过uv_write2间接发送给其他loop。



# 二、accept多线程替代方案

在官方例子中，**multi-echo-server**恰好实现了类似的需求。 不同的是，demo是将在父进程接收新的链接，将到来的接连client，通过uv_write2发送子进程， 子进程将接收到的client加入到自己的loop中。

但我们将接收到新的连接client，发到另外一个线程，而不是进程。为了满足这个需求，因此阅读了例子源码。

参考**multi-echo-server**, 这个例子内容挺多，挺复杂的，但是其主要原理大致如下：

* 首先使用socketpair创建一对已链接套接字，假设pfds, 用于父子进程之间通信。

* 父进程监听指定端口，将uv_accept收到的链接client，以pfds方式用uv_write2通知子进程。
* 内部uv_write2传递client到子进程，**本质上是使用sendmsg传递真正的fd（内部包含真正fd**), 即父进程sendmsg发送fd到子进程，**内核会将fd所对应的文件结构拷贝一份到子进程中**。
* 子进程使用pfds等待父进程消息。有消息到来，会使用uv_accept接收client, 然后将client加入到自己进程的loop中。 uv_accept接收client，由于pfds初始化指明接收这个消息是以ipc形式，**因此uv_accept本质上使用recvmsg接收父进程的fd，将其封装为client**。

这样就成功将父进程监听到的新连接client, 传递到子进程中，并且加入子进程自己的loop中。

通过分析，可以简单将其多进程改为多线程方式即可，需要用到socketpair，将创建一对连接用于线程之间的通信。需要注意点:

* 可以使用内部接口uv__make_socketpair， 间接调用socketpair创建一对已连接的socket. 用于线程通信。demo中包含uv_ex.h文文件， 其中声明uv_make_socektpair包含内部uv\_\_make_socketpair接口。
* 用于线程通信的句柄handle，必须指定ipc，即初始化参数指定1， 确保内部会使用sendmsg和recvmsg传递fd.
* 线程之间传递fd同样是使用sendmsg和recvmsg， 因此，不能确保fd值相同，但是fd所对应的文件节点是相同的。

我的实现代码，参考[<font color="#0000FF">multi-echo-server</font>](https://github.com/libuv/libuv/tree/v1.x/docs/code/multi-echo-server)

```c++
#include <inttypes.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <assert.h>

#include <uv.h>
#include "uv_ex.h"

uv_loop_t *loop[2];
uv_pipe_t pipe[2];
int fds[2];

uv_buf_t dummy_buf;

void echo_write(uv_write_t *req, int status) {
    if (status < 0) {
        fprintf(stderr, "Write error %s\n", uv_err_name(status));
    }
}

void echo_read(uv_stream_t *client, ssize_t nread, const uv_buf_t *buf) {
    if (nread > 0) {
    	uv_write_t *req = (uv_write_t*) malloc(sizeof(uv_write_t));
		uv_buf_t send_buf;
		send_buf.base =  buf->base;
		send_buf.len =  buf->len;
        uv_write((uv_write_t*) req, client, &send_buf, 1, echo_write);
        return;
    }

    if (nread < 0) {
        if (nread != UV_EOF)
            fprintf(stderr, "Read error %s\n", uv_err_name(nread));
        uv_close((uv_handle_t*) client, NULL);
    }

    free(buf->base);
}

void close_process_handle(uv_process_t *req, int64_t exit_status, int term_signal) {
    fprintf(stderr, "Process exited with status %" PRId64 ", signal %d\n", exit_status, term_signal);
    uv_close((uv_handle_t*) req, NULL);
}

void alloc_buffer(uv_handle_t *handle, size_t suggested_size, uv_buf_t *buf) {
  buf->base = malloc(suggested_size);
  buf->len = suggested_size;
}

void on_new_connection(uv_stream_t *server, int status) {
    if (status == -1) {
        // error!
        return;
    }
    printf("on_new_connection\n");
    uv_tcp_t *client = (uv_tcp_t*) malloc(sizeof(uv_tcp_t));
    uv_tcp_init(loop[0], client);
    if (uv_accept(server, (uv_stream_t*) client) == 0) {
        uv_write_t *write_req = (uv_write_t*) malloc(sizeof(uv_write_t));
        dummy_buf = uv_buf_init("a", 1);
        //向管道worker->pipe写入信息
        uv_write2(write_req, (uv_stream_t*) &pipe[0], &dummy_buf, 1, (uv_stream_t*) client, echo_write);
    }
    else {
        uv_close((uv_handle_t*) client, NULL);
    }
}

void child_on_new_connection(uv_stream_t *q, ssize_t nread, const uv_buf_t *buf) {
	printf(" child_on_new_connection \n");
    if (nread < 0) {
        if (nread != UV_EOF)
            fprintf(stderr, "Read error %s\n", uv_err_name(nread));
        uv_close((uv_handle_t*) q, NULL);
        return;
    }

    uv_pipe_t *pipe = (uv_pipe_t*) q;
    if (!uv_pipe_pending_count(pipe)) {
        fprintf(stderr, "No pending count\n");
        return;
    }

    uv_handle_type pending = uv_pipe_pending_type(pipe);
    assert(pending == UV_TCP);

    uv_tcp_t *client = (uv_tcp_t*) malloc(sizeof(uv_tcp_t));
    uv_tcp_init(loop[1], client);
    if (uv_accept(q, (uv_stream_t*) client) == 0) {
        uv_os_fd_t fd;
        uv_fileno((const uv_handle_t*) client, &fd);
        fprintf(stderr, "Worker %d: Accepted fd %d\n", getpid(), fd);
        uv_read_start((uv_stream_t*) client, alloc_buffer, echo_read);
    }
    else {
        uv_close((uv_handle_t*) client, NULL);
    }
}

void new_thread_handle_connection(void* arg){
	int fd = *(int *)arg;

	loop[1] = (uv_loop_t*)malloc(sizeof(uv_loop_t));
	uv_loop_init(loop[1]);

	uv_pipe_init(loop[1], &pipe[1], 1 /* ipc */);
	uv_pipe_open(&pipe[1], fd);
	uv_read_start((uv_stream_t*)&pipe[1], alloc_buffer, child_on_new_connection);

	uv_run(loop[1], UV_RUN_DEFAULT);

}

int main() {
	loop[0] = (uv_loop_t*)malloc(sizeof(uv_loop_t));
	uv_loop_init(loop[0]);

	uv_pipe_init(loop[0], &pipe[0], 1);

    //create a pair of connected sockets
    if (uv_make_socketpair(fds,0) != 0)
    	return -1;

    uv_pipe_open(&pipe[0], fds[0]);


    pthread_t tid;
	int error = pthread_create(&tid, NULL,
			new_thread_handle_connection, (void*) &fds[1]);
	if (0 != error)
		fprintf(stderr, "Couldn't run thread number %d, errno %d\n", tid, error);

    uv_tcp_t server;
    uv_tcp_init(loop[0], &server);

    struct sockaddr_in bind_addr;
    uv_ip4_addr("0.0.0.0", 7000, &bind_addr);
    uv_tcp_bind(&server, (const struct sockaddr *)&bind_addr, 0);
    int r;
    if ((r = uv_listen((uv_stream_t*) &server, 128, on_new_connection))) {
        fprintf(stderr, "Listen error %s\n", uv_err_name(r));
        return 2;
    }

    uv_run(loop[0], UV_RUN_DEFAULT);

    pthread_join(tid, NULL);

    return 0;
}
```



# 三、libuv实例源码解析

实例[<font color="#0000FF">multi-echo-server</font>](https://github.com/libuv/libuv/tree/v1.x/docs/code/multi-echo-server)大致分析。

1. 父进程使用setup_workers安装子进程任务，然后监听新的连接到来， 回调on_new_connection

```c++
int main() {
    loop = uv_default_loop();

    setup_workers();

    uv_tcp_t server;
    uv_tcp_init(loop, &server);

    struct sockaddr_in bind_addr;
    uv_ip4_addr("0.0.0.0", 7000, &bind_addr);
    uv_tcp_bind(&server, (const struct sockaddr *)&bind_addr, 0);
    int r;
    if ((r = uv_listen((uv_stream_t*) &server, 128, on_new_connection))) {
        fprintf(stderr, "Listen error %s\n", uv_err_name(r));
        return 2;
    }
    return uv_run(loop, UV_RUN_DEFAULT);
}
```

2. 当有新的连接到来，调用uv_accept，将连接client通过uv_write2发送给子进程（通过socketpair建立的IO）,

   在uv_write2内部会将收到client的fd以sendmsg发送给子进程。

```c++
void on_new_connection(uv_stream_t *server, int status) {
    if (status == -1) {
        // error!
        return;
    }
    uv_tcp_t *client = (uv_tcp_t*) malloc(sizeof(uv_tcp_t));
    uv_tcp_init(loop, client);
    if (uv_accept(server, (uv_stream_t*) client) == 0) {
        uv_write_t *write_req = (uv_write_t*) malloc(sizeof(uv_write_t));
        dummy_buf = uv_buf_init("a", 1);
        struct child_worker *worker = &workers[round_robin_counter];
        //向管道worker->pipe写入信息
        uv_write2(write_req, (uv_stream_t*) &worker->pipe, &dummy_buf, 1, (uv_stream_t*) client, echo_write);
        round_robin_counter = (round_robin_counter + 1) % child_worker_count;
    }
    else {
        uv_close((uv_handle_t*) client, NULL);
    }
}

```

3. setup_workers方法安装启动多个子进程，每个子进程都使用socketpair创建IO机制跟父进程通信。

```c++
  workers = calloc(sizeof(struct child_worker), cpu_count);
    while (cpu_count--) {
        struct child_worker *worker = &workers[cpu_count];
        uv_pipe_init(loop, &worker->pipe, 1);

        uv_stdio_container_t child_stdio[3];
        child_stdio[0].flags = UV_CREATE_PIPE | UV_READABLE_PIPE;
        //note, 注意这里stream绑定到管道了。
        child_stdio[0].data.stream = (uv_stream_t*) &worker->pipe;
        child_stdio[1].flags = UV_IGNORE;
        child_stdio[2].flags = UV_INHERIT_FD;
        child_stdio[2].data.fd = 2;

        worker->options.stdio = child_stdio;
        worker->options.stdio_count = 3;

        worker->options.exit_cb = close_process_handle;
        worker->options.file = args[0];
        worker->options.args = args;

        int ret = uv_spawn(loop, &worker->req, &worker->options);
        fprintf(stderr, "ret = %d, Started worker %d\n",ret,  worker->req.pid);
    }
```

4.  重点理解uv_spawn函数（内容很多，有点复杂），主要是内部fork出子进程使用execvp执行**worker**程序，使用socketpair创建IO机制，并且一端绑定父进程，一端绑定子进程，并且重定向子进程的IO, 使得父进程向worker->pipe写入数据， **worker**程序的从标准输入即可读取。源码过多，不展示。
5. 子进程主要是监听父进程的信息。uv_accept收到消息，内部会使用recvmsg将其封装句柄client中，然后添加到自己的loop.

```c++
void on_new_connection(uv_stream_t *q, ssize_t nread, const uv_buf_t *buf) {
    if (nread < 0) {
        if (nread != UV_EOF)
            fprintf(stderr, "Read error %s\n", uv_err_name(nread));
        uv_close((uv_handle_t*) q, NULL);
        return;
    }

    uv_pipe_t *pipe = (uv_pipe_t*) q;
    if (!uv_pipe_pending_count(pipe)) {
        fprintf(stderr, "No pending count\n");
        return;
    }

    uv_handle_type pending = uv_pipe_pending_type(pipe);
    assert(pending == UV_TCP);

    uv_tcp_t *client = (uv_tcp_t*) malloc(sizeof(uv_tcp_t));
    uv_tcp_init(loop, client);
    if (uv_accept(q, (uv_stream_t*) client) == 0) {
        uv_os_fd_t fd;
        uv_fileno((const uv_handle_t*) client, &fd);
        fprintf(stderr, "Worker %d: Accepted fd %d\n", getpid(), fd);
        uv_read_start((uv_stream_t*) client, alloc_buffer, echo_read);
    }
    else {
        uv_close((uv_handle_t*) client, NULL);
    }
}
```

# 四、总结

本次实现利用了libuv提供的接口uv_write2，将接收到的描述符fd，传递到另外地方，可以是进程，也可以是线程。然后接收端再调用accept封装成句柄添加到另外loop中。

顺便记录，几处代码阅读稍微卡住。

1. 对于指针赋值，不够敏感。一直在寻找socketpair一端信息，如何绑定到父进程的pipe中。 检查之后，才发现原来child_stdio[0].data.stream已经获取了pipe地址。
2. 对描述符属性UV_O_CLOEXEC，不够敏感。 使用UV__O_CLOEXEC创建了一对管道描述符，然后fork. 那么当子进程执行execvp系列函数， 对应的描述符会关闭。如果父进程有监听此描述符，那么会收到结束标志EOF, 即read正常返回0.

# 五、参考

[<font color="#0000FF">multi-echo-server</font>](https://github.com/libuv/libuv/tree/v1.x/docs/code/multi-echo-server)







