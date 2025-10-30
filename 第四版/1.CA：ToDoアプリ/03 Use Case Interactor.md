# 03 Use Case Interactor

# 🗄 Use Case（Interactor）

### `core/usecase/interactor/create_todo.py`

## 🧭 このクラスの役割

`TodoUseCase` は、このアプリケーションにおける**具体的なシナリオ（ユースケース）を1つ担当するクラス**です。

* Entityがビジネスの「材料」だとしたら、
* Use Caseはその材料を使って、ユーザーの目的（＝「やりたいこと」）を達成するための「レシピ」であり「シェフ」です。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

このクラスは、「TODOを追加する」というユースケースを実現するための一連の手順を定義し、`Todo`エンティティを扱い、Repositoryに保存を依頼し、その結果をPresenterに渡すまでを取りまとめます。
この「調整役」「指揮者」の役割を担うのがインタラクター（Interactor）です。

> ✅ ここに入るのは「アプリケーション固有のビジネスルール」です。
> ドメインの普遍的なルールは Entity 側に置きます。

---

## 🧑‍🍳 Use Case とは？

Use Case層は、アプリケーション固有のビジネスルールを実装する層です。

* Entities がビジネスの**名詞（Todo, Book, Loan...）と物理法則**なら、
* Use Caseはそれらを「どう使ってどんな業務を達成するか」という**動詞・手順書**です。

ユーザーの目的（「TODOを追加したい」「本を貸し出したい」など）を1つずつクラスとして表します。
この「1ユースケース＝1クラス」パターンは、テストしやすさ・変更のしやすさに直結します。

---

## ⚙️ このクラスにあるべき処理

✅ **含めるべき処理の例**

* エンティティの生成・状態遷移を指揮する
* アプリケーション固有の前処理・後処理（トリムや権限制御など）
* Repository・Presenter といった周辺コンポーネントを「抽象インターフェース越しに」呼び出す

❌ **含めてはいけない処理の例**

* 画面表示ロジック（それは Presenter の責務）
* DB固有の処理やSQL（それは infrastructure の責務）
* Web/CLIフレームワークに依存する処理（それは View/Controller の責務）
* 具体クラスへの依存（必ずインターフェースに依存する）

---

## 📂 ユースケース関連ファイルの配置

このユースケース（「TODOを追加する」）に関係するファイルは、以下のようにレイヤーごとに整理されます。

```text
clean_architecture_todo/
├─ core/
│   ├─ domain/
│   │   ├─ todo.py                # <E> Entity (Todo)
│   │   ├─ repository.py          # <I> TodoRepository 抽象 (Data Access Interface)
│   │   └─ errors.py              # ドメイン固有エラー（必要なら）
│   │
│   └─ usecase/
│       ├─ interactor/
│       │   └─ create_todo.py     # ← いま説明しているユースケース本体（Interactor）
│       │
│       └─ boundary/
│           ├─ dto.py             # <DS> Data Structures (TodoInputData, TodoOutputData, TodoViewModel)
│           ├─ input_boundary.py  # <I> TodoInputBoundary（Controllerが呼ぶ"入口"契約）
│           └─ output_boundary.py # <I> TodoOutputBoundary（Presenterが実装する"出口"契約）
│

```

この図で押さえてほしいことは1つだけです：

> UseCase（`create_todo.py`）は、
>
> * Presenterにも
> * Repositoryにも
>   具体型ではなく **インターフェース** だけで依存している。

これがテスト容易性と差し替え自由度の源です。

---

## 🔍 ソースコード（`core/usecase/interactor/create_todo.py`）

