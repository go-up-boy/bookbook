# Docker 部署 Redis集群

* 系统 windows11 WSL2
* 版本 Redis6.0.16
* 部署 6 个节点，3 master 3 slaves

1. 首先拉取redis docker镜像
```shell
docker pull redis:6.0.16
```
2. 运行 6 个容器
   * --cluster-enabled yes // 开启redis集群
   * --appendonly yes // 开启持久化
   * --net host 使用宿主机的ip和端口，windows不支持，这里手动映射端口
```shell
 docker run -d --name redis-node-1 -p 6381:6381 -p 16381:16381  --privileged=true -v C:/Users/z/redis/redis-node-1:/data redis:6.0.16 --cluster-enabled yes --appendonly yes --port 6381

 docker run -d --name redis-node-2 -p 6382:6382 -p 16382:16382 --privileged=true -v C:/Users/z/redis/redis-node-2:/data redis:6.0.16 --cluster-enabled yes --appendonly yes --port 6382
 
 docker run -d --name redis-node-3 -p 6383:6383 -p 16383:16383 --privileged=true -v C:/Users/z/redis/redis-node-3:/data redis:6.0.16 --cluster-enabled yes --appendonly yes --port 6383

 docker run -d --name redis-node-4 -p 6384:6384 -p 16384:16384 --privileged=true -v C:/Users/z/redis/redis-node-4:/data redis:6.0.16 --cluster-enabled yes --appendonly yes --port 6384

 docker run -d --name redis-node-5 -p 6385:6385 -p 16385:16385 --privileged=true -v C:/Users/z/redis/redis-node-5:/data redis:6.0.16 --cluster-enabled yes --appendonly yes --port 6385

 docker run -d --name redis-node-6 -p 6386:6386 -p 16386:16386 --privileged=true -v C:/Users/z/redis/redis-node-6:/data redis:6.0.16 --cluster-enabled yes --appendonly yes --port 6386
```
> 这里 需要注意的是 为什么要映射两个端口，因为Windows --net host模式不会生效
3. 进入redis-node-1 为集群建立集群关系
    - --cluster-replicas 1 表示为每个master创建一个slave节点
```shell
redis-cli --cluster create 192.168.1.12:6381 192.168.1.12:6382 192.168.1.12:6383 192.168.1.12:6384 192.168.1.12:6385 192.168.1.12:6386 --cluster-replicas 1
```
以下提示 集群建立成功
>[OK] All nodes agree about slots configuration.
>Check for open slots...                                                                           
> Check slots coverage...
>[OK] All 16384 slots covered.
4. 连接 redis-cli 测试
    * -c 参数 连接集群结点时使用，此选项可防止moved和ask异常
```shell
redis-cli -p 6381 -c
```
5. 集群常用命令
```shell
// 检查集群状态 任意一个容器
redis-cli --cluster check 192.168.1.12:6381
// 集群状态
redis-cli -h ip -p 6381 -a password cluster info
// 集群节点信息
redis-cli -h ip -p 6381 -a password cluster nodes
// 节点内存 cpu key数量等信息 每个节点都需查看
redis-cli -h ip -p 6381 -a password info
// 从redis集群中查看key 只需要连接master节点查看
// 查看该节点的所有key
redis-cli -h ip -p 6381 -a password keys *
// 查看key的value值
redis-cli -h ip -p 6381 -a password -c get key
// 查看key值得有效期
redis-cli -h ip -p 6381 -a password -c ttl key
```

