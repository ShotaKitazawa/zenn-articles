---
title: "ホストクラスタの構築"
---

# ホストクラスタの構築

まず kubeadm を用いてホストクラスタを構築します。環境情報は以下です。

* OS: Ubuntu 20.04
* Kubernetes v1.20.2
    * CRI: Containerd 1.3.3-0
    * CNI: Calico v3.16.6 (後ほど増えます)
* KubeVirt v0.35.0

