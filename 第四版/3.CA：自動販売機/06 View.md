# 06 View

### `interface_adapters/view_console.py` — `ConsoleView`

いよいよ一番ユーザー寄りの層、`View` です。
この章ではコンソールUI（ターミナル上の入出力）を例にして、`ConsoleView` を実装します。

`ConsoleView` は自販機にとっての「ボタンと小さな画面」。
ユーザーが押した操作を受け取り（入力）、結果メッセージを表示します（出力）。

## 🎯 役割

`ConsoleView` は、ユーザーと最も近い位置にある「画面（UI）」に相当します。責務は次の 2 点に限定されます。

1. 🧑‍💻 ユーザーからの操作（「A1ください」「100円入れる」など）を受け取る
   → `Controller`に伝える

2. 🖥 表示すべきメッセージを画面に出す
   → `Presenter`が準備した `ViewModel` の内容をそのまま出す

`ConsoleView` は、ビジネスルールもハードウェアの扱いも知りません。
ただ橋渡しをする「窓口」です。

このクラスは、ビジネスルール（貸出可否判断・金額計算など）を一切扱いません。また、データベースやハードウェアにも触れません。
役割としては「受付係」「掲示板」にあたります。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

## 🧭 プロジェクト内での位置づけ

フォルダ構成の中では、`ConsoleView` は `interface_adapters/` に属します。
View は Controller を呼び出し、Presenter が更新した ViewModel を読むだけで動作します。

```text
vending_machine/
├─ domain/
│   ├─ entities.py
│   └─ errors.py
│
├─ usecase/
│   ├─ dto.py                     # <DS> DTO / ViewModel
│   ├─ boundaries.py              # <I> 入出力インターフェースの定義
│   ├─ select_item_usecase.py     # <UC> ユースケース（例：商品を購入する）
│   └─ insert_coin_usecase.py     # <UC> ユースケース（例：お金を入れる）
│
├─ interface_adapters/
│   ├─ controller.py              # Controller（View→UseCase の橋渡し）
│   ├─ presenter.py               # Presenter（UseCase→ViewModel の整形）
│   ├─ view_console.py            # ← この章の主役（コンソールUI実装）
│   ├─ data_access.py             # 在庫などのデータアクセス実装（インメモリ等）
│   └─ hardware_adapter.py        # ハードウェア制御の具体実装
│
└─ main.py                        # すべてを組み立てて起動する場所
```
この図でいうと、`View` は最も外側の世界＝ユーザーと話す担当です。
でもビジネスルール（「買えるか？」「在庫あるか？」）は内側に聞きに行きます。直接決めません。

依存関係は次のようになります。

* View → Controller
  （ユーザーの操作を業務リクエストに変換してもらう）

* View → ViewModel
  （Presenter が更新した最新の表示内容を描画する）

View は UseCase や Entity を直接は知りません。
すなわち、ビジネスロジックの詳細から切り離されています。

---

## 💻 ソースコード（コンソール UI 実装例）

