# デプロイ設定確認のCI実装例

[デプロイ設定の確認](deployment-configuration.md) PlaybookをApplicationへ適用するときの、最小限のCI実装例です。

## 目的

CIでproductionのsecretを取得するのではなく、次を自動化します。

- 設定定義の妥当性を検証する
- `.env.example`などの変更を検知する
- 設定契約が変わったことをPRで明示する
- production secretをCIログやartifactへ持ち込まない

## `check-env.sh`

例えば、`.env.example`の定義自体を検証するスクリプトを用意します。

```bash
#!/usr/bin/env bash
set -euo pipefail

example_file=".env.example"

if [[ ! -f "$example_file" ]]; then
  echo "ERROR: $example_file not found" >&2
  exit 1
fi

keys=$(sed -n 's/^[[:space:]]*\([A-Za-z_][A-Za-z0-9_]*\)[[:space:]]*=.*/\1/p' "$example_file")
duplicates=$(printf '%s\n' "$keys" | sort | uniq -d)

if [[ -n "$duplicates" ]]; then
  echo "ERROR: duplicate environment variables:" >&2
  printf '%s\n' "$duplicates" >&2
  exit 1
fi

echo "Environment definition check passed."
```

これは最小例です。Application固有のrequired variables、Django settings、Compose設定などの検証はプロジェクト側で追加します。

## GitHub Actions

PRでスクリプトを実行します。

```yaml
name: Configuration Check

on:
  pull_request:

jobs:
  configuration:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check environment definition
        run: ./scripts/check-env.sh
```

既存のCI pipelineがある場合は、独立workflowを増やさず既存のquality/CI jobへ組み込んでも構いません。重要なのはPRの必須gateとして実行することです。

## `.env.example`変更の検知

設定契約が変更されたPRでは、レビューでproduction configurationへの影響を確認できるようにします。

```bash
changed_files="$(git diff --name-only "$BASE_SHA" "$HEAD_SHA")"

if printf '%s\n' "$changed_files" | grep -Eq '(^|/)(\.env\.example|compose.*\.ya?ml|Dockerfile)$'; then
  echo "Configuration impact: YES"
  echo "Review production/staging configuration before deployment."
else
  echo "Configuration impact: NO"
fi
```

実際のworkflowでは、`BASE_SHA`と`HEAD_SHA`をCI環境が提供するPRのbase/head SHAに合わせます。

出力例:

```text
Configuration impact: YES
Review production/staging configuration before deployment.
```

より強くする場合は、新規・削除されたenvironment variableのキーを抽出し、PR summaryへ表示します。ただしsecret値は絶対に表示しません。

## productionとのdrift検査

productionの実値をCIへコピーするのではなく、デプロイ先で検査します。

例えばrequired keysをApplication側で定義します。

```text
SECRET_KEY
DATABASE_URL
OIDC_CLIENT_ID
OIDC_CLIENT_SECRET
```

デプロイ先では、値を表示せず存在だけを検査します。

```bash
#!/usr/bin/env bash
set -euo pipefail

while IFS= read -r key; do
  [[ -z "$key" || "$key" == \#* ]] && continue

  if [[ -z "${!key:-}" ]]; then
    echo "ERROR: missing required environment variable: $key" >&2
    exit 1
  fi
done < scripts/required-env.txt

echo "Production configuration check passed."
```

`.env.example`と`required-env.txt`を両方手で管理すると二重管理になります。可能ならrequired keysの正本を一つにし、Applicationの設定定義から生成するか、どちらを正本とするかを明示してください。

## やってはいけない構成

CIでproduction `.env`を取得してdiffする方法は避けます。

```text
CI
 └── production .env
       ↓
     diff
       ↓
  CI log / artifact
```

production secretをCIへ持ち込むと、権限範囲と漏えい経路が増えます。

推奨する分離は次のとおりです。

```text
Git
 ├── .env.example
 └── configuration checks
       ↓
      CI
       └── 定義と変更を検証

Production
 └── .env / secret store
       ↓
  deploy-check
       └── 実環境の不足を検証
```

## 完了条件

- PRで設定契約の変更が検知される
- 設定定義のCI checkが必須gateになっている
- production secretをCIへ持ち込んでいない
- デプロイ時に不足したrequired variablesを検出できる
- エラー時にsecret値をログへ出さない
