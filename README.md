# dockernetwork
转载请注明出处：http://blog.csdn.net/gamer_gyt 
博主微博：http://weibo.com/234654758 
Github：https://github.com/thinkgamer

参考资料

1：容器网络那些事儿 
2：使用 Docker 容器网络 
3：Docker 1.9的新网络特性，以及Overlay详解（1） 
4：容器互联 
5：docker的网络-Container network interface(CNI)与Container network model(CNM)

写在前边的话

      突然发现好久没有更新博客了，像我这种频繁发表博客的人竟然也会放慢了更新的速度，其实不是说自己不去写，不去更新，只是不愿意去将就，去发表一些让别人看了没有多大帮助的文章，作为2017年的开篇博客，我想和你们一起学习下Docker容器网络的知识，首先声明，以下内容大部分都是来源网络，按照我对docker网络的理解，整理的一篇文章，一起学习。

docker容器网络概述

1：默认网络

      在默认情况下会看到三个网络，它们是Docker Deamon进程创建的。它们实际上分别对应了Docker过去的三种『网络模式』，可以使用docker network ls来查看

master@ubuntu:~$ sudo docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
18d934794c74        bridge              bridge              local
f7a7b763f013        host                host                local
697354257ae3        none                null                local

      这 3 个网络包含在 Docker 实现中。运行一个容器时，可以使用 the –net标志指定您希望在哪个网络上运行该容器。您仍然可以使用这 3 个网络。

bridge 网络表示所有 Docker 安装中都存在的 docker0 网络。除非使用 docker run –net=选项另行指定，否则 Docker 守护进程默认情况下会将容器连接到此网络。在主机上使用 ifconfig命令，可以看到此网桥是主机的网络堆栈的一部分。
none 网络在一个特定于容器的网络堆栈上添加了一个容器。该容器缺少网络接口。
host 网络在主机网络堆栈上添加一个容器。您可以发现，容器中的网络配置与主机相同。
2：自定义网络

      当然你也可以自定义网络来更好的隔离容器，Docker 提供了一些默认网络驱动程序来创建这些网络。您可以创建一个新 bridge 网络或覆盖一个网络。也可以创建一个网络插件或远程网络并写入您自己的规范中。您可以创建多个网络。可以将容器添加到多个网络。容器仅能在网络内通信，不能跨网络进行通信。一个连接到两个网络的容器可与每个网络中的成员容器进行通信。当一个容器连接到多个网络时，外部连接通过第一个（按词典顺序）非内部网络提供。

(1)：docker network 命令

执行 sudo docker network –help

master@ubuntu:~$ sudo docker network --help

Usage:  docker network COMMAND

Manage Docker networks

Options:
      --help   Print usage

Commands:
  connect     Connect a container to a network
  create      Create a network
  disconnect  Disconnect a container from a network
  inspect     Display detailed information on one or more networks
  ls          List networks
  rm          Remove one or more networks

Run 'docker network COMMAND --help' for more information on a command.

19
(2)：创建test-network网络

执行命令： sudo docker network create test-network 
查看： sudo docker network ls

master@ubuntu:~$ sudo docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
18d934794c74        bridge              bridge              local
f7a7b763f013        host                host                local
697354257ae3        none                null                local
c4f6d347c8b4        test-network        bridge              local

查看自己创建的网络的信息

master@ubuntu:~$ sudo docker network inspect test-network
[
    {
        "Name": "test-network",
        "Id": "c4f6d347c8b47471b97e1b5621dd2e90aff303bb7db632db86b0bbec6ffb91d4",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1/16"
                }
            ]
        },
        "Internal": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]

另外，还可以采用其他一些选项，比如 –subnet、–gateway和 –ip-range。

(3)：启动容器连接到test-network

sudo docker run -itd –name=test –net=test-network lt:1.0 /bin/bash

master@ubuntu:~$ sudo docker run -itd --name=test  --net=test-network lt:1.0 /bin/bash
09a9d7a9c37d691e0fc0f7cfdf3c9470b77f410592f9bf624fe90bff2b17e315
1
2
1
2
再次查看信息，可以看到挂载的容器

