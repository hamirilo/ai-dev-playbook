# DjangoとReact Islandsの接続

[Architecture Standard §2](https://github.com/hamirilo/ai-dev-standards/blob/main/standards/architecture/README.md) と
[ADR-0007](https://github.com/hamirilo/ai-dev-standards/blob/main/decisions/adr-0007-presentation-only-islands-and-template-classes.md) が定める境界を、
Django Templates + htmx + `application-ui-kit` で実装するときの手順です。

Componentのpropsと見た目はUI PlatformのStorybook（「はじめに / Django Islands」「基礎 / テンプレート用クラス」）を正とし、ここには書きません。

## 使う場面

- Django Templateから `application-ui-kit` のIsland（dialog、toast、date picker、tabs等）を初めて置く
- htmxの部分更新にIslandを混ぜる、または部分更新の結果をtoastで知らせる
- 削除確認、成功通知、CSRFの送り方がテンプレートごとにばらついているのを揃える
- 自前で書いたtab / 開閉 / 入力欄の出し分け / 確認modalをkitの汎用Islandへ置き換える
- 自前のtemplate用CSS（`.btn` / `.alert` / `.tabs` 等）をkitのtemplate用classへ寄せる

## 前提

- `application-ui-kit` を依存に持ち、`@import "application-ui-kit/styles.css"` でTokenとtemplate用classを読み込んでいる（Token / cn-*スキン / template用classは別ファイルに分かれており、`styles.css` が入口）
- htmxはkitの依存ではない。Templateの `<head>` で `window.htmx` として読み込む
- Islandを置く画面のroutingとFormはDjangoにある。Islandのために新しいJSON endpointを作らない（Architecture Standard §8）
- 業務domain固有のIslandはApplication側に置く（Application UI Standard §6）
- 見せ方だけの島とconfirm-hostはkit 6.3以上

## 手順

### 1. entryを1つ用意し、auto-mountを読み込む

```ts
// islands/main.ts
import 'application-ui-kit/islands/auto-mount'
```

- `base.html` の読み込み順は **htmx → entry**。auto-mountは読み込み時に `window.htmx` を見て、部分更新後の再mountを付ける。逆順にすると、htmxで差し替えた領域のIslandだけが動かない
- 自前のregistryを持つApplicationは、`application-ui-kit/islands` からIslandをimportして自分のregistryに登録する。default exportが要る場合は `export { TabsIsland as default } from 'application-ui-kit/islands'` のような1行のラッパーを書く
- Application固有のIslandは `registerIslandComponents({ 'my-widget': MyWidget })` で足す。kitの島と同じ `data-react` で使える

### 2. mount pointとpropsの渡し方

- `<div data-react="…">` を置く。propsは個別の `data-*` か `data-props`（JSON）で渡す
- Django側で `json.dumps()` した文字列を `data-props="{{ props_json }}"` と書き、autoescapeに任せる。`|safe` を付けない
- 個別 `data-*` はJSONとして解釈される（`data-count="3"` は数値、`"true"` は真偽値）。URL・token・大きな数値IDを文字列で保ちたい値は `data-props` 側に入れる
- 入力値を含むpropsをHTML属性へ文字列連結しない（[安全な実装とエラー処理](security-and-error-handling.md)「CSRFとブラウザ境界」）

### 3. toastを1本にする

- `base.html` に `data-react="toast-listener"` を1つだけ置く。表示領域を持つのはこの島だけ
- Django `messages` は `data-messages` へJSON化して渡す
- 素のJS / htmxからは `window.ApplicationToast.success(...)`。存在しないときに `?.` で握り潰すfallbackを書かない（出ない原因が見えなくなる）
- 既存コードが別名（例: `window.DxToast`）を呼んでいる場合は `data-global-aliases='["DxToast"]'` で同じ実体を登録し、呼び出し側を書き換えない
- Reactがmountする前に呼ばれる可能性がある場所は、`base.html` の先頭で `window.__applicationToastQueue = []` を用意し、`push(["success", "保存しました"])` しておく。mount時に消化される

### 4. htmxの部分更新

- 部分templateは `templates/<app>/partials/_<name>.html` に置き、full pageからも `{% include %}` する（[Django実装](django-implementation.md)「Form、View、部分更新」）
- Viewは `HX-Request` headerで部分HTMLとfull pageを分ける
- swap先の中にIslandがある場合は何もしなくてよい（`htmx:afterSwap` で再mountされる）。swap先の外にあるIslandは再mountされないので、Islandを差し替え範囲の外に置く
- 部分更新の結果をtoastで知らせる: `HX-Trigger: {"application-toast": {"type": "success", "title": "保存しました"}}`
- form-dialogの送信成功: `HX-Trigger: application-form-success`

### 5. CSRF

- htmxは `<body hx-headers='{"X-CSRFToken": "{{ csrf_token }}"}'>` に1箇所だけ書く
- IslandのfetchはCookieからtokenを読む。`CSRF_COOKIE_NAME` を変更しているprojectは `data-csrf-cookie-name` を **すべての** state変更Island（confirm-dialog / confirm-host）へ渡す。context processorで1箇所から出す
- cookie名をJavaScriptに直書きしない（`document.cookie.match(/csrftoken=/)` のような形は設定変更で静かに403になる）

### 6. 確認dialog

- `base.html` に `data-react="confirm-host" data-intercept-hx-confirm="true"` を1つ置く。`hx-confirm="…"` がそのままkitのdialogになる。文言は要素の `data-confirm-title` / `data-confirm-text` / `data-confirm-type` で上書きする
- 既存の `$dispatch('confirm-modal', {...})`（Alpine）はそのまま受かる。`confirmClass` と `onConfirm` 関数を含む形で互換
- 自前で `htmx:confirm` を握っているlistenerがある場合は、それを外してから `data-intercept-hx-confirm` を付ける（両方あると2回聞かれる、または2回飛ぶ）
- ボタンごとに宣言的に指定したい場合だけ `confirm-dialog` を使う
- 独自のCustomEvent名でdialogを開かない。受け手の無いeventは何も起こさず、押しても反応しないボタンとして残る

### 7. 見せ方だけの島

- `tabs` / `disclosure` / `field-visibility`。契約はStorybook「Django Islands」を正とする
- パネル・対象要素はDjango Templateが描く。初期非表示のものには **サーバー側で `hidden`** を付ける（付けないとmountまでの一瞬すべて見える）
- 隠れた入力欄の値もPOSTされる。捨てる判断はFormの `clean()` に置く
- tab一覧・ラベル・権限による有無はパネル側の属性が正。propsに二重で書かない
- JSを使わずに済む単純な開閉は `<details class="disclosure">` で足りる。件数の動的更新や見出しレベルの制御が要るときだけIslandにする

### 8. template用classへ寄せる

- `styles.css` を読み込んだうえで、**自前の同名class（`.btn-primary` / `.alert` / `.tabs` 等）を削除する**。残すと読み込み順で自前が勝ち、見た目が何も変わらない
- 対応表はUI Platformの `design-system/README.md`「Djangoテンプレートから使うクラス」。Storybook「基礎 / テンプレート用クラス」と実画面を並べて差分を確認する
- 状態バッジは `models.py` の `*_display_class` が `badge-active` のようなtone class名を返す。色をtemplateに書かない
- 名前が違う自前class（`.alert-error` → `.alert-danger`、`.dx-page-title` → `.page-header-title` 等）は採用時に置換する

### 9. Application固有のIslandを書くとき

- domain連携（endpoint、認証、マスタ取得）を持つものだけをApplicationで書き、`registerIslandComponents` で登録する
- 部品はkitのComponentから取る。`.btn` 直書きや自前overlay、`alert()` に戻さない

## 検証

- 画面をhtmxで差し替えた後もIslandが動く（consoleに `[React Islands] Component "…" not found` が出ない）
- `CSRF_COOKIE_NAME` を既定以外にしてもPOSTが403にならない
- messages / `HX-Trigger` / JS呼び出しの3経路でtoastが出る
- 見せ方だけの島: JS無効でも初期表示の1枚が見える。隠れたパネルの入力値がPOSTされ、Formが捨てる
- 確認dialogでEscapeとfocus移動が効く。ネイティブの `confirm()` が呼ばれない
- template用classへ寄せた画面で、Storybookの見本と同じ見た目になっている（自前classが勝っていない）
- 実browser確認は [テストとレビュー](testing-and-review.md) に従う

## 失敗時の確認

| 症状 | 確認すること |
|---|---|
| ボタンを押しても何も起きない | 発火しているevent名にlistenerがあるか（`confirm-modal` なら `confirm-host` が置かれているか）。`getRegisteredIslandComponents()` に該当Islandがあるか |
| POSTが403 | `CSRF_COOKIE_NAME` と `data-csrf-cookie-name` / `hx-headers` の値が一致しているか。cookie名をJSに直書きしていないか |
| toastが出ないがerrorも出ない | `toast-listener` がbase.htmlにあるか。`window.<独自名>` を参照しているなら `data-global-aliases` に登録したか。`?.` で存在確認して握り潰していないか |
| htmxで差し替えた領域のIslandだけ動かない | htmxがentryより前に読み込まれているか（auto-mountは読み込み時に `window.htmx` を見る） |
| Islandが二重に描かれる / `createRoot` の警告 | 同じ要素へ手動mountとauto-mountを両方かけていないか |
| 初期表示で全パネルが一瞬見える | server側で `hidden` を付けているか |
| 確認dialogが2回出る / 削除が2回飛ぶ | 自前の `htmx:confirm` listenerが残っていないか。`confirm-host` を2つ置いていないか（2つ目は警告を出して何もしない） |
| template用classを入れたのに見た目が変わらない | 自前の同名classを削除したか。`styles.css` の読み込みが自前CSSより前になっていないか |
| `data-props` がparseできない | `\|safe` を付けていないか。属性の引用符とJSONの引用符が衝突していないか |
| Islandが文字列のpropsを数値として受け取る | 文字列で保ちたい値を個別 `data-*` に置いていないか（`data-props` 側へ移す） |

## 扱わないもの

- 各Islandのprops一覧・見た目 → UI PlatformのStorybook
- React Componentを `.tsx` から使う方法 → UI Platform
- library選定 → Recommendations
- project固有のcookie名、URL、権限 → 各ApplicationのProject Context
