---
layout: post
title: kubernetes自签证书
<!-- subtitle: -->
date: 2020-10-20
categories: kubernetes cert
cover: 'assets/img/kubernetes-cert.jpg'
tags: kubernetes certification 
---

> 在kubernetes中有很多证书，设计到的证书文件有20多个，这些证书可以分为三类：kubernetes集群证书，etcd集群证书，API聚合功能使用的front proxy证书。本文将对这三类证书进行分析并利用开源工具自己制作证书。


#### 1 证书制作工具

证书制作有2种工具openssl 、cfssl ，本文使用cfssl。
下载工具并赋予执行权限

```
curl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -o /usr/local/bin/cfssl
curl https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -o /usr/local/bin/cfssljson
curl https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -o /usr/local/bin/cfsslinfo
chmod +x /usr/local/bin/cfssl*
```

#### 2 ca证书制作
```
cat >/tmp/cert/config.json<<EOF
{
    "signing": {
        "default": {
            "expiry": "87600h"
        },
        "profiles": {
            "kubernetes": {
                "expiry": "87600h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF
```

profiles中指定了不同的过期时间、使用场景等参数，可以定义多个profiles,在后续的签名中指定使用某一个<br>
expiry 表示证书的过期时间<br>
signing 表示该证书可用于签名其他证书<br>
key encipherment表示为密钥加密<br>
server auth 表示client可以用该CA对server提供的证书进行验证<br>
client auth 表示server可以用该CA对client提供的证书进行验证<br>

```
cat >/tmp/cert/csr.json<<EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH",
            "O": "k8s",
            "OU": "System"
        }
    ],
    "ca": {
        "expiry": "87600h"
    }
}
EOF
```
CN 请求的用户名<br>
L 地区<br>
ST 省<br>
O 请求用户所属组<br>
OU 组织单位，公司名<br>

>
本文kubernetes和etcd 使用同一套CA证书。在/tmp/cert目录下签发ca证书，将生成的ca证书分别copy到/etc/kubernetes/pki和/etc/kubernetes/pki/etcd，作为kubernetes和etcd的CA证书

```
cd /tmp/cert
cfssl gencert -initca csr.json|cfssljson -bare ca - && mv ca.pem ca.crt && mv ca-key.pem ca.key
rm -f ca.csr ca.json
cp ca.* /etc/kubernetes/pki/etcd
cp ca.* /etc/kubernetes/pki
```

#### 3 etcd集群证书制作
##### 3.1 etcd-server
```
cat >/tmp/cert/etcd-server.json<<EOF
{
    "CN": "etcd",
    "hosts": [
      "127.0.0.1",
      "192.168.20.87"
      ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH",
            "O": "k8s",
            "OU": "System"
        }
   ]
}
EOF
```

这里的etcd只有一台服务器，hosts中填写这个服务的ip即可，如果是多个服务器，要填上所有机器的IP，CN改为 etcd.
```
cd /tmp/cert
cfssl gencert -ca=ca.crt -ca-key=ca.key -config=config.json -profile=kubernetes etcd-server.json | cfssljson -bare etcd-server && mv etcd-server.pem etcd-server.crt && mv etcd-server-key.pem etcd-server.key
rm -f etcd-server.csr etcd-server.json
cp etcd-server.*  /etc/kubernetes/pki/etcd
```
将生成的etcd-server.crt 、etcd-server.key复制到etcd对应的目录中。

##### 3.2 etcd-peer 
> 
用于etcd集群内部节点之间的通信验证

```
cat >/tmp/cert/etcd-peer.json<<EOF
{
    "CN": "peer",
    "hosts": [
      "127.0.0.1",
      "192.168.20.87"
      ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH",
            "O": "k8s",
            "OU": "System"
        }
   ]
}
EOF
```

因为只有一台服务器做etcd,所以hosts填写这台机器的ip，如果是多个服务器，要填上所有机器的IP。

```
cd /tmp/cert
cfssl gencert -ca=ca.crt -ca-key=ca.key -config=config.json -profile=kubernetes etcd-peer.json | cfssljson -bare etcd-peer && mv etcd-peer.pem etcd-peer.crt && mv etcd-peer-key.pem etcd-peer.key
rm -f etcd-peer.csr etcd-peer.json
cp etcd-peer.* /etc/kubernetes/pki/etcd
```
将生成的etcd-peer复制到etcd集群对应的目录中

##### 3.3 etcd-client
>用于etcd客户端访问etcd集群使用，比如apiserver访问etcd集群

