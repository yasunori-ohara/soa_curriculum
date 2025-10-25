# 04 Input Boundary

# 🚪 Input Boundary
### `core/usecase/boundary/input_boundary.py`

## 🎯 このファイルの役割

`InputBoundary` は、**Controller（入力受付）と UseCase（業務ロジック）をつなぐ境界（Boundary）**を定義します。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

アプリケーションが「外部からどんな操作を受け付けるか」を宣言する場所であり、
Controller はこの契約に従って UseCase（Interactor）を呼び出します。

💡 **言い換えると、「UseCaseを呼び出すためのドアの形を決める」場所です。**

---

## 📁 ファイルの配置

```
├─ core/
│   └─ usecase/
│       └─ boundary/
│           ├─ input_boundary.py     # <I> 入力境界（Controllerが呼ぶ）
│           ├─ output_boundary.py    # <I> 出力境界（Presenterが実装）
│           └─ dto.py                # <DS> DataStructure（Input/Output/ViewModel）
```

---

## 🔍 ソースコード

```python
# --------------------------------------------------------------------
# File: core/usecase/boundary/input_boundary.py
# Layer: Boundary（UI → UseCase の入力契約）
#
# 目的:
#   - Controller（入力受付）が呼び出す「ユースケース操作」を定義する。
#   - 具体的な実装は UseCase（Interactor）が担当する。
#
# C言語の感覚に近い説明:
#   - ここは「関数ポインタの型定義」にあたる部分。
#   - Controllerはこの型に従って呼び出すだけで、実装の中身を知らなくてよい。
# --------------------------------------------------------------------

from abc import ABC, abstractmethod
from core.usecase.boundary.dto import TodoInputData


class TodoInputBoundary(ABC):
    """
    InputBoundary（入力境界インターフェース）

    - Controller が呼び出すための契約書。
    - UseCase（Interactor）はこのインターフェースを実装する。
    """

    @abstractmethod
    def execute(self, input_data: TodoInputData) -> None:
        """
        Controller → UseCase の呼び出し口。

        Parameters
        ----------
        input_data : TodoInputData
            Controller が整形した入力データ。
        """
        pass
```

---

## 🧭 この層の関係

| 呼び出し元      | 役割            | 呼び出される先             |
| ---------- | ------------- | ------------------- |
| Controller | 「ユーザー操作を受け取る」 | UseCase（Interactor） |
| UseCase    | 「ビジネスロジックを実行」 | -                   |

👉 Controller は、UseCaseの中身を知らずに呼び出せる。
👉 UseCase は、Controller の存在を知らない。

この「片方向の知識」が **依存性逆転の原則（DIP）** を実現します。

---

## 🧪 概念テスト（擬似実装）

```python
from core.usecase.boundary.input_boundary import TodoInputBoundary
from core.usecase.boundary.dto import TodoInputData


class FakeUseCase(TodoInputBoundary):
    """テスト用のダミーUseCase"""
    def __init__(self):
        self.last_called = None

    def execute(self, input_data: TodoInputData) -> None:
        self.last_called = input_data


# Controller側からの呼び出しイメージ
fake = FakeUseCase()
fake.execute(TodoInputData(title="学習タスク"))

assert fake.last_called.title == "学習タスク"
print("✅ InputBoundary 呼び出し契約テスト成功")
```

---

## 🛡 鉄則

> 「UseCase は外の世界を知らない」
> そのための防波堤が InputBoundary。

* Controller は UseCase の実装を知らずに使える。
* UseCase は Controller の存在を知らない。
* 双方は `InputBoundary`（契約書）だけで結ばれる。

