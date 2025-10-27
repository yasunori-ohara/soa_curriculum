# 05 Output Boundary

# 📤 Output Boundary
### `core/usecase/boundary/output_boundary.py`

---

## 🎯 このファイルの役割

`OutputBoundary` は、**Use Case（業務ロジック）と Presenter（表示用変換ロジック）をつなぐ「出口の契約」**です。

* Use Case はビジネスロジックを実行したあと、その結果をどこかに見せたいはずですよね。
* でも Use Case は「どう見せるべきか」（CLIでprintする？GUIでポップアップする？WebでHTML？）なんて知りたくありません。知ってしまうとUI技術に縛られ、差し替えが難しくなるからです。

そこで Use Case は UI を直接触らずに、**抽象的な「出口インターフェース」だけを呼び出します。**
その抽象インターフェースが `OutputBoundary` です。

> つまり、OutputBoundary は「結果出したいんで、あとはよろしく」という引き渡し口です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 📁 ファイルの配置

`OutputBoundary` は Use Case 層の中に置かれます。Presenter はこのインターフェースを実装します。

```text
├─ core/
│   ├─ domain/
│   │   ├─ book.py                     # <E> Book Entity
│   │   ├─ member.py                   # <E> Member Entity
│   │   ├─ loan.py                     # <E> Loan Entity
│   │   └─ repository.py               # <I> Repositoryインターフェース
│   │
│   └─ usecase/
│       ├─ boundary/
│       │   ├─ input_boundary.py       # <I> Controller→Use Case の入口契約
│       │   ├─ output_boundary.py      # <I> ← このページで説明
│       │   └─ dto.py                  # <DS> InputData / OutputData / ViewModel DTO
│       │
│       └─ interactor/
│           └─ check_out_book.py       # <UC> 「本を貸し出す」ユースケース実装
│
├─ interface_adapters/
│   ├─ presenters/
│   │   └─ checkout_presenter.py       # Presenter（OutputBoundaryを実装する）
│   ├─ controllers/
│   │   └─ checkout_controller.py      # Controller（InputBoundaryを呼ぶ）
│   └─ views/
│       └─ ...                         # CLI / GUI / Web のView
```

依存方向はこうなります：

* Use Case（内側）は `core/usecase/boundary/output_boundary.py` だけを知っている
* Presenter（外側）はその抽象インターフェースを**実装**する
* Use Case は Presenter の「中身」や「UI技術」をまったく知らない

👉 つまり、**内側が抽象を定義し、外側がそれを実装する**という、クリーンアーキテクチャの黄金パターンです。

---

## 🔍 ソースコード（正式版）

```python
# --------------------------------------------------------------------
# File: core/usecase/boundary/output_boundary.py
# Layer: Use Case Boundary（Use Case → Presenter の出力契約）
#
# 目的:
#   - Use Case が「処理結果はこれです。あとは表示用に整えてください」と
#     Presenter に渡すための公式な窓口（インターフェース）を定義する。
#
# C言語の感覚に近い説明:
#   - ここは「コールバック関数の型宣言」に近い。
#     Use Case はこの型どおりに呼び出すだけで、
#     何が起こるか（CLI出力？Webレスポンス？GUI更新？）は知らない。
# --------------------------------------------------------------------

from abc import ABC, abstractmethod
from core.usecase.boundary.dto import CheckOutBookOutputData


class CheckOutBookOutputBoundary(ABC):
    """
    出力境界（OutputBoundary）
    - Use Case が結果を通知するための契約（インターフェース）。
    - Presenter がこれを実装する。
    
    呼び出し側:
        Use Case（例: CheckOutBookUseCase）

    実装する側:
        Presenter（例: CheckOutBookPresenter）

    ポイント:
        Use Case は「どう見せるか」をまったく知らない。
        ただ、この present() を呼ぶだけ。
    """

    @abstractmethod
    def present(self, output_data: CheckOutBookOutputData) -> None:
        """
        Use Case が処理結果を Presenter に渡すためのメソッド。

        Parameters
        ----------
        output_data : CheckOutBookOutputData
            業務ロジックの結果（本のタイトル・会員名・返却期限など）
            - これはまだ「生のビジネス情報」
            - 人間に見せるための整形はここではしない
              （それはPresenterの責務）
        """
        raise NotImplementedError
```

