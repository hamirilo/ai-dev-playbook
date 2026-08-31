# AI支援開発の進め方

機能追加、bug修正、refactoringをAI agentと進めるときの実行手順です。

通常のApplicationでは、最初に [ai-dev-platform](https://github.com/hamirilo/ai-dev-platform) の `ai/ONBOARDING.md` を読み、そこからTaskに必要なStandard / Recommendation / Playbookだけへ進みます。

## 使う場面

- 変更範囲が複数fileにまたがるとき
- 既存実装の調査が必要なとき
- 動作確認やreview観点を毎回考え直しているとき

単純な1file修正では必要な手順だけを使います。

## 基本ループ

### 1. 目的と完了条件を固定する

最初に次を短く決めます。

- 何を変えるか
- 何を変えないか
- 利用者から見た完了条件
- 影響を受ける境界（認証、data、外部service、UI）
- 実行する検証

完了条件が曖昧なまま実装を始めません。

### 2. 必要なcontextだけを調べる

次の順で対象を絞ります。

1. Projectの `CLAUDE.md` 等の入口と `decisions/project-context.md` を読む。
2. Platform ONBOARDINGからTaskに必要なCore Standardを読む。
3. Optional Standard、Recommendation、Playbookは該当するものだけ読む。
4. 対象file、呼び出し元、類似実装、関連testを検索する。
5. 変更しない範囲を確認する。

repository全体や共有documentを最初から全量読み込みません。

### 3. 設計を小さく決める

実装前に、少なくとも次を確定します。

- 既存実装を使えるか
- 新しいlayerや依存が本当に必要か
- dataの所有者と更新経路
- 失敗時の表示・記録・再実行方法
- testで確認する振る舞い

全体の構成を作り直すのではなく、目的を満たす最小の変更にします。

### 4. 小さい単位で実装する

- 1stepごとに実行可能な状態を保つ。
- 無関係なformat、rename、moveを混ぜない。
- 既存Component、function、testを優先して使う。
- 仕様変更とrefactoringを同じ変更へ不用意に混ぜない。
- 自動生成codeやFormatterの出力を手作業で再調整しない。

### 5. 機械的な検証を先に実行する

Projectで用意されたFormatter、Linter、Type Checker、Build、基本testを実行します。
機械的に判定できる問題をAIの推論や人間の目視だけで確認しません。

自動修正できるものはtoolで直し、意味の判断が必要な問題だけをAIまたは人間が確認します。

### 6. 壊れるscenarioをreviewする

最低限、次を確認します。

- 正常系だけでなく入力error・権限error・外部障害が扱われるか
- 同時更新や二重送信でdataが壊れないか
- 削除・状態変更の影響範囲が把握できるか
- logに調査に必要な情報があり、秘密情報が出ていないか
- empty・loading・failure・大量dataのUIが成立するか
- 変更した契約を既存の利用側が壊さないか

### 7. 人間へ引き渡す

完了報告には次を含めます。

- 変更したfileと目的
- 実行した検証と結果
- 影響範囲と未確認事項
- 仕様上の判断、残るrisk、次の作業

「testが通った」だけで完了としません。

## コストを抑える原則

- Platform ONBOARDINGをrouterとして、共有documentはTaskに必要な部分だけ読む。
- 調査は検索語・対象directory・目的を先に決める。
- 同じ失敗を繰り返す場合だけPlaybookへ昇格する。
- FormatterやLinterの結果をAIに長文で説明させず、error箇所を修正させる。
- 1回の依頼に実装、全面refactoring、設計刷新を同時に含めない。

## よくある失敗

| 失敗 | 改善 |
|---|---|
| 最初に全documentを読ませる | Platformの入口からTaskに必要なdocumentだけ読む |
| 既存実装を探さず新しい抽象化を作る | 類似codeと利用実績を先に検索する |
| test成功だけで完了とする | UI、権限、失敗、data境界も確認する |
| formatをAIに手作業で合わせさせる | Formatterを正とし、自動修正する |
| 一度の変更で多くのfileを整理する | 目的ごとに変更を分ける |