master@ubuntu:~$ sudo docker network inspect test-network
[
    {
        "Name": "test-network",
        "Id": "c4f6d347c8b47471b97e1b5621dd2e90aff303bb7db632db86b0bbec6ffb91d4",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1/16"
                }
            ]
        },
        "Internal": false,
        "Containers": {
            "09a9d7a9c37d691e0fc0f7cfdf3c9470b77f410592f9bf624fe90bff2b17e315": {
                "Name": "test",
                "EndpointID": "7d1d57418a8ebde2d5404f37e27de68be41a917979fd74f394bc5fccc2601e08",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]

当然也可以动态的将容器挂载到某个网络上

master@ubuntu:~$ sudo docker run -itd --name=test1 lt:1.0 /bin/bash
b2aba703c5180819542d26e7bda784774ca87b896e8df612dd1b727218dde334
master@ubuntu:~$ sudo docker network connect test-network test1
master@ubuntu:~$ sudo docker network inspect test-network
[
    {
        "Name": "test-network",
        "Id": "c4f6d347c8b47471b97e1b5621dd2e90aff303bb7db632db86b0bbec6ffb91d4",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1/16"
                }
            ]
        },
        "Internal": false,
        "Containers": {
            "09a9d7a9c37d691e0fc0f7cfdf3c9470b77f410592f9bf624fe90bff2b17e315": {
                "Name": "test",
                "EndpointID": "7d1d57418a8ebde2d5404f37e27de68be41a917979fd74f394bc5fccc2601e08",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            },
            "b2aba703c5180819542d26e7bda784774ca87b896e8df612dd1b727218dde334": {
                "Name": "test1",
                "EndpointID": "5d18b876b62312698d4ce253e33b52552c9a3b1237bed90ec565d4cbd9f5710d",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]

“石器时代”的容器网络模型

      目前对于刚起步接触docker容器的筒子们（当然也包括我），大部分使用网络的方式应该是这样的，把需要暴漏的端口做端口映射（docker run -itd -p 81:80 …），例如一个主机内有很多Apache容器，每一个Apache要往外抛80的端口，那我怎么办？我需要针对第一个容器和主机80端口做映射，第二个和主机81端口做映射，依此类推，到最后发现非常混乱，没办法管理。这样的容器网络模型对于企业来说是基本没办法被采用。

这里写图片描述

      这是石器时代网络模型，它是Docker1.9之前的容器网络，实现方式是只针对单台主机进行IPAM管理，所有主机上的容器都会连接到主机内部的一个Linux Bridge，叫Docker0，主机的IP它默认会分配172.17网段中的一个IP，因为有Docker0，所以在一个主机上的容器可以实现互联互通。但是因为IP分配的范围是基于单主机的，所以你会发现在其他主机上，也会出现完全相同的IP地址。很明显，这两个地址肯定没办法直接通信。为了解决这个问题，在石器时代我们会用端口映射，实际上就是NAT的方法。比如说我有一个应用，它有Web和MySQL，分别在不同的主机上，Web需要去访问mysql，我们会把这个Mysql的3306端口映射到主机上的3306这个端口，然后这个服务实际上是去访问主机IP 10.10.10.3 的3306端口，这是过去的石器时代的一个做法。

      总结一下它的典型技术特征：基于单主机的IPAM；主机之内容器通讯实际上通过一个docker0的linux Bridge；如果服务想要暴露到外部的话需要做NAT，会导致端口争抢非常严重；当然它有一个好处，对大网IP消耗比较少。

docker容器的单主机通信

      docker容器的单主机通信使用的是容器互联技术，即 –link，映射网络端口不是吧Container彼此连接起来的唯一方法。Docker的linking系统允许你吧多个 container连接起来， 让他们彼此交互信息。Docker的linking会创建一种父子级别的关系。 父container可以看到他的子container提供的信息。 
创建一个数据库容器：

sudo docker run -d –name db training/postgres
然后创建一个新的 web 容器，并将它连接到 db 容器

sudo docker run -d -P –name web –link db:db training/webapp Python app.py
此时，db 容器和 web 容器建立互联关系。 
–link 参数的格式为 –link name:alias，其中 name 是要链接的容器的名称，alias 是这个连接的别名。

docker容器的跨主机通信

      早期大家的跨主机通信方案主要有以下几种：

容器使用host模式：容器直接使用宿主机的网络，这样天生就可以支持跨主机通信。虽然可以解决跨主机通信问题，但这种方式应用场景很有限，容易出现端口冲突，也无法做到隔离网络环境，一个容器崩溃很可能引起整个宿主机的崩溃。
端口绑定：通过绑定容器端口到宿主机端口，跨主机通信时，使用主机IP+端口的方式访问容器中的服务。显而易见，这种方式仅能支持网络栈的四层及以上的应用，并且容器与宿主机紧耦合，很难灵活的处理，可扩展性不佳。
docker外定制容器网络：在容器通过docker创建完成后，然后再通过修改容器的网络命名空间来定义容器网络。典型的就是很久以前的pipework，容器以none模式创建，pipework通过进入容器的网络命名空间为容器重新配置网络，这样容器网络可以是静态IP、vxlan网络等各种方式，非常灵活，容器启动的一段时间内会没有IP，明显无法在大规模场景下使用，只能在实验室中测试使用。
第三方SDN定义容器网络：使用Open vSwitch或Flannel等第三方SDN工具，为容器构建可以跨主机通信的网络环境。这些方案一般要求各个主机上的docker0网桥的cidr不同，以避免出现IP冲突的问题，限制了容器在宿主机上的可获取IP范围。并且在容器需要对集群外提供服务时，需要比较复杂的配置，对部署实施人员的网络技能要求比较高。
      上面这些方案有各种各样的缺陷，同时也因为跨主机通信的迫切需求，docker 1.9版本时，官方提出了基于vxlan的overlay网络实现，原生支持容器的跨主机通信。同时，还支持通过libnetwork的plugin机制扩展各种第三方实现，从而以不同的方式实现跨主机通信。就目前社区比较流行的方案来说，跨主机通信的基本实现方案有以下几种：

基于隧道的overlay网络：按隧道类型来说，不同的公司或者组织有不同的实现方案。docker原生的overlay网络就是基于vxlan隧道实现的。ovn则需要通过geneve或者stt隧道来实现的。flannel最新版本也开始默认基于vxlan实现overlay网络。
基于包封装的overlay网络：基于UDP封装等数据包包装方式，在docker集群上实现跨主机网络。典型实现方案有weave、flannel的早期版本。
基于三层实现SDN网络：基于三层协议和路由，直接在三层上实现跨主机网络，并且通过iptables实现网络的安全隔离。典型的方案为Project Calico。同时对不支持三层路由的环境，Project Calico还提供了基于IPIP封装的跨主机网络实现。
docker容器的CNI模型和CNM模型

      目前围绕着docker的网络，目前有两种比较主流的声音，docker主导的Container network model(CNM)和社区主导的Container network interface(CNI)。

1：CNI

(1) 概述

      Container Networking Interface(CNI)提供了一种linux的应用容器的插件化网络解决方案。最初是由rkt Networking Proposal发展而来。也就是说，CNI本身并不完全针对docker的容器，而是提供一种普适的容器网络解决方案。因此他的模型只涉及两个概念:

      容器(container) : 容器是拥有独立linux网络命名空间的独立单元。比如rkt/docker创建出来的容器。 
      这里很关键的是容器需要拥有自己的linux网络命名空间。这也是加入网络的必要条件。

      网络(network): 网络指代了可以相互联系的一组实体。这些实体拥有各自独立唯一的ip。这些实体可以是容器，是物理机，或者其他网络设备(比如路由器)等。

(2) 接口及实现

      CNI的接口设计的非常简洁，只有两个接口ADD/DELETE。

      以 ADD接口为例

      Add container to network

      参数主要包括:

Version. CNI版本号
Container ID. 这是一个可选的参数，提供容器的id
Network namespace path. 容器的命名空间的路径，比如 /proc/[pid]/ns/net。
Network configuration. 这是一个json的文档，具体可以参看network-configuration
Extra arguments. 其他参数
Name of the interface inside the container. 容器内的网卡名
      返回值:

IPs assigned to the interface. ipv4或者ipv6地址
DNS information. DNS相关信息
2：CNM

      相较于CNI，CNM是docker公司力推的网络模型。其主要模型如下图：

这里写图片描述

Sandbox

      Sandbox包含了一个容器的网络栈。包括了管理容器的网卡，路由表以及DNS设置。一种Sandbox的实现是通过linux的网络命名空间，一个FreeBSD Jail 或者其他类似的概念。一个Sandbox可以包含多个endpoints。

Endpoint

      一个endpoint将Sandbox连接到network上。一个endpoint的实现可以通过veth pair，Open vSwitch internal port 或者其他的方式。一个endpoint只能属于一个network，也只能属于一个sandbox。

Network

      一个network是一组可以相互通信的endpoints组成。一个network的实现可以是linux bridge，vlan或者其他方式。一个网络中可以包含很多个endpoints。

接口

      CNM的接口相较于CNI模型，较为复杂。其提供了remote plugin的方式，进行插件化开发。remote plugin相较与CNI的命令行，更加友好一些，是通过http请求进行的。remote plugin监听一个指定的端口，docker daemon直接通过这个端口与remote plugin进行交互。

鉴于CNM的接口较多，这里就不一一展开解释了。这里主要介绍下在进行docker的操作中，docker daemon是如何同CNM插件繁盛交互。

调用过程

Create Network
      这一系列调用发生在使用docker network create的过程中。

      /IpamDriver.RequestPool: 创建subnetpool用于分配IP 
      /IpamDriver.RequestAddress: 为gateway获取IP 
      /NetworkDriver.CreateNetwork: 创建neutron network和subnet

Create Container
      这一系列调用发生在使用docker run，创建一个contain的过程中。当然，也可以通过docker network connect触发。

      /IpamDriver.RequestAddress: 为容器获取IP 
      /NetworkDriver.CreateEndpoint: 创建neutron port 
      /NetworkDriver.Join: 为容器和port绑定 
      /NetworkDriver.ProgramExternalConnectivity: 
      /NetworkDriver.EndpointOperInfo

Delete Container
      这一系列调用发生在使用docker delete，删除一个contain的过程中。当然，也可以通过docker network disconnect触发。

      /NetworkDriver.RevokeExternalConnectivity 
      /NetworkDriver.Leave: 容器和port解绑 
      /NetworkDriver.DeleteEndpoint 
      /IpamDriver.ReleaseAddress: 删除port并释放IP

Delete Network
      这一系列调用发生在使用docker network delete的过程中。

      /NetworkDriver.DeleteNetwork: 删除network 
      /IpamDriver.ReleaseAddress: 释放gateway的IP 
      /IpamDriver.ReleasePool: 删除subnetpool

3：CNI与CNM的转化

      CNI和CNM并非是完全不可调和的两个模型。二者可以进行转化。比如calico项目就是直接支持两种接口模型。

      从模型中来看，CNI中的container应与CNM的sandbox概念一致，CNI中的network与CNM中的network一致。在CNI中，CNM中的endpoint被隐含在了ADD/DELETE的操作中。CNI接口更加简洁，把更多的工作托管给了容器的管理者和网络的管理者。从这个角度来说，CNI的ADD/DELETE接口其实只是实现了docker network connect和docker network disconnect两个命令。

      kubernetes/contrib项目提供了一种从CNI向CNM转化的过程。其中原理很简单，就是直接通过shell脚本执行了docker network connect和docker network disconnect命令，来实现从CNI到CNM的转化。
