# CIパイプライン

Application repositoryのCIを、PR・マージの必須gateとして組むための手順です。

[Governance Standard §4](https://github.com/hamirilo/ai-dev-standards/blob/main/standards/governance/README.md)は「projectで定義されている機械的な検証をPR・マージ・リリースの必須gateとする」と定め、具体的なtoolとcommandは各projectへ委ねています。このPlaybookはその**Workflowの組み方**だけを扱います。何を必須gateとするかはStandard、library・tool既定はRecommendationが正本です。

releaseのWorkflowは[リポジトリのリリース](repository-release.md)を参照します。

## 使う場面

- 既存のApplication repositoryへCIを新規に導入する
- 型チェック・Linter・test・buildを必須gateとして固定する
- CIが緑なのに壊れた成果物が出る原因を確認する
- CIの実行時間が長く、job分割を検討する

## 前提

- 依存がlock fileで固定されている
- projectのcommandがlocalで実行できる（`just test`等）
- 実行環境と同じDB engine、同じCPU architectureを指定できる

## 起動条件

GitHub ActionsにはCIを走らせない方法が2つあり、branch protection / rulesetのrequired checkとの相性が正反対です。

| | Workflow levelのskip（`on.paths` / `on.paths-ignore`） | job levelの`if:` |
|---|---|---|
| 起きること | Workflow自体が起動しない | Workflowは起動し、条件に合わないjobだけがskipされる |
| checkの報告 | **1つも報告されない** | `skipped`として報告される |
| required checkにした場合 | 対象外のPRでcheckが永久にpendingになり、mergeできない | skipは成功として扱われ、mergeできる |

required checkを使わない段階では`on.paths`で足ります。**required checkを導入する時点で、`on.paths`は変更検出jobと`if:`へ置き換えます**（[規模が大きい場合](#規模が大きい場合)）。起動条件を絞ったまま必須化すると、対象外のPRが永久にmergeできなくなります。

### どのcheckをrequiredにするか

required checkはcheckの**名前**で指定し、その名前のcheckが報告されないPRはpendingのままになります。

- skipされうるjobも含めて列挙します。skipは成功として扱われるため、変更検出で落としたjobがPRを止めることはありません。
- matrixを使うと、matrixの値がcheck名に含まれます。shardを増減させるたびにruleset側の登録を直すことになります。

job構成を変えるたびにruleset側を触りたくない場合は、**集約jobを1つだけrequiredにします**。

```yaml
  ci:
    # 先行jobがskip・失敗のどちらでも、このjobは必ず結果を報告する
    if: always()
    # 変更検出jobを含め、Workflowのすべてのjobを列挙する（後述）
    needs: [changes, backend, frontend]
    runs-on: ubuntu-latest
    steps:
      - name: 先行jobの結果を判定する
        env:
          RESULTS: ${{ join(needs.*.result, ',') }}
        run: |
          echo "$RESULTS"
          case "$RESULTS" in
            *failure*|*cancelled*) exit 1 ;;
          esac
```

`if: always()`が無いと、先行jobがskipされたときに集約job自体もskipされ、判定が行われません。この形ならrulesetへ登録するcheckは`ci`の1つで済みます。

**集約jobの`needs`には、そのWorkflowのすべてのjobを列挙します。** 中間のjobを省くと、その失敗が後続jobの`skipped`として伝播し、集約jobからは見えなくなります。例えば上の`needs`から`changes`を落とすと、変更検出jobが失敗したときに`backend` / `frontend`はskipされ、`RESULTS`が`skipped,skipped`になって**required checkが緑のままmergeできてしまいます**。

### 起動対象の選び方

`on.paths`を使う場合、Application directoryだけを列挙しません。**CIの結果を変え得るfileを種類で洗い出します。**

- Application code
- 依存のmanifestとlock file
- 配布物の作られ方を決めるfile（container定義、build設定）
- runtimeの版を固定するfile
- CIが実行するcommandの定義（task runner）
- そのWorkflow自身

pathはrepository構成によって変わります。次は列挙の粒度の例であり、そのまま写す対象ではありません。

```yaml
on:
  pull_request:
    paths:
      - "backend/**"                        # Application code
      - "pyproject.toml"                    # 依存のmanifest（rootに置く構成の場合）
      - "uv.lock"                           # 依存のlock file
      - "Dockerfile"                        # 配布物の作られ方
      - ".python-version"                   # runtimeの版
      - "justfile"                          # CIが実行するcommandの定義
      - ".github/workflows/backend-ci.yml"  # このWorkflow自身
```

判断に迷うfileは含めます。不要なrunが増える害は、検査されないままmergeされる害より小さいためです。

`paths`はpathをsourceへ固定します。directoryを移動・廃止するときは、同じPRでここも直します。**廃止したpathを指したままのfilterはerrorにならず、単に一度も起動しません。**

一覧は`push`と`pull_request`の両方へ同じものを書きます。YAMLのanchorは、置き場になる未知のtop level keyをActionsが拒否するため利用できません。

## 最小構成

### Server side

```yaml
name: Backend CI

# 依存のmanifestとlock fileはbackend/の下にある構成を前提にしている。
# rootに置く構成なら、それらも個別に列挙する（「起動対象の選び方」）。
on:
  push:
    paths:
      - "backend/**"
      - "Dockerfile"
      - ".python-version"
      - "justfile"
      - ".github/workflows/backend-ci.yml"
  pull_request:
    paths:
      - "backend/**"
      - "Dockerfile"
      - ".python-version"
      - "justfile"
      - ".github/workflows/backend-ci.yml"

defaults:
  run:
    working-directory: backend

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: app
          POSTGRES_USER: app
          POSTGRES_PASSWORD: app
        ports: ["5432:5432"]
        # 既定の10秒間隔では、最初のhealth checkを待つだけでcontainerの
        # 初期化に20秒以上かかる。短い間隔で回数を稼ぐ。
        options: >-
          --health-cmd pg_isready
          --health-interval 2s
          --health-timeout 3s
          --health-retries 20
    env:
      DATABASE_URL: postgresql://app:app@localhost:5432/app
      SECRET_KEY: ci-only-not-a-real-secret
      DEBUG: "False"
    steps:
      - uses: actions/checkout@v7

      # uv本体の版はbackend/pyproject.tomlの[tool.uv] required-versionが正本
      # （actionのpinはactionの版であり、uvの版ではない）
      - name: Install uv
        uses: astral-sh/setup-uv@v10.0.1
        with:
          version-file: backend/pyproject.toml
          enable-cache: true
          cache-dependency-glob: backend/uv.lock
      - run: uv sync --frozen

      - run: uv run ruff check .
      - run: uv run ruff format --check .

      # modelを変えてmigrationを作り忘れると、本番のmigrateまで誰も気づかない。
      # 開発機には既に適用済みのDBがあるため再現しない。
      - run: uv run python manage.py makemigrations --check --dry-run
      - run: uv run python manage.py migrate

      - run: uv run pytest --cov=apps --cov-fail-under=85

      # 本番設定での構成ミスを拾う。ERROR相当のときだけ失敗する。
      - run: uv run python manage.py check --deploy
        env:
          DEBUG: "False"
          ALLOWED_HOSTS: example.com

      # 型チェック・Linter・testの成功だけで「配布物が動く」とは判断しない。
      # pushはせず、imageが組み上がることだけを見る。
      - run: docker build -t app:ci .
```

DB engineは実行環境と揃えます。PostgreSQL固有のmigrationが1つでもあると、SQLiteではmigrate自体が失敗し、CIが丸ごと意味を失います。

`--cov-fail-under`は**大きく下がったときに気づくための床**で、上げることを目標にしません。数値のためのtestを増やさず、床に当たったら「どの振る舞いのtestが消えたか」を先に見ます。導入時は実測値から数point下げた値を置きます。

### Client side

```yaml
name: UI CI

on:
  push:
    paths: ["frontend/**", ".bun-version", ".github/workflows/ui-ci.yml"]
  pull_request:
    paths: ["frontend/**", ".bun-version", ".github/workflows/ui-ci.yml"]

permissions:
  contents: read
  packages: read

defaults:
  run:
    working-directory: frontend

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      # 版はrepositoryが持ち、CIはそれを参照する（後述「版を固定する」）
      - uses: oven-sh/setup-bun@v2
        with:
          bun-version-file: .bun-version

      # lock fileのとおりに入れる。lockとmanifestがずれていたら失敗させる。
      - run: bun install --frozen-lockfile
        env:
          PACKAGES_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - run: bun run typecheck
      - run: bun run test
      - run: bun run build
```

private registryのtokenを渡すenvへ`GITHUB_TOKEN`という名前を使いません。GitHub CLIの予約名であり、`read:packages`しか持たない値を入れると、同じ環境で動くCLIの他の操作が権限不足で失敗します。値そのものはActionsが発行する`secrets.GITHUB_TOKEN`で足ります。

test runnerは`bun test`ではなく`bun run test`で起動します。前者はbun内蔵のrunnerが動き、manifestの`test` scriptを無視します。

### 版を固定する

このPlaybookが求めるのは、bit単位で同一の実行環境ではなく、**意図しない更新でCIの結果が変わらないこと**です。同じcommitを再実行して結果が変わる原因のうち、自分たちで選べるものを固定します。

| 対象 | 固定の仕方 |
|---|---|
| 依存package | lock fileと、固定installのoption（`--frozen-lockfile`、`uv sync --frozen`） |
| language runtime、package manager本体 | repositoryのfileを版の正本にし、CIはそれを参照する |
| action | major versionにpinする。挙動が版に依存するものはexact versionでpinする |
| container image、runner image | tagで指定する。securityの更新は受け取る |

**actionをpinしても、そのactionが導入するtoolの版は固定されません。** `astral-sh/setup-uv@v10.0.1`はactionの版であり、入るuvの版ではありません。同じくbunもnodeも、setup系のactionは既定で最新を入れます。

toolの版はWorkflowへ直接書かず、repository側のfileを正本にしてCIから参照します。Workflowへ書くと、開発者の手元とCIで別々の版が動き、どちらが正なのか決まりません。

```yaml
- uses: oven-sh/setup-bun@v2
  with:
    bun-version-file: .bun-version        # package.json / .tool-versions も指定できる

- uses: astral-sh/setup-uv@v10.0.1
  with:
    version-file: backend/pyproject.toml  # [tool.uv] required-version を読む
```

`defaults.run.working-directory`はactionのinputには効きません。manifestがrepository rootに無い構成では、pathを明示しないとrootを探して見つけられません。

正本にできるfileがrepositoryへ無い場合は、暫定としてWorkflowへexact versionを書きます（`version: "0.9.2"`）。この場合、手元の版とずれてもCIは気づきません。project側へ`required-version`を置くところまで進めます。

container imageとrunner imageはtagのままにします。`postgres:16-alpine`や`runs-on: ubuntu-latest`をdigestで固定すると、securityの更新を明示的に取り込むまで受け取れなくなります。CIの実行環境そのものを証拠として残す必要がある場合（監査・規制対応）にだけdigest固定を検討します。

## 通す検証

PRでは、少なくとも次を通します。順序は原因の読みやすい順です。

1. lock fileどおりの固定install
2. Formatter / Linterの検査
3. 型チェック
4. migrationの欠落検査
5. unit / integration test
6. 本番設定の構成検査
7. 実行対象platformでのimage build
8. imageの中に期待する成果物があることの確認

7と8の詳細は[コンテナ配布](container-delivery.md)を参照します。image buildが成功しただけでは、静的成果物が1つも入っていないimageを配れます。中身の存在をimage内で確認します。

```yaml
- name: 成果物がimageに入っていることの確認
  run: |
    docker run --rm app:ci sh -c '
      set -e
      test -s static/css/output.css
      ls staticfiles/js/assets/*.js >/dev/null
    '
```

CIのrunnerから到達できない配信先へpushしません。runnerがnetwork的に届かない環境へ配る場合は、artifactとして置き、受け手が取りに来る向きにします。

## 規模が大きい場合

最小構成で実行時間が問題になってから、次を検討します。先回りして入れません。

### 変更検出でjobを分ける

server sideとclient sideのどちらかしか触っていない変更で、無関係なjobを待たされないようにします。`paths`と違い、必須checkにしてもskipされたjobは成功として扱われます。

```yaml
jobs:
  changes:
    runs-on: ubuntu-latest
    # pull_requestでは変更file一覧をGitHub APIから取るため、既定のread権限だけでは
    # "Resource not accessible by integration" で失敗する。このjobにだけ付与する。
    permissions:
      contents: read
      pull-requests: read
    outputs:
      backend: ${{ steps.filter.outputs.backend }}
      frontend: ${{ steps.filter.outputs.frontend }}
    steps:
      - uses: actions/checkout@v7
      - uses: dorny/paths-filter@v3
        id: filter
        with:
          filters: |
            # 起動条件の paths と同じ範囲を列挙する。片方だけ直すと、
            # Workflowは起動したのに対象jobがskipされる状態になる。
            backend:
              - 'backend/**'
              - 'Dockerfile'
              - '.python-version'
              - 'justfile'
            frontend:
              - 'frontend/**'
              - '.bun-version'

  backend:
    needs: changes
    if: needs.changes.outputs.backend == 'true'
```

### testを分割する

1台のrunner内のprocess並列は、標準runnerのCPU数で頭打ちになります。localの高core環境で測った実行時間は再現しません。頭打ちを確認してから、runnerを複数台へ分けます。

```yaml
strategy:
  fail-fast: false
  matrix:
    shard:
      - { name: shard1, paths: "apps/a apps/b" }
      - { name: shard2, paths: "apps/c apps/d" }
```

分割はtest件数をおおよそ均等にした固定listで足ります。自動分割toolはduration fileの生成・commit・定期更新を増やすため、偏りが実害になるまで導入しません。**setupがshardの数だけ重複するので、合計実行時間は増え、待ち時間だけが減ります。**

### 同じbranchの古い実行を打ち切る

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

## よくある失敗

| 症状 | 確認すること |
|---|---|
| 変更したのにCIが走らない | `paths`が現在の構成と一致しているか。廃止したpathを指したfilterはerrorにならず起動しないだけ |
| 設定fileだけ変えたPRが検査されずに通る | `paths`がApplication directoryだけになっていないか。lock file、container定義、task runnerの定義を含める |
| required checkが永久にpendingでmergeできない | `on.paths` / `on.paths-ignore`で起動を絞っていないか。required checkにするなら変更検出jobと`if:`へ変える |
| job構成を変えるたびにmergeできなくなる | required checkをjob名で個別に登録していないか。集約jobを1つだけrequiredにする |
| 途中のjobが失敗したのにrequired checkが緑 | 集約jobの`needs`にすべてのjobを列挙しているか。省いたjobの失敗は後続の`skipped`として伝播し、成功扱いになる |
| 変更検出jobが "Resource not accessible by integration" | そのjobへ`pull-requests: read`を付けているか |
| sourceを変えていないのにある日CIが落ちる | runtime・toolの版が`latest`追随になっていないか。actionのpinはactionの版であり、そのactionが入れるtoolの版ではない |
| 手元とCIでtoolの挙動が違う | 版の正本がrepository側のfileにあり、CIがそれを参照しているか。Workflowへ直接書くと二重管理になる |
| localでは通るがCIだけ落ちる | DB engine、timezone、実行時刻に依存していないか。日付境界の不具合は特定時間帯のrunだけ再現する |
| CIは緑なのに配布物が壊れている | image buildと、image内の成果物の存在確認がCIに含まれているか |
| lock fileを更新し忘れたPRが通る | installへ`--frozen-lockfile`相当を付けているか |
| test scriptが実行されていない | package managerの内蔵runnerがmanifestのscriptを横取りしていないか |
| tokenがimageに残った | build時のsecretを`--build-arg`ではなくbuild secretで渡しているか |
| container初期化だけで時間がかかる | serviceのhealth check間隔が既定のままになっていないか |

## 検証

- 対象pathを変更したPRでWorkflowが起動し、lock fileやtask runnerの定義だけを変えたPRでも起動する
- documentだけのPRでも、必須checkにしている場合はcheckが完了する
- lock fileとmanifestを意図的にずらすとinstallが失敗する
- modelを変更してmigrationを作らないと`makemigrations --check`が失敗する
- required checkにしたcheckが、対象外の変更だけのPRでも報告される（pendingで止まらない）
- 変更検出jobをわざと失敗させると、集約jobも失敗する（skipとして握り潰されない）
- runtime・toolの版をrepositoryのfileで変更すると、CIで動く版も変わる（CIのlogに出る版で確認する）
- image内の成果物確認を意図的に壊すとjobが失敗する

## 関連

- [リポジトリのリリース](repository-release.md)
- [Quality Recommendations](https://github.com/hamirilo/ai-dev-platform/blob/main/recommendations/quality.md) — 時刻の誤用等、CIで機械的に検出する対象
- [コンテナ配布](container-delivery.md)
- [テスト・レビュー・UI確認](testing-and-review.md)
- [品質確認](quality-checks.md)
