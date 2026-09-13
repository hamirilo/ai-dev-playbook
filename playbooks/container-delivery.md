# コンテナ配布

コンテナで配布するDjango + React Islandsアプリケーションを、CIで作成・検証し、実行環境では同一イメージをpullして起動するためのPlaybookです。

判断の根拠と必須事項は、[Architecture Standard: コンテナと配布](https://github.com/hamirilo/ai-dev-standards/blob/main/standards/architecture/README.md#10-コンテナと配布)を正とします。

## 使う場面

- 新しいコンテナ化アプリを用意するとき
- サーバ上でビルドする既存構成を、CIビルド・レジストリ配布へ移行するとき
- CIは成功するのに、イメージ作成・静的ファイル・private packageで失敗するとき

## 目標の流れ

1. CIがテストとイメージビルドを行う。
2. mainへの反映または明示的なリリースで、検証済みイメージをレジストリへpushする。
3. スキーマ変更がある場合は、トラフィック切替前に一度だけmigrationを実行する。
4. ステージング・本番はタグではなく、可能ならdigestでそのイメージをpullして起動する。
5. ロールバックは、DBスキーマとの互換性を確認したうえで直前に動作確認したdigestへ戻す。

開発機でのローカルビルドは許容します。ここで避けるのは、実行環境ごとに別の成果物をビルドしてデプロイする通常運用です。

## Django migrationとトラフィック切替

migrationを含むリリースでは、webコンテナの起動時に各replicaが`migrate`を実行する運用にしません。専用のrelease jobまたは一度だけ実行するコマンドとして扱います。

基本の順序は次のとおりです。

1. 旧アプリと新アプリの双方が扱える、後方互換な変更を先に追加する（expand）。
2. 新しいイメージの稼働前・トラフィック切替前に、migrationを一度だけ実行する。
3. 新しいイメージを起動してトラフィックを切り替える。
4. 旧イメージが不要になった後のリリースで、古い列・コードを削除する（contract）。

列の削除、意味の変更、大規模データ変換など、旧イメージと互換にならない変更ではimage digestだけで安全に戻せません。その場合は、メンテナンス時間、forward fix、DB復元のいずれを採るかをプロジェクト側で明示し、リリース前に確認します。

## 実装の基本

### Dockerfile

フロントエンドの成果物は、Dockerfile内のビルドステージで作ります。ホスト上で作った `static/dist` 等をコピーする運用にしません。

- 依存のインストールはロックファイルを使う。
- build stageでフロントエンドをビルドする。
- Djangoのstatic収集など、配布物を完成させる処理はビルド時に実行する。
- runtime stageには実行に必要なファイルだけを入れる。
- `.dockerignore` で仮想環境、node_modules、ローカル成果物、秘密情報を除外する。

ビルド時にアプリ設定を読む場合は、専用の環境変数または設定モジュールを用意します。実行時の設定を「DBなしでも通す」ために緩めないでください。

### 秘密情報とprivate package

- 認証情報はCIのsecretとして管理する。
- Docker buildで使うトークンは、BuildKit secret等で一時的に渡す。
- `ARG`、`ENV`、成果物、ログにトークンを残さない。
- パッケージの公開範囲と、CI・実行環境の読取権限を個別に確認する。

レジストリ、パッケージレジストリ、CIの実行基盤によって設定方法は異なるため、具体的な認証設定はプロジェクト側に置きます。GitHub Container Registryから実行環境がpullする場合の確認点は、[実行環境からのpullと認証](#実行環境からのpullと認証)にまとめます。

### プラットフォーム

CIでビルドする対象プラットフォームを、実行環境に合わせて明示します。Apple Siliconの開発機とLinuxサーバのようにアーキテクチャが異なる場合は、少なくとも実行対象のプラットフォームでイメージビルドを検証します。

## CIの分け方

PRでは、少なくとも次を通します。

- 依存関係の固定インストール
- 型チェック・Linter・基本テスト
- 実行対象プラットフォームでのイメージビルド
- 必要なら、イメージ内に期待する静的成果物があることの確認

mainまたはリリース時には、同じ手順で作成したイメージをレジストリへpushします。push用のジョブをPRで実行する必要はありません。

## 実行環境からのpullと認証

ここではレジストリをGitHub Container Registry（`ghcr.io`）とし、ステージング・本番がそこから同じイメージをpullする前提で書きます。他のレジストリを使う場合は、同じ観点をプロジェクト側で置き換えます。

### pull用の資格情報

- 実行環境へ置くのはpull専用の資格情報とし、CIのsecretやpushできるトークンを使い回さない。
- classic PATを使う場合、権限は`read:packages`だけにする。
- fine-grained PATやGitHub Appのinstallation tokenを使う場合は、対象のpackageを実際にpullできるところまで先に確認する。registryによって対応状況が異なる。
- `docker login ghcr.io -u <user> --password-stdin` の形で渡す。トークンをコマンド引数へ書かない。履歴とプロセス一覧に残る。
- ログイン結果は`~/.docker/config.json`へbase64で入るだけで、暗号化されない。デプロイ用ユーザーの領域に置き、読取権限を確認する。
- トークンには期限がある。期限と更新手順を決めて、切れる前に更新する。期限切れは次のデプロイまで気づけない。

### packageの可視性とrepositoryリンク

`ghcr.io`のpackageは、repositoryとは別にアクセス制御を持ちます。repositoryが見えていても、packageがprivateのままならpullできません。

- Dockerfileへ `LABEL org.opencontainers.image.source=https://github.com/<org>/<repo>` を入れ、packageをrepositoryへリンクする。リンクされたpackageはrepositoryの権限を引き継げる。
- 既にpush済みのpackageは、後からリンクしても可視性や権限が自動では変わらないことがある。package側の設定で、pullする側に読取権限が付いているかを確認する。
- organizationでSAML SSOを使っている場合、PATは作成後にSSOをauthorizeしないと拒否される。作成しただけでは通らない。

### 到達性とイメージ名

- 実行環境から`ghcr.io`へ到達できることを確認する。GHCRはmanifestとlayerで参照先が分かれるため、`pkg-containers.githubusercontent.com`への到達も必要になる。proxyやfirewallで絞っている環境では両方を許可する。
- イメージ名は小文字だけを使う。organization名やrepository名に大文字が入る場合、そのままの綴りではpullできない。

### 実行ユーザー

`docker compose pull`等は、実行ユーザーのDocker設定を読みます。手元のユーザーで`docker login`し、systemdやsudoで別ユーザーとして起動すると、認証していない状態でpullが走ります。デプロイを実行するユーザーでloginしてください。

CI等から遠隔でデプロイする場合は、デプロイのたびにloginして終了時に`docker logout`するか、資格情報の置き場所と保持期間をプロジェクト側で明示します。

## 移行チェックリスト

サーバビルドから移行する前に、以下を確認します。

- 実行環境がイメージレジストリへ到達できる
- pull用の最小権限の資格情報を実行環境へ安全に渡せる
- pull用トークンの期限と更新手順が決まっている
- pullするpackageへ読取権限が付いている（SSOが必要な組織ではauthorize済み）
- 実行環境のOS・CPUアーキテクチャが分かっている
- CIがprivate packageを取得できる
- Docker buildだけでフロント成果物とDjango staticが完成する
- 実行時にだけ必要な秘密情報と、ビルド時にだけ必要な値を分離できている
- 直前のイメージとDBスキーマの互換性を確認できる

満たせない項目がある場合は、制約・暫定手段・解消条件をプロジェクトのADRへ記録します。

## よくある失敗

| 症状 | 確認すること |
|---|---|
| テストは通るがイメージが作れない | CIにDocker buildが含まれているか。ビルド時設定・private package認証を確認する。 |
| 起動したが静的ファイルがない | Dockerfile内で成果物を作り、runtime stageへコピーしているか確認する。失敗を`|| true`で隠さない。 |
| サーバだけで依存取得に失敗する | サーバビルドをやめ、CIで完成イメージを作れるか確認する。 |
| Macでは動くがサーバで起動しない | 実行対象プラットフォームでビルド・起動確認しているか確認する。 |
| トークンがイメージに残った | Docker historyとビルドログを確認し、ARG/ENVではなくsecret注入へ変更する。 |
| 本番でだけ`denied`・`unauthorized`でpullできない | デプロイを実行するユーザーで`docker login`できているか。packageに読取権限が付いているか。SSOのauthorizeが済んでいるか。 |
| 昨日まで通っていたデプロイが認証で失敗する | pull用トークンの期限が切れていないか。期限と更新手順を決めているか。 |
| 認証は通るが`manifest unknown`になる | 認証ではなくタグ・digestの指定の問題。CIがpushした綴りと一致しているか。 |
| `docker login`は成功するのに起動時のpullが失敗する | 起動する側のユーザーが違っていないか（systemd、sudo、遠隔デプロイ）。 |
| pullがtimeoutやTLSで失敗する | `ghcr.io`だけでなく`pkg-containers.githubusercontent.com`へ到達できるか。 |

## リリース前の確認

- CIでイメージビルドが成功している。
- 配布するタグとdigestを記録できる。
- ステージングで対象digestを起動できる。
- 実行環境へ渡す資格情報で、対象のイメージをpullできる。
- 静的ファイル、DB migration、外部依存を含む最低限の動作を確認した。
- ロールバック対象のdigestと、DBスキーマに応じた復旧またはforward fixの方針が分かる。