```python
# --------------------------------------------------------------------
# File: core/usecase/interactor/create_todo.py
# Layer: Use Case（アプリケーション固有のビジネスルール）
#
# [クラス図] UseCaseInteractor（Create）
# [同心円] UseCase層
#
# 目的:
#   - 「TODOを追加する」というユースケースの処理手順を定義する。
#   - Entity（材料）を生成・操作し、
#     永続化（保存）と表示準備（Presenter）をオーケストレーションする。
#
# C言語にたとえると:
#   - このクラスは「手続きのメイン関数」に近い。
#   - RepositoryやPresenterは「関数ポインタ」として外から渡されるイメージ。
#     （依存性注入 = DI）
# --------------------------------------------------------------------

from core.domain.todo import Todo
from core.domain.repository import TodoRepository           # <I> Data access 抽象
from core.usecase.boundary.input_boundary import TodoInputBoundary
from core.usecase.boundary.output_boundary import TodoOutputBoundary
from core.usecase.boundary.dto import TodoInputData, TodoOutputData


class TodoUseCase(TodoInputBoundary):
    """
    「TODOを追加する」というユースケースを実装するクラス。

    - TodoInputBoundary（入力境界インターフェース）を実装することで、
      Controllerなどの呼び出し側は
      「TodoUseCaseがこの契約どおりに動く」と信じて呼び出せる。
    """

    def __init__(self, presenter: TodoOutputBoundary, repository: TodoRepository):
        """
        [依存性の注入 / Dependency Injection]

        このクラスが動くために必要な依存物を、
        具体クラスではなく抽象インターフェースとして受け取る。

        - presenter: TodoOutputBoundary
            UseCaseの結果をどうユーザー向けに整形するかはPresenterの責務。
            UseCaseは「結果を渡す」ことだけ知ればいい。

        - repository: TodoRepository
            データをどこに保存するか（メモリ？DB？ファイル？）は
            UseCaseの関心ではない。Repositoryインターフェース越しに依頼する。
        """
        self._presenter = presenter
        self._repository = repository

    def execute(self, input_data: TodoInputData) -> None:
        """
        [ユースケースの手順そのもの]

        1. 入力データを受け取る
        2. エンティティを生成する
        3. Repositoryに「保存してくれ」と依頼する
        4. Presenterに結果を渡す

        この流れこそが「TODOを1つ登録する」というビジネスシナリオ。
        """

        # --- Step 1: 入力の正規化（アプリ固有の前処理） ---
        normalized_title = input_data.title.strip()
        # ※ 空文字チェックなどの厳しいバリデーション自体は
        #    Todoエンティティのコンストラクタに任せる。

        # --- Step 2: エンティティの生成 ---
        # IDは本来ユニークであるべきだが、
        # その採番戦略はインフラ側（Repository実装）の責務にする。
        # ここでは「未採番」を意味する仮IDとして id=0 を渡す。
        new_todo = Todo(id=0, title=normalized_title)

        # --- Step 3: 永続化の依頼 ---
        # Repositoryは保存時に正式なIDを付与して返す想定。
        saved_todo = self._repository.save(new_todo)

        # --- Step 4: 出力データを用意 ---
        # Presenterに渡すための「素の結果オブジェクト」（DTO）を組み立てる。
        output_data = TodoOutputData(
            id=saved_todo.id,
            title=saved_todo.title,
        )

        # --- Step 5: Presenterに引き渡し ---
        # View向けのメッセージ整形やViewModel更新はPresenterの仕事なので、
        # UseCaseはただ「これが結果だよ」と渡すだけでOK。
        self._presenter.present(output_data)
```

---

## 🧪 ユニットテストはこの段階ですでに書けます

ここがクリーンアーキテクチャのうまみです。
UseCaseは Presenter と Repository に「抽象として」しか依存していないので、テストでは簡単なフェイクを注入できます。

以下はユースケースの単体テスト例です。

📄 `tests/unit/test_create_todo_usecase.py`