```
cat >/tmp/cert/etcd-client.json<<EOF
{
    "CN": "client",
    "hosts": [""],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

```
cd /tmp/cert
cfssl gencert -ca=ca.crt -ca-key=ca.key -config=config.json -profile=kubernetes etcd-client.json | cfssljson -bare etcd-client && mv etcd-client.pem etcd-client.crt && mv etcd-client-key.pem etcd-client.key
rm -f etcd-client.csr etcd-client.json
cp etcd-client.* /etc/kubernetes/pki/etcd
```
将生成的etcd-client.crt,etcd-client.key 复制到apiserver的对应目录下

#### 4 admin证书
>
admin证书用于管理员操作kubernetes集群时的验证，O值为system:masters ，在kubernetes中system:masters组默认和cluster-admin角色绑定，该角色拥有对kubernetes所有资源的所有操作权限。





![1](https://github.com/x2y2/x2y2.github.io/blob/master/assets/img/20201029-1.png?raw=true)
![2](https://github.com/x2y2/x2y2.github.io/blob/master/assets/img/20201029-2.png?raw=true)

```
cat > /tmp/cert/admin.json<<EOF
{
    "CN": "kubernetes-admin",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH",
            "O": "system:masters",
            "OU": "System"
        }
    ]
}
EOF
```
```
cd /tmp/cert
cfssl gencert -ca=ca.crt -ca-key=ca.key -config=config.json -profile=kubernetes admin.json | cfssljson -bare admin && mv admin.pem admin.crt && mv admin-key.pem admin.key
rm -f admin.csr admin.json
cp admin.* /etc/kubernetes/pki
```
将admin.crt,admin.key 复制到kubernetes集群对应的目录下

#### 5 apiserver证书
用于apiserver TLS验证
```
cat >/tmp/cert/apiserver.json<<EOF
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "192.168.20.90",
      "10.244.0.1",
      "10.96.0.1",
      "10.96.0.10",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```
apiserver的hosts填写apiserver的ip，service_ip_range的第一个ip，cluster_cidr的第一个ip以及dns server 的ip。
```
cd /tmp/cert
cfssl gencert -ca=ca.crt -ca-key=ca.key -config=config.json -profile=kubernetes apiserver.json | cfssljson -bare apiserver && mv apiserver.pem apiserver.crt && mv apiserver-key.pem apiserver.key
rm -f apiserver.csr apiserver.json
cp apiserver.* /etc/kubernetes/pki
```
#### 6 controller-manager证书
>
kube-controller-manager访问apiserver使用的证书

```
cat >/tmp/cert/controller-manager.json<<EOF
{
    "CN": "system:kube-controller-manager",
    "hosts": [""],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

```
cd /tmp/cert
cfssl gencert -ca=ca.crt -ca-key=ca.key -config=config.json -profile=kubernetes controller-manager.json | cfssljson -bare controller-manager && mv controller-manager.pem controller-manager.crt && mv controller-manager-key.pem controller-manager.key 
rm -f controller-manager.csr controller-manager.json
cp controller-manager.* /etc/kubernetes/pki
```

#### 7 scheduler证书
> kube-scheduler访问apiserver使用的证书


```
cat >/tmp/cert/scheduler.json<<EOF
{
    "CN": "system:kube-scheduler",
    "hosts": [""],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

```
cd /tmp/cert
cfssl gencert -ca=ca.crt -ca-key=ca.key -config=config.json -profile=kubernetes scheduler.json | cfssljson -bare scheduler && mv scheduler.pem scheduler.crt && mv scheduler-key.pem scheduler.key
rm -f scheduler.csr scheduler.json
cp scheduler.* /etc/kubernetes/pki
```

#### 8 apiserver-kubelet-client证书
>
apiserver访问kubelet使用的证书

```
cat >/tmp/cert/apiserver-kubelet-client.json<<EOF
{
    "CN": "apiserver-kubelet-client",
    "hosts": [""],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH",
            "O": "system:masters",
            "OU": "System"
        }
    ]
}
EOF
```
```
cd /tmp/cert
cfssl gencert -ca=ca.crt -ca-key=ca.key -config=config.json -profile=kubernetes apiserver-kubelet-client.json | cfssljson -bare apiserver-kubelet-client && mv apiserver-kubelet-client.pem apiserver-kubelet-client.crt && mv apiserver-kubelet-client-key.pem apiserver-kubelet-client.key
rm -f apiserver-kubelet-client.csr apiserver-kubelet-client.json
cp apiserver-kubelet-client.* /etc/kubernetes/pki
```

#### 9 front-proxy证书
>
该证书用于API Aggregation(聚合),用户自定义的API可以通过API Aggregation代理动态、无缝地注册到kubernetes API Server中进行使用。

```
cat >/tmp/cert/front-proxy-ca.json<<EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

```
cd /tmp/cert
cfssl gencert -ca=ca.crt -ca-key=ca.key -config=config.json -profile=kubernetes front-proxy-ca.json | cfssljson -bare front-proxy-ca && mv front-proxy-ca.pem front-proxy-ca.crt && mv front-proxy-ca-key.pem front-proxy-ca.key
rm -f front-proxy-ca.csr front-proxy-ca.json
cp front-proxy-ca.* /etc/kubernetes/pki
```

#### 10 front-proxy-client证书

```
cat >/tmp/cert/front-proxy-client.json<<EOF
{
    "CN": "front-proxy-client",
    "hosts": [""],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SH",
            "ST": "SH",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```

```
cd /tmp/cert
cfssl gencert -ca=ca.crt -ca-key=ca.key -config=config.json -profile=kubernetes front-proxy-client.json | cfssljson -bare front-proxy-client && mv front-proxy-client.pem front-proxy-client.crt && mv front-proxy-client-key.pem front-proxy-client.key
rm -f front-proxy-client.csr front-proxy-client.json
cp front-proxy-client.* /etc/kubernetes/pki
```
#### 11 serviceaccount证书
> 用于kubernetes集群内部POD访问apiserver时采用service account token方式进行验证，token就是apiserver启动时--service-account-key-file参数指定的值签署生成的。

```
cd /tmp/cert
openssl genrsa -out sa.key 1024
openssl rsa -in sa.key -pubout -out sa.pub
cp sa.* /etc/kubernetes/pki
```

#### 证书有效期查看
用kubeadm搭建的kubernetes证书的有效期默认是一年，在证书过期之前要重新签发证书或是延长有效期，如何查看证书有效期，可以用openssl命令
openssl x509 -in  ca.crt -noout -dates
![cert-expiry](https://github.com/x2y2/x2y2.github.io/blob/master/assets/img/cert-expiry.png?raw=true)
可以看到ca.crt 的有效时间为10年