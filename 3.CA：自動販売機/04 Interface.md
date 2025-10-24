# 04 Interface

# 🔌 Interfaces / Boundaries : application/boundaries.py

`UseCase`が仕事を依頼する相手との「契約書」にあたる、`boundaries.py` (Interfaces / Boundaries) を解説します。

## 🎯 このファイルの役割

このファイルは、`UseCase`と外側の層（`Adapters`）との間の「契約」を定義するインターフェースを集めたものです。

`UseCase`は、具体的な`Presenter`や`DataAccess`、そして今回は`Hardware`の実装を直接知る代わりに、ここで定義された抽象的な「契約書」にのみ依存します。

特に組み込みシステムでは、ビジネスロジックと物理的なハードウェアとの間にこの「契約」を設けることが、システムのテスト容易性と柔軟性を飛躍的に向上させます。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

## ✅ このファイルにあるべき処理

⭕️ 含めるべき処理の例:

- 抽象クラス (`ABC`) と 抽象メソッド (`@abstractmethod`) の定義のみ。

❌ 含めてはいけない処理の例:

- メソッド内の具体的なロジックの実装。すべての中身は`raise NotImplementedError`または`pass`であるべきです。

## 💻 ソースコードの詳細解説

```python
# application/boundaries.py
from abc import ABC, abstractmethod
from typing import Optional

# 先ほど定義したDataStructuresと、内側のEntityをインポート
from .data_structures import SelectItemInputData, SelectItemOutputData
from domain.entities import Item, PaymentManager

# -----------------------------------------------------------------------------
# Boundaries / Interfaces (<I>)
# - 同心円図の位置: 円と円の境界線そのもの
# -----------------------------------------------------------------------------

# --------------------------------------------------------------------
# - クラス図の位置: <I>InputBoundary / <I>OutputBoundary
# --------------------------------------------------------------------
class SelectItemInputBoundary(ABC):
    @abstractmethod
    def handle(self, input_data: SelectItemInputData, payment_manager: PaymentManager):
        raise NotImplementedError

class SelectItemOutputBoundary(ABC):
    @abstractmethod
    def present(self, output_data: SelectItemOutputData):
        raise NotImplementedError

# --------------------------------------------------------------------
# - クラス図の位置: <I>DataAccessInterface
# --------------------------------------------------------------------
class ItemDataAccessInterface(ABC):
    @abstractmethod
    def find_by_slot_id(self, slot_id: str) -> Optional[Item]:
        raise NotImplementedError

    @abstractmethod
    def save(self, item: Item) -> Item:
        raise NotImplementedError

# --------------------------------------------------------------------
# - クラス図の位置: <I>HardwareInterface (DataAccessInterfaceの一種)
# --------------------------------------------------------------------
class HardwareInterface(ABC):
    @abstractmethod
    def dispense_item(self, slot_id: str):
        """指定されたスロットの商品を物理的に排出する"""
        raise NotImplementedError

    @abstractmethod
    def return_change(self, amount: int):
        """指定された金額のお釣りを物理的に排出する"""
        raise NotImplementedError

```

- `HardwareInterface`: これがこの題材における最も重要なインターフェースです。`UseCase`は、商品の排出やコインの返却といった物理的な操作を、このインターフェースを通じて\_抽象的に\_指示します。`UseCase`は、モーターをどう制御するか、コインをどう数えるか、といったハードウェアの具体的な仕組みを一切知る必要がありません。

## 💡 ユニットテストは必要？

**原則として、このファイル自体に専用のユニットテストは不要です。**

なぜなら、これらはロジックを持たない「定義」や「契約書」だからです。テストすべき振る舞いがありません。これらのインターフェースが正しく使われているかは、`UseCase`や`Presenter`、`Adapters`といった、それらを利用・実装する側のクラスのテストで間接的に検証されます。

## 🐍 PythonとC言語の比較（初心者の方へ）

- Python (オブジェクト指向): `ABC`を使って抽象クラス（インターフェース）を明確に定義し、「このインターフェースを実装するクラスは、必ず`dispense_item`メソッドを持たなければならない」という契約をコードで表現できます。
- C言語 (手続き型): C言語にはインターフェースの概念がありません。似た役割を果たすのはヘッダーファイル (`.h`) です。ヘッダーファイルに`void dispense_item(const char* slot_id);`のように関数のシグネチャを宣言することで、「このヘッダーをインクルードすれば、この関数を呼び出せる」という一種の契約を定義します。

## 🛡️ このファイル群の鉄則

> 契約を定義せよ、実装は他に任せよ。 (Define the contract, delegate the work.)
> 

このファイルは、システムの柔軟性とテスト容易性を支える土台です。特に`HardwareInterface`があるおかげで、私たちは`UseCase`のテストをする際に、高価な実機や複雑なシミュレーターを用意することなく、単純な偽物（モック）を差し込むだけでロジックを100%検証できます。