# リポジトリのリリース

共有ドキュメント資産をSemVerでversioningし、Release Pleaseでversion更新、CHANGELOG、tag、GitHub Releaseを管理するための手順です。

## 使う場面

- `ai-dev-standards`、`ai-dev-playbook`、`ai-dev-platform`へRelease Pleaseを導入する
- release PRを確認して正式versionを公開する
- Release PleaseのWorkflowが失敗した原因を確認する
- Standards / PlaybookのreleaseをPlatformへ取り込む
- forkでupstreamのrelease tagを同期する

`ui-platform`はrepository releaseとnpm package releaseが分かれているため、この手順をそのまま適用しません。

## 前提

- latestのGitHub Releaseとtagが存在する
- rootの`version.txt`がlatest versionと一致している
- rootに`CHANGELOG.md`がある
- PRは原則としてsquash mergeし、PR titleを`main`へ入るcommit messageとして扱う

既存repositoryへ導入する場合、`version.txt`を新しいversionへ先に進めません。現在のlatest Releaseが`v1.0.0`なら`1.0.0`から開始します。

## GitHubの設定

`Settings → Actions → General → Workflow permissions`で、次を有効にします。

- Actionsがwrite権限を利用できること
- `Allow GitHub Actions to create and approve pull requests`

Workflow側で`pull-requests: write`を指定していても、後者が無効ならrelease PRの作成は拒否されます。

## 最小Workflow

`.github/workflows/release-please.yml`を作成します。

```yaml
name: Release Please

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: write
  pull-requests: write
  issues: write

jobs:
  release:
    # forkでは実行しない。forkはupstreamのtagを同期するだけで独自のversionを切らない（docs/release.md）
    if: github.event.repository.fork == false
    runs-on: ubuntu-latest
    steps:
      - uses: googleapis/release-please-action@v4
        with:
          release-type: simple
```

`if`により、forkではWorkflowが起動しても`release` jobがskipされ、fork独自のversionやrelease PRが作られません。

自動化を揃えることだけを目的に共通Workflow repositoryや追加設定を増やしません。複数packageや独自tagが必要なrepositoryは個別に設計します。

## 変更種別

| `main`へ入るcommit | version | 例 |
|---|---|---|
| `fix:` | PATCH | 誤記、既存手順の訂正、互換性を保つ明確化 |
| `feat:` | MINOR | 新しいStandard、Recommendation、Playbookの追加 |
| `!`または`BREAKING CHANGE` | MAJOR | 既存利用者の変更が必要な破壊的変更 |
| `docs:` / `chore:` | 原則releaseなし | release対象外の内部整理 |

文書repositoryでも、利用者に影響する追加は`feat:`、訂正は`fix:`として扱います。

## Release手順

1. release候補のPRをreviewし、関連repositoryとの矛盾がないことを確認する。
2. PRを`main`へmergeする。
3. Release Please Workflowが成功したことを確認する。
4. Release Pleaseが作成または更新したrelease PRでversionと`CHANGELOG.md`を確認する。
5. 破壊的変更、移行作業、参照先versionがRelease notesへ反映されていることを確認する。
6. release PRをmergeする。
7. 次のWorkflowでtagとGitHub Releaseが作成されたことを確認する。

すべてのmergeでrelease PRをmergeする必要はありません。関連変更をrelease PRへ蓄積し、まとまった時点で公開できます。

## Platformへ取り込む場合

Standards / PlaybookをPlatformへ採用する順序は次のとおりです。

1. candidate commit間の整合性をrelease前に確認する。
2. Standards / Playbookを必要な順序でreleaseする。
3. Platformのsubmoduleを各release tagが指すcommitへ更新する。
4. `git diff --submodule=log`で変更元commitを確認する。
5. submodule pointer更新をPRにし、採用versionと移行要否を記載する。
6. Platformをreleaseする。

ApplicationはStandards / Playbookを個別にlatestへ更新せず、採用したPlatform tagまたはcommitが指す組合せを利用します。

## forkで利用する場合

forkは配布・検証・upstreamとの往復に使い、独自のreleaseを行いません。`release` jobはfork判定でskipされるため、forkではrelease PR、tag、GitHub Releaseのいずれも作成されません。`version.txt`と`CHANGELOG.md`はupstreamが所有し、Release notesの正本もupstreamのGitHub Releaseです。forkにはtagがあれば足ります。

forkで必要なのは、採用するupstreamのrelease tagの同期だけです。`git push --tags`はforkにしか無いtagも公開するため使わず、採用するtagを1つ指定してpushします。upstreamを`upstream` remoteとして登録している場合の例です。

```bash
RELEASE_TAG="v1.2.0"
git fetch upstream tag "$RELEASE_TAG"
git push origin "$RELEASE_TAG"
```

forkがupstreamより先行する変更を直ちに利用する場合は、tagではなくfork上のcommitで固定し、upstreamがreleaseした後にtag固定へ戻します。

## 失敗時の確認

### Release PRを作成できない

次のerrorはrepository設定が原因です。

```text
GitHub Actions is not permitted to create or approve pull requests.
```

`Allow GitHub Actions to create and approve pull requests`を有効にし、失敗したWorkflowの`Re-run failed jobs`を実行します。Release Pleaseが作成済みのbranchは再実行時に再利用されます。

### GitHub APIのerrorで失敗する

次のerrorはGitHub側の一時的な障害であることが多く、repository設定やcommit messageは原因ではありません。

```text
Something went wrong while executing your query ... Please include <id> when reporting this issue
```

時間を置いてから、失敗したWorkflowの`Re-run all jobs`を実行します。2026-09-09に`ai-dev-platform`で2回連続して発生し、約10時間後の再実行で成功しました。

再実行しても再現する場合は、最終手段として手動でreleaseします。着手前に、Release Pleaseが作成したrelease PRとbranch（`release-please--branches--main`）が残っていないかを確認します。残っている場合は手動releaseと競合するため、release PRをcloseしbranchを削除してから進めます。

1. `version.txt`と`CHANGELOG.md`を次のversionへ更新するPRを作り、`chore(main): release <version>`としてmergeする。
2. merge後のcommitへ`v<version>`のtagを作成しpushする。
3. 同じtagでGitHub Releaseを作成し、Release notesへ`CHANGELOG.md`の該当分を記載する。
4. 次回のRelease Pleaseが手動作成したlatest tagを基準に、期待どおりの次versionのrelease PRを作ることを確認する。古いstateが再利用されていないことを、release PRのversionと差分で確認する。

### versionが意図と異なる

- latest tagとGitHub Releaseが一致しているか
- `version.txt`がlatest Releaseと一致しているか
- squash後に`main`へ入ったcommit messageが`fix:` / `feat:` / breaking changeのどれとして判定されたか
- 未releaseのcommitが前回tag以降に残っていないか

### CHANGELOGが不自然

`CHANGELOG.md`は次の最小構造から開始します。release見出しを手動で先回りして作りません。

```markdown
# Changelog

このファイルはRelease Pleaseにより更新します。
```

## 検証

- Workflowが成功している
- release PRのversionがSemVer判断と一致する
- `CHANGELOG.md`に対象変更が含まれる
- release PR merge後にtagとGitHub Releaseが作成される
- forkでWorkflowを実行すると`release` jobがskipされる
- Platformではsubmodule pointerがrelease tagのcommitと一致する
