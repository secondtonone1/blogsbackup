---
title: python学习(九)网络编程学习--简易网站服务器
date: 2017-09-05 15:39:49
categories: 技术开发
tags: [python]
---
python `网络编程`和其他语言都是一样的，服务器这块步骤为：
`1. 创建套接字`
`2. 绑定地址`
`3. 监听该描述符的所有请求`
`4. 有新的请求到了调用accept处理请求`

Python Web服务器网关接口（Python Web Server Gateway Interface，简称`“WSGI”`），可以保证同一个服务器响应不同应用框架的请求，WSGI的出现，让开发者可以将网络框架与网络服务器的选择分隔开来，例如，你可以使用Gunicorn或Nginx/uWSGI或Waitress服务器来运行Django、Flask或Pyramid应用。下面简单实现一个机遇WSGI协议的服务器。
<!--more-->
``` python
import socket
from io import StringIO
import sys


class WSGIServer(object):

    address_family = socket.AF_INET
    socket_type = socket.SOCK_STREAM
    request_queue_size = 1

    def __init__(self, server_address):
        # Create a listening socket
        self.listen_socket = listen_socket = socket.socket(
            self.address_family,
            self.socket_type
        )
        # Allow to reuse the same address
        listen_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        # Bind
        listen_socket.bind(server_address)
        # Activate
        listen_socket.listen(self.request_queue_size)
        # Get server host name and port
        host, port = self.listen_socket.getsockname()[:2]
        self.server_name = socket.getfqdn(host)
        self.server_port = port
        # Return headers set by Web framework/Web application
        self.headers_set = []
```
定义了一个WSGIServer类，并且在类的init函数完成了套接字的创建、绑定、监听等。
下面实现WSGIServer的轮询检测新的连接并处理连接：
``` python
def set_app(self, application):
        self.application = application

def serve_forever(self):
    listen_socket = self.listen_socket
    while True:
        # New client connection
        self.client_connection, client_address = listen_socket.accept()
        # Handle one request and close the client connection. Then
        # loop over to wait for another client connection
        self.handle_one_request()
```

实现处理请求的函数
``` python
def handle_one_request(self):
    self.request_data = request_data = self.client_connection.recv(1024)
    # Print formatted request data a la 'curl -v'
    print(''.join(
            '< {line}\n'.format(line=line)
            for line in request_data.splitlines()
    ))

    self.parse_request(request_data)

    # Construct environment dictionary using request data
    env = self.get_environ()

    # It's time to call our application callable and get
    # back a result that will become HTTP response body
    result = self.application(env, self.start_response)

    # Construct a response and send it back to the client
    self.finish_response(result)
```
解析请求
``` python
def parse_request(self, text):
    request_line = text.splitlines()[0]
    request_line = request_line.rstrip('\r\n')
    # Break down the request line into components
    (self.request_method,  # GET
    self.path,            # /hello
    self.request_version  # HTTP/1.1
    ) = request_line.split()
```
返回当前服务器wsgi版本等信息
``` python
def get_environ(self):
    env = {}
    # The following code snippet does not follow PEP8 conventions
    # but it's formatted the way it is for demonstration purposes
    # to emphasize the required variables and their values
    #
    # Required WSGI variables
    env['wsgi.version']      = (1, 0)
    env['wsgi.url_scheme']   = 'http'
    env['wsgi.input']        = StringIO.StringIO(self.request_data)
    env['wsgi.errors']       = sys.stderr
    env['wsgi.multithread']  = False
    env['wsgi.multiprocess'] = False
    env['wsgi.run_once']     = False
    # Required CGI variables
    env['REQUEST_METHOD']    = self.request_method    # GET
    env['PATH_INFO']         = self.path              # /hello
    env['SERVER_NAME']       = self.server_name       # localhost
    env['SERVER_PORT']       = str(self.server_port)  # 8888
    return env
```
填写app所需的回调函数
``` python
def start_response(self, status, response_headers, exc_info=None):
    # Add necessary server headers
    server_headers = [
            ('Date', 'Tue, 31 Mar 2015 12:54:48 GMT'),
            ('Server', 'WSGIServer 0.2'),
    ]
    self.headers_set = [status, response_headers + server_headers]
    # To adhere to WSGI specification the start_response must return
    # a 'write' callable. We simplicity's sake we'll ignore that detail
    # for now.
    # return self.finish_response
```

发送数据并且关闭连接
``` python
def finish_response(self, result):
    try:
        status, response_headers = self.headers_set
        response = 'HTTP/1.1 {status}\r\n'.format(status=status)
        for header in response_headers:
            response += '{0}: {1}\r\n'.format(*header)
        response += '\r\n'
        for data in result:
            response += data
            # Print formatted response data a la 'curl -v'
        print(''.join(
                '> {line}\n'.format(line=line)
                 for line in response.splitlines()
        ))
        self.client_connection.sendall(response)
    finally:
        self.client_connection.close()
```

