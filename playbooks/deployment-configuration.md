# デプロイ設定の確認

Gitで管理しない設定や実行環境固有の依存関係を、デプロイ前に発見するためのPlaybookです。

複数Applicationを運用すると、`.env`、secret、host側ディレクトリ、外部サービス設定などがGitの変更から分離されます。特に `.env.example` に新しい必須変数が追加されても、本番環境の `.env` は自動では更新されないため、「コードは正しいのに本番だけ動かない」という事故につながります。

このPlaybookでは、**設定の定義・実環境・変更検知を分離し、デプロイ時に不足を機械的に検出する**ことを基本とします。

## 使う場面

- 新しいApplicationを本番へデプロイするとき
- `.env.example`、Compose、Django設定などに変更を入れるとき
- Git管理外のsecretやhost設定を追加・変更するとき
- 複数Applicationを同じサーバで運用するとき
- 「初回は動くが、次のリリースで突然動かなくなる」問題を防ぎたいとき

## 基本方針

### 1. Gitには値ではなく必要条件を残す

秘密値や本番固有値はGitへ入れません。一方で、**何が必要なのかはGitから分かる状態**にします。

代表的には次を使います。

- `.env.example`: 必要な環境変数名と説明可能なデフォルト値
- `docs/deployment.md`: Git管理外の設定、host依存、初回作業
- Composeの設定: 必須変数を明示し、不足時は早く失敗させる
- アプリケーションの設定検証: 起動に必要な値を起動時に検証する

`.env.example`にはsecretの実値を入れません。また、本番専用の値をそのままサンプルへコピーしないでください。

### 2. `.env.example`だけを正本にしない

`.env.example`は「環境変数の一覧」を表すには便利ですが、すべての実行時依存関係を表せるとは限りません。

例えば次のようなものは別途管理します。

- Docker volumeやhost側ディレクトリ
- 外部DB、Object Storage、OIDC等の接続先
- DNSや証明書
- Docker registryのpull認証
- OS、Docker、ネットワークなどhost側の前提
- 初回migration、superuser作成、データ投入

そのため、デプロイ時には「環境変数」「secret」「host設定」「外部サービス」をまとめて確認します。

## 設定変更を見逃さない仕組み

### 必須: 変更をPRで見えるようにする

環境変数を追加・削除・変更するPRでは、アプリケーションコードだけでなく、次を確認します。

- `.env.example`の変更
- 本番・stagingでの反映が必要か
- 既存環境への移行手順が必要か
- secretの新規発行・更新が必要か
- 不要になった変数を削除するか
- `docs/deployment.md`等の記述を更新するか

PR本文またはチェックリストに「Production configuration impact」を設けると、コードレビューに埋もれにくくなります。

### 推奨: 必須変数を機械的に検査する

デプロイ時に、アプリケーションが必要とする変数が実行環境に存在するかを検査します。

例えばComposeでは、必須値を明示できます。

```yaml
environment:
  SECRET_KEY: ${SECRET_KEY:?SECRET_KEY is required}
  DATABASE_URL: ${DATABASE_URL:?DATABASE_URL is required}
```

Django側でも、本番設定の読み込み時に必須値を検証します。

```text
Environment check failed

Missing required variables:
  OIDC_CLIENT_ID
  OIDC_CLIENT_SECRET

Deployment aborted.
```

重要なのは、**本番でリクエストを受けてから失敗するのではなく、デプロイ処理の早い段階で失敗させること**です。

### より強くする: 定義と実環境のdriftを検査する

`.env.example`に依存するだけでなく、次の差分を検査できるようにします。

```text
required configuration
        │
        ├── .env.example
        ├── application settings
        └── deployment manifest
                │
                ▼
        production environment
```

少なくとも次を区別します。

| 状態 | 判定 |
|---|---|
| 必須変数が本番にない | ERROR |
| 必須変数が空 | ERROR（空を許可する場合を除く） |
| 本番に不要な古い変数が残っている | WARNINGまたはERROR |
| 本番だけに存在する許可済み変数 | OK |
| `.env.example`で変数名が変更された | ERRORまたはmigration対象 |

