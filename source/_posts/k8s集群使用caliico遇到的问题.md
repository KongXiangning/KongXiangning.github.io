title: k8s集群使用caliico遇到的问题
tags:
  - k8s
  - calico
  - ''
categories: []
date: 2018-10-12 10:52:00
---
搭建kubernetes集群，服务器由openstack提供，通过kubeadm安装高可用。采用coredns，calico网络组件。

# calico搭建：

## 一：calico服务启动
根据官网教程 <a href="https://docs.projectcalico.org/v3.2/getting-started/kubernetes/installation/calico">官方文档</a>  
先添加RBAC  

	kubectl apply -f \ https://docs.projectcalico.org/v3.2/getting-started/kubernetes/installation/rbac.yaml
    
下载calico.yaml模板并进行更改

	curl \ https://docs.projectcalico.org/v3.2/getting-started/kubernetes/installation/hosted/calico.yaml \ -O
vim calico.yaml
主要对ConfigMap部分进行配置,由于kubernetes集群使用的是外部etcd集群,calico也统一使用该etcd集群。  
calico.yaml中
> <pre><code>kind: ConfigMap
apiVersion: v1
metadata:
  name: calico-config
  namespace: kube-system
data:
  etcd_endpoints: "https://192.168.21.11:2379,https://192.168.21.12:2379,https://192.168.21.13:2379"
  etcd_ca: "/calico-secrets/etcd-ca"
  etcd_cert: "/calico-secrets/etcd-cert"
  etcd_key: "/calico-secrets/etcd-key"
  calico_backend: "bird"
  veth_mtu: "1440"
  cni_network_config: |-
    {
      "name": "k8s-pod-network",
      "cniVersion": "0.3.0",
      "plugins": [
        {
          "type": "calico",
          "log_level": "info",
          "etcd_endpoints": "https://192.168.21.11:2379,https://192.168.21.12:2379,https://192.168.21.13:2379",
          "etcd_key_file": "/etc/etcd/ssl/etcd-key.pem",
          "etcd_cert_file": "/etc/etcd/ssl/etcd.pem",
          "etcd_ca_cert_file": "/etc/etcd/ssl/ca.pem",
          "mtu": 1500,
          "ipam": {
              "type": "calico-ipam"
          },
          "policy": {
              "type": "k8s"
          },
          "kubernetes": {
              "kubeconfig": "/etc/kubernetes/kubelet.conf"
          }
        },
        {
          "type": "portmap",
          "snat": true,
          "capabilities": {"portMappings": true}
        }
      ]
   }
\---
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: calico-etcd-secrets
  namespace: kube-system
data:
  etcd-key: `cat /etc/etcd/ssl/etcd-key.pem | base64 | tr -d '\n'` #对应etcd的ssl文件路径
  etcd-cert: `cat /etc/etcd/ssl/etcd.pem | base64 | tr -d '\n'`
  etcd-ca: `cat /etc/etcd/ssl/ca.pem | base64 | tr -d '\n'`
</code></pre>  

接着执行

	kubectl apply -f calico.yaml
稍等一段时间可以看到coredns及calico服务都起来了

	kubectl get po -n kube-system
此时如果使用官方的测试教程进行测试,可以发现网络是正常的。

## 二: k8s高可用集群搭建完成后
将高可用的master节点起好后,calico的节点会自动部署至各个master节点上.此时会发现:calico-node的节点两个容器才有一个容器是启动成功的。  

	kubectl describe pod calico-node-id -n kube-system
看到错误:Readiness probe failed: caliconode is not ready: BIRD is not ready: BGP not established with 172.18.0.1
出现了奇怪的ip  
在三台master节点机中查找,发现node02中  

	32: br-fac7e8026658: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
    link/ether 02:42:7e:92:0b:14 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 scope global br-fac7e8026658
       valid_lft forever preferred_lft forever
    inet6 fe80::42:7eff:fe92:b14/64 scope link 
    valid_lft forever preferred_lft forever
    
现在还不知道这个ip是怎么产生的//TODO
   
##### 安装calicoctl工具: 
同样根据官方文档操作(注意版本对应)

	cd /usr/local/bin
    curl -O -L https://github.com/projectcalico/calicoctl/releases/download/v3.2.3/calicoctl
    chmod +x calicoctl

因为我们使用的是外部etcd集群,所以需要对calicoctl进行配置,使其能读取calico配置信息。配置文件的位置:/etc/calico/calicoctl.cfg   
模板:

> 	apiVersion: projectcalico.org/v3
	kind: CalicoAPIConfig
	metadata:
	spec:
  	etcdEndpoints: 		https://192.168.21.11:2379,https://192.168.21.12:2379,https://192.168.21.13:2379
  	etcdKeyFile: /etc/etcd/ssl/etcd-key.pem #对应etcd的ssl文件路径
  	etcdCertFile: /etc/etcd/ssl/etcd.pem
  	etcdCACertFile: /etc/etcd/ssl/ca.pem

接下来就可以使用calicoctl工具来对calico进行更改。

	calicoctl get nodes  #查看节点  **
	calicoctl get node node02 -o yaml
可以看见输出:

> 	apiVersion: projectcalico.org/v3
	kind: Node
	metadata:
  	creationTimestamp: 2018-10-12T08:16:05Z
  	name: cloud02
  	resourceVersion: "219244"
  	uid: 1087f828-cdf7-11e8-bc0e-fa163e88cdac
	spec:
  	bgp:
    	ipv4Address: 172.18.0.1/16
    	ipv4IPIPTunnelAddr: 192.168.160.128
  	orchRefs:
  	- nodeName: node02
    	orchestrator: k8s

这就确认了错误ip的节点

	calicoctl get node node02 -o yaml > caliconode02.yaml
    vim caliconode02.yaml
将ip更改正确

	calicoctl apply -f caliconode02.yaml
    kubectl get po -n kube-system
可以看到calico-node的节点都正常启动


## 备注:  
\* 此时用

	calicoctl node status
可以发现,并没有分配BGP对等体

	calicoctl get node nodeName
查看ipv4Address也是正确的  
\**  此时用

	calicoctl node status
可以看到BGP对等体已经分配,但是有错误

> 	Calico process is running.
	IPv4 BGP status
	+--------------+-------------------+-------+----------+--------------------------------+
	| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |              INFO              |
	+--------------+-------------------+-------+----------+--------------------------------+
	| 192.168.21.13  | node-to-node mesh | up    | 08:16:14 | Established                    |
	| 172.18.0.1   | node-to-node mesh | start | 09:22:01 | Active Socket: Connection      |
	|              |                   |       |          | refused                        |
	+--------------+-------------------+-------+----------+--------------------------------+
	IPv6 BGP status
	No IPv6 peers found.

其中Established即为连同状态.