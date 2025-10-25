# 11 Main

# 🚀 Main
### `main.py`


## 🧭 このファイルの役割

`main.py` は、アプリ全体を「起動可能な形」に束ねる場所です。

ここでは、アプリを動かすために必要な部品たちをすべて生成し、正しい順番でお互いを接続します。

登場する主なオブジェクトは以下の通りです：

* `InMemoryBookRepository` / `InMemoryMemberRepository` / `InMemoryLoanRepository`
  永続化（Repositoryインターフェースの具体実装：データの保存先）。技術側の担当。
* `CheckOutBookUseCase`
  ビジネスフローの中心（「本を貸し出す」という業務シナリオの実行者）。
* `CheckOutBookPresenter`
  Use Case の出力結果を人間が読めるメッセージに整形し、ViewModelに反映する翻訳者。
* `CheckOutBookController`
  View から渡された入力値（book_idなど）を Use Case が受け取れる形（InputData）にして引き渡す交通整理員。
* `ConsoleView`
  ユーザーと直接対話する層。入力を受け取り、結果を表示する最前線。
* `BookViewModel`
  Presenter と View が共有する、「画面に今なにを表示すべきか」を保持する状態オブジェクト。

この1ファイルで、ユーザーの入力が `Entity` まで届き、結果がまた画面に戻ってくるループが成立します。

---

## 🔁 全体の依存の流れ（おさらい）

ユーザーの「貸出したい」という操作が結果表示まで届く流れはこうなります：

1. View（`ConsoleView`）がユーザーから `book_id` と `member_id` の入力を受け取る
2. Controller（`CheckOutBookController`）がその値を `CheckOutBookInputData` に詰めて、Use Case をコールする
3. Use Case（`CheckOutBookUseCase`）が

   * `BookRepository` から本を取得
   * `MemberRepository` から会員を取得
   * `Book` Entity に `check_out()` を実行させ、状態を「貸出中」にする
   * 新しい `Loan` Entity を作る
   * `BookRepository` / `LoanRepository` に保存（永続化）を依頼
   * 結果を `CheckOutBookOutputData` にまとめ、Presenter を呼ぶ
4. Presenter（`CheckOutBookPresenter`）が

   * 返却期限などを「人間が読むメッセージ」に整形
   * `BookViewModel` を更新
5. View（`ConsoleView`）が `BookViewModel` の内容をコンソールに表示する

この一連のパイプラインを、`main.py` が配線してスタートさせます。

---

## 📁 全体の最終フォルダ構成（ここまでで完成させたい形）

```text
clean_architecture_library/
├─ core/
│   ├─ domain/
│   │   ├─ book.py                  # 01 Entities（Book, BookStatus）
│   │   ├─ member.py                # 01 Entities（Member）
│   │   ├─ loan.py                  # 01 Entities（Loan）
│   │   └─ repository.py            # 02 Repository Interface
│   │
│   └─ usecase/
│       ├─ boundary/
│       │   ├─ dto.py               # 03 Data Structures (InputData / OutputData / ViewModel)
│       │   ├─ input_boundary.py    # 04 InputBoundary（Controllerが呼ぶ）
│       │   └─ output_boundary.py   # 05 OutputBoundary（Presenterが実装）
│       │
│       └─ interactor/
│           └─ check_out_book.py    # 06 Use Case (CheckOutBookUseCase)
│
├─ interface_adapters/
│   ├─ data_access/
│   │   └─ in_memory_repositories.py    # 07 Data Access（InMemoryBookRepositoryなど）
│   ├─ presenters/
│   │   └─ checkout_presenter.py        # 08 Presenter (CheckOutBookPresenter)
│   ├─ controllers/
│   │   └─ checkout_controller.py       # 09 Controller (CheckOutBookController)
│   └─ views/
│       └─ view_console.py              # 10 View (ConsoleView)
│
└─ main.py                              # 11 Main（Composition Root）
```

* `core/` はビジネス側（内側）
* `interface_adapters/` は技術側（外側）
* `main.py` は最外層の「配線係」

---

## 🔍 ソースコード（main.py）

組み立ての順番には意味があります。

* ViewModel は Presenter と View の間で共有される状態なので最初に作る
* Repository は Use Case に注入されるので先に作る
* Presenter は Use Case から呼ばれるので先に作る
* Use Case は Controller から呼ばれるのでそのあと
* Controller は View から呼ばれるのでそのあと
* 最後に View を作って、`run()` でアプリを開始する

