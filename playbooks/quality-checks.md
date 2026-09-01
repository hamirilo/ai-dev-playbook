# 品質確認プレイブック

性能、accessibility、responsive対応、実browser上の操作性を、実装後に確認するための手順です。全画面に同じ確認を要求するものではなく、画面の重要度と変更範囲に応じて使います。

このPlaybookでは必須gateと品質推奨を分けて扱います。対象projectで定義されている型チェック、Linter、Build、基本testは必須gateです。Lighthouse、Core Web Vitals、N+1 query対策、高度なperformance最適化はSoft Targetです。

AI agentは品質推奨を常時適用せず、ユーザーから明示的な指示がある場合、または大規模なUI・performance変更を行う場合に参照・適用します。

## 使う場面

- 新しい主要画面や導線を追加したとき
- 一覧、検索、Form、React Islandを大きく変更したとき
- JavaScript、CSS、画像、font、外部scriptを追加・変更したとき
- 共通UI Componentを変更したとき
- layout崩れ、遅い操作、keyboard操作の問題が報告されたとき
- release前に代表的な画面を確認するとき

## 手順

### 1. 代表pageと完了条件を決める

最低限、変更の影響を受けるpageを1つ以上選びます。規模が大きい場合は、top、一覧・検索、Form、詳細から代表pageを選びます。

次を作業開始時に決めます。

- どの利用者・device widthを対象にするか
- 主要操作を何とするか
- Lighthouseやbrowserで何を確認するか
- 今回は確認しない範囲と、その理由

### 2. 生成物を使って測定する

開発serverだけでなく、本番に近いbuildと設定で確認します。

- 最適化済みのCSS、JavaScript、画像を使う
- staging等、本番に近いserver性能で確認する
- 認証画面はtest用accountで確認し、認証情報をlogへ出さない
- 同じpageを同じ条件で複数回測定し、極端な一回の結果だけで判断しない

### 3. Lighthouseを実行する

代表pageについて、次を確認します。

- Performance
- Accessibility
- Best Practices
- SEO（外部公開や検索流入が必要な場合のみ）

Performance、Accessibility、Best Practicesは100を目標にします。ただし100点そのものを機械的な合格条件にせず、90未満の項目、重大なaccessibility問題、利用者の主要操作を妨げる問題を優先して直します。

scoreを上げるために機能や意味のあるaccessibility情報を削りません。未達を残す場合は、未達項目、理由、利用者への影響、再確認の条件を報告します。

### 4. Core Web Vitalsを確認する

目安として次を確認します。

| 指標 | 良好な目安 |
|---|---:|
| LCP | 2.5秒以内 |
| INP | 200ms以内 |
| CLS | 0.1以下 |

lab計測は回帰確認に使います。可能であれば実利用環境のdataも確認し、初期表示だけでなく操作後の遅延やlayout shiftも見ます。

### 5. 実装品質を確認する

#### N+1 query

一覧・検索・詳細画面で、表示件数や関連dataの増加に比例してquery数が増えないか確認します。

- ForeignKeyやOneToOneは必要に応じて `select_related()` を使う
- 多対多や逆参照は必要に応じて `prefetch_related()` を使う
- 取得した関連をTemplate、Serializer、Presenter等で再度遅延取得していないか確認する
- paging後の件数だけでなく関連dataの取得queryも確認する
- 重要な一覧・検索ではquery数を固定するtestや計測を検討する

すべてを先回りして取得するのではなく、画面が必要とするdataと計測結果に基づいて取得範囲を決めます。

#### TypeScriptと型検査

Reactの共有実装はTypeScriptを基本とし、ComponentはTSX、logicはTSで記述します。

- JSXを追加する前にTSXへする必要がないか確認する
- typecheckを実行し、暗黙のanyや不整合なPropsを残さない
- anyでerrorを隠さず、unknownや適切な型、type guardを検討する
- API response、Djangoから渡るJSON、localStorage等の外部境界はruntime validationを検討する
- TypeScriptの型検査はruntime validationやserver-side validationの代替にしない

#### 機械的な検証

projectで用意されている次の検証を、意味の判断より先に実行します。

- formatter
- lint
- typecheck
- unit / integration test
- Storybook build
- 必要に応じた依存関係・bundle size確認

### 6. 実browserで主要操作を確認する

自動testやscoreに加えて、次を手動またはbrowser automationで確認します。

- keyboardだけで主要操作を完了できる
- focusが見え、Dialog / Menuを閉じた後の戻り先が自然である
- loading、empty、error、success、大量dataのstateが成立する
- 長い文字列やserver-side validation errorで崩れない
- mobile width・狭い画面幅で操作が隠れない
- 連打、二重送信、back、reloadでstateが壊れない
- console error、不要なnetwork failure、layout shiftがない
- reduced motion設定を尊重できる

### 7. 結果を報告する

PRや完了報告には必要に応じて次を含めます。

- 対象page、利用者、device width、測定環境
- Lighthouseの各category結果
- Core Web Vitalsの結果
- 実施した主要操作と結果
- 未確認事項、未達項目、残るrisk
- 再確認が必要になる条件

## 関連

- [Quality Recommendations](https://github.com/hamirilo/ai-dev-platform/blob/main/recommendations/quality.md)
- [テスト・レビュー・UI確認](testing-and-review.md)
