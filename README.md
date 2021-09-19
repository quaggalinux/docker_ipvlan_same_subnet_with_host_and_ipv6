# docker_ipvlan_same_subnet_with_host_and_ipv6  
docker容器配置ipvlan及设置容器与宿主机同一ipv4网段并通信，另外还配置ipvlan获得公网IPv6地址  
  
前面我的repo已经分享了docker容器配置macvlan及设置容器与宿主机同一ipv4网段并通信，  
另外还配置macvlan获得公网IPv6地址的方法，下面再分享另外一个ipvlan模式，  
它也是解决容器实例的ipv4需要与宿主机同一网段，而且还有互通的需求，所以也把我成功的操作流程记录一下  
  
  
操作流程假设你已经从服务供应商获得ipv6公网地址段::/64，当前局域网的ip网段10.0.0.0/24  
部署容器的宿主机对外网卡为ens33，宿主机系统为ubuntu 18  
  
首先查看宿主机网卡ipv6地址段  
#ip -f inet6 addr show ens33  
  
选取的是公网IP，也就是后面scope global指示的，下面的ipv6子网为2a01:53c0:ff0e:2e::/64  
inet6 2a01:53c0:ff0e:2e::b139/128 scope global dynamic noprefixroute  
inet6 2a01:53c0:ff0e:2e:20c:29ff:feff:1453/64 scope global dynamic mngtmpaddr noprefixroute  
  
编辑或追加配置文件内容，至少要定义一个ipv6子网段，而且这个子网段必须在::/64内，否则报错  
#nano /etc/docker/daemon.json  
写入下面内容  
  
{  
"ipv6": true,  
"fixed-cidr-v6": "2a01:53c0:ff0e:2e:2::/80"  
}  
  
保存退出并重启容器服务  
#systemctl restart docker  
  
  
创建容器网络模式ipvlan并使用当前局域网的ip网段及公网ipv6网段，前提是必须配置好/etc/docker/daemon.json文件  
必须加上--ipv6参数，否则即使在管理界面看到网络模式名字分配了网段，而容器实例即使指定了ipv6地址，  
但容器实例ipv6地址实际为空，ipv6的网关参数可以不写，系统会自动指定::1，子网段不能用::/64，否则报错，  
也不要与已经定义的其他网络模式的网段重叠，  
--subnet：当前局域网的ip网段，--aux-address：排除本宿主机的ip地址，parent：指定使用的网卡名字  
如果已经创建macvlan，那么--subnet必须不重叠，但理论上如果都有这种与宿主机同一ipv4网段需求的话，  
各位只能选择一种方式了，因为两个模式都要写当前局域网的ipv4网段，但又必须不重叠，所以这个是悖论  
参数--aux-address：可以多次引用，列出所有排除的ip地址  
如--aux-address="my-router=10.0.0.5" --aux-address="my-switch=10.0.0.6" --aux-address="my-printer=192.170.1.5" --aux-address="my-nas=192.170.1.6"
  
#docker network create -d ipvlan --ipv6 --subnet=2a01:53c0:ff0e:2e:4::/80 --subnet=10.0.0.0/24 --gateway=10.0.0.1 --aux-address="exclude_host=10.0.0.206" -o parent=ens33 -o ipvlan_mode=l2 ip6ipvlan  
  
检查创建是否成功  
#docker network ls  
  
创建两个容器实例并使用ipvlan模式，指定ipv6地址，指定IPv4地址，如果不指定的话会自动分配，  
建议自己指定，可避免IPv4地址冲突问题，验证指定的ipv4地址是否能够与宿主机ipv4通信  
  
#docker run -dit --restart=always --network=ip6ipvlan --ip6=2a01:53c0:ff0e:2e:4::2 --ip=10.0.0.202 --name=u18ip6ipvlan2 -v /data:/data ubuntu:bionic-20210827 /bin/bash -c "/etc/init.d/cron start;/etc/init.d/run;/bin/bash"  
  
#docker run -dit --restart=always --network=ip6ipvlan --ip6=2a01:53c0:ff0e:2e:4::3 --ip=10.0.0.203 --name=u18ip6ipvlan3 -v /data:/data ubuntu:bionic-20210827 /bin/bash -c "/etc/init.d/cron start;/etc/init.d/run;/bin/bash"  
  
无论如何宿主机的原生ipv4是不能直接和本机新创建的容器通信的，所以另外创建一个ipvlan并把它设置成桥接组，  
如果各位没有宿主机与本机容器互通需求，那么可以忽略下面的配置了，  
因为这时容器已经可以和任何局域网内10.0.0.0/24网段的机器互通，除了宿主机  
#ip link add ipvlan0 link ens33 type ipvlan mode l2  
  
在ipvlan模式下必须设置这个接口ip地址，否则不能转发ip包，而且这个地址要与宿主机原来ip地址不同，  
以作为访问宿主机的ip地址，配置完成后容器实例仍然不能访问宿主机原来的ip地址，只能访问这个宿主机新加的ip地址  
#ip address add 10.0.0.207/24 dev ipvlan0  
  
#ip link set ipvlan0 up  
  
修改路由，使宿主机到上面容器ip地址10.0.0.202及203的通信全部转发给ipvlan0进行ip转发  
#ip route add 10.0.0.202 dev ipvlan0  
#ip route add 10.0.0.203 dev ipvlan0  
  
  
最后是在宿主机ping容器的ip地址及容器内部ping宿主机ip地址，验证配置正确  
  
由于ip命令在linux重启后会丢失，所以需要把命令写在宿主机的定时任务@reboot或启动脚本，然后设置开机执行这个脚本  
  
用了ipvlan模式后，ping的反应明显比macvlan快，而且像croc这个应用也完全没有问题，  
但缺点是宿主机必须多占用一个ip地址才能与容器通讯，而且ipvlan外访都是使用宿主机的mac地址，  
这样也可能导致有些应用有问题  
  
至于time curl -o -v github.com命令结果差别不大  
  
对于ipvlan模式下的容器ipv6地址不需要设置任何的ND proxy，ipv6互访性与普通局域网的ipv6机器一样，  
如果各位没有ipv6需求，那么只需要去掉上面流程中所有关于ipv6的配置及命令  
  

