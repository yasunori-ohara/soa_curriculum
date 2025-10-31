# 07 Presenter

# 🎨 Presenter
### `interface_adapters/presenters/checkout_presenter.py` - `CheckOutBookPresenter`


## 🎯 このクラスの役割

`CheckOutBookPresenter` は、**Use Case の実行結果を、人間が読める表示用テキストに整形し、View が参照する ViewModel に反映する担当者**です。

もう少し分解すると：

* Use Case は、ビジネス的な事実だけを返します
  例：「この本タイトルは 'クリーンアーキテクチャ'」「この会員は '田中太郎'」「返却期限は日付型で 2025-10-27」。

* Presenter は、それをユーザーに伝える文章に変換します
  例：「貸出処理が完了しました。返却期限は2025年10月27日です。」

* View は、その文章をそのまま表示するだけになります（printする / ラベルに貼る / HTMLに差し込む など）。

この役割分担によって、

* Use Case は UI 表現のことを一切考えなくていい
* View はビジネスロジックを一切考えなくていい
  というきれいな分離が成立します。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

---

## 📁 フォルダの配置（正式版）

Presenter は **Interface Adapters 層**に属します。
依存方向は「外側 → 内側」で、内側の層は Presenter を知りません。

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
│       │   ├─ dto.py                # <DS> DTO / ViewModel
│       │   ├─ input_boundary.py     # <I> InputBoundary(Controllerが呼ぶ)
│       │   └─ output_boundary.py    # <I> OutputBoundary(Presenterが実装)
│       │
│       └─ interactor/
│           └─ check_out_book.py     # <UC> 本を貸し出すユースケース本体
│
└─ interface_adapters/
    ├─ presenters/
    │   └─ checkout_presenter.py     # ← この章の主役 (Presenter)
    ├─ controllers/
    │   └─ checkout_controller.py    # Controller (Boundary呼び出し)
    └─ views/
        ├─ view_console.py           # CLI View
        ├─ view_gui.py               # GUI View
        └─ view_web.py               # Web View
```

ポイント：

* Presenter は `core` には置きません。`interface_adapters` 側に置きます。
* Presenter は `core.usecase.boundary.output_boundary` で定義された契約（OutputBoundary）を実装します。
* Presenter は `core.usecase.boundary.dto` に定義された `CheckOutBookOutputData` と `BookViewModel` を扱います。

---

## 💻 ソースコード（正式版）

```python
# --------------------------------------------------------------------
# File: interface_adapters/presenters/checkout_presenter.py
# Layer: Interface Adapters（Presenter）
#
# 責務:
#   - Use Case から受け取った OutputData を
#     View がそのまま表示できる ViewModel に変換する。
#
# 依存:
#   - <I> CheckOutBookOutputBoundary（core.usecase.boundary.output_boundary）
#   - <DS> CheckOutBookOutputData / BookViewModel（core.usecase.boundary.dto）
#
# 同心円図での位置:
#   - Interface Adapters 層
#   - 内側(core)には依存するが、内側はこのクラスを知らない一方向依存
# --------------------------------------------------------------------

from core.usecase.boundary.output_boundary import CheckOutBookOutputBoundary
from core.usecase.boundary.dto import (
    CheckOutBookOutputData,
    BookViewModel,
)


class CheckOutBookPresenter(CheckOutBookOutputBoundary):
    """
    Use Case からの結果(OutputData)を、画面表示用のViewModelに変換する「翻訳担当」。

    - OutputBoundary(抽象インターフェース)を実装する具体クラス。
    - ViewModelの中身を書き換えることで View に「こう表示してね」と伝える。
    """

    def __init__(self, view_model: BookViewModel):
        """
        View と共有する ViewModel を受け取る。

        ViewModelは View 側でも参照される同じインスタンスを渡す想定。
        Presenterはそれを更新するだけで、Viewは render() 時に読むだけ。
        """
        self._view_model = view_model

    def present(self, output_data: CheckOutBookOutputData) -> None:
        """
        OutputBoundary の契約に従った実装。

        Use Case が計算した「事実」(OutputData)をもらい、
        ユーザーが読める日本語のメッセージ文 (ViewModel.display_text) に整形する。
        """

        # 1. 日付を人間向けの文字列へ
        due_date_str = output_data.due_date.strftime("%Y年%m月%d日")

        # 2. 表示用メッセージ組み立て
        #    ※ この「文言をどう見せたいか」は Presenter の責務
        display_text = (
            "貸出処理が完了しました。\n"
            f"  書籍: 『{output_data.book_title}』\n"
            f"  会員: {output_data.member_name} 様\n"
            f"  返却期限: {due_date_str}"
        )

        # 3. ViewModel を更新
        #    View はこの値をそのまま表示するだけでよい
        # 注意:
        #   - BookViewModel が NamedTuple (不変) なら再生成して置き換える
        #   - BookViewModel が単なる可変オブジェクトならフィールドを書き換える
        try:
            # 可変オブジェクト想定パターン
            self._view_model.display_text = display_text
        except AttributeError:
            # NamedTupleなどイミュータブルな場合は再束縛
            self._view_model = BookViewModel(display_text=display_text)

    # Presenterから現在のViewModelを取り出したい場合に使えるヘルパ
    # （Viewが直接 view_model を持っているなら不要）
    def get_view_model(self) -> BookViewModel:
        return self._view_model
