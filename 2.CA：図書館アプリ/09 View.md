# 09 View

# 🎨 View
### `interface_adapters/views/view_console.py` - `ConsoleView`

---

## 🎯 このクラスの役割

`ConsoleView` は、**ユーザーが直接触るインターフェース（UI）そのもの**です。責務はきれいに 2 つだけ。

1. **表示（Output）**
   `Presenter` によって更新された `ViewModel` の内容を、コンソールに描画する。

2. **入力（Input）**
   ユーザーからの入力を受け取り、その入力を `Controller` に渡す。

これ以上のことはしません。ビジネスルールは扱わないし、DBにも触れません。
UIの「受付係」と思ってください。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 📁 フォルダ構成の中での View の位置

`ConsoleView` は **Interface Adapters 層**に属します。
Controller と同じ層にいますが、役割が違います。

```text
├─ core/
│   ├─ domain/
│   │   ├─ book.py
│   │   ├─ member.py
│   │   ├─ loan.py
│   │   └─ repository.py
│   │
│   └─ usecase/
│       ├─ boundary/
│       │   ├─ dto.py                # <DS> InputData / OutputData / ViewModel
│       │   ├─ input_boundary.py     # <I> InputBoundary (Controllerが呼ぶ)
│       │   └─ output_boundary.py    # <I> OutputBoundary (Presenterが実装)
│       │
│       └─ interactor/
│           └─ check_out_book.py     # <UC> 「本を貸し出す」ユースケース本体
│
└─ interface_adapters/
    ├─ controllers/
    │   └─ checkout_controller.py    # Controller（View → Use Caseの橋渡し）
    ├─ presenters/
    │   └─ checkout_presenter.py     # Presenter（Use Case → ViewModelの翻訳）
    └─ views/
        └─ view_console.py           # ← この章の主役（Console UI）
```

View は Controller を呼びます。
Presenter は ViewModel を更新します。
View はその ViewModel を読むだけです。
つまり View は Presenter や UseCaseの中身を知らないまま動けます。

---

## 💻 ソースコード（コンソール版 View）

```python
# --------------------------------------------------------------------
# File: interface_adapters/views/view_console.py
# Layer: Interface Adapters（View）
#
# 責務:
#   - ユーザーとの直接的なやり取り（入力受付・画面表示）
#   - Console という具体的なUI技術に依存する部分
#
# 依存:
#   - CheckOutBookController (同じ Interface Adapters 層のコントローラ)
#   - BookViewModel (core.usecase.boundary.dto で定義された表示用データ構造)
#
# 同心円図での位置:
#   - Frameworks & Drivers 相当 / Interface Adapters のUIエッジ
#   - 内側（Use Case, Entity）には依存させない
# --------------------------------------------------------------------

from interface_adapters.controllers.checkout_controller import CheckOutBookController
from core.usecase.boundary.dto import BookViewModel


class ConsoleView:
    """
    コンソール版のView。
    ユーザーとの入出力にだけ責任を持つ。

    - run(): 入力を受け取って Controller に渡す
    - render(): ViewModel の内容を画面に表示する
    """

    def __init__(self, controller: CheckOutBookController, view_model: BookViewModel):
        """
        controller:
            ユーザー操作を業務的なリクエストに変換して Use Case を呼ぶ係
        view_model:
            Presenter が更新する「表示専用のデータ入れ物」
        """
        self._controller = controller
        self._view_model = view_model

    def run(self) -> None:
        """
        [入力処理／1サイクル分]
        ユーザーから入力を受け取り、Controller に伝える。
        その後、結果を画面に描画する。
        """

        try:
            book_id_str = input("貸し出す本のIDを入力してください: ")
            member_id_str = input("貸し出す会員のIDを入力してください: ")

            # UIとして必要な最小限の前処理（文字列→数値など）はViewの責務
            book_id = int(book_id_str)
            member_id = int(member_id_str)

            # ビジネスロジックは Controller → Use Case 側に任せる
            self._controller.check_out(book_id=book_id, member_id=member_id)

        except ValueError:
            # 入力が数字でなかったなどの「UIレベルの問題」はここで処理してOK
            # （ビジネスルールではないので Use Case や Presenter を呼ぶ必要はない）
            self._view_model.display_text = "エラー: IDは数字で入力してください。"

        except Exception as e:
            # Use Case や Repository から上がってきた例外など
            # ここではユーザーにわかる形にして表示用状態に入れる
            self._view_model.display_text = f"エラー: {e}"

        # 最後に、現在のViewModelの状態を表示
        self.render()

    def render(self) -> None:
        """
        [表示処理]
        ViewModelの内容をもとに実際の画面出力を行う。
        """
        print("\n--- 処理結果 ---")
        print(self._view_model.display_text)
        print("----------------")
```

