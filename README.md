# compose_web

### 前提：安装好Docker Engine和Docker Compose

安装compose：
```
$ sudo apt-get install python-pip
$ sudo pip install docker-compose
```
## step1: 配置过程

### 1.1 创建项目目录
```
$ mkdir composetest
$ cd composetest
```

### 1.2 在目录中创建app.py文件，内容如下：
```python
from flask import Flask
from redis import Redis

app = Flask(__name__)
redis = Redis(host='redis', port=6379)

@app.route('/')
def hello():
    redis.incr('hits')
    return 'Hello World! I have been seen %s times.' % redis.get('hits')

if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True)
```

### 1.3 在目录中创建requirements.txt文件，内容如下：
```
flask
redis
```

## step2: 创建Docker镜像

### 2.1 在目录中创建Dockerfile文件，内容如下：
```Dockerfile
FROM python:2.7
ADD . /code
WORKDIR /code
RUN pip install -r requirements.txt
CMD python app.py
```
- 基础镜像Python2.7
- 将当前路径下的文件复制到镜像中的/code目录下
- 在镜像中工作路径切换到/code
- 安装Python依赖包(flask、redis)
- 设置容器启动后默认执行的命令

### 2.2 基于Dockerfile构建镜像
```
$ sudo docker build -t journeycheng/web .
```

## step3: 定义服务
使用docker-compose.yml定义一整套服务。

在目录中创建docker-compose.yml文件，内容如下：
```docker
version: '2'
services:
  web:
    build: .
    ports:
      - "5000:5000"
    volumes:
      - .:/code
    depends_on:
      - redis
  redis:
    image:redis
```
compose文件定义了两种服务，web和redis。
- web服务：
    - 使用的镜像根据当前目录中的Dockerfile创建
    - 暴露主机端口5000，并连接到容器内的5000端口
    - 将主机上的项目目录挂载到容器中，修改代码后就不必重建镜像了
    - 连接web服务和redis服务
- redis服务：
    - 直接使用最新的官方redis镜像
  
## step4: 创建和运行应用
### 4.1 启动应用
```
$ sudo docker-compose up
Creating network "composetest_default" with the default driver
Pulling redis (redis:latest)...
...
redis_1  | 1:M 03 Nov 07:38:46.813 * The server is now ready to accept connections on port 6379
web_1    |  * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
web_1    |  * Restarting with stat
web_1    |  * Debugger is active!
```

在浏览器中输入http://host-ip:5000 就可以看到运行的应用了:
Hello World! I have been seen 1 times.
（此处应有图片一张...）

刷新网页，数字会增加
Hello World! I have been seen 2 times.
同时，在终端可以看到请求的记录：
```
web_1    | 192.168.140.197 - - [03/Nov/2016 07:46:00] "GET / HTTP/1.1" 200 -
web_1    | 192.168.140.197 - - [03/Nov/2016 07:47:31] "GET / HTTP/1.1" 200 -
web_1    | 192.168.140.197 - - [03/Nov/2016 07:47:52] "GET / HTTP/1.1" 200 -
web_1    | 192.168.140.197 - - [03/Nov/2016 07:48:36] "GET / HTTP/1.1" 200 -
```

退出Ctrl + c
```
^CGracefully stopping... (press Ctrl+C again to force)
Stopping composetest_web_1 ... 
Stopping composetest_redis_1 ... 
Killing composetest_web_1 ... done
Killing composetest_redis_1 ... done
```

### 4.2 启动应用，后台运行
如果你想在后台运行服务，可以在docker-compose up命令后增加－d （detached模式）
```
$ sudo docker-compose up -d
Starting composetest_redis_1
Starting composetest_web_1
$ sudo docker-compose ps
       Name                      Command               State           Ports          
-------------------------------------------------------------------------------------
composetest_redis_1   docker-entrypoint.sh redis ...   Up      6379/tcp               
composetest_web_1     /bin/sh -c python app.py         Up      0.0.0.0:5000->5000/tcp
```
除此之外，还可以运行一些一次性的命令，比如查看web服务的环境变量
```
$ sudo docker-compose run web env
PATH=/usr/local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=55b0348a8d6d
TERM=xterm
LANG=C.UTF-8
GPG_KEY=C01E1CAD5EA2C4F0B8E3571504C367C218ADD4FF
PYTHON_VERSION=2.7.12
PYTHON_PIP_VERSION=8.1.2
HOME=/root
```
### 4.3 停止后台运行的服务
```
$ sudo docker-compose stop
Stopping composetest_web_1 ... done
Stopping composetest_redis_1 ... done
```