```

### 重要な点

* `CheckOutBookPresenter` は **`CheckOutBookOutputBoundary` を実装する具体クラス**です。
  → Use Case は「OutputBoundary に present(output_data) しているだけ」。Presenterの存在（実装クラス名）は知らないまま動ける、ということ。

* `CheckOutBookOutputData` はビジネス寄りの“事実”。日付も `datetime.date` のまま等身大。

* `BookViewModel` はユーザー寄りに整形された“見せ方”。日付も整形済みテキストの一部。

この「事実」→「見せ方」の変換が Presenter の本業です。

---

## ✅ このファイルにあるべき処理

⭕️ **含めるべき処理の例:**

* 日付や数値を、人が読める文字列にフォーマットする
* 文言・敬称・装飾を含めた表示用テキストを組み立てる
* ViewModelを更新する

❌ **含めてはいけない処理の例:**

* ビジネスルールの判断（「貸し出せるか？」など）→ Use Case の仕事
* 画面への直接出力（print / Tkinterの `.config()` / Flaskのレスポンス返却など）→ View の仕事
* データベースへの保存や検索 → Repository/DataAccess の仕事

Presenterは「翻訳と整形」に徹します。

---

## 🧪 ユニットテスト例

Presenterは単体でテストできます。
やりたいことは一つだけ：`present()` を呼んだら ViewModel の `display_text` が期待どおりになるか？

```python
# tests/interface_adapters/presenters/test_checkout_presenter.py

from datetime import date
from core.usecase.boundary.dto import (
    CheckOutBookOutputData,
    BookViewModel,
)
from interface_adapters.presenters.checkout_presenter import CheckOutBookPresenter


def test_presenterは_OutputData_を_人間向けメッセージに変換して_ViewModelを更新する():
    # Arrange: 空のViewModelを用意
    # BookViewModel が NamedTuple の場合は一時的にダミー値を入れておく
    initial_vm = BookViewModel(display_text="")

    presenter = CheckOutBookPresenter(initial_vm)

    output_data = CheckOutBookOutputData(
        book_title="クリーンアーキテクチャ",
        member_name="田中太郎",
        due_date=date(2025, 10, 27),
    )

    # Act
    presenter.present(output_data)

    # Assert
    vm_after = presenter.get_view_model()

    expected_text = (
        "貸出処理が完了しました。\n"
        "  書籍: 『クリーンアーキテクチャ』\n"
        "  会員: 田中太郎 様\n"
        "  返却期限: 2025年10月27日"
    )

    assert vm_after.display_text == expected_text
```

テストの狙いはシンプルです：

* 日付が `"2025年10月27日"` という書式で埋め込まれているか？
* 敬称（"様"）や文言が期待通りか？
* ViewModel がちゃんと更新されるか？

Use Case も DB も View も起動しないので、超速・超安定のテストになります。

---

## 🐍 Python vs C言語でのイメージ

* Python版（いまやってるやつ）

  * Presenter はクラス
  * ViewModel はオブジェクト
  * Presenter が ViewModel の中身を書き換えることで View に「表示用データできたよ」と伝える

* C言語でやるなら

  * `format_checkout_message(output_data, char *buffer)` のような関数で文字列を組み立てて、
    その `buffer` を UI 側に渡すイメージ
  * 役割は似ているけど、「責務として分ける」という設計の話を C で徹底するのはけっこう難しい

---

## 🛡 鉄則

> 表示のために翻訳せよ。ただしビジネスを決めるな。

* Presenter は「どう見せるか」を決めるところ。
* 「何が正しいビジネス結果なのか」は Use Case が決める。
* 「どう実際にユーザーに見せるか（コンソール？Web画面？GUIラベル？）」は View が決める。

この三者分離のおかげで、

* UIをCLI→GUI→Webに変えても Use Case はノータッチ
* 文言や日付フォーマットを変えても Use Case はノータッチ
* ビジネスルールが変わっても Presenter / View は最小限の変更で済む

という、変更にめっぽう強いアプリになります。
