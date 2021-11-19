
##资源背景说明##

A.k8s 版本：V1.22.2 #2021/9当前最新版本；

B.操作系统： Centos7.8 64bit；

C.虚机：购买的腾讯云CVM虚拟机，其中Master节点：4C8G，Node节点：2C4G，都具备外网访问功能，方便装镜像；

1.机器标准化

部署k8s集群，linux机器必须要先进行标准化配置才能通过集群创建之前的precheck阶段，主要包括hostname，yum源，firewall等配置项；

1.1 关闭防火墙

systemctl stop firewalld && systemctl disable firewalld 
1.2 关闭selinux

sed -i 's/SELINUX=enforcing/SELINUX=disabled/g'/etc/sysconfig/selinux 
setenforce 0 
1.3 配置iptables

cat <<EOF >  /etc/sysctl.d/k8s.conf 
net.bridge.bridge-nf-call-ip6tables = 1 
net.bridge.bridge-nf-call-iptables = 1 
EOF 
sysctl -p #加载生效
1.4 关闭swap分区

swapoff -a 
vi /etc/fstab    #注释swap配置那一行 
free -m  #没有swap显示，说明配置成功；
1.5 修改hostname

echo "" > /etc/hostname 
echo $target_hostname > /etc/hostname    #确保每台唯一,不能有_字符，建议全部小写； 
sed -i 's/$current_hostname/$target_hostname/g' /etc/hosts     #修改host解析文件； 
cat /etc/hostname > /proc/sys/kernel/hostname   #确保修改后永久生效
systemctl restart network 
#退出当前的ssh登录然后重新登录，就能看到修改后的hostname；

1.6 安装docker

通过yum方式安全docker，默认部署docker1.13.1版本，如果觉得版本不符合要求，可以手动下载tar包安装；

yum install -y docker
#使用中科大作为新的docker镜像源；
vim /etc/docker/daemon.json  
#写入如下镜像网址： {"registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"]}
配置完成后，让docker开机自动启动；

systemctl enable docker && systemctl start docker
1.7 安装kubeadm套件

国内的机器机器由于某些不可描述的原因，没办法直接访问google上的k8s镜像源，单独配置k8s的yum源后，使用yum的方式自动安装当前最新版本的kubelet，kubectl，kubeadm等套件；

vim /etc/yum.repos.d/kubernetes.repo 
[kubernetes] 
name=kubernetes 
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64 
enabled=1 
gpgcheck=1 
repo_gpgcheck=1 
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg 
更新yum源信息：

yum clean all
yum makecache
yum repolist
安装kubeadm套件：

yum install -y kubelet kubeadm kubectl kubernetes-cni
安装完成后检查版本为1.22.2：

[root@master ~]# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.2", GitCommit:"8b5a19147530eaac9476b0ab82980b4088bbc1b2", GitTreeState:"clean", BuildDate:"2021-09-15T21:37:34Z", GoVersion:"go1.16.8", Compiler:"gc", Platform:"linux/amd64"}
[root@master ~]# kubelet --version
Kubernetes v1.22.2
此时kubelet因为没有配置文件，状态无法加载是正常的，后面使用kubeadm init后会自动拉起；

1.8 cgourpfs配置【可选】

docker默认使用cgroup做资源限制，正常情况下需要配置系统为cgroupfs，不过以我的实际部署过程来看，这里使用默认的systemd也可以搭建，只要kubelet和docker保持一致就行；

vim /usr/lib/systemd/system/docker.service
将native.cgroupdriver从systemd改为cgroupfs；
######################################

机器标准化对master和所有的node节点都需要执行；

######################################

2.搭建master节点

前面已经讲机器标准化做完了，现在就使用kubeadm来真正的进行k8s master的一键部署；

2.1 kubeadm init

使用kubeadm真的只用这一条命令就可以了，区别是你可以通过配置文件的方式来指定多个参数，或者直接在命令行中指定配置参数：

A. 命令行：

对于国内的朋友，无法直接访问google的镜像源，因此image-repsitory这个选项必须指定，否则会因为无法访问到http://gcr.io，下载镜像失败而部署失败；其他的参数通过字面意思很好理解，看自己是否有这个特殊需求，没有的话可以不用写；

#如果你的机器有多网卡，建议指定apiserver；

kubeadm init --kubernetes-version=1.22.2 \
--image-repository=registry.aliyuncs.com/google_containers \
--service-cidr=10.1.0.0/16 \
--pod-network-cidr=10.244.0.0/16 \
--apiserver-advertise-address=192.168.1.2
B. 配置文件