```python
import unittest

from core.domain.todo import Todo
from core.domain.repository import TodoRepository
from core.usecase.boundary.dto import TodoInputData, TodoOutputData
from core.usecase.boundary.output_boundary import TodoOutputBoundary
from core.usecase.interactor.create_todo import TodoUseCase


# Presenterのフェイク（実際はViewModelを更新する役割だが、ここでは記録だけする）
class FakePresenter(TodoOutputBoundary):
    def __init__(self):
        self.last_output: TodoOutputData | None = None

    def present(self, output_data: TodoOutputData) -> None:
        self.last_output = output_data


# Repositoryのフェイク（メモリ上に積むだけの簡易版）
class InMemoryTodoRepository(TodoRepository):
    def __init__(self):
        self._store: list[Todo] = []
        self._next_id = 1

    def save(self, todo: Todo) -> Todo:
        todo.id = self._next_id
        self._next_id += 1
        self._store.append(todo)
        return todo

    def find_all(self) -> list[Todo]:
        # ユースケース側で呼んでないが、実用時に便利なので用意しておく
        return list(self._store)


class TestTodoUseCase(unittest.TestCase):
    def setUp(self):
        self.presenter = FakePresenter()
        self.repo = InMemoryTodoRepository()
        self.usecase = TodoUseCase(self.presenter, self.repo)

    def test_add_first_todo(self):
        # Arrange
        input_data = TodoInputData(title="  最初のタスク  ")

        # Act
        self.usecase.execute(input_data)

        # Assert: Presenterに通知されたか？
        self.assertIsNotNone(self.presenter.last_output)
        self.assertEqual(self.presenter.last_output.title, "最初のタスク")
        self.assertEqual(self.presenter.last_output.id, 1)

        # Assert: Repositoryに保存されたか？
        all_todos = self.repo.find_all()
        self.assertEqual(len(all_todos), 1)
        self.assertEqual(all_todos[0].title, "最初のタスク")
        self.assertEqual(all_todos[0].id, 1)
        self.assertFalse(all_todos[0].completed)

    def test_add_second_todo_increments_id(self):
        # 先に1件保存（ID=1）
        self.repository.save(Todo(id=0, title="既存タスク"))

        # 次を追加（ID=2になるはず）
        input_data = TodoInputData(title="次のタスク")
        self.use_case.execute(input_data)

        self.assertEqual(self.presenter.last_output.id, 2)

    def test_empty_title_raises_value_error_from_entity(self):
        # Arrange
        input_data = TodoInputData(title="   ")

        # Act & Assert
        with self.assertRaises(ValueError):
            self.usecase.execute(input_data)


if __name__ == "__main__":
    unittest.main()
```

意図としてはこうです：

* ✅ UseCaseがRepositoryに保存を依頼していること
* ✅ ID採番はRepository側に任せていること
* ✅ Presenterに正しい結果（DTO）が渡ること
* ✅ 空のタイトルなどドメインルール違反はEntity側で例外になること

どれも UI や DB を一切立ち上げずにテストできます。これが本当に大きい。

---

## 🛡 このクラスの鉄則

> ビジネスロジックを指揮せよ。ただし詳細には関わるな。

* UseCaseは「何をするか（What）」だけを書く。

  * 何をチェックするか
  * いつ保存するか
  * どんなデータを返すか
* 「どうやるか（How）」は外側の責務。

  * どう保存するか → Repository実装（infrastructure）
  * どう表示するか → Presenter / View
  * どう呼ばれるか → Controller / View

---

## 📝 ちょっと大事な順番の話

このアーキテクチャでは、**内側の層が外側に「こういう形で呼んでください」と宣言します。**

1. UseCaseが「このユースケースは `execute(input_data)` で呼んでください」と決める
2. その形を `TodoInputBoundary` / `TodoInputData` としてコード化する
3. Controller はその契約どおりに呼ぶ
4. UseCaseは、結果を `TodoOutputBoundary.present(output_data)` で外に返す
5. Presenter はそれを受け取って ViewModel を更新する

つまり、
**主導権は常に内側（UseCase）にあります。**
外側は合わせに来る側です。これは現実の業務フローの設計責任とよく似ています。

---

## 🔧 実務での追加ポイント（参考）

* 例外（ドメインエラーやバリデーションエラー）をOutputBoundary経由でPresenterに渡し、エラーメッセージをViewModelに詰める、という設計もよく使われます。
* 複数のリポジトリ呼び出しが1トランザクションであるべき場合（例：図書館の「本を貸し出す」処理など）は、UseCase側が「このユースケースは1トランザクションでやってください」と宣言し、インフラ側（Unit of Workなど）にそれを実現させることがあります。

---

次はこのUseCaseに関連する「Data Structures（InputData / OutputData / ViewModel）」と「InputBoundary / OutputBoundary」のページを、`core/usecase/boundary/` 配下の正式な構成に沿ってアップデートしていきます。