ポイントまとめ：

* `ConsoleView` は Controller と ViewModel だけを知っていれば動ける。
* Use Case／Presenter／Repository／Entity は import しない（＝依存しない）。
* これは「一番外側の層ほど、内側のことを意識しないで差し替えたいから」という設計意図に沿っています。

  * 例：同じ `CheckOutBookController` と `BookViewModel` を使いまわして、GUIやWeb版の View に差し替えることが可能になる。

---

## ✅ このファイルにあるべき処理

⭕️ **含めるべき処理の例:**

* `input()` や `print()` のような入出力
* 数字変換など、UIとして必要な表面的バリデーション
* Controller のメソッド呼び出し
* ViewModel の表示

❌ **含めてはいけない処理の例:**

* 「この会員は貸出可能か？」の判断 → それは Use Case の仕事
* 日付フォーマットや敬称などの整形 → それは Presenter の仕事
* データベースへのアクセスや永続化処理 → Repository の仕事
* 例外的な業務判断（貸出可能冊数の上限など）→ Entity / Use Case の仕事

View は「見せて、渡す」。それだけで十分。

---

## 🧪 View のユニットテストはどう書く？

`View`は I/O を直接やるのでちょっとだけ工夫します。
考えたいことはただひとつ：

> ユーザーが "100" と "200" と打ったら、
> `controller.check_out(book_id=100, member_id=200)` が呼ばれるか？

それだけを検証します。
`input()` と `print()` はテスト中だけ差し替えます。

```python
# tests/interface_adapters/views/test_view_console.py

from unittest.mock import patch, MagicMock
from core.usecase.boundary.dto import BookViewModel
from interface_adapters.views.view_console import ConsoleView


def test_viewはユーザー入力をcontrollerに正しく伝える():
    # Arrange: Controller は MagicMock で代用する
    mock_controller = MagicMock()

    # ViewModel は Presenter が本来更新する入れ物
    view_model = BookViewModel(display_text="")

    view = ConsoleView(controller=mock_controller, view_model=view_model)

    # Act: input() を差し替えて run() を実行
    # side_effect の順番通りに input() が返る
    with patch("builtins.input", side_effect=["100", "200"]):
        # print() を無効化してテスト出力を静かにすることもできる
        with patch("builtins.print"):
            view.run()

    # Assert: Controller が正しい引数で呼ばれたか
    mock_controller.check_out.assert_called_once_with(book_id=100, member_id=200)
```

ここでやっていないこと：

* Use Case の動作確認 → 別章でテストする
* Presenter のメッセージ整形の確認 → 別章でテストする
  （View はそれらに立ち入らないので）

ここでもう一度、「薄い責務」のおかげで単体テストがしやすいことがわかります。

---

## 🐍 Python と C のイメージ的な違い

* **Python / クリーンアーキテクチャ的世界観**

  * View はクラス化されていて、Controller と ViewModel を外から渡して（依存性注入）、外部UIを入れ替えできるようにする。
  * つまり「コンソールUI」「GUI」「Web UI」が全部きれいに差し替え可能。

* **C 言語的世界観**

  * だいたい `main()` の中の while ループが「View＋Controller＋ビジネスロジックぜんぶ入り」になりやすい。
  * ユーザー入力を `scanf` で受けて、すぐにビジネスロジック関数を呼び、すぐに `printf` で結果を出す。
    責務が混ざりやすいので、差し替えやテストが大変になる。

Python版では、このごちゃ混ぜを分離しているイメージです。

---

## 🛡️ このクラスの鉄則

> 愚直であれ (Be Dumb)

* View は「受付」と「掲示板」に徹する。考えない。
* 表示する文言そのものは Presenter が用意したものを使う。
* ビジネスロジックは Controller → Use Case に任せる。
* そうすることで、**UI技術だけを安心して差し替えられる**（コンソール → GUI → Web）。

これが、クリーンアーキテクチャにおける View の立場です。