主函数和参数解析，创建服务器
``` python
SERVER_ADDRESS = (HOST, PORT) = '', 8888


def make_server(server_address, application):
    server = WSGIServer(server_address)
    server.set_app(application)
    return server


if __name__ == '__main__':
    if len(sys.argv) < 2:
        sys.exit('Provide a WSGI application object as module:callable')
    app_path = sys.argv[1]
    module, application = app_path.split(':')
    module = __import__(module)
    application = getattr(module, application)
    httpd = make_server(SERVER_ADDRESS, application)
    print('WSGIServer: Serving HTTP on port {port} ...\n'.format(port=PORT))
    httpd.serve_forever()
```
将上面的文件保存为webserver.py
下面搭建虚拟环境，并且安装Pyramid、Flask和Django等框架开发的网络应用。
``` shell
$ [sudo] pip install virtualenv
$ mkdir ~/envs
$ virtualenv ~/envs/lsbaws/
$ cd ~/envs/lsbaws/
$ ls
bin  include  lib
$ source bin/activate
(lsbaws) $ pip install pyramid
(lsbaws) $ pip install flask
(lsbaws) $ pip install django
```
编写pyramidapp.py，主要是调用pyramidapp接口生成app
``` python
from pyramid.config import Configurator
from pyramid.response import Response


def hello_world(request):
    return Response(
        'Hello world from Pyramid!\n',
        content_type='text/plain',
    )

config = Configurator()
config.add_route('hello', '/hello')
config.add_view(hello_world, route_name='hello')
app = config.make_wsgi_app()
```
可以通过自己开发的网络服务器来启动上面的Pyramid应用。
`python webserver.py pyramidapp:app`

![1.png](1.png)
![2.png](2.png)
同样可以创建Flask应用
``` python
from flask import Flask
from flask import Response
flask_app = Flask('flaskapp')


@flask_app.route('/hello')
def hello_world():
    return Response(
        'Hello world from Flask!\n',
        mimetype='text/plain'
    )

app = flask_app.wsgi_app
```
![3.png](3.png)

上述代码的工作原理：

`1 网络框架提供一个命名为application的可调用对象`。
`2 服务器每次从HTTP客户端接收请求之后，调用application。它会向可调用对象传递一个名叫environ的字典作为参数，其中包含了WSGI/CGI的诸多变量，以及一个名为start_response的可调用对象`。
`3 框架/应用生成HTTP状态码以及HTTP响应报头（HTTP response headers），然后将二者传递至start_response，等待服务器保存。此外，框架/应用还将返回响应的正文。
服务器将状态码、响应报头和响应正文组合成HTTP响应，并返回给客户端`。

可以采用多进程的方式处理多个客户端请求,将上述代码稍作修改
``` python
import errno
import os
import signal
import socket

SERVER_ADDRESS = (HOST, PORT) = '', 8888
REQUEST_QUEUE_SIZE = 1024


def grim_reaper(signum, frame):
    while True:
        try:
            pid, status = os.waitpid(
                -1,          # Wait for any child process
                 os.WNOHANG  # Do not block and return EWOULDBLOCK error
            )
        except OSError:
            return

        if pid == 0:  # no more zombies
            return


def handle_request(client_connection):
    request = client_connection.recv(1024)
    print(request.decode())
    http_response = b"""\
HTTP/1.1 200 OK

Hello, World!
"""
    client_connection.sendall(http_response)


def serve_forever():
    listen_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    listen_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    listen_socket.bind(SERVER_ADDRESS)
    listen_socket.listen(REQUEST_QUEUE_SIZE)
    print('Serving HTTP on port {port} ...'.format(port=PORT))

    signal.signal(signal.SIGCHLD, grim_reaper)

    while True:
        try:
            client_connection, client_address = listen_socket.accept()
        except IOError as e:
            code, msg = e.args
            # restart 'accept' if it was interrupted
            if code == errno.EINTR:
                continue
            else:
                raise

        pid = os.fork()
        if pid == 0:  # child
            listen_socket.close()  # close child copy
            handle_request(client_connection)
            client_connection.close()
            os._exit(0)
        else:  # parent
            client_connection.close()  # close parent copy and loop over

if __name__ == '__main__':
    serve_forever()
```
grim_reaper 函数为捕捉子进程退出的回调函数，父进程等待所有子进程退出后再退出，避免僵尸进程。由于子进程退出父进程捕获到消息，调用grim_reaper处理，由于父进程之前阻塞在accept上，捕获子进程销毁消息后，父进程accept失败，所以增加了errno.EINTR错误判断，如果是由于信号中断导致accept失败，就让父进程继续调用accept即可。

![4.png](4.png)
谢谢关注我的微信公众平台：
![1.jpg](1.jpg)
