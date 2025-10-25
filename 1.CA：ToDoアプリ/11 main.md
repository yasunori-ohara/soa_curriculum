# 11 Main

# 🚀 Main
### `main.py`

---

## 🧭 このファイルの役割

`main.py` は、アプリ全体を「起動可能な形」に束ねる場所です。

ここでは以下のオブジェクトを生成し、正しく接続します：

* `InMemoryTodoRepository`
  永続化の実装（インフラ層）
* `TodoPresenter`
  UseCaseの出力をViewModelに反映する
* `TodoController`
  Viewからの入力をUseCaseへ橋渡しする
* `ConsoleView`
  入出力の最終フロント（ユーザーと対話）
* `TodoUseCase`
  ビジネスフローの中心（Interactor）
* `TodoViewModel`
  Viewが表示するための状態
* `TodoInputBoundary` / `TodoOutputBoundary`
  （インターフェース。直接インスタンス化はしないが、依存形として重要）

これらが正しくつながると、ユーザーの入力がEntityまで届き、結果がまたViewに戻ります。

---

## 🔁 全体の依存の流れ（おさらい）

ユーザー入力から表示までの流れはこうなります：

1. View（`ConsoleView`）がユーザーからタイトル文字列を受け取る
2. Controller（`TodoController`）が `TodoInputData` を組み立て、UseCaseを呼び出す
3. UseCase（`TodoUseCase`）が

   * `Todo` Entityを作る
   * Repository（`InMemoryTodoRepository`）に保存する
   * 結果を `TodoOutputData` にまとめる
   * OutputBoundary（`TodoOutputBoundary`）を呼ぶ
4. Presenter（`TodoPresenter`）が ViewModel（`TodoViewModel`）を書き換える
5. View が ViewModel の内容を画面に表示する

この一連のパイプラインを、`main.py` で組み上げて実際に回します。

---

## 📁 全体の最終フォルダ構成（ここまでの完成形）

```
clean_architecture_todo/
├─ core/
│   ├─ domain/
│   │   ├─ todo.py                 # 01 Entity
│   │   ├─ repository.py           # 02 Repository Interface
│   │   └─ errors.py               # （必要ならドメイン固有エラー）
│   │
│   └─ usecase/
│       ├─ interactor/
│       │   └─ create_todo.py      # 03 UseCase (TodoUseCase)
│       └─ boundary/
│           ├─ input_boundary.py   # 04 InputBoundary
│           ├─ output_boundary.py  # 05 OutputBoundary
│           └─ dto.py              # 06 Data Structures (TodoInputData等)
│
├─ interface_adapters/
│   ├─ presenters/
│   │   └─ todo_presenter.py       # 07 Presenter
│   ├─ controllers/
│   │   └─ todo_controller.py      # 08 Controller
│   └─ views/
│       └─ view_console.py         # 09 View
│
├─ infrastructure/
│   └─ repositories/
│       └─ in_memory_todo_repository.py  # 10 Data Access実装
│
└─ main.py                         # 11 Main (Composition Root)
```

---

## 🔍 ソースコード（main.py）

ここでは、依存の内側から順にインスタンスを用意していきます。

* ViewModel は Presenter と View の間で共有される「状態」なので、最初に作る
* Repository の実体はインフラ層（InMemory版）を使う
* UseCase は Presenter と Repository に依存するので、それらを渡して生成
* Controller は UseCase（正確には InputBoundary としてのUseCase）に依存する
* View は Controller と ViewModel に依存する

この順番で組み立てます。

```python
# --------------------------------------------------------------------
# File: main.py
# Layer: Composition Root (アプリの組み立てと起動)
#
# 目的:
#   - 全コンポーネントを生成して依存関係を接続し、
#     アプリを実際に動作可能な状態にする。
#
# 注意:
#   - ここは「アプリ起動のための配線場所」なので、
#     ビジネスロジックは書かないこと。
#   - main.py はインフラ層にも依存してよい（一番外側だから）。
# --------------------------------------------------------------------

from core.usecase.boundary.dto import TodoViewModel
from core.usecase.interactor.create_todo import TodoUseCase
from interface_adapters.presenters.todo_presenter import TodoPresenter
from interface_adapters.controllers.todo_controller import TodoController
from interface_adapters.views.view_console import ConsoleView
from infrastructure.repositories.in_memory_todo_repository import InMemoryTodoRepository


def main():
    # 1. ViewModelを作成
    #    - Presenterが更新し、Viewが参照する「表示用の状態オブジェクト」
    view_model = TodoViewModel()

    # 2. Repositoryの具体実装を用意
    #    - ここではインメモリ実装（DBなし）を使う
    repository = InMemoryTodoRepository()

    # 3. Presenterを作成
    #    - PresenterはOutputBoundaryを実装しており、
    #      UseCaseから結果を受け取ってViewModelを書き換える
    presenter = TodoPresenter(view_model)

    # 4. UseCase（Interactor）を作成
    #    - UseCaseは InputBoundary を実装している（＝Controllerから呼び出される）
    #    - UseCaseは OutputBoundary と Repository に依存する
    use_case = TodoUseCase(presenter, repository)

    # 5. Controllerを作成
    #    - ControllerはViewから呼ばれ、UseCaseを起動する
    controller = TodoController(use_case)

    # 6. Viewを作成
    #    - Viewはユーザーとやり取りし、結果を表示する最前線
    view = ConsoleView(controller, view_model)

    # 7. 実行開始
    #    - Viewに制御を渡す
    view.run()


if __name__ == "__main__":
    main()
```

---

## 🌱 実際に動かすとこうなるイメージ

1. アプリが起動すると `view.run()` が呼ばれる
2. ユーザーに
   「追加するTODOのタイトルを入力してください: 」
   と聞かれる
3. 入力した文字列が Controller → UseCase → Repository へ流れる
4. Todoが保存され、PresenterがViewModelを更新する
5. Viewが `render()` を呼んで、例のメッセージを表示する

   * 例：
     `✅ タスク『買い物に行く』 (ID: 1) を追加しました。`

この一連の動きに、WebフレームワークもDBもまだいりません。
それでも「クリーンアーキテクチャとしての依存方向」はすでに完成しています。

---

## 🤔 main.py にロジックを書かないのはなぜ？

`main.py` は「結線する場所」です。
ここにビジネスロジックを混ぜると、依存関係が壊れていきます。

* `main.py` はいちばん外側の層なので、内側のものは全部importしてOK
* 逆は絶対にNG

  * たとえばUseCaseが `main.py` を import してしまうと設計崩壊

この「外側で束ねる」「内側は束ねられるだけ」という関係が、システムの寿命を大きく伸ばします。

---

## 🛡 この章の鉄則

> mainは「配線の場」。ロジックは書かない。
> あなたのアプリはここで初めて「動く」。

* main.pyは、各層のオブジェクト同士を接続する責務だけを持つ
* main.pyはもっとも外側なので、インフラ層を含む全ての実装詳細を知っていてよい
* UseCase・Entity側はmain.pyを一切知らない（ここが本当に大事）

---

## 🎉 ここまででできること

* クリーンアーキテクチャの各レイヤーが、理論だけでなく**実際のPythonファイルとしてすべて揃った**
* それらがどう依存しているか
  （内側を守り、外側を差し替える設計）
  が実感できた
* `python main.py` で動く最低限のTodoアプリの骨格が完成した

