+++
Categories = ["AWS"]
Tags = ["AWS", "Rust", "Cost", "FinOps", "AI-DLC"]
date = "2026-04-06T00:00:00+09:00"
title = "AWS費用監視ツール（Rust版：AI-DLC体験記）"
archives = ["2026", "2026-04", "2026-04-06"]
+++

本記事は「AWS費用監視ツール」シリーズの番外編です。

- [前編：AWSマネジメントコンソール & AWS CLI](https://t-tkm.github.io/blog/posts/2026/03/aws_costcheck_cli/)
- [後編：Lambda活用](https://t-tkm.github.io/blog/posts/2026/04/aws_costcheck_lambda/)

## 1. はじめに

前編・後編では、Cost Explorer APIをAWS CLIで叩くところから、Python（boto3）＋ AWS SAMによるLambdaデイリー通知まで、一通りの仕組みを整えました。

ふと、「このツール、Rustで書いたらどうなるんだろう？」と思い立ちました。Python版はLambda上で動くサーバーサイドの仕組みでしたが、Rustで書けばランタイム不要のネイティブバイナリとして配布できます。手元のPCにPythonもboto3もSAMも不要で、バイナリ1本を置けば動く形を試してみたくなったのです。

せっかくならと、今回はツールの実装に **AI-DLC**（AI-Driven Development Life Cycle）を活用してみました。「要件をドキュメントに書いて、AIにコードを生成させる」という構造化された開発フローを、シンプルなCLIツールで体験してみたレポートを兼ねています。

今回作成したプロジェクトはこちらです。

<a href="https://github.com/t-tkm/awscosttool-rust" target="_blank" rel="noopener">
    <img src="https://gh-card.dev/repos/t-tkm/awscosttool-rust.svg">
</a>

### 1.1 この記事で扱うこと

| 項目 | 内容 |
|------|------|
| ツールの概要 | Rust製CLI。バイナリ1本でAWS費用をサービス別に表示 |
| 認証方式 | IAM Identity Center（SSO）推奨。前編・後編と共通 |
| AI-DLC | ワークショップを踏まえてCLIツールで試した体験記。ドキュメントが残る心強さ |
| コード解説 | モジュール構成の概要 |

Lambda版（後編）との使い分けイメージとしては、「毎日自動通知はLambdaに任せて、手元でサッと確認したいときはRust版CLIを使う」という感じでしょうか。

## 2. AI-DLCとは

[AI-DLC（AI-Driven Development Life Cycle）](https://github.com/awslabs/aidlc-workflows)は、AWS Labsがオープンソースで公開している開発メソドロジーです。「人間がレビュー・承認しながら、AIと協調して開発ライフサイクル全体を進める」という考え方を、Inception（要件定義）→ Construction（設計・実装）という段階的なフローとして体系化しています。

AI-DLC自体は特定のIDEに依存しておらず、Cursor、Claude Code、Amazon Q Developerなど複数のAIツールで利用できます（今回のリポジトリには `.kiro/` ディレクトリが残っており、開発中にKiroも試しました）。

セットアップや詳しい使い方は、AWSの公式ワークショップが非常によくまとまっています。

> [AI-Driven Development Life Cycle Workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/e1a0e9ed-f484-4d68-ba0e-357d2e134ad1/en-US)

本記事では「このメソドロジーをCLIツール開発に試してみた」という観点でポイントを絞って紹介します。

## 3. ツールの概要

### 3.1 できること

実行すると、当月1日から実行日までのAWS費用を、**クレジット適用後・適用前** の2本立てでサービス別に表示します。

```
------------------------------------------------------
AWSアカウント 123456789012
03/01～03/30のクレジット適用後費用は、45.23 USD です。
- Amazon EC2: 30.10 USD
- Amazon S3: 10.05 USD
- AWS Lambda: 5.08 USD
------------------------------------------------------

------------------------------------------------------
AWSアカウント 123456789012
03/01～03/30のクレジット適用前費用は、50.00 USD です。
- Amazon EC2: 35.00 USD
- Amazon S3: 10.00 USD
- AWS Lambda: 5.00 USD
------------------------------------------------------
```

前編で紹介したCLIコマンド（`aws ce get-cost-and-usage`）の出力と同じ情報を、バイナリ1本で取得できます。

### 3.2 仕様

| 項目 | 内容 |
|------|------|
| 集計期間 | 当月1日〜実行日 |
| メトリクス | AmortizedCost |
| グループ化 | サービス別 |
| リージョン | us-east-1 固定（Cost Explorer APIの制約） |
| 最小表示閾値 | 0.01 USD 未満は非表示 |

> **月初実行時の補正について**
> Cost Explorer の `TimePeriod.End` は排他指定で、`Start` より後の日付でなければなりません。月初（1日）に実行すると `Start=2026-04-01, End=2026-04-01` となり無効になるため、この場合だけ `End` を翌日（`2026-04-02`）に補正しています。表示ラベルは「End − 1日」で計算するため、画面上は正しい日付が表示されます。

## 4. インストールと実行

### 4.1 バイナリのダウンロード（推奨）

[Releases](https://github.com/t-tkm/awscosttool-rust/releases) から、環境に合ったバイナリをダウンロードします。

| ファイル名 | 対象環境 |
|-----------|----------|
| `aws-cost-report-*-aarch64-apple-darwin.zip` | M1/M2/M3 Mac |
| `aws-cost-report-*-x86_64-apple-darwin.zip` | Intel Mac |
| `aws-cost-report-*-x86_64-unknown-linux-gnu.tar.gz` | Linux x86_64 |
| `aws-cost-report-*-aarch64-unknown-linux-gnu.tar.gz` | Linux ARM64 |
| `aws-cost-report-*-x86_64-pc-windows-msvc.zip` | Windows x86_64 |

展開後、実行権限を確認してください。

```bash
chmod +x aws-cost-report
```

### 4.2 ソースからビルド

Rust環境（stable 1.75以上）があれば、ソースからビルドすることもできます。

```bash
git clone https://github.com/t-tkm/awscosttool-rust.git
cd awscosttool-rust
cargo build --release
```

ビルドが完了すると `target/release/aws-cost-report` が生成されます。

### 4.3 認証情報の準備

認証方式は前編・後編と共通です。IAM Identity Center（SSO）の利用を推奨します。まだセットアップが済んでいない方は [前編の付録A・B](https://t-tkm.github.io/blog/posts/2026/03/aws_costcheck_cli/) を参照してください。

必要なIAM権限は最小限の2アクションです。

```json
{
  "Effect": "Allow",
  "Action": [
    "ce:GetCostAndUsage",
    "sts:GetCallerIdentity"
  ],
  "Resource": "*"
}
```

前編で作成した `billing-user` プロファイルをそのまま使えます。

### 4.4 実行前チェック

初回実行の前に、次の2点を確認してください。

**① macOSの隔離属性を外す（ブラウザでダウンロードした場合のみ）**

ブラウザ経由で取得したバイナリには隔離（quarantine）が付いており、そのままでは実行できないことがあります。

```bash
xattr -d com.apple.quarantine aws-cost-report
```

この属性がない場合はエラーになりますが、無視してかまいません。

**② AWS SSOでログインする**

```bash
aws sso login --profile billing-user
export AWS_PROFILE=billing-user
```

一時認証情報の有効期間は最大12時間です。期限が切れたら再度 `aws sso login` を実行してください。

### 4.5 実行

```bash
./aws-cost-report
```

ログレベルを上げると、0.01 USD 未満で除外されたサービスの一覧もDEBUGログで確認できます。

```bash
RUST_LOG=debug ./aws-cost-report
```

## 5. コードの構成（概要）

コードは5つのファイルに分かれています。

```
src/
├── main.rs      # 起動・処理の流れを制御
├── config.rs    # 定数（リージョン名、メトリクス名など）
├── cost.rs      # Cost Explorer APIの呼び出し
├── date.rs      # 集計期間の計算
└── report.rs    # 結果の表示
```

前編で `aws ce get-cost-and-usage` コマンドに渡していたパラメータ（期間・メトリクス・フィルタ）が、それぞれRustのコードとして表現されているイメージです。AWS SDKを使うことで、CLIコマンドと同じAPIを直接呼び出しています。

コードの詳細に興味がある方は [GitHub](https://github.com/t-tkm/awscosttool-rust) のソースを参照してください。

## 6. 試してみた感想

### 6.1 「ドキュメントが残る」のが一番の収穫

AI-DLCを試してみて一番良かったのは、**設計ドキュメントが自然に残ること**です。

`aidlc-docs/` 以下に、要件定義・機能設計・NFR設計・ビルド手順まで一式が出力されます。今回のリポジトリで言うと、こんな構成になっています。

```
aidlc-docs/
├── inception/
│   └── requirements/
│       └── requirements.md   ← 要件定義書（FR・NFR・スコープ外）
└── construction/
    └── aws-cost-report/
        ├── functional-design/ ← 機能設計
        ├── nfr-requirements/  ← 非機能要件
        └── plans/             ← 実装計画
```

通常、個人ツールを作るとき、ドキュメントは後回しになりがちです。「動いてるんだからいいや」で済ませてしまい、数カ月後に「なんでこう実装したんだっけ？」となった経験、あるあるではないでしょうか。

AI-DLCのフローではコードより先にドキュメントを作ることが前提なので、出来上がったときには設計の意図まで含めたドキュメントが揃っています。これは個人開発でも、後から見返したときにかなり心強いです。

### 6.2 「やらないこと」を先に決める効果

要件定義で、Lambda対応・Teams通知・Secrets Manager連携・構造化ログをOut of Scopeとして明文化しました。「シンプルなCLIバイナリ」という軸がブレなかったのは、この一手間のおかげだと感じています。

曖昧なまま実装を始めると、AIは「便利そうな機能」を盛り込もうとします。スコープを言語化しておくと、AIへの指示も通りやすくなり、レビューもしやすくなりました。

> 「AIが全部書いてくれる魔法ツール」ではありません。要件定義の精度がそのままコード品質に直結するのは、人間が書く場合と変わらないなと感じました。

### 6.3 4/1に実行したらバグが発覚——要件から直した話

ツールが完成したと思っていた4月1日、実際に実行してみたらAPIエラーが返ってきました。

原因はこうです。Cost Explorer の `TimePeriod.End` は**排他指定**で、`Start` より後の日付でなければなりません。月の初日に実行すると `Start=2026-04-01, End=2026-04-01` となり、同じ日を指定することになってAPIが弾くのです。

```
Error: Failed to call Cost Explorer GetCostAndUsage
```

コード自体の修正はシンプルで、「月初のときだけ `End` を翌日に補正する」という処理を `date.rs` に追加しました（[該当コミット](https://github.com/t-tkm/awscosttool-rust/commit/a7bfa8daeebbc306d4726ce29de9e8732c7b7bed)）。

```rust
let end_date = if today > month_start {
    today
} else {
    month_start + Duration::days(1)  // 月初のみ翌日に補正
};
```

ここからがAI-DLCらしいポイントです……と言いたいところなのですが、正直に白状すると、**実際にはコードを先に直してしまいました**。バグを目の前にすると、つい「とりあえず直したい」という気持ちが勝ってしまうんですよね。

本来のAI-DLCフローは「要件を修正 → コードを更新」の順番です。でも実際にやったのは「コードを修正 → 後からドキュメントを整合させる」でした。

それでも、ドキュメントを後追いで更新したことで、要件定義書（FR-06）もきちんと直しました。

**修正前のFR-06:**
```
- 開始日: 当月1日
- 終了日: 当日
```

**修正後のFR-06:**
```
- 開始日: 当月1日（TimePeriod.Start）
- API用終了日（TimePeriod.End）: 通常は実行日（当日）。
  Cost Explorer は End を排他とし、Start より後である必要があるため、
  実行日が当月1日のときだけ End を「月初の翌日」に補正する
- 表示ラベル: End の日付 − 1日として扱う
```

さらに業務ルール・業務モデル・ドメインエンティティ・コードサマリー・NFR・ビルド手順・`audit.md`（変更履歴）まで、関連する13ファイルが一式更新されました（[該当コミット](https://github.com/t-tkm/awscosttool-rust/commit/0d1cbce0f010b929fb50a639ed30d3d15be52c63)）。

通常の個人開発では、バグを直して「終わり」になりがちです。順番は前後しましたが、AI-DLCのフローがあったからこそ「ドキュメントも直さないと」という意識が自然に働きました。「なぜこの実装になっているのか」が要件レベルから追跡できる状態——これが、冒頭で「心強い」と言った理由です。

> フローを完璧に守れなくても、ドキュメントを更新する習慣が残ります。それだけでも十分に価値があると感じました。

## 7. まとめ

本記事では、「AWS費用監視ツール」シリーズの番外編として、Rust製CLIバイナリの紹介と、AI-DLCを使った開発体験をまとめました。

- **Rust版ツール**: ランタイム不要のネイティブバイナリ。前編・後編と同じIAM Identity Center認証で動作し、クレジット適用後・適用前のコストをサービス別に表示
- **AI-DLC体験**: 設計ドキュメントがフローの産物として `aidlc-docs/` に残るのが心強い。「やらないこと」をスコープ外に明文化することで、AIへの指示が通りやすくなる

AI-DLCの詳細はワークショップで体験するのがおすすめです。

> [AI-Driven Development Life Cycle Workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/e1a0e9ed-f484-4d68-ba0e-357d2e134ad1/en-US)

前編のCLI確認、後編のLambdaデイリー通知、そして今回のRust版CLIと、同じAWS Cost Explorer APIを軸に異なるアプローチを試してきました。費用監視という地味なテーマですが、ツールの作り方・開発フローまで含めると、学べることが意外と多いと感じています。

## [付録] GitHub Actions によるマルチプラットフォームリリース

`cargo build --release` は実行マシンのアーキテクチャ向けバイナリしか生成しません。他プラットフォーム向けはクロスコンパイルが必要で、ツールチェーンの追加インストールが必要になります。

今回はGitHub Actionsのリリースワークフロー（`.github/workflows/release.yml`）に任せることにしました。タグをpushすると、全プラットフォーム分のバイナリが自動ビルドされてReleasesに添付されます。

```bash
git tag v0.1.0
git push origin v0.1.0
```

個人ツールでも「リリースバイナリを配布できる」状態にしておくと、他の環境へ持ち運ぶときに便利です。ぜひ試してみてください。

---

*Amazon Web Services、およびその他のAWS商標は、米国およびその他の諸国におけるAmazon.com, Inc.またはその関連会社の商標です。*
