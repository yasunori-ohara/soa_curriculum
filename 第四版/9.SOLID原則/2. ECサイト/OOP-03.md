# OOP-03 : CAによるレイヤー分離 - エンティティと境界(Interface)の定義

`OOP-02`の評価で、`OOP-01`のコードは主に2つの大きな問題を抱えていることがわかりました。

1.  **SRP（単一責任の原則）違反**:
    `Store`クラスが「商品の在庫管理（リポジトリの役割）」「注文処理（ユースケースの役割）」「注文記録の保存（リポジトリの役割）」という、**3つ以上の異なる責任**を併せ持つ「神クラス（God Class）」と化している。
2.  **DIP（依存性逆転の原則）違反**:
    `Store`（高レベルな方針）が、`Product`（低レベルな詳細）という**具象クラス**に直接依存している。

クリーンアーキテクチャ（CA）は、まさにこれらの問題を解決するために生まれました。CAの核心は「**関心の分離 (Separation of Concerns)**」と「**依存性のルール (Dependency Rule)**」です。

  * **SRPの解決**: 責任（関心）ごとにコードをレイヤー（Entities, Use Cases, Adapters）に分離する。
  * **DIPの解決**: 依存性のルール（依存は常に内側へ向かう）を強制し、レイヤー間は必ずインターフェース（境界）を介して通信する。

この章では、`OOP-01`のモノリシックな`logic.py`を、CAの「**ドメイン層（Entities）**」と「**アプリケーション層（Boundaries）**」に分離することから始めます。

## 🎯 この章のゴール

  * `OOP-01`のコードをCAのレイヤーに分類する。
  * ドメイン層（エンティティ）を定義し、ビジネスの核となるルールをカプセル化する。
  * アプリケーションの「境界（Boundary）」＝「インターフェース」を定義し、DIPを適用する準備を整える。
  * この分離によって、`Store`クラスのSRP違反がどのように解消に向かうかを理解する。

-----

## 1\. ドメイン層 (Entities) の定義

CAで最も内側にある、\*\*エンタープライズ・ビジネス・ルール（ドメイン）\*\*の層です。ここは、アプリケーションの振る舞い（ユースケース）から独立した、ビジネスそのものの「概念」と「ルール」を定義します。

`OOP-01`の`logic.py`から、純粋なビジネス概念を抽出します。

> **`domain.py` (新設)**

```python
import datetime

class Product:
    """
    【エンティティ】
    「商品」というビジネス概念。
    自身のデータ（在庫）に関するルール（在庫チェック、削減）を持つ。
    OOP-01のProductクラスとほぼ同じ。
    """
    def __init__(self, product_id: str, name: str, price: int, stock: int):
        self.product_id = product_id
        self.name = name
        self.price = price
        self._stock = stock
        
        # (DDD適用前なので、バリデーションはまだ甘い)

    def check_stock(self, quantity: int) -> bool:
        """在庫が十分かを確認する（自身のルール）"""
        return self._stock >= quantity

    def reduce_stock(self, quantity: int):
        """在庫を減らす（自身のルール）"""
        if self.check_stock(quantity):
            self._stock -= quantity
            print(f"在庫更新: {self.name}の在庫が{self._stock}になりました。")
            return True
        return False

    def get_stock(self) -> int:
        return self._stock

class Order:
    """
    【新設エンティティ】
    「注文」というビジネス概念。
    OOP-01では曖昧な「辞書(dict)」だったものを、明確なクラスとして定義。
    """
    def __init__(self, product_name: str, quantity: int, total_price: int):
        self.product_name = product_name
        self.quantity = quantity
        self.total_price = total_price
        self.order_date = datetime.datetime.now().isoformat()
    
    def __repr__(self):
        return f"<Order: {self.product_name}, Qty: {self.quantity}>"
```

### 💡 この時点での改善

  * `OOP-01`の`Store`クラスは、エンティティではない（複数の責務が混在している）ため、**ここには含めません**。
  * `OOP-01`の`order_record`（辞書）を、`Order`**エンティティ**に昇格させました。これにより、「注文とは何か」がコード上で明確になりました。

-----

## 2\. アプリケーション境界 (Interfaces / Boundaries) の定義