命令行方式指定参数执行起来非常冗余，尤其参数比较多的情况，此时可以将所有的配置要求写到配置文件中，部署的时候指定对应的配置文件即可，假设取名kubeadm.yaml：

apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
controllerManager:
  ExtraArgs:
    horizontal-pod-autoscaler-use-rest-clients: "true"
    horizontal-pod-autoscaler-sync-period: "10s"
    node-monitor-grace-period: "10s"
apiServer:
  ExtraArgs:
    runtime-config: "api/all=true"
kubernetesVersion: "v1.22.2"
imageRepository: registry.aliyuncs.com/google_containers
networking:
  podSubnet: x.x.x.x/x
写好配置文件后，直接执行：

kubeadm init --config kubeadm.yaml
无论是命令行，还是配置文件的方式，整个部署过程几分钟就完成了，我部署使用的是命令行方式，执行过程如下：

[root@master kubernetes]# kubeadm init --image-repository=registry.aliyuncs.com/google_containers
[init] Using Kubernetes version: v1.22.2
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master] and IPs [10.96.0.1 10.114.1.39]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost master] and IPs [10.114.1.39 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost master] and IPs [10.114.1.39 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 7.003706 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.22" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node master as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: ajk7gh.zqo5ldmo84subl5h
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.114.1.39:6443 --token ajk7gh.zqo5ldmo84subl5h \
	--discovery-token-ca-cert-hash sha256:24e85b981d7ebc7999b4e073bbd0e58fd5652c87e75eaca6bd7adeab2fcb9c73 
注意事项：

A. 将最后一行的kubeadm join字段cp出来单独保存，因为后面node节点加入需要用到这里的token信息；

B.如果部署过程中因为kubelet进程启动失败等某些原因导致部署失败，不需要重装装机，使用kubeadm reset命令清除已有的部署痕迹，反复重试发起部署，非常好用；

kubeadm reset
执行完成后检查maste节点状态：

[root@master kubernetes]# kubectl get nodes
NAME     STATUS     ROLES                  AGE   VERSION
master   NotReady   control-plane,master   86s   v1.22.2
此时状态NotReady是正常的，暂时不用管，有兴趣的可以通过kubectl describe命令来查看具体的信息；

检查kube组件的状态：

[root@master kubernetes]# kubectl get pods -n kube-system
NAME                             READY   STATUS    RESTARTS   AGE
coredns-7f6cbbb7b8-swg7n         0/1     Pending   0          104s
coredns-7f6cbbb7b8-xcfc9         0/1     Pending   0          104s
etcd-master                      1/1     Running   0          112s
kube-apiserver-master            1/1     Running   0          109s
kube-controller-manager-master   1/1     Running   0          109s
kube-proxy-thvxf                 1/1     Running   0          105s
kube-scheduler-master            1/1     Running   0          109s
coredns由于依赖网络组件，当前处于pending状态是正常符合预期的；

配置初始化：

在部署过程中会打印如下三条命令，照做即可；

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
2.2 安装weave网络组件

部署网络插件也非常简单，只需要执行一条命令即可；

kubectl apply -n kube-system -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
执行完成后检查：

[root@master ~]# kubectl get pods -A
NAMESPACE              NAME                                         READY   STATUS    RESTARTS      AGE
kube-system            coredns-7f6cbbb7b8-swg7n                     1/1     Running   0             24h
kube-system            coredns-7f6cbbb7b8-xcfc9                     1/1     Running   0             24h
kube-system            etcd-master                                  1/1     Running   0             24h
kube-system            kube-apiserver-master                        1/1     Running   0             24h
kube-system            kube-controller-manager-master               1/1     Running   0             24h
kube-system            kube-proxy-ldr5n                             1/1     Running   0             75m
kube-system            kube-proxy-thvxf                             1/1     Running   0             24h
kube-system            kube-scheduler-master                        1/1     Running   0             24h
kube-system            weave-net-5qmcs                              2/2     Running   0             75m
kube-system            weave-net-kch9p                              2/2     Running   1 (20h ago)   20h
此时可以看到coredns对应的容器状态已经切换为running；

2.3 安装dashboard可视化组件

同样是执行一条命令就能安装完成；

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc6/aio/deploy/recommended.yaml
执行完成后，检查组件状态，确保处于runnjing：

[root@master ~]# kubectl get pods -A
NAMESPACE              NAME                                         READY   STATUS    RESTARTS      AGE
kube-system            coredns-7f6cbbb7b8-swg7n                     1/1     Running   0             25h
kube-system            coredns-7f6cbbb7b8-xcfc9                     1/1     Running   0             25h
kube-system            etcd-master                                  1/1     Running   0             25h
kube-system            kube-apiserver-master                        1/1     Running   0             25h
kube-system            kube-controller-manager-master               1/1     Running   0             25h
kube-system            kube-proxy-ldr5n                             1/1     Running   0             139m
kube-system            kube-proxy-thvxf                             1/1     Running   0             25h
kube-system            kube-scheduler-master                        1/1     Running   0             25h
kube-system            weave-net-5qmcs                              2/2     Running   0             139m
kube-system            weave-net-kch9p                              2/2     Running   1 (21h ago)   21h
kubernetes-dashboard   dashboard-metrics-scraper-6ddd77bc75-dvmsw   1/1     Running   0             20h
kubernetes-dashboard   kubernetes-dashboard-656997fc56-ch2sl        1/1     Running   0             20h
如果想通过当前机器的IP直接访问dashboard主页的话，需要将默认的clusterIP方式配置为Nodeport(type:Nodeport)，并手动指定访问的端口为32443(nodeport:32443)；

[root@master ~]# kubectl edit svc -n kubernetes-dashboard kubernetes-dashboard

# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"k8s-app":"kubernetes-dashboard"},"name":"kubernetes-dashboard","namespace":"kubernetes-dashboard"},"spec":{"ports":[{"port":443,"targetPort":8443}],"selector":{"k8s-app":"kubernetes-dashboard"}}}
  creationTimestamp: "2021-09-28T08:34:45Z"
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
  resourceVersion: "25687"
  uid: d2924d34-86bf-4ea7-8982-2d672c5d93aa
spec:
  clusterIP: 10.111.122.194
  clusterIPs:
  - 10.111.122.194
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - nodePort: 32443
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
修改完成后，service会自动加载生效，检查修改后的服务状态：

[root@master ~]# kubectl get svc -n kubernetes-dashboard
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
dashboard-metrics-scraper   ClusterIP   10.101.199.8     <none>        8000/TCP        20h
kubernetes-dashboard        NodePort    10.111.122.194   <none>        443:32443/TCP   20h
获取dashborad主页访问所需的token信息，执行如下两条命令：

kubectl get secret -n kubernetes-dashboard | grep token
kubectl describe secret -n kubernetes-dashboard kubernetes-dashboard-token-r6cmj
token信息保存在secret对应的pod当中，通过describe获取后复制出来：

[root@master ~]# kubectl get secret -n kubernetes-dashboard | grep token
default-token-4mkzg                kubernetes.io/service-account-token   3      20h
kubernetes-dashboard-token-r6cmj   kubernetes.io/service-account-token   3      20h
[root@master ~]# 
[root@master ~]# kubectl describe secret -n kubernetes-dashboard kubernetes-dashboard-token-r6cmj
Name:         kubernetes-dashboard-token-r6cmj
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: kubernetes-dashboard
              kubernetes.io/service-account.uid: 840b0795-5321-4fef-9df8-f2288eb4f256

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1099 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6Ijh3Wm1DbkVoOGw2YWdxZTF6UEZNZy0xc19oSGpEYkZDRkxxcG
通过手动指定的nodeport32443端口访问主页，粘贴刚刚获取的token，点击登录：



此时，主页就能正常对外访问了，只是没有数据而已；

3.加入node节点

###如果node节点已经按照第1章的部分完成机器的标准化，那么加入master集群非常简单，就几分钟时间；

在node节点上，执行kubeadm join命令，示例如下：

kubeadm join 10.114.1.39:6443 --token ajk7gh.zqo5ldmo84subl5h --discovery-token-ca-cert-hash sha256:24e85b981d7ebc7999b4e073bbd0e58fd5652c87e75eaca6bd7adeab2fcb9c73
执行完毕后等待2分钟，回到master节点检查，即可看到node节点已加入并且状态流转为Ready；

[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES                  AGE     VERSION
master   Ready    control-plane,master   22h     v1.22.2
node1    Ready    <none>                 4m46s   v1.22.2
#注意：这里的token信息只有24h有效，如果master节点部署完毕后没有保证这个token，或者说token过期了，手动执行命令重新生成即可；

[root@master ~]# kubeadm token create --print-join-command
kubeadm join 10.114.1.39:6443 --token pjaitq.996gqzivvqyrngqy --discovery-token-ca-cert-hash sha256:24e85b981d7ebc7999b4e073bbd0e58fd5652c87e75eaca6bd7adeab2fcb9c73
至此：包含一个master，一个node节点的k8s集群就搭建好了，你可以在这里非常方便部署你的应用；
