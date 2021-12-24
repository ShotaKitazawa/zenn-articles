---
title: "PR が出るたびに Kubernetes 上で dev 環境を立ち上げるための Kubernetes Operator"
emoji: "⚙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes","operator","CICD","GitOps"]
published: true
---

この記事は、[Kubernetes Advent Calendar](https://qiita.com/advent-calendar/2021/kubernetes) 21 日目の記事です。

[reviewapp-operator](https://github.com/cloudnativedaysjp/reviewapp-operator) という、アプリケーション開発リポジトリへ PR が出るたびに Kubernetes 上で dev 環境を立ち上げるための Kubernetes Operator を自作した話になります。

この記事は、以前 [CloudNative Days Tokyo 2021 プレイベント](https://cloudnativedays.connpass.com/event/226567/) にて `CloudNative Days を支える CI/CD ワークフローの変遷` というタイトルで話したものをより一般的な内容にしました。
上記発表は「CloudNative Days というイベントの裏方として CI/CD ワークフローを整備した」ことに焦点を当てた内容となっているため、もしよければ [発表資料](https://speakerdeck.com/shotakitazawa/cd-wakuhurofalsebian-qian) も合わせてご覧ください。

# 背景

GitHub 等のリモートリポジトリ上の Kubernetes マニフェストファイルを参照し Argo CD を用いてアプリケーションの Continuous Delivery する場合、以下のいずれかの方法で「リモートリポジトリ上の Kubernetes マニフェストファイルの継続的な更新」を行う必要があります。

* アプリケーションを開発しているリポジトリ (以降 `アプリケーションリポジトリ`) で動作する CI にて、マニフェストを配置しているリポジトリ (以降 `マニフェストリポジトリ`) 上にある該当マニフェストのバージョン情報を書き換える
* [Argo CD Image Updater](https://github.com/argoproj-labs/argocd-image-updater) を用いて、コンテナレジストリの更新契機でマニフェストリポジトリ上にある該当マニフェストのバージョン情報を書き換える

この方法により「あるブランチの更新契機で、そのブランチに紐づく環境への CD」を実現できます。

しかしながら、上記から一歩進んで「アプリケーションリポジトリに PR が出るたびに新規 Namespace を作成しそこに新規環境を立ち上げる」というように監視対象ブランチが動的に変わる場合、 Argo CD Image Updater では実現できないです。

上記のように動的に新規環境を立ち上げる仕組みは、特に複数人で同じアプリケーションを開発する際において、共用利用のステージング環境へマージする前に自分の実装の動作を試す事ができるためとても便利です。 例えば有名な PaaS である Heroku では [Review Apps](https://devcenter.heroku.com/articles/github-integration-review-apps) という機能でこれを実現しています。

そのため Kubernetes を利用している場合においても同じことを実現したいというのがモチベーションとしてありました。

# reviewapp-operator

[reviewapp-operator](https://github.com/cloudnativedaysjp/reviewapp-operator) は「アプリケーションリポジトリに PR が出るたびに dev 用の新規環境 (以下、Review Apps 環境) を立ち上げる」ことを実現するために、Argo CD と協調して動作します。
reviewapp-operator の責務は主に「アプリケーションリポジトリの PR の更新契機でマニフェストリポジトリのマニフェストを作成・削除する」ことであり、実際にマニフェストリポジトリからマニフェストを Kubernetes に適用するのは Argo CD の責務となります。

![reviewapp-operator の workflow](https://raw.githubusercontent.com/ShotaKitazawa/zenn-articles/master/images/about-reviewapp-operator/workflow.jpg)
*reviewapp-operator の workflow*

**なお、以降この記事は [reviewapp-operator v0.0.5](https://github.com/cloudnativedaysjp/reviewapp-operator/releases/tag/v0.0.5) を前提に話します。**

## 使い方

reviewapp-operator の使い方について説明します。

本当はドキュメントのリンクを貼り付けるだけで済めば良かったのですが、 reviewapp-operator のドキュメントは現状皆無なので、ここで各リソースの役割の説明や使い方について説明します。(後日、公式ドキュメントを作成予定です🙇)

### reviewapp-operator のインストール

Controller をデプロイするための Namespace の作成と、CRD や Custom Controller の動作する Deployment 等が書かれたマニフェストの適用だけです。 (いつもの)

```bash
kubectl create namespace reviewapp-operator-system
kubectl apply -n reviewapp-operator-system -f https://github.com/cloudnativedaysjp/reviewapp-operator/releases/download/v0.0.5/install.yaml
```

### reviewapp-operator の提供する Custom Resources

reviewapp-operator は以下の 4 つの Custom Resource を提供します。

| リソース名            | 説明                                                                                                                                                                                                                                                                                                                            |
|:---------------------:|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|
| `ReviewAppManager`    | ユーザが宣言するリソース①。監視対象のアプリケーションリポジトリ・マニフェストリポジトリと、Review Apps 環境を構築するための各種マニフェストのテンプレート (`ApplicationTemplate`, `ManifestsTemplate`) を指定する。                                                                                                            |
| `ReviewApp`           | `ReviewAppManager` オブジェクトを親リソースとして、アプリケーションリポジトリの PR ごとに `ReviewAppManager` コントローラより作成される。reviewapp-operator のメインロジック (マニフェストのテンプレーティング、マニフェストリポジトリへマニフェストファイルの push、etc...) はすべて `ReviewApp` コントローラより実行される。 |
| `ApplicationTemplate` | ユーザが宣言するリソース②。Review Apps 環境を構築するための Argo CD Application のテンプレートを記述するためのリソース。                                                                                                                                                                                                       |
| `ManifestsTemplate`   | ユーザが宣言するリソース③。Review Apps 環境を構築するための各種マニフェストファイルのテンプレートを記述するためのリソース。                                                                                                                                                                                                    |

![reviewapp-operator の提供する Custom Resources](https://raw.githubusercontent.com/ShotaKitazawa/zenn-articles/master/images/about-reviewapp-operator/resources.jpg)
*reviewapp-operator の提供する Custom Resources*

また、現状 `ReviewAppManager`, `ApplicationTemplate`, `ManifestsTemplate` の値を記述する際に利用できるテンプレート変数には以下があります。
テンプレート変数がどのように利用されるかついては [実際の設定例](#実際の設定例) をご覧ください。

| テンプレート変数名         | 説明                                                                                                                           |
|:--------------------------:|:------------------------------------------------------------------------------------------------------------------------------:|
| `.Apprepo.Organization`    | ReviewApp が監視するアプリケーションリポジトリの組織名                                                                         |
| `.Apprepo.Repository`      | ReviewApp が監視するアプリケーションリポジトリのリポジトリ名                                                                   |
| `.Apprepo.Branch`          | ReviewApp が監視するアプリケーションリポジトリのブランチ名                                                                     |
| `.Apprepo.PrNumber`        | ReviewApp が監視するアプリケーションリポジトリの PR 番号                                                                       |
| `.Apprepo.LatestCommitSha` | ReviewApp が監視するアプリケーションリポジトリの PR における最新 commit の hash                                                |
| `.InfraRepo.Organization`  | ReviewApp が管理するマニフェストリポジトリの組織名                                                                             |
| `.InfraRepo.Repository`    | ReviewApp が管理するマニフェストリポジトリのリポジトリ名                                                                       |
| `.Variables.<任意の名前>`  | `ReviewAppManager.spec.variables` 以下に定義された KeyValue の組。これにより ReviewAppManager リソースから任意の値を設定可能。 |


### 実際の設定例

以下は [cloudnativedaysjp/dreamkast](https://github.com/cloudnativedaysjp/dreamkast) の Review Apps 環境を管理するための実際のマニフェストです。

* [dreamkast.yaml](https://github.com/cloudnativedaysjp/dreamkast-infra/blob/501263c85fe05f72ebea7ece0d6d0f8fd75edb49/manifests/reviewapps/dreamkast.yaml)

上記ファイルには `ReviewAppManager`, `ApplicationTemplate`, `ManifestsTemplate` リソースの宣言がそれぞれ記述されています。以降この記述について説明していきます。

* 8~14行目: ReviewAppManager.spec.appRepoTarget
    * 監視対象のアプリケーションリポジトリへの接続情報を記述するフィールド
    * この情報をもとに、PR が 1 つ増えるたびに `ReviewAppManager` オブジェクトから `ReviewApp` オブジェクトが 1 つ生成される

:::details クリックで展開
```yaml
  appRepoTarget:
    username: showks-containerdaysjp
    organization: cloudnativedaysjp
    repository: dreamkast
    gitSecretRef:
      name: git-creds
      key: token
```
:::

* 18~22行目: ReviewAppManager.spec.appRepoConfig
    * アプリケーションリポジトリに対する接続情報以外の設定をするフィールド
    * テンプレート変数を記述可能
    * v0.0.5 現在、「PR に紐づく ReviewApp 環境がデプロイされたときに PR に対して出力するメッセージ」の定義のみ可能
        * 例: https://github.com/cloudnativedaysjp/dreamkast/pull/1057#issuecomment-997184923

:::details クリックで展開
```yaml
  appRepoConfig:
    message: |
      Review app
      https://dreamkast-{{.Variables.AppRepositoryAlias}}-{{.AppRepo.PrNumber}}.dev.cloudnativedays.jp
```
:::

* 23~30行目: ReviewAppManager.spec.infraRepoTarget
    * 管理対象のマニフェストリポジトリへの接続情報を記述するフィールド

:::details クリックで展開
```yaml
  infraRepoTarget:
    username: showks-containerdaysjp
    organization: cloudnativedaysjp
    repository: dreamkast-infra
    branch: main
    gitSecretRef:
      name: git-creds
      key: token
```
:::

* 31~41行目: ReviewAppManager.spec.infraRepoConfig
    * マニフェストリポジトリのどのパスにマニフェストを配置するかを記述するフィールド
    * テンプレート変数を記述可能
    * ReviewAppManager.spec.infraRepoConfig.manifests には複数個の `ManifestsTemplate` の指定が、ReviewAppManager.spec.infraRepoConfig.argocdApp には 1 つの `ApplicationTemplate` の指定がそれぞれ可能
    * **`ReviewAppManager.spec.infraRepoConfig.argocdApp.filepath` の dirname を監視する Argo CD Application を事前に定義しておくことで以下の図のように Argo CD Application を多段で管理でき Review Apps 環境の構築を実現可能**

:::details クリックで展開
```yaml
  infraRepoConfig:
    manifests:
      templates:
        - namespace: reviewapp-operator-system
          name: dreamkast
      dirpath: "manifests/app/dreamkast/overlays/development/{{.Variables.AppRepositoryAlias}}-{{.AppRepo.PrNumber}}"
    argocdApp:
      template:
        namespace: reviewapp-operator-system
        name: dreamkast
      filepath: "manifests/app/argocd-apps/development/dreamkast-{{.Variables.AppRepositoryAlias}}-{{.AppRepo.PrNumber}}.yaml"
```
:::

![Review Apps を実現するディレクトリ構成](https://raw.githubusercontent.com/ShotaKitazawa/zenn-articles/master/images/about-reviewapp-operator/directory.png)
*Review Apps を実現するディレクトリ構成*

* 42~43行目: ReviewAppManager.spec.variables
    * テンプレート変数として `{{.Variables.<key>}}` で呼び出し可能な KeyValue を定義

:::details クリックで展開
```yaml
  variables:
      - AppRepositoryAlias=dk
```
:::

* 51~70行目: ApplicationTemplate.spec.stable
    * ReviewApp からマニフェストリポジトリに push する Argo CD Application 用マニフェストのテンプレートを定義するフィールド
    * テンプレート変数を記述可能

:::details クリックで展開
```yaml
  stable: &application |
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: "dreamkast-development-{{.Variables.AppRepositoryAlias}}-{{.AppRepo.PrNumber}}"
      namespace: argocd
      finalizers:
        - resources-finalizer.argocd.argoproj.io # cascade deletion on this App deletion
    spec:
      destination:
        namespace: "dreamkast-{{.Variables.AppRepositoryAlias}}-{{.AppRepo.PrNumber}}"
        server: https://kubernetes.default.svc
      project: app
      source:
        repoURL: https://github.com/cloudnativedaysjp/dreamkast-infra
        path: "manifests/app/dreamkast/overlays/development/{{.Variables.AppRepositoryAlias}}-{{.AppRepo.PrNumber}}"
        targetRevision: main
      syncPolicy:
        automated:
          prune: true
```
:::

![ApplicationTemplate が配置される箇所](https://raw.githubusercontent.com/ShotaKitazawa/zenn-articles/master/images/about-reviewapp-operator/directory-at.png)
*ApplicationTemplate が配置される箇所*

* 71行目: ApplicationTemplate.spec.candidate
    * アプリケーションリポジトリの PR に `candidate-template` というラベルが付いている場合、 ApplicationTemplate.spec.stable の代わりにこちらのテンプレートが利用される
        * ApplicationTemplate 自体をバージョンアップしたい際の検証用として利用を想定
    * dreamkast では yaml の安価を利用して stable と同値としている

:::details クリックで展開
```yaml
  candidate: *application
```
:::

* 78~612行目: ManifestsTemplate.spec.stable
    * ReviewApp からマニフェストリポジトリに push する Review Apps 環境構築用マニフェストのテンプレートを定義するフィールド
    * テンプレート変数を記述可能
    * [dreamkast の場合](https://github.com/cloudnativedaysjp/dreamkast-infra/blob/501263c85fe05f72ebea7ece0d6d0f8fd75edb49/manifests/reviewapps/dreamkast.yaml#L79-L98)、kustomization.yaml も ManifestsTemplate に記述している
        * **kustomization.yaml の images.newTag に `{{.AppRepo.LatestCommitSha}}` を記述することで、アプリケーションリポジトリ PR の HEAD のコンテナイメージを常に利用するようになっている**
        * [この ManifestsTemplate より作成されたディレクトリ](https://github.com/cloudnativedaysjp/dreamkast-infra/tree/501263c85fe05f72ebea7ece0d6d0f8fd75edb49/manifests/app/dreamkast/overlays/development/dk-1042) を見ると、 `kustomize build .` で解決可能なディレクトリ構成になっていることが確認できる

:::details クリックで展開
[行数が多いのでリンクを貼っておきます](https://github.com/cloudnativedaysjp/dreamkast-infra/blob/501263c85fe05f72ebea7ece0d6d0f8fd75edb49/manifests/reviewapps/dreamkast.yaml#L78-L612)
:::

![ManifestsTemplate が配置される箇所](https://raw.githubusercontent.com/ShotaKitazawa/zenn-articles/master/images/about-reviewapp-operator/directory-mt.png)
*ManifestsTemplate が配置される箇所*

* 613行目: ManifestsTemplate.spec.candidate
    * ApplicationTemplate.spec.candidate と同様の利用方法

:::details クリックで展開
```yaml
    candidate: *manifests
```
:::

# reviewapp-operator を3ヶ月ほど運用してみての感想など

上記で紹介した reviewapp-operator は 2021/10 あたりから [Dreamkast](https://github.com/cloudnativedaysjp/dreamkast) というオンラインカンファレンスプラットフォームの開発環境を管理するのに利用されています。
ここからは、reviewapp-operator を実際に運用してみて分かったことやその感想などを書いていきます。

### ManifestsTemplate リソースのマニフェストが非常に読みにくい

[実際の設定例](#実際の設定例) にて実際の ManifestsTemplate の書きっぷりをお見せしましたが、string なフィールドにマニフェストを列挙する形になっており非常に読みにくいです。
この問題を解決するために、以下の方針を考えています。

* 複数の yaml 形式のマニフェストファイルを引数に与えることで ManifestsTemplate リソースを出力する CLI ツールを実装
* Argo CD の [Config Management Plugins](https://argo-cd.readthedocs.io/en/stable/user-guide/config-management-plugins/) を利用して、Argo CD による apply の前に上記 CLI ツールで ManifestsTemplate リソースの記述されたマニフェストを生成
    * ちなみに Argo CD v2.2 で Config Management Plugins V2 がリリースされ、 `ConfigManagementPlugin` カスタムリソースとして設定を記述できるようになった。([参考](https://blog.argoproj.io/argo-cd-v2-2-release-candidate-4e16e985b486))

なおこの問題は [#50 ManifestsTemplate, ApplicationTemplate が見にくいのをどうにかする](https://github.com/cloudnativedaysjp/reviewapp-operator/issues/50) という Issue で管理しています。

### 新機能の実装ネタが思ったよりもある

実は reviewapp-operator を実装するよりも前にも GitHub Actions で reviewapp-operator 相当のものを実現していたのですが ([発表資料](https://speakerdeck.com/shotakitazawa/cd-wakuhurofalsebian-qian)を参照)、「Review Apps 環境を構築・クリーンアップする他に様々な追加機能が欲しくなった際、その開発をちゃんとテスタブルな言語で行いたい」というのが reviewapp-operator を実装するモチベーションの 1 つとしてありました。

個人的にこの理由は「言うても機能追加はそうそうないだろう」と高をくくっていたのですが、いざ運用してみると様々な追加機能が欲しくなりました。

* [アプリケーションリポジトリの該当 PR に紐づく CI が pass している場合のみ reviewapp-operator でマニフェストの更新を行うようにする](https://github.com/cloudnativedaysjp/reviewapp-operator/issues/3)
* [reviewapp-operator による Review Apps 環境の構築・クリーンアップ時に任意のスクリプトをフックしたい](https://github.com/cloudnativedaysjp/reviewapp-operator/issues/69)
* [reviewapp-operator が Review Apps 環境を構築・クリーンアップしたことを通知する](https://github.com/cloudnativedaysjp/reviewapp-operator/issues/71)

機能の案は [GitHub Issue](https://github.com/cloudnativedaysjp/reviewapp-operator/issues) で管理しているため、自分の手が空いたときにやりたいもの順で実装していこうと思っています。

### 統合テストに時間がかかる

これは reviewapp-operator の話というよりは、Kubernetes Operator の「Custom Resource オブジェクトの状態を監視し、更新などを契機に該当 Custom Resource オブジェクトや外部リソース (例. GitHub PullRequest) を CRUD する」という、宣言的 API を実現するための実装の難しさの話です。

宣言的でない API (例. 単純な CRUD が可能な Web API サーバ) に対する統合テストの場合は外部リソースの収束を待たずともハンドラがエラー落ちしたらテスト失敗とみなせる場合が多いですが、Kubernetes Operator の場合は Reconciler メソッドが複数回呼び出されることで宣言された状態に収束するような実装をするため、ただ単純に Reconciler がエラー落ちするかどうかではなく、数秒以内に外部リソースが期待する状態に収束しているかというテストの書き方をする必要があります。

このようなテストにおいて Operator が期待通り動作すれば問題ないのですが、実装が間違っている場合はいくら待っても外部リソースが期待する状態に収束しないため、タイムアウトするまで待つことになり、個人の感想としてテストの実行時間が長いと感じることが良くあります。

このあたりについて、今の所「短すぎず長すぎないタイムアウトの時間を見つける」以上の案がないので、もし何か良い手があればコメント等で教えていただきたいです。

### 「reviewapp-operator 何もわからん」

使い方の項で述べたとおり現状 reviewapp-operator のドキュメントが皆無であるため、チームメンバーであっても reviewapp-operator 周りについて触れないという現状になってしまっていると感じています。

幸い reviewapp-operator で立ち上げる Review Apps 環境のマニフェスト自体を更新したい機会が現状そこまで多くないためその機会が訪れるたびに自分が作業者になれば良いとして回っていますが、この作業が属人化してしまってる現状はあまり健全でないので是正していきたいです。

reviewapp-operator は OSS として公開されているため、他の方にも利用してもらえるよう、ドキュメントや CRD のフィールドに対する description の拡充はできるだけ優先度を高くしてやっていこうと思っています。
// 本当はこの記事の公開までにドキュメントを整備したかった...orz

# 最後に

「アプリケーションリポジトリに PR が出るたびに新規 Namespace を作成しそこに新規環境を立ち上げる」ことを実現する Kubernetes Operator である [reviewapp-operator](https://github.com/cloudnativedaysjp/reviewapp-operator) について、開発したモチベーションやその使い方を紹介しました。
また、reviewapp-operator を複数人で開発されるアプリケーションの開発環境構築へ実際に導入してみて、分かったことや感想を書きました。

CloudNative Days の特定のユースケースというよりはより汎用的に利用できるものとして実装したつもりなので、この記事を読んでくれた方が実際にそう感じてもらえたらならば幸いです。

reviewapp-operator は現状 [Dreamkast](https://github.com/cloudnativedaysjp/dreamkast) でのみ利用されていますが、上述したとおり動的に dev 環境を立ち上げるという用途でより汎用的に利用可能なので、引き続き機能開発を勧めつつどなたにでも reviewapp-operator を使ってもらえるようドキュメントを充実させていこうと思っています。
