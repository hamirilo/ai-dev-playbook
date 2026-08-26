# AI Development Playbook

複数のWebアプリケーションをAI支援で開発するときに、毎回同じ調査・設計・検証を繰り返さないための実装プレイブックです。

## 役割

このリポジトリは、`ai-dev-standards` が定める判断を、実装時にどう適用するかを扱います。

| 資産 | 答える問い |
|---|---|
| [AI Development Standards](https://github.com/hamirilo/ai-dev-standards) | 何を守るか・何を選ぶか |
| **このリポジトリ** | どう進めるか・どう検証するか |
| UI / 実装リポジトリ | 実際に動くコードをどう再利用するか |
| プロジェクト側 | ドメイン固有の判断・実装・運用 |

StandardとPlaybookが矛盾する場合は、Standardとプロジェクト側の明示的な決定を優先します。

## 入口

最初から全ファイルを読まず、タスクに必要なものだけを参照してください。

| タスク | 参照するPlaybook |
|---|---|
| 機能追加・バグ修正・リファクタリング | [AI支援開発](playbooks/ai-assisted-development.md) |
| Djangoの実装・変更 | [Django実装](playbooks/django-implementation.md) |
| テスト・レビュー・UI確認 | [テストとレビュー](playbooks/testing-and-review.md) |
| 品質・性能・アクセシビリティ確認 | [品質確認](playbooks/quality-checks.md) |
| 認証・認可・エラー・外部通信 | [安全な実装とエラー処理](playbooks/security-and-error-handling.md) |
| APIを追加・変更 | [API実装](playbooks/api-implementation.md) |
| コンテナ化・イメージ配布・既存環境の移行 | [コンテナ配布](playbooks/container-delivery.md) |

## 増やさないもの

このリポジトリには、次のものを置きません。

- 全プロジェクトが守るべき少数の原則（`ai-dev-standards` の責務）
- 現時点のライブラリ選定（Recommendationsの責務）
- UIコンポーネント、デザインシステム、Storybook
- 新規プロジェクト用テンプレートやボイラープレート
- 再利用可能な実装パッケージ
- 社員・組織・認証基盤などの業務ドメイン固有情報
- AIエージェント定義、モデル設定、複雑なワークフロー実行基盤

コード例と失敗例は、独立した `examples/` を増やさず、関係するPlaybook内に必要な範囲で含めます。

## Playbook追加・昇格の基準

1. 複数のプロジェクトまたは複数回の実装で、同じ手順や失敗が繰り返されたことを確認する。
2. 特定プロジェクトのモデル名・URL・社内設定を除いても役に立つ形にする。
3. 「使う場面」「前提」「手順」「検証方法」「失敗時の確認」を一つの文書に含める。
4. 既存Standardと同じ判断を再記述せず、Standardへのリンクで参照関係を作る。
5. 実装コードが複数箇所で安定して再利用される場合は、Playbookではなく別の実装資産として昇格を検討する。

### 配置先の判断

| 内容 | 配置先 |
|---|---|
| 全体で守る判断原則 | `ai-dev-standards` |
| 現時点のライブラリの既定 | `ai-dev-standards/recommendations` |
| 実装手順・検証・失敗例 | `ai-dev-playbook` |
| コピーして使う雛形 | 将来のTemplateリポジトリまたはプロジェクト側 |
| 実行可能な共通コード | 実際の利用実績を確認した後、別Packageリポジトリ |
| 業務ドメインUI | そのドメインを所有するプロジェクト |

## 公開範囲

公開リポジトリとして扱える一般化された内容だけを置きます。社内URL、認証情報、個人情報、組織固有のモデル・運用、非公開の障害情報は追加しません。