```python
# --------------------------------------------------------------------
# File: interface_adapters/view_console.py
# Layer: Interface Adapters（View）
#
# 目的:
#   - ユーザーとの直接のやり取り（入力と表示）を担う。
#   - 具体的なUI技術（ここではコンソールI/O）に依存する処理をまとめる。
#
# 責務:
#   - ユーザーから値を受け取る
#   - Controller に処理を依頼する
#   - Presenter が整形したメッセージ(ViewModel)を表示する
#
# 依存:
#   - Controller（interface_adapters.controller）
#     -> ビジネスロジックの呼び出し窓口
#   - ViewModel（usecase.dto で定義される表示用データ構造）
#     -> Presenter が更新する「表示専用の状態」
#
# 注意:
#   - View はビジネスルールを決めない
#   - View は DB / ハードウェアを直接触らない
# --------------------------------------------------------------------

from vending_machine.interface_adapters.controller import VendingMachineController
from vending_machine.usecase.dto import VendingMachineViewModel


class ConsoleView:
    """
    コンソール版の View（UI 表現そのもの）。

    - run()    : ユーザーと1ステップ分の対話を進める
    - render() : 現在の表示用メッセージを出力する

    View は Controller と ViewModel に依存するが、
    UseCase や Entity の中身は知らない。
    """

    def __init__(self, controller: VendingMachineController, presenter_view_model: VendingMachineViewModel):
        """
        Parameters
        ----------
        controller : VendingMachineController
            ユーザーの操作をユースケース呼び出しにつなぐ窓口。
            例: 「100円を入れたい」「商品A1を買いたい」といった要求を
            正式な入力DTOにまとめ、UseCase（業務ロジック）へ渡す。

        presenter_view_model : VendingMachineViewModel
            Presenter が更新する表示用データオブジェクト。
            最新のメッセージを View はこれから読み取って表示する。
        """
        self._controller = controller
        self._view_model = presenter_view_model

    def run(self) -> None:
        """
        1サイクルぶんのUI処理を担当する。
        - ユーザーに操作メニューを提示する
        - 入力を受け取る
        - Controller に依頼する
        - 最終的なメッセージを render() で表示する
        """

        print("自動販売機へようこそ。")
        print("操作を入力してください:")
        print("  i <金額>    ... コインを投入する   例: i 100")
        print("  s <スロット> ... 商品を購入する     例: s A1")
        print("  q           ... 終了する")

        raw = input("> ").strip()

        # 例:
        # "i 100" → ["i", "100"]
        # "s A1"  → ["s", "A1"]
        parts = raw.split()

        try:
            if not parts:
                # 何も入力されなかった場合は特別なメッセージを用意する
                self._view_model.message = "入力が確認できませんでした。"
                self.render()
                return

            command = parts[0]

            if command == "i":
                # コイン投入
                # 期待フォーマット: i <amount>
                amount_str = parts[1]
                amount = int(amount_str)  # 文字列→数値 などの軽い変換は View 側で行ってよい
                self._controller.insert_coin(amount)

            elif command == "s":
                # 商品選択
                # 期待フォーマット: s <slot_id>
                slot_id = parts[1]
                self._controller.select_item(slot_id)

            elif command == "q":
                # 終了コマンド
                self._view_model.message = "ありがとうございました！"
                self.render()
                return

            else:
                # 未知コマンド
                self._view_model.message = f"不明なコマンドです: {command}"

        except (IndexError, ValueError):
            # 例:
            # - "i" だけで金額が指定されなかった
            # - "i abc" のように数値でない値が入力された
            #
            # これはあくまで「UIレベルの形式エラー」なので、
            # ビジネスロジック層（UseCase）まで渡す必要はない。
            self._view_model.message = "入力形式が正しくありません。使い方を確認してください。"

        except Exception as e:
            # UseCase層やRepository層など、内側から伝播してきたエラーを
            # ユーザー向けの短いメッセージに変換して出力用に格納する。
            #
            # Viewは「どう見せるか」にのみ責任を持つので、
            # ここでは例外オブジェクトをそのまま文字列化して保存している。
            self._view_model.message = f"エラー: {e}"

        # 最終的な状態を必ず描画する
        self.render()

    def render(self) -> None:
        """
        現在の ViewModel の内容を画面に表示する。
        View は「何が起きたのか」を判断しない。ただ表示するだけ。
        """
        print("\n--- 現在の状態 ---")
        print(self._view_model.message)
        print("------------------")
```

---

## 🔍 コードで押さえるべき点

### 1. View は Controller を呼び出すだけ

View はユーザーの操作を Controller に伝えることに集中します。
例：`self._controller.insert_coin(amount)` / `self._controller.select_item(slot_id)`

Controller 側で DTO を作り、UseCase の正式な `handle()` を呼びます。
この分業により、View はユースケースの詳細やデータアクセスの仕組みを知る必要がありません。

---

### 2. View は ViewModel を直接表示するだけ

`self._view_model.message` のように、Presenter が整形した「人に見せる形の文言」をそのまま表示します。
View 自身が日付フォーマットや敬称付けなどの整形をすることはありません。

これは責務分離のポイントです。

* 「どのように表示するかの文章をつくる」のは Presenter
* 「その文章を出力するだけ」のが View

---

### 3. View は「軽い入力変換」までは担当してよい