本番のsecret値そのものをCIログやPRへ出力する必要はありません。比較対象は**キーの存在**と、必要に応じて値が空でないことに限定します。

## デプロイ前チェック

新規デプロイまたは設定変更を含むリリースでは、次を確認します。

### Configuration

- [ ] `.env.example`と必要な環境変数の定義が一致している
- [ ] 新しく追加された必須変数を本番へ追加した
- [ ] 削除・改名された変数を本番から整理した
- [ ] secretの発行・更新が必要なら完了している
- [ ] secretをGit、Docker image、ログへ含めていない

### Infrastructure

- [ ] 必要なhost側ディレクトリが存在する
- [ ] volume / mount先が正しい
- [ ] 必要なDocker networkが存在する
- [ ] 外部サービスへ到達できる
- [ ] registryからimageをpullできる

### Application initialization

- [ ] migrationが必要か確認した
- [ ] 初回のみ必要な処理を確認した
- [ ] static files等の成果物を確認した
- [ ] health checkが成功する

## `.env.example`を変更したときの扱い

特に見落としやすいのは「既存Applicationへの新しい必須変数の追加」です。

例えば次の変更があった場合、コード変更だけでPRを完了させません。

```diff
 DATABASE_URL=
 SECRET_KEY=
+OIDC_CLIENT_ID=
+OIDC_CLIENT_SECRET=
```

PR作成時に、少なくとも次を確認します。

1. 新しい変数が本番で必須か判断する。
2. 必須なら、どこへ登録するかを確認する。
3. secretなら安全なsecret管理場所へ登録する。
4. デプロイ前checkで不足を検出できることを確認する。
5. デプロイ後にhealth checkまたは起動確認を行う。

**「`.env.example`を更新したので完了」ではなく、「実行環境が新しい契約を満たすことを確認して完了」**とします。

## 変更を検知するCIの考え方

CIではsecretの実値を取得せず、設定定義の変更を検出するだけでも効果があります。

例えばPRで次のファイルが変更された場合、configuration impact checkを必須にできます。

```text
.env.example
compose*.yaml
Dockerfile
config/settings*.py
*/settings*.py
docs/deployment.md
```

CIの結果として、次のようなチェックを出すとレビューで見落としにくくなります。

```text
Configuration impact

.env.example changed: YES
New required variables: 2
Removed variables: 0
Production configuration update required: YES

Deployment checklist must be completed.
```

このCIはsecretそのものを検証するものではありません。**「設定契約が変わったので、実環境の確認が必要」というシグナルを必ず出すこと**が目的です。

## 設定の責務を分ける

Applicationが増えた場合、全Applicationで同じ`.env`ファイルを共有するような運用は避けます。

各Applicationは自分の設定契約を持ちます。

```text
Application A
  ├── .env.example
  └── deployment.md

Application B
  ├── .env.example
  └── deployment.md

Application C
  ├── .env.example
  └── deployment.md
```

共通Playbookは確認方法を定義し、**実際に必要な変数・host設定・外部サービスは各Applicationが定義する**形にします。

## 失敗時の確認

「本番だけ動かない」場合は、まずGitのコード差分だけを見るのではなく、次を確認します。

1. `.env.example`の直近変更
2. 必須環境変数の追加・削除・改名
3. production environmentのキー一覧との差分
4. Docker Composeのenvironment / env_file変更
5. Docker image内に必要な設定を誤って要求していないか
6. host側volume、directory、permission
7. 外部サービスの認証情報・到達性
8. secretの期限切れ・rotation
9. migrationや初回処理の不足

設定値そのものをログへ出すのではなく、**「変数が存在するか」「どの設定が不足しているか」だけを安全に記録**します。

## 完了条件

デプロイは、次を満たした状態を完了とします。

- 必須設定がGit上で定義されている
- Git管理外の値・secretの配置場所が明確である
- 設定変更がPRで検知できる
- デプロイ時に不足設定を機械的に検出できる
- 本番環境のsecret値を公開せずにdriftを確認できる
- 設定変更後のhealth checkが成功している
