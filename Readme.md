Ceph 15 (Octopus)- ubuntu20.04

1.基础环境配置

1.1 主机规划
节点名称
IP
CPU
内存
系统盘
ceph硬盘
ceph角色
k8s01
192.168.1.210
4c
8G
40G
60G
mon,osd
k8s02
192.168.1.211
4c
8G
40G
60G
mon,osd
k8s03
192.168.1.212
4c
8G
40G
60G
osd
k8s04
192.168.1.213
4c
8G
40G
60G
mon,osd

1.2 配置服务器配置文件/etc/hosts（所有主机）
192.168.1.210 k8s01
192.168.1.211 k8s02
192.168.1.212 k8s03
192.168.1.213 k8s04

1.3 安装chrony时间同步服务（所有主机）
apt install chrony -y

1.4 配置ssh免密 （k8s01主机到其他节点root用户免密登录）
1.所有节点配置ssh容许root登陆（所有节点）
// 编辑/etc/ssh/sshd_config，修改PermitRootLogin为yes，修改完后，执行systemctl restart sshd.service或者/etc/init.d/ssh restart重启ssh服务。

PermitRootLogin yes

2.生成密钥
ssh-keygen -t rsa -b 4096 -N '' -f ~/.ssh/id_rsa

 3.ssh配置
tee -a ~/.ssh/config<<EOF
Host *
    UserKnownHostsFile /dev/null
    StrictHostKeyChecking no
    IdentitiesOnly yes
    ConnectTimeout 0
    ServerAliveInterval 300
EOF

4.公钥发布
for i in {1..4}; do ssh-copy-id k8s0$i; done;

5.验证使用root用户
ssh $HOSTNAME 能成功即可

2.引导新的集群（k8s01）
1.创建配置文件目录（重建时需要删除改目录内的文件）
mkdir -p /etc/ceph

2.引导集群（创建第一个 mon 节点）

cephadm bootstrap \
  --mon-ip 192.168.1.210 \
  --initial-dashboard-user admin \
  --initial-dashboard-password supernode@2021
  



3.安装ceph工具（所有节点）
cephadm add-repo --release octopus
cephadm install ceph-common

备注：里面包含了所有的ceph命令，其中包括ceph，rbd，mount.ceph（用于安装CephFS文件系统）等

4.部署 mon 节点
1.复制Ceph SSH密钥
ssh-copy-id -f -i /etc/ceph/ceph.pub root@k8s02
ssh-copy-id -f -i /etc/ceph/ceph.pub root@k8s04

2.禁用自动监视器部署
ceph orch apply mon --unmanaged

3.增加节点到集群
ceph orch host add k8s02
ceph orch host add k8s04

4.标记mon节点
ceph orch host label add k8s01 mon
ceph orch host label add k8s02 mon
ceph orch host label add k8s04 mon

//查看标记结果
ceph orch host ls

HOST   ADDR   LABELS  STATUS
k8s01  k8s01  mon
k8s02  k8s02  mon
k8s05  k8s04  mon

5.配置cephadm根据标签部署监视器
ceph orch apply mon label:mon

6.确保将监视器应用于这三个主机
ceph orch apply mon "k8s01,k8s02,k8s04"



5.部署Ceph OSD节点（k8s01）
1.复制Ceph SSH密钥
ssh-copy-id -f -i /etc/ceph/ceph.pub root@k8s03

2.增加节点到集群
ceph orch host add k8s03

3.标记osd节点
ceph orch host label add k8s01 osd
ceph orch host label add k8s02 osd
ceph orch host label add k8s03 osd
ceph orch host label add k8s04 osd

4.查看标记结果
ceph orch host ls

HOST   ADDR   LABELS  STATUS
k8s01  k8s01  mon,osd
k8s02  k8s02  mon,osd
k8s03  k8s03  osd
k8s04  k8s04  mon,osd

6.查看存储节点上的所有设备( k8s01 稍等片刻20分钟左右)
ceph orch device ls

7.部署OSD（k8s01）
ceph orch daemon add osd k8s01:/dev/sdb
ceph orch daemon add osd k8s02:/dev/sdb
ceph orch daemon add osd k8s03:/dev/sdb
ceph orch daemon add osd k8s04:/dev/sdb
访问链接
https://192.168.1.210:8443
