## Ubuntu安装subspace

ubuntu版本为22.04

### 一、安装

#### 1.安装Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```



#### 2.安装其它相关工具

```
sudo apt-get install git protobuf-compiler llvm clang cmake
```



#### 3.git 方式编译安装

##### a. clone源码

```
git clone https://github.com/subspace/subspace.git
```

##### b. checkout到合适的版本或tag, 如当前最新是gemini-3h-2024-mar-20

```
cd subspace

git checkout gemini-3h-2024-mar-20
```

##### c.编译

```bash
cargo build \
    --profile production \
    --bin subspace-node \
    --bin subspace-farmer
```

国内环境安装时可能出现github访问超时问题

```
Cloning into 'subspace'...
fatal: unable to access 'https://github.com/subspace/subspace.git/': Failed to connect to github.com port 443 after 129521 ms: Connection timed out
```

如果有此问题，可以尝试修改主机hosts解决：

```
vi /etc/hosts
```

添加：

```
140.82.113.3 github.com 

199.232.5.194 github.global.ssl.fastly.net 

54.231.114.219 github-cloud.s3.amazonaws.com

source /etc/hosts
```

编译完成后，二进制文件在target/production目录下，可以将subspace-node和subspace-farmer mv到/usr/bin下



### 二、启动和配置

#### 1.启动node

```
  ./subspace-node \
    run \
    --chain gemini-3h \
    --base-path /var/subspace/node/ \
    --listen-on /ip4/0.0.0.0/tcp/30333 \
    --dsn-listen-on /ip4/0.0.0.0/udp/30433/quic-v1 \
    --dsn-listen-on /ip4/0.0.0.0/tcp/30433 \
    --rpc-cors all \
    --rpc-methods unsafe \
    --rpc-listen-on 0.0.0.0:9944 \
    --farmer \
    --name "node"
```

--base-path 为同步数据存放的目录，请自行设置，空间预留200G。

--rpc-listen-on 为node对外提供rpc服务的端口设置，默认9944，如被占用可修改。

node启动后会同步整个链的数据，时间比较长，通常要几个小时。



#### 2.启动farmer

```
  ./subspace-farmer \
    farm \
    --node-rpc-url ws://0.0.0.0:9944 \
    --reward-address WALLET_ADDRESS \
    path=/var/subspace/data,size=100G
```

--node-rpc-url 这里设置之前启动的node的rpc地址。如果farmer和node同机器，默认ws://0.0.0.0:9944即可，如果不同机器，则设置为node所在机器的ip地址:ws://x.x.x.x:9944

--reward-address WALLET_ADDRESS为您的获取reward的钱包地址，具体获取方式参见：[Wallets | Farm from Anywhere (subspace.network)](https://docs.subspace.network/docs/category/wallets)

path为plot的存储目录，plot文件的大小通过size设置。

这里path可以设置多个，以空格分隔，比如：

path=/var/subspace/data,size=100G path=/var/subspace/data2,size=120G path=/var/subspace/data3,size=1T



### 三、优化farmer

如果想充分发挥多核CPU的强大能力，更快的完成plotting，farmer也提供了相关的option用于自定义或优化。

具体的option可以通过help查询：

```
./subspace-farmer farm --help
```

我们这里主要关心cpu core相关的参数plotting-cpu-cores：

      --plotting-cpu-cores <PLOTTING_CPU_CORES>
          Specify exact CPU cores to be used for plotting bypassing any custom logic farmer might use otherwise. It replaces both
          `--sector-encoding-concurrency` and `--plotting-thread-pool-size` options if specified. Requires `--replotting-cpu-cores` to be
          specified with the same number of CPU cores groups (or not specified at all, in which case it'll use the same thread pool as
          plotting).
          
          Cores are coma-separated, with whitespace separating different thread pools/encoding instances. For example "0,1 2,3" will result in
          two sectors being encoded at the same time, each with a pair of CPU cores.

通过这个参数，可以将cpu中的多个core应用于一个farmer上，使其发挥最大效率。

比如您的cpu为24核心48线程处理器。

numa node两个，

core的分布为：

numa node1:  0-11,24-35

numa node2: 12-23,36-47

现在启动一个farmer, 同时plot 4个path.我想为每个path分配4个core，总共使用16个core, 则可以这样设置：

```bash
  ./subspace-farmer \
    farm \
    --plotting-cpu-cores '0-1,24-25 2-3,26-27 12-13,36-37 14-15,38-39'
    --node-rpc-url ws://0.0.0.0:9944 \
    --reward-address WALLET_ADDRESS \
    path=/var/subspace/data1,size=500G \
    path=/var/subspace/data2,size=500G \
    path=/var/subspace/data3,size=500G \
    path=/var/subspace/data4,size=500G
```

参数值**0-1,24-25 2-3,26-27 12-13,36-37 14-15,38-39**以空格分隔成4组

这里的数字对应的就是numa node1:  0-11,24-35,numa node2: 12-23,36-47中的core编号

注意同属于一个numa node的core要分配在一组，且有对应关系，

如**0-1,24-25**，都属于numa node1, 并且0对应24，1对应25。

而**14-15,38-39**则属于numa node2，14对应38，15对应39。以此类推。

查看cpu numa信息可以使用lscpu命令



### 四、安装监控仪表盘

#### 1. 给需要监控的node和farmer添加启动参数用以开启metrics

   for a node: --prometheus-listen-on 0.0.0.0:9080

   for a farmer: --metrics-endpoints 0.0.0.0:9081
   如果同机器起多个farmer，则需要给每个farmer设置不同端口，如0.0.0.0:9081，0.0.0.0:9082，0.0.0.0:9083......



#### 2. docker compose方式启动Grafana和Prometheus

   启动docker compose之前，需要在宿主机上设置grafana的data目录和prometheus配置

   a.  新建一个工作目录如 $HOME/subspace_monitor

   b.  在此目录下创建文件docker-compose.yaml和prometheus.yml，可直接使用后面的链接中的文件（[点击docker-compose.yaml](docker-compose.yaml), [点击prometheus.yml](prometheus.yml)）, 创建子目录  grafana/

   c.  找到prometheus.yml文件，在scrape_configs添加修改对应需要监控的目标地址，可设置多个地址，以逗号分隔:
```bash
- job_name: "subspace"
  static_configs:
    - targets: ["localhost:9080"]
      labels:
        group: 'node'
    - targets: ["localhost:9081","localhost:9082"]
      labels:
        group: 'farmer'


```

  d. 启动容器：docker compose up -d



#### 3. 配置Grafana

浏览器打开：http://主机IP:3000, 默认用户名密码 admin/admin

a. 首先配置data source, 选择prometheus:
    <image src="images/image-1.png">
    因为是docker compose 启动，这里prometheus server url用container name而不是ip : http://prometheus:9090

b. Import Grafana dashbord

输入Grafana仪表板ID=20433，然后单击 “加载”:

<image src="images/image-2.png">

如果页面一直没数据，则可以打开promeheus的web页面，看状态是否都正常（up）
地址：http://主机ip:9090
<image src="images/image-4.png">

c. 访问Grafana dashbord 
<image src="images/image-3.png">

