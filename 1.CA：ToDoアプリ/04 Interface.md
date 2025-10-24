# 04 Interface

# 🔌 Interfaces / Boundaries : `boundaries.py`

## 🧭 このファイルの役割

このファイルは、レイヤー間、特に `Use Case` と外側の層（`Adapters`）との間の「契約」を定義するインターフェースを集めたものです。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

これは、クリーンアーキテクチャの心臓部である**依存性反転の原則**を実現するための最も重要な部品です。

`Use Case` は、具体的な `Presenter` や `DataAccess` のクラスを直接知る代わりに、ここで定義された抽象的な「契約書」にのみ依存します。

💡 **イメージ**

壁の**電源コンセント**に例えると分かりやすいです。電化製品（`Use Case`）は、壁の向こう側にある発電所（`DataAccess`）の仕組みを知る必要はありません。ただ、標準規格のコンセント（インターフェース）にプラグを差し込むだけで、正しく動作します。

このファイルは、その「コンセントの規格」を定義する場所です。

---

## ✅ このファイルにあるべき処理

⭕️ **含めるべき処理の例**

- **抽象クラス** (`ABC`) と **抽象メソッド** (`@abstractmethod`) の定義のみ。
- メソッドの名前、引数、返り値の型といった、契約に必要なシグネチャ（署名）の定義。

❌ **含めてはいけない処理の例**

- メソッド内の具体的なロジックの実装。すべてのメソッドの中身は `pass` であるべきです。

---

## 🔍 ソースコード

```python
# --------------------------------------------------------------------
# File: boundaries.py
# Layer: Boundaries（レイヤー間の契約定義）
#
# 目的:
#   - Use Case と外部層（UI, DB）との間のインターフェースを定義する。
#   - 依存性反転の原則を実現し、柔軟でテストしやすい設計を可能にする。
#
# C言語の感覚に近い説明:
#   - ABC（Abstract Base Class）は「関数ポインタの型定義」に近い。
#   - 実装は持たず、関数の名前・引数・戻り値だけを定義する。
# --------------------------------------------------------------------

from abc import ABC, abstractmethod
from data_structures import TodoInputData, TodoOutputData
from domain.todo import Todo

# --------------------------------------------------------------------
# <I> Input Boundary に相当
#
# Controller → Use Case の契約。
# Controllerはこのインターフェースを通じてUse Caseを呼び出す。
# --------------------------------------------------------------------
class TodoInputBoundary(ABC):
    """
    Controller → Use Case の契約インターフェース。

    - execute(): ユーザー入力を受け取り、Use Caseを実行する。
    - input_data: UI層から渡される公式なデータ構造（TodoInputData）
    """
    @abstractmethod
    def execute(self, input_data: TodoInputData) -> None:
        pass

# --------------------------------------------------------------------
# <I> Output Boundary に相当
#
# Use Case → Presenter の契約。
# Use Caseはこのインターフェースを通じて結果をPresenterに渡す。
# --------------------------------------------------------------------
class TodoOutputBoundary(ABC):
    """
    Use Case → Presenter の契約インターフェース。

    - present(): 処理結果をPresenterへ渡す。
    - output_data: Use Caseが生成した公式な出力データ（TodoOutputData）
    """
    @abstractmethod
    def present(self, output_data: TodoOutputData) -> None:
        pass

# --------------------------------------------------------------------
# <I> Data Access Interface に相当
#
# Use Case → Repository の契約。
# Use Caseはこのインターフェースを通じてデータ保存・取得を依頼する。
# --------------------------------------------------------------------
class TodoDataAccessInterface(ABC):
    """
    Use Case → Repository の契約インターフェース。

    - save(): Todoを保存し、保存後の状態（ID付与済み）を返す。
    - find_all(): 現在保存されているTodo一覧を取得する。
    """
    @abstractmethod
    def save(self, todo: Todo) -> Todo:
        pass

    @abstractmethod
    def find_all(self) -> list[Todo]:
        pass

```

---

## 🧩 Python初心者向け補足

- **`ABC`（Abstract Base Class）と `@abstractmethod`**
    - JavaやC#の `interface` に近い仕組み。
    - `@abstractmethod` を付けたメソッドは、**必ずサブクラスで実装しなければならない**。
    - 抽象クラスは直接インスタンス化できず、**「契約書」としての役割に徹する**。
- **なぜこうするのか？**
    - Use Caseが `Presenter` や `Repository` の具体的なクラスを知らないことで、**柔軟性とテスト容易性が向上**する。
    - 依存関係の方向が「内側 → 外側」になることで、**内側の安定性が保たれる**。

---

## 🧪 この段階でユニットテストをすることができます

```python
# --------------------------------------------------------------------
# ユニットテスト例: Boundaries の抽象性確認
# - 抽象クラスは直接インスタンス化できないことを確認
# --------------------------------------------------------------------

import unittest
from boundaries import TodoInputBoundary

class TestBoundaries(unittest.TestCase):
    def test_cannot_instantiate_abstract_class(self):
        """抽象クラスは直接インスタンス化できないことを確認"""
        with self.assertRaises(TypeError):
            TodoInputBoundary()

if __name__ == "__main__":
    unittest.main()

```

💡 **テストの意図**

- 抽象クラスは直接インスタンス化できない → インターフェースとして正しく機能している証拠。

---

## 🛡 鉄則

> 契約を定義せよ、実装は他に任せよ。
> 
- ロジック禁止（計算・条件分岐などは一切書かない）
- 依存性反転を徹底（内側が外側に依存する）
- 柔軟性とテスト容易性を確保（差し替え可能な設計）