### ここで登場する型：`CheckOutBookOutputData`

* `CheckOutBookOutputData` は `core/usecase/boundary/dto.py` に定義される DTO（単なるデータの箱）です。
* Use Case はこれを組み立てて `present()` に渡します。
* Presenter はそれを受け取り、UIで使いやすい `ViewModel` を組み立てます（例：人間向けのメッセージ文字列）。

---

## 🧠 なぜ OutputBoundary が必要なの？

短くいうと：**Use Case が UI 技術を知らずに済ませるため**です。

もし OutputBoundary がなかったら、Use Case はこうなってしまいます：

* `print()` を直接呼ぶ
* Flask の `render_template()` を直接呼ぶ
* GUI のラベルを書き換える処理を直接持つ

これをやると「ビジネスロジック（貸出処理）」が「表示方法（CLI/Web/GUI）」に縛られてしまいます。
UIを差し替えようとした瞬間に、Use Case を書き換える羽目になります。つまり壊れます。

OutputBoundary を使えば、Use Case はこう言えます：

> 「はい結果できました。あとは頼んだ（present() 呼んだのであとは外側よろしく）」
> （UIのことは何も知らない）

この分離によって、たとえば CLI から Web UI に変えても、Use Case のコードは一文字も変えずに済みます。

---

## 🧪 Presenter 側のテストイメージ

`OutputBoundary` 自体にはロジックがないので、それ単体をテストすることは基本しません。
代わりに「これを実装する Presenter が、期待どおりに ViewModel を更新できるか」をテストします。

たとえば Presenter はこんな感じになります（イメージ）：

```python
# interface_adapters/presenters/checkout_presenter.py

from core.usecase.boundary.output_boundary import CheckOutBookOutputBoundary
from core.usecase.boundary.dto import CheckOutBookOutputData, BookViewModel

class CheckOutBookPresenter(CheckOutBookOutputBoundary):
    """
    Use Case から渡された生の業務データを、
    View がそのまま使える形（ViewModel）に整形する。
    """

    def __init__(self, view_model: BookViewModel):
        self._view_model = view_model  # View と共有する描画用データ置き場

    def present(self, output_data: CheckOutBookOutputData) -> None:
        # 例: due_date（日付型）を、人間向けの文字列に整形する責務は Presenter 側にある
        message = (
            f"📕『{output_data.book_title}』を {output_data.member_name} さんに貸し出しました。\n"
            f"返却期限: {output_data.due_date:%Y-%m-%d}"
        )

        # View が後で参照する表示用データを更新
        self._view_model.display_text = message
```

テストではこの Presenter を直接呼んで、`view_model.display_text` が想定どおりになったかを確認すればOKです。

👉 ここで注目してほしいのは、Presenter が `CheckOutBookOutputBoundary` を「実装」している点。
つまり **OutputBoundary は Presenter の “インターフェースの親玉” になっている**ということです。

---

## 🧭 依存方向まとめ（ここが一番大事）

* `check_out_book.py`（Use Case / Interactor）は
  `CheckOutBookOutputBoundary` に依存する（＝この抽象を呼ぶ）

* `checkout_presenter.py`（Presenter, UI側）は
  `CheckOutBookOutputBoundary` を実装する（＝具象で応える）

これ、めちゃくちゃ重要です。
これによって**内側（Use Case）が外側（Presenter/UI）を支配**します。
本来なら UI が「内側よりえらそう」に見えがちですが、ここでは逆です。
ビジネスロジックこそが権威になるように依存方向を設計している、ということなんです。

---

## 🛡 鉄則

> 見せ方は外側に任せろ。内側は結果だけ語れ。
> (Tell the result, let others decide how to show it.)

* Use Case は「何が起こったか」だけをまとめて OutputBoundary に渡す
* Presenter は「どう見せるか」の責任を単独で引き受ける
* UI の技術（CLI / GUI / Web / API…）は Presenter と View 側に閉じ込められる
* だから UI を差し替えても、Use Case は一切変更不要になる

これが OutputBoundary の存在理由であり、クリーンアーキテクチャのうまみが最も実感できるポイントの一つです。
