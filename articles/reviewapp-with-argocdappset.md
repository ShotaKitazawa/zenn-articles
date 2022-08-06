---
title: "ArgoCD ApplicationSet で実現する Review Apps"
emoji: "⚙"
type: "tech"
topics: ["kubernetes","argocd","CICD","GitOps"]
published: true
---

以前 [PR が出るたびに Kubernetes 上で dev 環境を立ち上げるための Kubernetes Operator](https://zenn.dev/kanatakita/articles/about-reviewapp-operator) という記事を書きましたが、 Operator を自作しなくとも Argo CD の提供する機能のみで Review App を実現することが可能になったため、その方法について書きます。

## TL;DR

* ArgoCD ApplicationSet Controller の [Pull Request Generator](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Pull-Request/) 機能を使った
* Review App 環境ごとに変わる変数を設定するために ArgoCD Plugin と Kustomize を駆使した

## 背景

背景とこの記事で実現したいことについては [以前の記事](https://zenn.dev/kanatakita/articles/about-reviewapp-operator#%E8%83%8C%E6%99%AF) と同様のため、ここでは省略します。

## ArgoCD ApplicationSet Controller

ArgoCD ApplicationSet は複数の ArgoCD Application をまとめて管理することができる機能です。

ApplicationSet の CustomResource スキーマは大きく分けて `Generators` と `Template fields` の2つに分かれており、 Generators の出力の数だけ Template fields に書かれた Application リソースのマニフェストを適用するというのが大まかな流れになります。

[`Cluster Generator`](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Cluster/) をもとに具体例をあげると、以下のような ApplicationSet マニフェストを適用すると ArgoCD 管理下のクラスタの数だけ Application リソースが作成されることになります。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
spec:
  generators:
  - clusters: {} # Automatically use all clusters defined within Argo CD
  template:
    metadata:
      name: '{{name}}-guestbook' # 'name' field of the Secret
    spec:
      project: "default"
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps/
        targetRevision: HEAD
        path: guestbook
      destination:
        server: '{{server}}' # 'server' field of the secret
        namespace: guestbook
```

## Pull Request Generator

[Pull Request Generator](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Pull-Request/) は Generators の一種で、これを利用すると GitHub などの SCMaaS における Pull Request の数だけ Application リソースを生成することができます。

また、Review App 対象の GitHub リポジトリに [Webhook の設定](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Pull-Request/#webhook-configuration) をすることで Pull Request の作成や削除などのイベントが起きるとすぐに Review App 環境も更新することができます。

これを用いることで Pull Request 毎に Review App 環境を構築することができるようになりました。しかしこの方法には **Review App 環境毎に異なる値をマニフェストに与えることができない** 課題点があります。

例えば「Review App 環境ごとに異なる FQDN で Ingress リソースを作成する」ことがこの方法だけだと実現することができないです。

## ArgoCD Plugin + Kustomize replacements





## まとめ

ArgoCD ApplicationSet を利用して Review App を実現する方法について書きました。

もともとこれを実現するために Custom Controller を自作していた ([前回の記事](https://zenn.dev/kanatakita/articles/about-reviewapp-operator)) のですが、今回の方法では Review App の仕組みの大部分に Argo CD の提供する機能を使うことができたため、自前実装の保守をする必要性がなくなったのは良いことだと思います。