CAのドメイン層（内側）とユースケース層（外側）の間には、「**境界（Boundary）**」が存在します。これは、\*\*DIP（依存性逆転の原則）\*\*を適用するための「**インターフェース（Pythonでは`ABC`）**」です。

ユースケース（例：注文処理）が、外部の仕組み（例：DB、Web）とどう連携するかを「契約」として定義します。

> **`application_boundaries.py` (新設)**

```python
from abc import ABC, abstractmethod
from domain import Product, Order # 依存は内側(domain)へ

# --- DTO (Data Transfer Objects) ---
# ユースケースの入出力データを運ぶためだけの、構造化されたデータクラス
# (CAでは Input/Output Data と呼ばれる)
class ProcessOrderRequest:
    """ユースケースへの入力データ"""
    def __init__(self, product_id: str, quantity: int, store_name: str):
        self.product_id = product_id
        self.quantity = quantity
        self.store_name = store_name # どの店舗での処理かも必要

class ProcessOrderResponse:
    """ユースケースからの出力データ"""
    def __init__(self, order: Order, updated_stock: int):
        self.order = order
        self.updated_stock = updated_stock


# --- 1. ユースケース（Interactor）の境界 ---
class IProcessOrderUseCase(ABC):
    """
    「注文処理」というユースケースの入力ポート（Input Port）。
    コントローラ(Adapter)は、このインターフェースの handle を叩く。
    """
    @abstractmethod
    def handle(self, request: ProcessOrderRequest):
        # 処理の実行。戻り値はPresenter経由で返される
        pass

class IProcessOrderPresenter(ABC):
    """
    「注文処理」の出力ポート（Output Port）。
    ユースケース(Interactor)は、このインターフェースを通じて結果を返す。
    """
    @abstractmethod
    def present_success(self, response: ProcessOrderResponse):
        pass

    @abstractmethod
    def present_failure(self, error: str):
        pass

# --- 2. ゲートウェイ（Gateway / Repository）の境界 ---
# ユースケースが「永続化」を必要とする場合に定義するインターフェース
# (CAの依存ルール: ユースケースは、このインターフェースに依存する)

class IProductRepository(ABC):
    """
    「商品」の永続化を担当するゲートウェイ（リポジトリ）の契約。
    OOP-01のStoreが持っていた「商品管理」の責任。
    """
    @abstractmethod
    def find_by_id(self, product_id: str, store_name: str) -> Product | None:
        pass
    
    @abstractmethod
    def save(self, product: Product, store_name: str):
        # 在庫更新などを反映する
        pass

class IOrderRepository(ABC):
    """
    「注文」の永続化を担当するゲートウェイ（リポジトリ）の契約。
    OOP-01のStoreが持っていた「注文記録」の責任。
    """
    @abstractmethod
    def save(self, order: Order, store_name: str):
        pass
```

-----

## ✨ この変更が解決したこと（設計上の進捗）

コードの実装はまだですが、「設計」は大きく進みました。

1.  **SRP（単一責任の原則）の解決**:
    `OOP-01`の巨大な`Store`クラスが持っていた複数の責任が、**インターフェースとして明確に分離**されました。

      * `IProductRepository`: 商品の保存・検索の責任
      * `IOrderRepository`: 注文の保存の責任
      * `IProcessOrderUseCase`: 注文処理のビジネスロジック（アプリケーション・ルール）の責任
      * `Store`というクラスは、ドメイン層からもインターフェース層からも消え去りました（`Store`は「リポジトリのコンテキスト」を示す`store_name`という引数になりました）。

2.  **DIP（依存性逆転の原則）の解決**:
    `OOP-02`では「`Store`が具象の`Product`に依存している」ことが問題でした。
    新しい設計では、`Product`（エンティティ）を扱うのは`IProductRepository`の実装（インフラ層）と`IProcessOrderUseCase`の実装（ユースケース層）になります。
    次の`OOP-04`で実装するユースケースは、`IProductRepository`や`IOrderRepository`という\*\*抽象（インターフェース）\*\*にのみ依存します。

    これにより、「高レベルな方針（ユースケース）」が「低レベルな詳細（DBやリストへの保存方法）」に依存するのではなく、両者が「**抽象（インターフェース）**」に依存する、というDIPが完全に達成されます。