たとえば、ユーザーが `"i 100"` と入力したとき `100` は文字列として渡ってきます。
これを `int(100)` にする程度の処理は View の責務に含めても構いません。
これは UI の形式的な都合であり、ビジネス判断ではないからです。

一方で、「この金額なら購入可能か？」といった判断は View の責務ではありません。
それは UseCase（ビジネスロジック）が行います。

---

### 4. 例外処理の方針

`try`〜`except` のブロックでエラーを捕まえ、ユーザーに分かりやすいメッセージを ViewModel に格納しています。

* 入力形式エラー（例：数字でない金額）が起きた場合
  → View 内で処理してよい（UIの問題だから）

* ビジネスルール違反など、UseCase 側から上がってきた例外
  → 文字列化してメッセージとして表示する

ここでも、View は「説明用のメッセージを提示するだけ」にとどまり、ビジネスロジックの修正や再計算は行いません。

---

## ✅ このクラスに含めてよいこと / 含めてはいけないこと

### ⭕ 含めてよいこと

* `input()` / `print()` による入出力
* 簡単なパース（例：コマンドと引数を分割する）
* 文字列→数値変換のような軽い整形
* Controller メソッドの呼び出し
* ViewModel の内容の表示

### ❌ 含めてはいけないこと

* ビジネスルールの判断（購入可能か？貸出許可か？在庫はあるか？など）
* 結果テキストの整形（「返却期限は〇年〇月〇日です」などの体裁づくり）
* DB・ハードウェアへのアクセス
* エンティティ（`Item`, `PaymentManager` など）の直接操作

View は「渡す・表示する」ことに徹します。
これ以上の責務を抱えさせると、UI をコンソールからWebに差し替えるときにビジネスロジックごと書き直すことになってしまいます。

---

## 🧪 単体テストのイメージ

View は I/O を伴うので、テストでは `input()` や `print()` を一時的に差し替えます。
目的は「Controller が正しい引数で呼ばれるか」「ViewModel に結果が反映されるか」を確認することです。

```python
# tests/interface_adapters/test_view_console.py

from unittest.mock import patch, MagicMock
from vending_machine.usecase.dto import VendingMachineViewModel
from vending_machine.interface_adapters.view_console import ConsoleView


def test_viewはユーザー入力を_controllerに正しく伝える():
    # Arrange
    fake_controller = MagicMock()
    vm = VendingMachineViewModel(message="")

    view = ConsoleView(controller=fake_controller, presenter_view_model=vm)

    # Act
    # 入力 "i 100"（100円投入）をシミュレートする
    with patch("builtins.input", return_value="i 100"):
        with patch("builtins.print"):  # コンソール出力を抑制
            view.run()

    # Assert
    # insert_coin が正しい値で呼ばれているか？
    fake_controller.insert_coin.assert_called_once_with(100)
```

このテストでは、ユースケースそのものは動いていません。
Repository も Presenter も不要です。
View 単体で期待どおりに Controller を呼べるかを確認できる、ということです。

---

## 🐍 Python と C の比較（イメージ）

* 従来の手続き的な書き方では、`main()` 相当の処理の中に
  「ユーザー入力の読み取り」「ビジネスロジックの実行」「結果メッセージ整形」「printによる表示」
  がすべて混在しがちです。

* クリーンアーキテクチャでは、それらを役割ごとに分割します。

  * View … 入力と表示だけ
  * Controller … 入力をユースケースの正式なリクエスト形式に変換する
  * UseCase … ビジネスルールの適用と状態変更
  * Presenter … 人間が読むためのメッセージ整形
  * View … 整形済みメッセージの表示

結果として、UI技術（コンソール → GUI → Web）を差し替えるだけなら View や Presenter だけを交換すればよく、内側のユースケースやエンティティをまったく変更しなくて済みます。

---

## 🛡 鉄則

> View は「入力を受ける場所」と「結果を表示する場所」。
> 判断やロジックは持たない。

* View は Controller と ViewModel だけを相手にすればよい
* UI 技術が変わっても（CLI / Web / GUI）、ユースケース本体は無傷で再利用できる
* ビジネスロジックは「一番内側」に閉じ込め、UI に漏れ出させない

この分離こそが、クリーンアーキテクチャの「UIは詳細である」という考え方を具体的に体現する要素です。
