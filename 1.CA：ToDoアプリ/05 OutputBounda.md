# 05 Output Boundary

# 📤 Output Boundary
### `core/usecase/boundary/output_boundary.py`

## 🎯 このファイルの役割

`OutputBoundary` は、**UseCase（業務ロジック）から Presenter（出力変換）への出口**を定義します。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

UseCase は結果を Presenter に渡しますが、Presenter の実装（CLI / Web / GUI）を知ってはいけません。
そのため、OutputBoundary を「出口の契約書」として挟みます。

💡 **言い換えると、「結果をどのように通知するか」を抽象化したドア**です。

---

## 📁 ファイルの配置

```
├─ core/
│   └─ usecase/
│       └─ boundary/
│           ├─ input_boundary.py     # <I> InputBoundary（Controllerが呼ぶ）
│           ├─ output_boundary.py    # <I> OutputBoundary（Presenterが実装）
│           └─ dto.py                # <DS> DataStructure（Input/Output/ViewModel）
```

---

## 🔍 ソースコード

```python
# --------------------------------------------------------------------
# File: core/usecase/boundary/output_boundary.py
# Layer: Boundary（UseCase → Presenter の出力契約）
#
# 目的:
#   - UseCase が Presenter へ結果を渡すための「出口」を定義する。
#   - Presenter の具体的な実装を UseCase から完全に切り離す。
#
# C言語の感覚に近い説明:
#   - ここは「コールバック関数の型定義」にあたる部分。
#   - UseCase はこの関数ポインタを呼ぶだけで、実装は知らない。
# --------------------------------------------------------------------

from abc import ABC, abstractmethod
from core.usecase.boundary.dto import TodoOutputData


class TodoOutputBoundary(ABC):
    """
    OutputBoundary（出力境界インターフェース）

    - UseCase（Interactor）が結果を Presenter に渡すための契約書。
    - Presenter はこのインターフェースを実装する。
    """

    @abstractmethod
    def present(self, output_data: TodoOutputData) -> None:
        """
        UseCase → Presenter の呼び出し口。

        Parameters
        ----------
        output_data : TodoOutputData
            UseCase が生成した出力データ。
        """
        pass
```

---

## 🧭 この層の関係

| 呼び出し元               | 役割               | 呼び出される先   |
| ------------------- | ---------------- | --------- |
| UseCase（Interactor） | 「結果を通知する」        | Presenter |
| Presenter           | 「結果を整形してViewへ渡す」 | View      |

👉 UseCase は Presenter を知らない。
👉 Presenter は UseCase の存在を意識せず、結果を整形する。

---

## 🧪 概念テスト（擬似実装）

```python
from core.usecase.boundary.output_boundary import TodoOutputBoundary
from core.usecase.boundary.dto import TodoOutputData


class FakePresenter(TodoOutputBoundary):
    """テスト用ダミーPresenter"""
    def __init__(self):
        self.last_output = None

    def present(self, output_data: TodoOutputData) -> None:
        self.last_output = output_data


# UseCase側から呼び出すイメージ
fake = FakePresenter()
fake.present(TodoOutputData(id=1, title="完了したタスク"))

assert fake.last_output.title == "完了したタスク"
print("✅ OutputBoundary 呼び出し契約テスト成功")
```

---

## 🛡 鉄則

> 「Presenter は UseCase の外にある。UseCase は結果をただ渡すだけ。」

* UseCase は出力フォーマットを知らない。
* Presenter は UseCase の内部構造を知らない。
* 双方は `OutputBoundary`（契約）で結ばれるだけ。

