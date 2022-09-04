---
title: "私的 Go 言語での SlackBot 開発のベストプラクティス 2022"
emoji: "⚡️"
type: "tech"
topics: ["slackbot","go"]
published: true
---

個人やコミュニティ活動等で Go 言語を用いて何回か Slack Bot を作り個人的にしっくりと来た開発方法を確立できたため、備忘録として残します。

なお参考までに、ここに書かれている話を元に実装したソフトウェアは以下になります。以降の説明でも具体例として利用します。

* https://github.com/cloudnativedaysjp/chatbot

## 利用する Slack クライアントや API

この章では Slack がいくつか提供している機能のうち何を利用するべきか、また何のライブラリをどのように利用すべきかについて記述します。
なおこの記事は 2022 年 9 月に書いたものであるため、 Slack の公開する API の変更などで情報が古くなる可能性があることにご留意ください。

### Socket Mode を利用する

2022 年現在、 Slack Bot で調べると過去の情報から最新の情報まで様々な記事が見つかり、また [公式サイト](https://api.slack.com/apis) を見ても様々な API が提供されていることが確認できるため、何を利用するべきか迷ってしまうと思います。

個人的によく利用するのは Socket Mode で利用する方法です。
Socket Mode は WebSocket を用いることで HTTP server を外部に expose せずに Bot を動かすことが出来ます。またドキュメントに書かれている通り、Socket Mode では [Events API](https://api.slack.com/events-api) や [Slack プラットフォームの interactive components](https://api.slack.com/interactivity) を HTTP server を外部に expose した場合と同じように利用することが可能です。

Socket Mode について詳しくは [公式ドキュメント](https://api.slack.com/apis/connections/socket) をご覧ください。

### github.com/slack-go/slack の提供するハンドラ機能を利用する

Go 言語で Slack Bot を開発する際に真っ先に上がるライブラリは [slack-go/slack](https://github.com/slack-go/slack) だと思います。私も Go 言語で Slack Bot を開発する際はこれを用います。

[slack-go/slack](https://github.com/slack-go/slack) は上述した Socket Mode の利用をサポートしています。

## 開発で意識すること

この章では開発で意識することのうち、特にツールに依存しない一般的な話について記述します。

### パッケージ構成はシンプルに

自分の体感として、 Slack Bot は特に新しい機能を欲しいと思う人間が開発者である場合が多いため、複数人で開発する機会が多いです。
そのためアプリケーションのどこで何をやっているかを明確にするために、ペライチのソースコードではなく、ある程度パッケージ分けは必要になります。
しかしながら世の中にあるソフトウェアアーキテクチャのパターンは様々で、中には機能の拡張性やテスタビリティを上げる代わりに複雑な構成であるパターンも数多く存在します。

個人的な意見として、 Slack Bot に実装する機能は以下の特性を持つことが大半です。

* 過度なモデル化の必要がない
* TODO
* 

そのため私は、以下の構成で Slack Bot を実装することが多いです。

| パッケージ名 | 役割                                                                                                                        |
|:------------:|:---------------------------------------------------------------------------------------------------------------------------:|
| `controller` | 入力に対するバリデーションや必要な値の取得を行う、十分単純な処理しか行わない場合は service を経由せずに外部問い合わせを行う |
| `service`    | 複数の外部問い合わせや、内部の詳細なロジックを記述する                                                                      |
| `view`       | Slack に送信する json を組み立てる                                                                                          |
| `model`      | DTO やグローバル変数の配置先                                                                                                |
| others       | 外部への問い合わせを行うためのパッケージ (例: [`gitcommand`](TODO), [`githubapi`](TODO))                                    |

![](TODO)

なお、具体例で示したコードでは外部問い合わせを行うパッケージ内で interface を定義し、 それらのパッケージを利用している controller や service パッケージの構造体に対しては main から DI しています。
これにより controller や service に少し複雑なロジックを書きたくなったときにモックを利用したユニットテストをすることが可能になるため、自分は外部問い合わせを行うパッケージに対しては常に interface を提供し抽象化するようにしています。

### Bot から送るメッセージは Bot を動作させなくても分かるようにする

Slack には [Block Kit](https://api.slack.com/block-kit) という機能が提供されており、Bot が Slack に任意の json を送信することにより、単純なテキストメッセージだけでなくボタンやプルダウンメニューを表示させることが出来ます。

![](TODO)

また、上記の json を視覚的に組み立てるための [Block Kit Build](https://app.slack.com/block-kit-builder) というページも提供されています。これにより Bot から送信したいメッセージを簡単に組み立てることが可能です。

しかしながら
TODO

### CI は最初に用意する

上述の [Bot から送るメッセージは Bot を動作させなくても分かるようにする](TODO) にて、 view に対するユニットテストを通すことで Bot から Slack に送信されるメッセージが意図通りであることを検証しました。しかしながらこのテストを実行する方法が手動実行しかない場合、いつの間にかテストが実際のコードに追従されていないことが起こりえます。

またテストのためにモックを利用している場合は、そのモックが実際のコードに追従され忘れることも起こりえます。

そのため上記が追従されていることを main ブランチにマージする前に常に確認するよう、 CI でテストを実行するべきです。

GitHub のパブリックリポジトリで開発する場合 GitHub Action を無料で利用することが可能です。
私がよく書くのは、[Action](https://github.com/cloudnativedaysjp/chatbot/blob/main/.github/workflows/test.yml) にて `make test; git diff --exit-code` のみを実行するようにし、 [Makefile](https://github.com/cloudnativedaysjp/chatbot/blob/main/Makefile) にてモックの生成及びテストの実行をすることで、テストが通らなかったときかファイル生成コマンドにより push したコミットと diff が出たときに CI が失敗するようにしています。

### ヘルプメッセージにドキュメントのリンクを記載する

TBW

## まとめ

以上が個人的な Go 言語での Slack Bot 開発のベストプラクティスでした。
もっとこうした方が良いよ等の意見があればぜひコメントしてくださると幸いです。