```python
# --------------------------------------------------------------------
# File: main.py
# Layer: Composition Root（アプリの組み立てと起動）
#
# 目的:
#   - 図書館アプリ全コンポーネントを生成し、
#     正しい依存関係でつなげて、アプリを動かす。
#
# 注意:
#   - ここは「配線する場所」。業務ロジックは書かないこと。
#   - main.py は最外層なので、内側(core)にも外側(interface_adapters)にも
#     依存してよい。
# --------------------------------------------------------------------

from core.usecase.boundary.dto import BookViewModel
from core.usecase.interactor.check_out_book import CheckOutBookUseCase

from interface_adapters.data_access.in_memory_repositories import (
    InMemoryBookRepository,
    InMemoryMemberRepository,
    InMemoryLoanRepository,
)
from interface_adapters.presenters.checkout_presenter import CheckOutBookPresenter
from interface_adapters.controllers.checkout_controller import CheckOutBookController
from interface_adapters.views.view_console import ConsoleView


def main():
    # 1. ViewModelを作成
    #    - Presenterが更新し、Viewが参照する「表示用の状態オブジェクト」
    view_model = BookViewModel(display_text="")  # 初期表示は空メッセージ

    # 2. Repository（Data Access: 永続化の具体実装）を用意
    #    - ここではインメモリ(fake DB)版を使う
    book_repo = InMemoryBookRepository()
    member_repo = InMemoryMemberRepository()
    loan_repo = InMemoryLoanRepository()

    # 3. Presenterを作成
    #    - OutputBoundaryを実装しており、
    #      Use Case からの結果を ViewModel に反映する
    presenter = CheckOutBookPresenter(view_model)

    # 4. Use Case（Interactor）を作成
    #    - ビジネスフローの司書役
    #    - Repository群とPresenterに依存する
    use_case = CheckOutBookUseCase(
        presenter=presenter,
        book_repository=book_repo,
        member_repository=member_repo,
        loan_repository=loan_repo,
    )

    # 5. Controllerを作成
    #    - Viewから呼ばれ、Use Case を起動する
    controller = CheckOutBookController(use_case)

    # 6. Viewを作成
    #    - 人間と会話する最前線
    view = ConsoleView(controller, view_model)

    # 7. アプリ開始
    view.run()


if __name__ == "__main__":
    main()
```

---

## 🌱 実際に動かすとこうなるイメージ

1. アプリが起動すると `view.run()` が呼ばれます
2. コンソールで聞かれます：
   「貸し出す本のIDを入力してください: 」
   「貸し出す会員のIDを入力してください: 」
3. あなたが `1` と `2` を入力すると…

   * Controller → Use Case → Repository の順に処理が流れ、
   * 本が「貸出中」に更新され、
   * 新しい Loan が登録され、
   * Presenter が返却期限つきのメッセージを組み立てて ViewModel に書きこみます
4. View がその ViewModel をそのまま表示します👇

例：

```text
--- 処理結果 ---
貸出処理が完了しました。
  書籍: 『クリーンアーキテクチャ』
  会員: Alice様
  返却期限: 2025年10月27日
----------------
```

（※日付は `Loan` Entity が「貸出日＋14日」で計算するビジネスルールでしたね 📅）

---

## 🤔 なぜ `main.py` にビジネスロジックを書いてはいけないの？

`main.py` は「配線する場所」、いわゆる Composition Root です。
ここに「貸出できるか判定する処理」などを書いてしまうと、せっかく分離したレイヤーが一気にごちゃまぜになります。

* `main.py` はいちばん外側の層だから、内側（core）も外側（interface_adapters）も import してOK。
* でも逆方向はNG。
  たとえば Use Case（core/usecase/...）が `main.py` を import したら設計崩壊です。

この「外側が内側を束ねる／内側は外側を知らない」という非対称性こそが、クリーンアーキテクチャの強みです。

---

## 🛡 この章の鉄則

> mainは「配線の場」。ロジックは書かない。
> あなたのアプリはここで初めて「動く」。

* `main.py` は各層のオブジェクト同士を接続するだけ。
* アプリの依存関係をここで初めて具現化する。
* ビジネスルールは core 側に閉じ込める。ここに持ち込まない。

---

## 🎉 ここまででできたこと

* クリーンアーキテクチャの全レイヤーが、図書館の貸出ドメインに沿ってフルに揃った 📚
* それぞれの責務が分離されたまま、`python main.py` で実際に動くところまで到達した
* Webフレームワークも本物DBもまだ使っていないのに、
  「会員が本を借りる」というリアルな業務フローをすでに表現できている

次のステップでは、この配線を保ったまま、UIをFastAPIに差し替えたり、ストレージをPostgreSQLに差し替えたりしていけます。
それができるのは、ここまでのレイヤー分離が正しく守られているからです 👍
