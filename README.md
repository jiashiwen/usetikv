准备tikv开发环境的时候遇到点小坑

# 麻烦是啥
tikv是一个分布式的kv存储系统，开发环境免不了部署多个节点。手头资源不够的同学可以用pingcap官方给出的docker-compose方案（https://github.com/pingcap/tidb-docker-compose）。
麻烦就从这里开始了。这个方案暴露到本地的端口只有tidb、grafana的端口（4000、 9090、3000）；pd、tikv的端口并没有暴露到本地。需要修改的地方有几处
* 为了保证容器内时间和宿主机时间一致，最好在每个"volumes"下添加"- /etc/localtime:/etc/localtime:ro"
* 为容器pd0、pd1、pd2添加"ports"暴露端口号,由于我们要吧端口全部映射到本地所以三个pd节点使用不同的端口号。修改完的pd大概长这个样子
```
  pd1:
    image: pingcap/pd:latest
    ports:
      - "2378:2378"
    volumes:
      - ./config/pd.toml:/pd.toml:ro
      - ./data:/data
      - ./logs:/logs
      - /etc/localtime:/etc/localtime:ro
    command:
      - --name=pd1
      - --client-urls=http://0.0.0.0:2378
      - --peer-urls=http://0.0.0.0:2380
      - --advertise-client-urls=http://pd1:2378
      - --advertise-peer-urls=http://pd1:2380
      - --initial-cluster=pd0=http://pd0:2380,pd1=http://pd1:2380,pd2=http://pd2:2380
      - --data-dir=/data/pd1
      - --config=/pd.toml
      - --log-file=/logs/pd1.log
    restart: on-failure

```
* 暴露tikv的端口号，前面改了pd的端口应声，需要修改"command"，"- --pd=pd0:2379,pd1:2378,pd2:2377"
```
  tikv2:
    image: pingcap/tikv:latest
    ports:
      - "20162:20162"

    volumes:
      - ./config/tikv.toml:/tikv.toml:ro
      - ./data:/data
      - ./logs:/logs
      - /etc/localtime:/etc/localtime:ro
    command:
      - --addr=0.0.0.0:20162
      - --advertise-addr=tikv2:20162
      - --data-dir=/data/tikv2
      - --pd=pd0:2379,pd1:2378,pd2:2377
      - --config=/tikv.toml
      - --log-file=/logs/tikv2.log
    depends_on:
      - "pd0"
      - "pd1"
      - "pd2"
    restart: on-failure

```
* 修改好的docker-compose.yml在这里（https://github.com/jiashiwen/usetikv/blob/master/docker-compose.yml）,想省事儿的同学直接覆盖官方的docker-compse文件就可以了
* 最后修改一下本地/etc/hosts文件，pd及tikv都是通过hostname绑定的“advertise-client-urls”，不绑定hosts找不到找不到pd和tikv的节点
```
127.0.0.1  pd0
127.0.0.1  pd1
127.0.0.1  pd2
127.0.0.1  tikv0
127.0.0.1  tikv1
127.0.0.1  tikv2
```

至此，环境搭建完毕，愉快的run demo吧。