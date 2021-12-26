---
title: "ネットワーク構成"
---

今回は ホストネットワークと L2 でつながるネットワークを構築する
(OpenStack 用語で [Provider Network](https://docs.openstack.org/install-guide/launch-instance-networks-provider.html))


* Multus 使って ovs-cni 使えるようにする
* ovs-cni で Pod に Provider Network アドレスが振れていることを確認
* TODO: これじゃ駄目、なぜ？
* 暫定的な解決 → cloud-init で static IP を振る
    * https://github.com/kubevirt/kubevirt/issues/4564