## ✨ 補足：なぜ `OOP-01` の `Store` はエンティティではないのか？

ご指摘の通り、「複数の責務が混在している」だけではエンティティになれない理由として不十分です。「分離すればよい」というのはその通りです。

`OOP-01`の`Store`が`domain.py`（エンティティ層）に属せない真の理由は、その混在している責務をクリーンアーキテクチャ（CA）のレイヤーに基づいて分離していくと、**ドメイン・エンティティ層（`domain.py`）に残るものがほとんどない**からです。

`OOP-02`では、`Store`（特に`process_order`）が少なくとも3つの責任を持っていると指摘しました。これらをCAのレイヤーにマッピングしてみましょう。

#### 💡 1. 責任1: 注文のフローを実行する（在庫チェック・削減指示）

`process_order`メソッドが実行する「1. 商品を探し → 2. 在庫を確認し → 3. 在庫を減らす」という**手順**です。

* **該当レイヤー**: **ユースケース (Use Cases)** 層
* **理由**: これは、`Product`エンティティが持つ純粋なルール（「在庫はマイナスになれない」など）とは異なります。これは、そのルールを**利用**して、「注文を処理する」という**アプリケーション特有の目的**を達成するための**手順（＝ポリシー）**です。CAにおいて、このロジックは`domain.py`（エンティティ層）には属しません。

#### 💡 2. 責任2: 注文記録をどのような形式で作成するか

`order_record = { ... }` の部分です。

* **該当レイヤー**: **ユースケース (Use Cases)** 層
* **理由**: 「注文処理の結果として、どのようなデータ（注文記録）を生成するか」は、**ユースケース（責任1）の処理フローの一部**です。`OOP-01`では、これが曖昧な`dict`（辞書）として定義されていました。
* **補足**: `OOP-03`では、この`dict`を`Order`**エンティティ**（`domain.py`で定義）に昇格させました。しかし、`Order`エンティティを**インスタンス化（生成）する**のは、依然としてユースケース層の責務です。`OOP-01`の`Store`は、この「生成ロジック」を持ってしまっています。

#### 💡 3. 責任3: 注文記録をどこに保存するか（今はリスト）

`self._orders = []` というデータの保持と、`self._orders.append(order_record)` という保存処理です。

* **該当レイヤー**: **インフラストラクチャ (Infrastructure)** 層
* **理由**: これは「メモリ上のリストに保存する」という**具体的なデータ永続化の実装**です。CAにおいて、データが「どのように保存されるか」（DBか、ファイルか、メモリ上の辞書か）というロジックは、最も外側（インフラ層）の責務です。エンティティ（ドメイン層）は、「**永続化について無知（Persistence Ignorant）**」でなければなりません。

#### 💡 4. (指摘外の責務): 商品のデータ管理

`self._products = {}`, `add_product`, `_find_product` も同様です。

* **該当レイヤー**: **インフラストラクチャ (Infrastructure)** 層
* **理由**: これも「メモリ上の辞書に商品を保存する」という**具体的なデータ永続化の実装**（リポジトリの責務）です。

---
#### 💡 結論：`Store`を分解すると、エンティティ層には何も残らない

`OOP-01`の`Store`から、
* **責任1と2（ユースケース）**を`IProcessOrderUseCase`（アプリケーション層）に分離し、
* **責任3と4（インフラ）**を`IProductRepository`と`IOrderRepository`（インフラ層）に分離すると、

`Store`には**ドメイン層（`domain.py`）で定義すべき中核的なビジネスルールが何も残らない**ことがわかります。（唯一残る`self.name`も、`OOP-01`の文脈ではルールを持っていません）。

これが、`OOP-01`の`Store`をエンティティとして扱わず、`OOP-03`では`store_name`という単なる「コンテキスト（リポジトリを絞り込むための引数）」として扱った理由です。
（もちろん、ビジネスが成長し「店舗」自体がルールを持ち始めたら、その時点で`Store`エンティティを`domain.py`に新設することになります）

-----

## 🚧 次の課題

`OOP-03`では、CAの「設計図（エンティティと境界）」を定義しました。
次の章では、この設計図に基づいて、\*\*アプリケーション層（ユースケース）**と**インフラストラクチャ層（リポジトリの実装）\*\*を具体的に実装していきます。