## 集群扩容、缩容
### 集群扩容
1. 运行两个容器 master 和 slave
```shell
 docker run -d --name redis-node-7 -p 6387:6387 -p 16387:16387 --privileged=true -v C:/Users/z/redis/redis-node-7:/data redis:6.0.16 --cluster-enabled yes --appendonly yes --port 6387
 
 docker run -d --name redis-node-8 -p 6388:6388 -p 16388:16388 --privileged=true -v C:/Users/z/redis/redis-node-8:/data redis:6.0.16 --cluster-enabled yes --appendonly yes --port 6388
```
2. 进入到新扩容的容器，并且 作为master节点加入集群
```shell
 redis-cli --cluster add-node 192.168.1.12:6387 192.168.1.12:6381
```
3. 为新加入的master节点分配槽位
```shell
redis-cli --cluster reshard 192.168.1.12:6381
```
> How many slots do you want to move (from 1 to 16384)? 4096 <br>
> What is the receiving node ID? <br> 
> 47ce2b86e1501d2076eaec8cb54b3e021cc803da // 新master节点的hash <br>
> Please enter all the source node IDs. <br>
> Type 'all' to use all the nodes as source nodes for the hash slots. <br>
> Type 'done' once you entered all the source nodes IDs. <br>
> Source node #1: all

> 等待槽位分配完成即可

4. 为新master节点分配 slave节点
    * --cluster-master-id 根据实际节点id
```shell
redis-cli --cluster add-node 192.168.1.12:6388 192.168.1.12:6387 --cluster-slave --cluster-master-id 47ce2b86e1501d2076eaec8cb54b3e021cc803da
```
### 集群缩容

1. 删除 slave 节点
    * ip:port slave_id
```shell
redis-cli --cluster del-node 192.168.1.12:6388 18c67b76bc2c9f07d2800fb4120b933dda22fe92
```
2. 将要缩容master节点槽号清空，重新分配，本次都分配给6381
```shell
redis-cli --cluster reshard 192.168.1.12:6381
```
* 选择要分配的数量
> How many slots do you want to move (from 1 to 16384)? 4096
* 分配给 6381 ，填写接收槽号的 node id
> What is the receiving node ID? aea82a556231dc3573412b6c6f7443376a829fe3
* 输入需要缩容的master节点ID，本案例中的 6387
> Source node #1: 47ce2b86e1501d2076eaec8cb54b3e021cc803da <br>
> Source node #1: done
3. 删除 已清空槽号的master 节点
```shell
redis-cli --cluster del-node 192.168.1.12:6387 47ce2b86e1501d2076eaec8cb54b3e021cc803da
```
4. 最后检查一下节点
```shell
root@292b9949f373:/data# redis-cli --cluster check 192.168.1.12:6381
192.168.1.12:6381 (aea82a55...) -> 0 keys | 8192 slots | 1 slaves.
172.17.0.4:6383 (3d410141...) -> 1 keys | 4096 slots | 1 slaves.
172.17.0.3:6382 (cbc4d3cf...) -> 0 keys | 4096 slots | 1 slaves.
[OK] 1 keys in 3 masters.
0.00 keys per slot on average.
>>> Performing Cluster Check (using node 192.168.1.12:6381)
M: aea82a556231dc3573412b6c6f7443376a829fe3 192.168.1.12:6381
   slots:[0-6826],[10923-12287] (8192 slots) master
   1 additional replica(s)
M: 3d41014104245657c1c43aa0c7904c586d32eab8 172.17.0.4:6383
   slots:[12288-16383] (4096 slots) master
   1 additional replica(s)
S: 94ded12b7df9071fc07d2a6d46cdd75f05f5758b 172.17.0.5:6384
   slots: (0 slots) slave
   replicates 3d41014104245657c1c43aa0c7904c586d32eab8
M: cbc4d3cf29dc64f19373449473affe7555d8f14c 172.17.0.3:6382
   slots:[6827-10922] (4096 slots) master
   1 additional replica(s)
S: ae12fbf44924cf308f79ad4b6789bbe7a0927c5f 172.17.0.7:6386
   slots: (0 slots) slave
   replicates cbc4d3cf29dc64f19373449473affe7555d8f14c
S: 8e8d85c56593d8abdc5a6ec813895cef5288272e 172.17.0.6:6385
   slots: (0 slots) slave
   replicates aea82a556231dc3573412b6c6f7443376a829fe3
```
> 一切正常，OK