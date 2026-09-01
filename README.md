# AI Development Playbook

複数のWeb ApplicationをAI支援で開発するときに、毎回同じ実装・移行・検証・障害対応を考え直さないためのPlaybookです。

## 位置づけ

通常のApplicationは [ai-dev-platform](https://github.com/hamirilo/ai-dev-platform) の `ai/ONBOARDING.md` を統合入口として利用し、タスクに必要なPlaybookだけを参照します。

このリポジトリは **「どう実施するか」** の正本です。StandardやRecommendationの判断を再定義しません。

| 資産 | 答える問い | 正本 |
|---|---|---|
| Standard | 何を守るか | [ai-dev-standards](https://github.com/hamirilo/ai-dev-standards) |
| Recommendation | 普段は何を選ぶか | `ai-dev-platform/recommendations` |
| **Playbook** | どう実装・移行・検証・復旧するか | このリポジトリ |
| UI Platform | UIをどう設計し、何を再利用するか | [ui-platform](https://github.com/hamirilo/ui-platform) |
| Project Context / ADR | このApplication固有ではどうするか | 各Application |

StandardとPlaybookが矛盾する場合はStandardを優先します。Project固有の明示的な決定がある場合は、その適用範囲ではProject側の決定を優先します。

## 入口

最初から全ファイルを読まず、タスクに必要なものだけを参照してください。

| タスク | 参照するPlaybook |
|---|---|
| 機能追加・bug修正・refactoring | [AI支援開発](playbooks/ai-assisted-development.md) |
| Djangoの実装・変更 | [Django実装](playbooks/django-implementation.md) |
| test・review・UI確認 | [テストとレビュー](playbooks/testing-and-review.md) |
| 品質・性能・accessibility確認 | [品質確認](playbooks/quality-checks.md) |
| 認証・認可・error・外部通信 | [安全な実装とエラー処理](playbooks/security-and-error-handling.md) |
| APIを追加・変更 | [API実装](playbooks/api-implementation.md) |
| container化・image配布・既存環境の移行 | [コンテナ配布](playbooks/container-delivery.md) |
| Docker Composeのhost port公開を整理 | [Docker Compose ポート公開](playbooks/docker-compose-port-exposure.md) |

UIのComponent、Pattern、Template、Storybook Catalog、design referenceはUI Platformを参照します。ReactとDjangoの接続、security、error handling、実browser検証等、UIに関係していても一般化できる **実施方法** はPlaybookで扱います。

## このリポジトリに置かないもの

- 全Applicationが守る技術選定・責務境界・制約（Standards）
- 現時点のlibrary / tool既定（Recommendations）
- UI Component、Pattern、Template、design-system、Storybook（UI Platform）
- 新規Application用の巨大なboilerplate
- 再利用可能な実行package
- 社員・組織・認証基盤等の業務domain固有情報
- AI agent定義、model設定、workflow実行基盤

code例と失敗例は独立した `examples/` を増やさず、関係するPlaybook内へ必要な範囲で含めます。

## Playbook追加の基準

1. 複数projectまたは複数回の実装で、同じ手順や失敗が繰り返されたことを確認する。
2. project固有のmodel名、URL、社内設定を除いても役に立つ形にする。
3. 「使う場面」「前提」「手順」「検証」「失敗時の確認」を一つの文書へまとめる。
4. Standardの判断を再記述せず、必要な場合はStandardを参照する。
5. 実装codeそのものが安定して複数箇所で再利用される場合は、Playbookではなく実装資産への昇格を検討する。UI実装はUI PlatformのComponent候補とする。

### 配置先の判断

| 内容 | 配置先 |
|---|---|
| 全体で守る判断・制約 | `ai-dev-standards` |
| 現時点のlibrary既定 | `ai-dev-platform/recommendations` |
| 実装・移行・検証・失敗例 | `ai-dev-playbook` |
| UI design候補・Pattern・画面構成例 | `ui-platform` |
| 再利用可能な汎用UI実装 | `ui-platform/components` |
| UI以外のproject固有雛形 | 各project。繰り返しが確認されてから別資産化を検討 |
| UI以外の実行可能な共通code | 利用実績を確認した後、別package |
| 業務domain UI | そのdomainを所有するproject |

## 公開範囲

公開repositoryとして扱える一般化された内容だけを置きます。社内URL、認証情報、個人情報、組織固有model・運用、非公開の障害情報は追加しません。

## リリース

正式なversionはSemVer形式のGitHub Release（`v<major>.<minor>.<patch>`）で示します。tagだけを単独で作らず、利用者が変更内容を確認できるReleaseを作成します。

現時点では自動release workflowを持たないため、releaseが必要な変更をまとめ、SemVerに従ってGitHub Releaseを手動で作成します。各mergeのたびにreleaseする必要はありません。

Playbookをreleaseしただけでは、Platform利用者の組合せは変わりません。Standardsとの整合を確認した後、`ai-dev-platform`のsubmodule pointerを更新してPlatformをreleaseした時点で、推奨する組合せが確定します。

SemVerの判断、共有資産間のrelease順序、Platform releaseの扱いは[AI Development Platformのリリース方針](https://github.com/hamirilo/ai-dev-platform/blob/main/docs/release.md)を参照してください。
