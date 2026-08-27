# Docker Compose ポート公開の整理

既存のDocker Compose構成で `ports:` を棚卸しし、必要な入口だけをホストへ公開するためのPlaybookです。

判断原則は [Architecture Standard: Docker Compose ネットワークとポート公開](https://github.com/hamirilo/ai-dev-standards/blob/main/standards/architecture/container-network.md) を正とします。

## 使う場面

- PostgreSQL、Redis、Celery、Flower等を含む既存Composeの公開ポートを整理するとき
- 複数プロジェクトのホストポート競合を減らしたいとき
- リバースプロキシ導入前に、内部サービスをCompose networkへ閉じたいとき
- 意図しないLAN公開がないか確認するとき

このPlaybook自体はCaddy / Traefik、HTTPS、LAN DNS、共有Docker networkの導入を要求しません。

## 目標

ホストまたはLANから直接必要な入口だけをpublishし、PostgreSQL、Redis、Celery等はCompose標準のdefault network上で通信させます。

```text
Host / LAN
    |
    | 必要なWebポートのみ
    v
Django Web
    |
    | Docker Compose default network
    +--> PostgreSQL :5432
    +--> Redis :6379
    +--> Celery

PostgreSQL / Redis / Celery
    X  Host / LANへ不要なpublishをしない
```

## 1. 変更前の調査

まず、リポジトリ内でComposeと接続先の定義を確認します。ファイル名を決め打ちせず、存在するものだけを対象にします。

- `compose.yml`, `docker-compose.yml`, `docker-compose.*.yml`
- `.env` と環境変数定義
- DjangoのDATABASE設定
- Redis接続設定
- Celery broker / result backend設定
- Vite等の開発サーバ設定
- nginx等が存在する場合の設定
- README、Makefile、justfile、開発用スクリプト等に記載されたポート番号

すべての `ports:` を洗い出し、各サービスについて次を確認します。

1. なぜ現在publishされているか。
2. ホストOSから直接アクセスする必要があるか。
3. LAN内の別端末からアクセスする必要があるか。
4. 同一Compose内のコンテナ間通信だけでよいか。

必要性が不明なポートは推測だけで削除せず、コード、設定、ドキュメント、スクリプトから利用状況を確認します。

## 2. 公開境界を決める

### Webアプリ

Caddy等をまだ導入しない段階では、ブラウザやLANからアクセスするWebサービスのホストポートを維持します。

```yaml
services:
  web:
    ports:
      - "8001:8000"
```

複数プロジェクトではホスト側の `8001` を変えて競合を避けて構いません。コンテナ内部の `8000` は理由がなければ変更しません。

### PostgreSQL / Redis

DjangoやCelery等、Compose内からしか利用しない場合は `ports:` を削除します。

```yaml
services:
  db:
    image: postgres:16

  redis:
    image: redis:7
```

接続先もホスト公開ポートではなくサービス名へ合わせます。

```text
PostgreSQL: db:5432
Redis:      redis://redis:6379/0
```

`localhost` はそのコンテナ自身を指すため、別コンテナへの接続先として使いません。

### Celery worker / beat

通常は外部から接続を受けるサービスではありません。`ports:` が存在する場合は用途を確認し、コンテナ間通信だけなら削除します。

### Flower・DB管理UI等

ホストから利用するがLANへ公開する必要がない管理UIは、必要性を確認してlocalhostへ限定します。

```yaml
services:
  flower:
    ports:
      - "127.0.0.1:5555:5555"
```

現在利用していない場合は、単にlocalhostへ変更して残すのではなく、公開自体を削除することを優先します。

## 3. 不要な追加設定を避ける

- 同一Compose内のサービス間通信だけなら通常 `expose:` は不要です。
- default networkで足りるなら独自 `networks:` を追加しません。
- この整理と同時にリバースプロキシ、HTTPS、DNS、共有network等へ範囲を広げません。

変更目的を「必要な入口だけをpublishする」に限定すると、既存環境への影響を小さくできます。

## 4. 設定を更新する

`ports:` を削除したサービスについて、アプリケーションがホスト経由で接続していないことを確認します。

代表例:

```text
DB_HOST=db
DB_PORT=5432
CELERY_BROKER_URL=redis://redis:6379/0
CELERY_RESULT_BACKEND=redis://redis:6379/1
```

実際のサービス名やRedis DB番号はプロジェクトの既存設定を維持します。ポート整理を理由に別の設定変更まで行いません。

## 5. 検証

変更後は少なくとも次を確認します。

- `docker compose config` が成功し、最終的な `ports` が意図どおりである。
- `docker compose up` で必要なサービスが正常に起動する。
- DjangoからPostgreSQLへ接続できる。
- Django / CeleryからRedisへ接続できる。
- Celery workerが正常に起動し、必要なら代表的なtaskを実行できる。
- Webアプリへ従来のホストポートからアクセスできる。
- 必要なLAN内アクセスが維持されている。
- PostgreSQLやRedis等、不要なサービスがホスト/LANへpublishされていない。
- 既存テストが通る。

必要に応じて `docker compose ps` やホストOSのlisten状態も確認します。Composeファイル上の意図だけで完了と判断せず、最終構成を確認します。

## 6. 変更結果を記録する

レビュー時には、公開境界が変わったサービスをBefore / Afterで整理します。

| Service | Before | After | 理由 |
|---|---|---|---|
| Django | `0.0.0.0:8001 -> 8000` | 維持 | LANからアクセスするため |
| PostgreSQL | `0.0.0.0:5432 -> 5432` | 非公開 | DjangoからDocker network経由で接続 |
| Redis | `0.0.0.0:6379 -> 6379` | 非公開 | コンテナ間通信のみ |
| Flower | `0.0.0.0:5555 -> 5555` | 必要なら `127.0.0.1:5555 -> 5555` | 管理用途。未使用なら公開削除 |

## よくある失敗

| 症状 | 確認すること |
|---|---|
| DBの `ports:` を消したらDjangoが接続できない | `DB_HOST=localhost` やホスト側ポートを参照していないか。Composeサービス名へ変更する。 |
| Redisを非公開にしたらCeleryが起動しない | broker/backend URLが `localhost` やホストIPを参照していないか確認する。 |
| LANからWebへ接続できなくなった | Webサービスまでlocalhost bindにしていないか。LANアクセスが必要な入口はホストへpublishする。 |
| Flowerをlocalhost bindしたら別端末から見えない | 意図どおり。LAN利用が本当に必要なら要件として再評価する。 |
| `expose:` を大量に追加した | Compose内通信に必要か再確認する。通常はサービス名と内部ポートだけで通信できる。 |
| 独自networkが増えた | 分離要件がなければdefault networkへ戻し、今回の変更範囲を小さくする。 |

## 完了条件

変更後の構成について「外部から必要な入口」と「Compose内部だけのサービス」を説明でき、`docker compose config` と実動作の双方でその境界を確認できれば完了です。
