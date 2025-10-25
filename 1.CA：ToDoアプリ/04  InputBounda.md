# 04 Input Boundary

# 🚪 Input Boundary

### `core/usecase/boundary/input_boundary.py`


## 🎯 このファイルの役割

`InputBoundary` は、**Controller（ユーザー入力を受け取る役）と UseCase（ビジネスロジックを実行する役）の「境界（Boundary）」**を定義します。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

もっとラフに言うと、これはこういう場所です：

> 「このユースケースを呼びたいなら、この形で呼んでくださいね」という“入口の契約書”。

* Controller は、この契約（インターフェース）に従って UseCase を呼び出す。
* UseCase（Interactor）は、この契約を実装する。

こうすることで、Controller は UseCase の中身を知らなくていいし、UseCase は Controller の存在を一切知らなくてよくなります。
これはクリーンアーキテクチャにおける依存性逆転（DIP）を実現するための超重要ポイントです。

---

## 📁 ファイルの配置

まず、関連するファイルがどこに置かれるかをもう一度確認しておきます。
特に `InputBoundary` は UseCase層の一部であり、Controller から呼ばれる「入り口の契約」を定義します。

```text
clean_architecture_todo/
├─ core/
│   ├─ domain/
│   │   ├─ todo.py                 # <E> Entity（ビジネスルールの本体）
│   │   ├─ repository.py           # <I> Repository 抽象（データアクセス契約）
│   │   └─ errors.py               # ドメイン固有エラー（必要に応じて）
│   │
│   └─ usecase/
│       ├─ interactor/
│       │   └─ create_todo.py      # UseCase本体（Interactor: TodoUseCase）
│       │
│       └─ boundary/
│           ├─ input_boundary.py   # ← 今見ている「入力境界」(<I>) Controllerが呼ぶ契約
│           ├─ output_boundary.py  # 「出力境界」(<I>) Presenterが実装する契約
│           └─ dto.py              # <DS> InputData / 
```

観察してほしいのは依存方向です：

* `todo_controller.py`（外側）は `input_boundary.py`（内側）を import する
* 逆（UseCase側がControllerをimportする）は絶対に起きない

これが「内側は外側を知らない」という同心円モデルを守るコツです。

---

## 🔍 ソースコード

```python
# --------------------------------------------------------------------
# File: core/usecase/boundary/input_boundary.py
# Layer: Use Case / Boundary（UI → UseCase の“入口”契約）
#
# 目的:
#   - Controller（入力受付）が呼び出すユースケース操作の“公式な入り口”を定義する。
#   - 具体的な実装は UseCase（Interactor）が担当する。
#
# C言語の感覚に近い説明:
#   - ここは「関数ポインタのシグネチャ宣言」に相当する。
#   - Controllerはこの“型”どおりに呼び出すだけで、中身を知らなくてよい。
# --------------------------------------------------------------------

from abc import ABC, abstractmethod
from core.usecase.boundary.dto import TodoInputData


class TodoInputBoundary(ABC):
    """
    InputBoundary（入力境界インターフェース）

    - Controller が呼び出すための契約書。
    - UseCase（Interactor）がこのインターフェースを実装する。
    """

    @abstractmethod
    def execute(self, input_data: TodoInputData) -> None:
        """
        Controller → UseCase の呼び出し口。

        Parameters
        ----------
        input_data : TodoInputData
            Controller が整形した、ユースケースに渡すための公式な入力データ。
        """
        pass
```

ポイント：

* `TodoInputBoundary` は**抽象クラス**であり、実装を持ちません。
* 実装するのは `core/usecase/interactor/create_todo.py` の `TodoUseCase`。
* Controller は `TodoInputBoundary` 型に依存することで、実際にどの実装（どのユースケースクラス）が呼ばれるかを知る必要がありません。

---

## 🧭 この層の関係まとめ

| 呼び出し元                | 呼び出す相手               | 呼び出しの手段                          |
| -------------------- | -------------------- | -------------------------------- |
| Controller           | UseCase (Interactor) | `TodoInputBoundary.execute(...)` |
| UseCase (Interactor) | -                    | （呼ばれる側 / 処理を実行する側）               |

ここでのポイントは2つ。

1. Controller は UseCase の「中身」を知らない
   → `TodoUseCase` のクラス名すら知らなくていい。
   → 契約（`TodoInputBoundary`）だけ知っていればOK。

2. UseCase は Controller の存在を知らない
   → 「どのUIから呼ばれたか」は関係ない。
   → CLIからでもGUIからでもWebアプリからでも、同じUseCaseが使える。

これが依存性逆転の原則（DIP: Dependency Inversion Principle）を守っている状態です。
**外側（UI）が内側（UseCase）に従う**のであって、逆ではありません。

---

## 🧪 ミニテスト（概念テスト）

InputBoundaryそのものは抽象インターフェースなので、単独でのロジックテストは基本的に不要です。
ただし、「Controllerがこの契約どおりに呼べるか？」という観点で、最小のフェイクUseCaseを用意して振る舞いを確認することはできます。

```python
from core.usecase.boundary.input_boundary import TodoInputBoundary
from core.usecase.boundary.dto import TodoInputData


class FakeUseCase(TodoInputBoundary):
    """テスト用のダミーUseCase実装（Interactorの代役）"""
    def __init__(self):
        self.last_called_with: TodoInputData | None = None

    def execute(self, input_data: TodoInputData) -> None:
        self.last_called_with = input_data


# Controller側からの呼び出しイメージ
fake_uc = FakeUseCase()
fake_uc.execute(TodoInputData(title="学習タスク"))

assert fake_uc.last_called_with.title == "学習タスク"
print("✅ InputBoundary の契約どおりに呼び出せました")
```

このテストで学べることは「Controllerは`execute(input_data)`という形さえ守ればUseCaseを呼べる」という事実です。
つまり Controller の役目は「ユーザー入力を TodoInputData に詰めて、execute に渡すこと」だけなんだ、と実感できます。

---

## 🛡 鉄則

> 「UseCase は外の世界を知らない」
> そのための防波堤が InputBoundary。

* Controller は UseCase の実装を知らずに呼ぶ（疎結合）
* UseCase は Controller を知らない（UIに依存しない）
* 双方は `TodoInputBoundary` という契約だけを共有する

この境界を守ることで、

* ViewやControllerをCLI→GUI→Webに差し替えても、
* UseCase（業務ロジック）は一切変更不要になります。

これがクリーンアーキテクチャの「UIはいつでも取り替えられる」という約束の正体です。
