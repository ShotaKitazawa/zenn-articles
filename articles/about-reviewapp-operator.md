---
title: "reviewapp-operator"
emoji: "⚙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kubernetes","operator","CICD","GitOps"]
published: false
---

この記事は、[Kubernetes Advent Calendar](https://qiita.com/advent-calendar/2021/kubernetes) 21 日目の記事です。

[reviewapp-operator](https://github.com/cloudnativedaysjp/reviewapp-operator) という、アプリケーション開発リポジトリへ PR が出るたびに Kubernetes 上で dev 環境を立ち上げるための Kubernetes Operator を自作した話になります。

この記事は、以前 [CloudNative Days Tokyo 2021 プレイベント](https://cloudnativedays.connpass.com/event/226567/) にて `CloudNative Days を支える CI/CD ワークフローの変遷` というタイトルで話したものをより一般的な内容にしました。
上記発表では「CloudNative Days というイベントの裏方として開発した」ことに焦点を当てた内容となっているため、もしよければ [発表資料](https://speakerdeck.com/shotakitazawa/cd-wakuhurofalsebian-qian) も合わせてご覧ください。

# 背景

GitHub 等のリモートリポジトリ上の Kubernetes マニフェストファイルを参照し Argo CD を用いてアプリケーションの Continuous Delivery する場合、以下のいずれかの方法で「リモートリポジトリ上の Kubernetes マニフェストファイルの継続的な更新」を行う必要があります。

* アプリケーションを開発しているリポジトリ (以降 `アプリケーションリポジトリ`) で動作する CI にて、マニフェストを配置しているリポジトリ (以降 `マニフェストリポジトリ`) 上にある該当マニフェストのバージョン情報を書き換える
* [Argo CD Image Updater](https://github.com/argoproj-labs/argocd-image-updater) を用いて、コンテナレジストリの更新契機でマニフェストリポジトリ上にある該当マニフェストのバージョン情報を書き換える

この方法により「あるブランチの更新契機で、そのブランチに紐づく環境への CD」を実現できます。

しかしながら、上記から一歩進んで「アプリケーションリポジトリに PR が出るたびに新規 Namespace を作成しそこに新規環境を立ち上げる」というふうに監視対象ブランチが動的に変わる場合、 Argo CD Image Updater では実現できないです。

上記のように動的に新規環境を立ち上げる仕組みは、特に複数人で同じアプリケーションを開発する際において、共用利用のステージング環境へマージする前に自分の実装の動作を試す事ができるためとても便利です。 例えば有名な PaaS である Heroku では [Review Apps](https://devcenter.heroku.com/articles/github-integration-review-apps) という機能でこれを実現しています。

そのため Kubernetes を利用している場合においても同じことを実現したいというのがモチベーションとしてありました。

# reviewapp-operator

reviewapp-operator は「アプリケーションリポジトリに PR が出るたびに新規 Namespace を作成しそこに新規環境を立ち上げる」ことを実現するために、Argo CD と協調して動作します。
reviewapp-operator の責務は主に「アプリケーションリポジトリの PR の更新契機でマニフェストリポジトリのマニフェストを作成・削除する」ことであり、実際にマニフェストリポジトリからマニフェストを Kubernetes に適用するのは Argo CD の責務となります。

![reviewapp-operator の workflow](https://raw.githubusercontent.com/ShotaKitazawa/zenn-articles/master/images/about-reviewapp-operator/workflow.jpg)

**なお、以降この記事は `reviewapp-operator v0.0.5` を前提に話します。**

## 使い方

reviewapp-operator の使い方について説明します。

本当はドキュメントのリンクを貼り付けるだけで済めば良かったのですが、 reviewapp-operator のドキュメントは現状皆無なので、ここで各リソースの役割の説明や使い方について説明します。(後日、公式ドキュメントを作成予定です🙇)

### reviewapp-operator のインストール

TODO

### reviewapp-operator の提供する Custom Resources

TODO

### 実際の設定例

TODO

# reviewapp-operator を3ヶ月ほど運用してみての感想

上記で紹介した reviewapp-operator は 2021/10 あたりから [Dreamkast](https://github.com/cloudnativedaysjp/dreamkast) というオンラインカンファレンスプラットフォームの開発環境を管理するのに利用されています。
ここからは、reviewapp-operator を実際に運用してみて分かったことやその感想を書いていきます。

## ManifestsTemplate リソースのマニフェストが非常に読みにくい

[実際の設定例](#実際の設定例) にて実際の ManifestsTemplate の書きっぷりをお見せしましたが、string なフィールドにマニフェストを列挙する形になっており非常に読みにくいです。
この問題を解決するために、以下の方針を考えています。

* 複数の yaml 形式のマニフェストファイルを引数に与えることで ManifestsTemplate リソースを出力する CLI ツールを実装
* Argo CD の [Config Management Plugins](https://argo-cd.readthedocs.io/en/stable/user-guide/config-management-plugins/) を利用して、Argo CD による apply の前に上記 CLI ツールで ManifestsTemplate リソースの記述されたマニフェストを生成
    * ちなみに Argo CD v2.2 で Config Management Plugins V2 がリリースされ、 `ConfigManagementPlugin` カスタムリソースとして設定を記述できるようになった。([参考](https://blog.argoproj.io/argo-cd-v2-2-release-candidate-4e16e985b486))

なおこの問題は [#50 ManifestsTemplate, ApplicationTemplate が見にくいのをどうにかする](https://github.com/cloudnativedaysjp/reviewapp-operator/issues/50) という Issue で管理しています。

## 新機能の実装ネタが思ったよりもある

実は reviewapp-operator を実装するよりも前にも GitHub Actions で reviewapp-operator 相当のものを実現していたのですが ([発表資料](https://speakerdeck.com/shotakitazawa/cd-wakuhurofalsebian-qian)を参照)、「Review Apps 環境を構築・クリーンアップする他に様々な追加機能が欲しくなった際、その開発をちゃんとテスタブルな言語で行いたい」というのが reviewapp-operator を実装するモチベーションの 1 つとしてありました。

個人的にこの理由は「言うても機能追加はそうそうないだろう」と高をくくっていたのですが、いざ運用してみると様々な追加機能が欲しくなりました。

* [アプリケーションリポジトリの該当 PR に紐づく CI が pass している場合のみ reviewapp-operator でマニフェストの更新を行うようにする](https://github.com/cloudnativedaysjp/reviewapp-operator/issues/3)
* [reviewapp-operator による dev 環境の構築・クリーンアップ時に任意のスクリプトをフックしたい](https://github.com/cloudnativedaysjp/reviewapp-operator/issues/69)
* [reviewapp-operator が dev 環境を構築・クリーンアップしたことを通知する](https://github.com/cloudnativedaysjp/reviewapp-operator/issues/71)

機能の案は [GitHub Issue](https://github.com/cloudnativedaysjp/reviewapp-operator/issues) で管理しているため、自分の手が空いたときにやりたいもの順で実装していこうと思っています。

## 統合テストに時間がかかる

これは reviewapp-operator の話というよりは、Kubernetes Operator の「Custom Resource オブジェクトの状態を監視し、更新などを契機に該当 Custom Resource オブジェクトや外部リソース (例. GitHub PullRequest) を CRUD する」という、宣言的 API を実現するための実装の難しさの話です。

この Operator の一連の動作を統合テストするためには、外部リソースが期待する状態に収束するまで待つ必要があります。このとき、 Operator の実装が期待通り動作すれば問題ないのですが、実装が間違っている場合はいくら待っても外部リソースが期待する状態に収束しないため、タイムアウトするまで待つことになります。
このタイムアウトの値が曲者で、長すぎると実装が誤っていた際の待ち時間が長くなるのですが、短すぎると Reconciliation Loop により期待する状態になる前にテストが落ちてしまうことがたまに発生してしまいます。

このあたりについて、今の所「うまい時間を見つける」以上の案がないので、もし何か良い手があればコメント等で教えていただきたいです。

## 「reviewapp-operator 何もわからん」

使い方の項で述べたとおり現状 reviewapp-operator のドキュメントが皆無であるため、チームメンバーであっても reviewapp-operator 周りについて触れないという現状になってしまっていると感じています。

幸い reviewapp-operator で立ち上げる dev 環境のマニフェスト自体を更新したい機会が現状そこまで多くないためその機会が訪れるたびに自分が作業者になれば良いとして回っていますが、この作業が属人化してしまってる現状はあまり健全でないので是正していきたいです。

reviewapp-operator は OSS として公開されているため、他の方にも利用してもらえるよう、ドキュメントや CRD のフィールドに対する description の拡充はできるだけ優先度を高くしてやっていこうと思っています。
// 本当はこの記事の公開までにドキュメントを整備したかった...orz

## まとめ

「アプリケーションリポジトリに PR が出るたびに新規 Namespace を作成しそこに新規環境を立ち上げる」ことを実現する Kubernetes Operator である [reviewapp-operator](https://github.com/cloudnativedaysjp/reviewapp-operator) について紹介しました。
また、reviewapp-operator を複数人で開発されるアプリケーションの開発環境構築へ実際に導入してみて、分かったことや感想を書きました。

reviewapp-operator は現状 [Dreamkast](https://github.com/cloudnativedaysjp/dreamkast) でのみ利用されていますが、動的に dev 環境を立ち上げるという用途でより汎用的に利用可能なので、引き続き機能開発やドキュメントの充実をしていこうと思っています。
