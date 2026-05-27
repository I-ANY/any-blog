+++
title = "k8s集群证书过期更换"
date = "2026-05-28T00:01:08+08:00"
draft = false
+++

# kubadm



使用kubeadm部署的k8s集群默认证书有效期为1年，可以通过更新k8s版本自动更新证书或修改源码更改证书有效期。下面为手动更换证书步骤总结。

- **查看证书日期**

```bash
# kubeadm alpha certs check-expiration
kubeadm certs check-expiration
```

- **备份**

```bash
cp -rp /etc/kubernetes /etc/kubernetes.bak
cp -rp /var/lib/etcd /var/lib/etcd.bak
```

- **生成新的证书**

集群每台master节点都需要操作

```bash
# kubeadm alpha certs renew all
kubeadm certs renew all
```

此时所有证书都已重新生成，在 /etc/kubernetes/pki下。
/etc/kubernetes/admin.conf、/etc/kubernetes/controller-manager.conf、/etc/kubernetes/scheduler.conf 也已经更新了。

- **再次查看证书**

```bash
kubeadm certs check-expiration
```

- **重启组件**

集群每台master节点都需要操作

```bash
docker ps | grep -E 'k8s_kube-apiserver|k8s_kube-controller-manager|k8s_kube-scheduler|k8s_etcd_etcd' | awk -F ' ' '{print $1}' | xargs docker restart
```

- **更新config**

```bash
# 此时kubectl 报权限问题，无法执行（每台master都要操作）
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

- **重启kubelete**

```bash
systemctl restart kubelete
```

