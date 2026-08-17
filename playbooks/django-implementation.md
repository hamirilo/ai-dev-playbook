# Django実装プレイブック

[Architecture Standard](https://github.com/hamirilo/ai-dev-standards/blob/main/standards/architecture/README.md) の判断を、Djangoで実装するときの具体的な進め方です。

## 責務の置き場所

| 内容 | 基本の置き場所 |
|---|---|
| フィールド、不変条件、状態遷移 | Model |
| 再利用するクエリ | QuerySet / Manager |
| 入力の検証・整形 | FormまたはSchema |
| 認可 | 専用のpermission module |
| リクエスト処理・画面遷移 | View |
| テンプレートへのデータ組み立て | Viewまたは明示的なPresenter |
| 複数モデル、外部通信、明確なトランザクション | 必要な場合だけService |

処理を置く場所が曖昧だからという理由で、Repository、UseCase、Selectorなどの層を先に増やしません。

## 実装前の確認

1. 対象Modelと関連Modelを読む
2. URL、View、Form、Template、既存テストを追う
3. 同じ種類の画面や処理を検索する
4. データの所有者、認可境界、状態遷移を確認する
5. 追加する検証と、変更しない契約を決める

## Modelとデータ整合性

- どの入口から保存されても守るべき不変条件はModelまたはDB制約に置く
- Viewだけ、Formだけの検証にしない
- 「1件だけ有効」「重複禁止」など競合に関わる条件はDB制約を検討する
- 状態遷移は許可される遷移と不正な遷移を明示する
- 削除時の `CASCADE`、`SET_NULL`、`PROTECT` が利用者へ与える影響を確認する

```python
class Record(models.Model):
    state = models.CharField(max_length=20)

    def clean(self):
        if self.state == "closed" and not self.close_reason:
            raise ValidationError({"close_reason": "終了理由が必要です。"})
```

## QuerySetと一覧処理

一覧を実装するときは、表示する関連データと件数を先に確認します。

```python
records = (
    Record.objects.filter(owner=request.user)
    .select_related("category")
    .prefetch_related("labels")
    .order_by("-created_at")
)
```

- ForeignKeyは `select_related()` を検討する
- 多対多・逆参照は `prefetch_related()` を検討する
- OR条件でJOINが発生する場合は重複行と `.distinct()` の要否を確認する
- 大量データはページングし、関連データを無制限に展開しない
- クエリの再利用はQuerySet / Managerへ置き、呼び出し側で同じ条件を複製しない

## Form、View、部分更新

- サーバー側の検証を必ず残す。クライアント側検証はUX改善として追加する
- Formにモデルの不変条件をコピーせず、Modelの検証結果を利用する
- Django Formで完結する処理に、理由なくJSON APIを追加しない
- 部分更新では、フルページと部分HTMLの責務を分け、同じ部分テンプレートを再利用する
- 保存中は二重送信を防ぎ、失敗時に入力内容を失わない

## 認可とリソース取得

業務権限は専用の認可層へ集約します。`is_staff` や `is_superuser` を業務機能の権限として直接流用しません。

```python
record = get_object_or_404(
    Record,
    id=record_id,
    owner=request.user,
)
```

リソースの存在自体を隠す必要がある場合は404、存在を隠す必要がない単なる権限不足は403を基本とします。
ビュー、テンプレート、APIで認可ロジックを重複させないでください。

## React Islandsとの境界

- ルーティング、ページシェル、Django Form、認可はDjango側を基本にする
- UI状態を持つ複雑な操作はReact Islandへ置く
- React Islandを使うことだけを理由にAPIを増やさない
- DjangoからReactへデータを渡すときは安全なシリアライズ手段を使い、文字列連結でJSONを埋め込まない
- 状態変更ではCSRF保護を維持する

## 変更後の検証

- Model制約と状態遷移
- 未認証・一般ユーザー・権限あり・対象外ユーザー
- 正常系、入力エラー、空データ、重複送信
- 一覧のクエリ数とページング
- 部分更新とフルページ表示
- 既存の呼び出し元とURL/API契約

## 避けること

- Viewに業務ルールを集める
- FormとModelに同じ不変条件を二重に書く
- 一覧で関連を遅延取得したままにする
- 小さな処理のために新しい抽象レイヤーを作る
- React側とDjango側に同じ業務ロジックを複製する

