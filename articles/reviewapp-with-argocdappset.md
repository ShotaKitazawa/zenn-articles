---
title: "Argo CD ApplicationSet で実現する Review Apps"
emoji: "⚙"
type: "tech"
topics: ["kubernetes","argocd","CICD","GitOps"]
published: true
---

以前 [PR が出るたびに Kubernetes 上で dev 環境を立ち上げるための Kubernetes Operator](https://zenn.dev/kanatakita/articles/about-reviewapp-operator) という記事を書きましたが、 Operator を自作しなくとも Argo CD の提供する機能のみで Review App を実現することが可能になったため、その方法について書きます。

## TL;DR

* Argo CD ApplicationSet Controller の [Pull Request Generator](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Pull-Request/) 機能を使った
* Review App 環境ごとに変わる変数を設定するために Argo CD Plugin と Kustomize を駆使した

## 背景

背景とこの記事で実現したいことについては [以前の記事](https://zenn.dev/kanatakita/articles/about-reviewapp-operator#%E8%83%8C%E6%99%AF) と同様のため、ここでは省略します。

## Argo CD ApplicationSet Controller

Argo CD ApplicationSet は複数の Argo CD Application をまとめて管理できる機能です。

ApplicationSet の CustomResource スキーマは大きく分けて `Generators` と `Template fields` の 2 つに分かれており、 Generators の出力の数だけ Template fields に書かれた Application リソースのマニフェストを適用するというのが大まかな流れになります。

[`Cluster Generator`](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Cluster/) をもとに具体例をあげると、以下のような ApplicationSet マニフェストを適用すると Argo CD 管理下のクラスタの数だけ Application リソースが作成されることになります。

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

[Pull Request Generator](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Pull-Request/) は Generators の一種で、これを利用すると GitHub などの SCMaaS における Pull Request の数だけ Application リソースを生成できます。

また、Review App 対象の GitHub リポジトリに [Webhook の設定](https://argocd-applicationset.readthedocs.io/en/stable/Generators-Pull-Request/#webhook-configuration) をすることで Pull Request の作成や削除などのイベントが起きるとすぐに Review App 環境も更新できます。

これを用いることで Pull Request 毎に Review App 環境を構築できるようになりました。しかしこの方法には **Review App 環境毎に異なる値をマニフェストに与えることができない** という課題点があります。

例えば「Review App 環境ごとに異なる FQDN で Ingress リソースを作成する」ことがこの方法だけだと実現できないです。

## Argo CD Plugin + Kustomize replacements

上記を実現する方法を探っていると、Argo CD のリポジトリ内に[まさにこの課題について言及している Discussion](https://github.com/argoproj/argo-cd/discussions/9042) を見つけました。
つまり、 Kustomize の `configMapGenerator` と `replacements` の機能を使うことでマニフェストの任意のフィールドを書き換えられるようです。

ただ、 Argo CD Application のサポートしている [kustomize](https://argo-cd.readthedocs.io/en/stable/user-guide/kustomize/) の機能を見る限り、この kustomize に環境変数を与えることが出来ません。

これを解決するために、 [Argo CD Plugins](https://argo-cd.readthedocs.io/en/stable/user-guide/config-management-plugins/) の機構を利用して以下のように「任意の環境変数を渡せる kustomize plugin」を定義しました。

* 参考: [実際の定義箇所](https://github.com/cloudnativedaysjp/dreamkast-infra/blob/1a19e187a444744672a7cb26c1d8131b7cffef90/manifests/argocd/overlays/dev/argocd-cm.yaml#L25-L37)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  configManagementPlugins: |
    - name: kustomize-with-replacements
      init:
        command: ["bash", "-euxc"]
        args:
        - export IFS=",";
          for image in ${ARGOCD_ENV_IMAGES}; do
            kustomize edit set image $image;
          done;
          kustomize edit set namespace $ARGOCD_ENV_NAMESPACE;
      generate:
        command: ["kustomize", "build"]
      lockRepo: true
```

ApplicationSet リソースを宣言するマニフェストにて、この Plugin を利用するよう以下のように記述します。

* 参考: [実際の定義箇所](https://github.com/cloudnativedaysjp/dreamkast-infra/blob/1a19e187a444744672a7cb26c1d8131b7cffef90/manifests/reviewapps/dreamkast-dk.yaml#L30-L40)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: test
spec:
  generators: (snip)
  template:
    metadata:
      name: 'dreamkast-dk-{{number}}'
    spec:
      source:
        repoURL: https://github.com/cloudnativedaysjp/dreamkast-infra.git
        targetRevision: main
        path: manifests/app/dreamkast/overlays/development/template-dk
        plugin:
          name: kustomize-with-replacements
          env:
            - name: FQDN
              value: 'dreamkast-dk-{{number}}.dev.cloudnativedays.jp'
            - name: NAMESPACE
              value: 'dreamkast-dk-{{number}}'
            - name: IMAGES
              value: >-
                dreamkast-ecs=607167088920.dkr.ecr.ap-northeast-1.amazonaws.com/dreamkast-ecs:{{head_sha}},
                dreamkast-ui=607167088920.dkr.ecr.ap-northeast-1.amazonaws.com/dreamkast-ui:main
      destination:
        server: https://kubernetes.default.svc
        namespace: 'dreamkast-dk-{{number}}'
```

上記のようにマニフェストを記述することで、該当の ApplicationSet から生成される Application 毎に異なる変数 (例えば. `FQDN='dreamkast-dk-{{number}}.dev.cloudnativedays.jp'`) を kustomize 実行時の環境変数として設定することが出来るようになりました。

実際にこの Application が読み込む先の kustomization.yaml に以下のように `configMapGenerator` と `replacements` を記述することで、Review App として PR 毎に適用されるマニフェストの任意のフィールドを任意の値に更新することが出来ます。

* 参考: [実際の定義箇所](https://github.com/cloudnativedaysjp/dreamkast-infra/blob/1a19e187a444744672a7cb26c1d8131b7cffef90/manifests/app/dreamkast/overlays/development/template-dk/kustomization.yaml#L17-L27)

```yaml
# in kustomization.yaml
generatorOptions:
  disableNameSuffixHash: true
configMapGenerator:
- envs:
  - .env
  name: replacement-rules
replacements:
- path: .replacement_ns.yaml
- path: .replacement_ingress-hostname.yaml

# in .env : 以下に列挙した環境変数から `replacement-rules` という name の ConfigMap を生成
ARGOCD_ENV_NAMESPACE
ARGOCD_ENV_FQDN

# in .replacement_ns.yaml : Namespace の metadata.name を ConfigMap `replacement-rules` の data.ARGOCD_ENV_NAMESPACE の値に置換する
source:
  version: v1
  kind: ConfigMap
  name: replacement-rules
  fieldPath: data.ARGOCD_ENV_NAMESPACE
targets:
- select:
    kind: Namespace
    name: __REPLACEMENT__
  fieldPaths:
    - metadata.name
```

注意点として Argo CD 2.4 以上から、この方法で Application リソース側にて指定した環境変数に `ARGOCD_ENV_` というプレフィックスが付きます。この Review App の仕組みを導入している https://github.com/cloudnativedaysjp/dreamkast-infra でもこの記事を書いている今現在 Argo CD 2.4 以上を利用しているため、Plugin 内で環境変数を参照する際は上記のように `ARGOCD_ENV_` プレフィックスを付与しています。


## まとめ

ArgoCD ApplicationSet を利用して Review App を実現する方法について書きました。

もともとこれを実現するために Custom Controller を自作していた ([前回の記事](https://zenn.dev/kanatakita/articles/about-reviewapp-operator)) のですが、今回の方法では Review App の仕組みの大部分に Argo CD の提供する機能を使うことができたため、自前実装の保守をする必要性がなくなったのは良いことだと思います。

