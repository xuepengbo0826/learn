# <center>linxu下redis集群</center>
下载redis3.0以上版本<br/>
下载ruby2.2.2以上版本<br/>
下载rubygems<br/>
以上操作自行百度<br/><br/>

1. 创建集群文件夹

```
mkdir redisCluster
```

2. 创建集群中单个redis文件夹

```
cd redisCluster;
mkdir redisClusterOne;
mkdir redisClusterTwo;
mkdir redisClusterThree;
mkdir redisClusterFour;
mkdir redisClusterFive;
mkdir redisClusterSix;
```

3. 复制redis到集群中的每个redis文件夹

```
cp -r redis redisCluster/redisClusterOne;
cp -r redis redisCluster/redisClusterTwo;
cp -r redis redisCluster/redisClusterThree;
cp -r redis redisCluster/redisClusterFour;
cp -r redis redisCluster/redisClusterFive;
cp -r redis redisCluster/redisClusterSix;
```

4. 修改集群中单个redis配置文件<br/>
由于我们是在单机环境下搭建的，所以必须修改每个redis单体的port端口号
进入每个redis单体的文件夹

```
vi redis.conf
```
需要修改一下配置文件
> #端口号 <br/>
port 7000 <br/>
#开启集群 <br/>
cluster-enabled yes<br/>
#集群配置文件，可以写为nodes-你设置的端口号.conf
cluster-config-file nodes.conf<br/>
#集群连接超时时间 /毫秒
cluster-node-timeout 5000<br/>
#后台启动<br/>
appendonly yes<br/>
bind 当前集群ip地址

5. 创建集群<br/>
把redis单体src文件夹下的redis.trib.rb文件拷贝到redisCluster文件夹下，执行该文件
```
redis-trib.rb create --replicas 1 10.100.31.209:30001 10.100.31.209:30002 10.100.31.209:30003 10.100.31.209:30004 10.100.31.209:30005 10.100.31.209:30006
```
参数解释：<br/>
代码中的1，指的是master/slave

6. <a name="6">集群中重新分配hash槽</a>
首先连接到当前集群
```
redis-trib.rb reshard ip:端口号
```

> How many slots do you want to move (from 1 to 16384)? 1000
What is the receiving node ID?

1000：要分配的hash槽数量<br/>
What is the receiving node ID：要分配给的集群节点id<br/>

> Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1:

Source node #1：输入某一个节点id或者all<br/>
all：从全部节点中取出一部分空闲分配给当前节点。<br/>
输入某一个节点id：从该id节点分配空闲hash槽给当前节点。<br/>

7. <a name="lianjie">在集群中添加主节点</a><br/>
新建一个全新redis单体<br/>
需要修改一下配置文件<br/>

> #端口号 <br/>
port 7000 <br/>
#开启集群 <br/>
cluster-enabled yes<br/>
#集群配置文件，可以写为nodes-你设置的端口号.conf
cluster-config-file nodes.conf<br/>
#集群连接超时时间 /毫秒
cluster-node-timeout 5000<br/>
#后台启动<br/>
appendonly yes<br/>
bind 当前集群ip地址

开启新的redis节点

```
src/redis-server redis.conf
```
添加主节点到集群

```
redis-trib.rb add-nodes 10.100.31.209:30007 10.100.31.209:30001

```
<a href="#6">添加主节点完成后要重新分配hash槽，否则新添加的主节点没有存储空间</a>
8. 在集群中添加从节点<br/>
<a href="#lianjie">添加节点前的操作同7</a>
```
redis-trib.rb add-node --slave --master-id 主节点id 添加节点的ip和端口 集群中已存在节点ip和端口
```

9. 从集群中删除节点<br/>
<a href="#6">删除节点前首先把要删除节点的hash擦分配出去</a>
分配完成后

```
redis-trib.rb del-node ip:端口号 节点id
```
