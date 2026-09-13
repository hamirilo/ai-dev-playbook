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

### サーバ側に用意するもの

pullして起動するだけの環境に必要なものは次のとおりです。ビルドに要るtoolは入れません。

- Docker Engineとcompose plugin。言語runtime、package manager、ビルド用toolは不要。
- デプロイを実行するユーザーと、そのユーザーがdockerを使える権限。
- そのユーザーから読めるregistryの資格情報（`~/.docker/config.json`、または`DOCKER_CONFIG`で指定した場所）。
- `ghcr.io`と`pkg-containers.githubusercontent.com`への443到達。
- 起動するイメージの参照（tagではなくdigest）を持つファイル。デプロイのたびにここを書き換える。
- 実行時にだけ必要な環境変数・secret。ビルド時にだけ必要な値とは分けて置く。
- イメージを置くディスクの余裕と、古いイメージを整理する手順。ディスク不足はpullの失敗として現れる。
- 正しい時刻（NTP）。ずれると証明書やトークンの検証で落ちる。

### 使うトークンの種類

本番サーバからpullするときに使えるものと、使えないものを分けます。

| 種類 | サーバからのpullに使えるか | 必要な権限 | 期限と注意 |
|---|---|---|---|
| personal access token (classic) | 使える。Container registryの認証として案内されている方式 | `read:packages`だけ。`repo`や`write:packages`を足さない | 期限を設定し、更新手順と担当を決める。SSOのorganizationではauthorizeが必要 |
| fine-grained personal access token | 対応状況が変わるため、使う前に本番と同じ経路でpullできるか確認する | organization permissionsの`Packages: Read-only` | 期限が必須。更新の運用が要る |
| GitHub Appのinstallation token | 使えるが、短時間で失効するため取得の仕組みが要る | Appの`Packages: Read-only`と、対象organizationへのinstall | 都度取得する前提。取得処理自体の資格情報（App IDと秘密鍵）の管理が増える |
| Actionsの`GITHUB_TOKEN` | Actionsのjob内だけ。サーバへ持ち出さない | workflowの`permissions:`で`packages: read` | jobの終了で失効する。値をサーバのfileへ保存しない |
| deploy key・SSHの鍵 | 使えない | — | registryの認証はHTTPSで、SSHの鍵を受け付けない。gitのcloneの認証と混同しやすい |

迷う場合は、pull専用のアカウントにclassic PATを`read:packages`だけで作り、それをサーバへ置きます。期限と更新担当を決めたうえで、[packageの可視性とrepositoryリンク](#packageの可視性とrepositoryリンク)の読取権限を確認します。

### pull用の資格情報

- 実行環境へ置くのはpull専用の資格情報とし、CIのsecretやpushできるトークンを使い回さない。
- `docker login ghcr.io -u <user> --password-stdin` の形で渡す。トークンをコマンド引数へ書かない。履歴とプロセス一覧に残る。
- `-u`にはトークンを発行したGitHubアカウント名を入れる。権限はトークン側で決まるため、ここを変えても足りない権限は補えない。
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

### 実行ユーザーとSSHでの作業

`docker compose pull`等は、実行ユーザーのDocker設定を読みます。認証は「サーバに対して」ではなく「ユーザーのhome配下の設定に対して」残るため、誰がどのhomeで実行するかで結果が変わります。

- SSHでログインした自分のユーザーで`docker login`し、systemdやsudoで別ユーザーとして起動すると、認証していない状態でpullが走る。デプロイを実行するユーザーでloginする。
- `sudo docker ...`は`HOME`が`root`側へ変わる構成が多く、`~/.docker/config.json`を読む先も変わる。`sudo`を挟むなら、`sudo`を挟んだ状態でloginする。
- rootless Dockerとsystem daemonが混在する環境では、`DOCKER_HOST`の指す先によって読む設定が変わる。どちらへ繋いでいるかを確認する。
- homeが揮発する構成（都度作り直すサーバ、共有アカウント）では、`docker login`の結果を前提にしない。`DOCKER_CONFIG`で置き場所を明示するか、デプロイ手順の中でloginする。

`ssh server "docker compose pull"`のように非対話で実行する場合は、対話ログインとは環境が変わります。

- 非対話shellでは`.bashrc`等のprofileが読まれず、`PATH`や環境変数が対話時と違う。トークンを環境変数から渡している手順は、この差で静かに失敗する。
- `~/.docker/config.json`の`credsStore`がGUI前提のcredential helper（鍵束、`pass`等）を指していると、SSH越しでは鍵束を開けず`error getting credentials`で落ちる。デプロイ用ユーザーではhelperを使わないか、非対話で開ける方式にする。
- 認証の確認は、実際にデプロイで使う経路（同じユーザー、同じ非対話コマンド）で行う。対話ログインで`docker pull`が通ることは、非対話で通る根拠にならない。

CI等から遠隔でデプロイする場合は、デプロイのたびにloginして終了時に`docker logout`するか、資格情報の置き場所と保持期間をプロジェクト側で明示します。

## 移行チェックリスト

サーバビルドから移行する前に、以下を確認します。

- 実行環境がイメージレジストリへ到達できる
- pull用の最小権限の資格情報を実行環境へ安全に渡せる
- pull用トークンの期限と更新手順が決まっている
- デプロイで実際に使うユーザーと実行方法（sudo、systemd、非対話SSH）でpullを確認できる
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
| SSHで入れば通るが、`ssh server "..."`の非対話実行だと認証で落ちる | profileが読まれず環境変数が変わっていないか。実際に使う経路で確認する。 |
| `error getting credentials`で止まる | `config.json`の`credsStore`がGUI前提のhelperを指していないか。SSH越しでは開けない。 |
| `sudo`を付けたときだけpullできない | `sudo`で`HOME`が変わり、別の`config.json`を読んでいないか。 |
| gitのcloneはできるのにpullできない | SSHの鍵とregistryの認証は別。tokenで`docker login`しているか。 |
| 途中まで進んでpullが失敗する | ディスクが埋まっていないか。古いイメージを整理する手順があるか。 |

## リリース前の確認

- CIでイメージビルドが成功している。
- 配布するタグとdigestを記録できる。
- ステージングで対象digestを起動できる。
- 実行環境へ渡す資格情報で、対象のイメージをpullできる。
- 静的ファイル、DB migration、外部依存を含む最低限の動作を確認した。
- ロールバック対象のdigestと、DBスキーマに応じた復旧またはforward fixの方針が分かる。

