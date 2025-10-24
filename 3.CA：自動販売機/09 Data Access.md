# 09 Data Access

# 💾 Data Access : adapters/data_access.py

`UseCase`が仕事を依頼する相手との「契約書」ができたので、次はそれを具体的に実装する`Adapters`層を見ていきましょう。

まずは、データベースとのやり取りを担当する`Data Access`を解説します。今回は商品在庫と、取引中の投入金額という2種類のデータを永続化（保持）する必要があります。

## 🎯 このファイルの役割

このファイルは、`boundaries.py`で定義されたデータアクセスインターフェースを、具体的な技術（今回はインメモリ）で実装するアダプターです。

`UseCase`からの「スロットID:A1の商品を探して」「現在の投入金額を保存して」といった抽象的な要求を、Pythonのオブジェクトや辞書を操作するという具体的なストレージへの読み書き処理に変換して実行します。

`UseCase`という「設計部門」と、データを保持する「倉庫」の間に立つ、「倉庫管理人」のような役割を果たします。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

## ✅ このファイルにあるべき処理

⭕️ 含めるべき処理の例:

- `DataAccessInterface`で定義されたメソッドの具体的な実装。
- `Entity`を永続化するためのデータソースとのやり取り。

❌ 含めてはいけない処理の例:

- ビジネスロジック（`UseCase`の責務）。
    - 例：「この商品は現在セール中か？」といった判断は行いません。ただ「在庫を探して」「保存して」という指示を忠実に実行するだけです。

## 💻 ソースコードの詳細解説

原則として、*1つのEntityに対して、1つのData Accessクラス（リポジトリ）が作られます。* 今回は`Item`と`PaymentManager`、2つのEntityに対応するリポジトリを実装します。

```python
# adapters/data_access.py

from typing import Dict, Optional

# 内側の世界のEntityと「境界」にのみ依存する
from domain.entities import Item, PaymentManager
from application.boundaries import ItemDataAccessInterface, PaymentManagerAccessInterface

# -----------------------------------------------------------------------------
# DataAccess (Repository)
# - クラス図の位置: DataAccess
# - 同心円図の位置: Adapters (Gateways)
# -----------------------------------------------------------------------------

class InMemoryItemDataAccess(ItemDataAccessInterface):
    """ItemDataAccessInterfaceの「インメモリDB」版実装"""
    _database: Dict[str, Item] = {}

    def __init__(self):
        # サンプルとして初期データをいくつか用意しておく
        self._database["A1"] = Item(slot_id="A1", name="お茶", price=160, stock=5)
        self._database["B2"] = Item(slot_id="B2", name="コーヒー", price=130, stock=3)
        self._database["C3"] = Item(slot_id="C3", name="水", price=110, stock=0) # 在庫切れ

    def find_by_slot_id(self, slot_id: str) -> Optional[Item]:
        """[インターフェースの実装] 'find_by_slot_id'を辞書検索で実現"""
        return self._database.get(slot_id)

    def save(self, item: Item) -> Item:
        """[インターフェースの実装] 'save'を辞書への上書き保存で実現"""
        self._database[item.slot_id] = item
        return item

class InMemoryPaymentManagerAccess(PaymentManagerAccessInterface):
    """
    PaymentManagerAccessInterfaceのインメモリ版実装。
    アプリケーション全体で唯一の投入金額状態を保持するため、シングルトンとして振る舞う。
    """
    _instance: PaymentManager = PaymentManager() # アプリケーションの状態を保持

    def get(self) -> PaymentManager:
        """現在のPaymentManagerインスタンスを取得する"""
        return self._instance

    def save(self, payment_manager: PaymentManager):
        """PaymentManagerの状態を保存（更新）する"""
        self._instance = payment_manager

```

- `InMemoryItemDataAccess`: 在庫情報を管理します。`__init__`でサンプル商品を辞書に格納しています。
- `InMemoryPaymentManagerAccess`: 取引中の投入金額を管理します。自動販売機全体で投入金額の状態は一つだけなので、クラス変数`_instance`に`PaymentManager`オブジェクトを保持する、シンプルなシングルトンパターンとして実装しています。

## 💡 ユニットテストでDataAccessの正しさを証明する

DataAccessのテストでは、永続化ロジックが正しく機能するかを検証します。`save`した後に`get`や`find`で正しく取得できるかを確認します。

```python
# tests/adapters/test_data_access.py の例

def test_InMemoryItemDataAccessは商品の保存と検索ができる():
    # 1. Arrange (準備)
    item_repo = InMemoryItemDataAccess()
    new_item = Item(slot_id="D4", name="コーラ", price=160, stock=10)

    # 2. Act (実行)
    item_repo.save(new_item)
    found_item = item_repo.find_by_slot_id("D4")

    # 3. Assert (検証)
    # 意図: 「saveしたものが、ちゃんとfind_by_slot_idで見つかるか？」をテスト
    assert found_item is not None
    assert found_item.name == "コーラ"

def test_InMemoryPaymentManagerAccessは状態を保持できる():
    # 1. Arrange (準備)
    pm_repo = InMemoryPaymentManagerAccess()
    pm = pm_repo.get()
    pm.insert_coin(100)

    # 2. Act (実行)
    pm_repo.save(pm)
    # 別の場所で再度取得したと仮定
    retrieved_pm = pm_repo.get()

    # 3. Assert (検証)
    # 意図: 「saveした状態が、次にgetした時も維持されているか？」をテスト
    assert retrieved_pm.current_amount == 100

```

## 🐍 PythonとC言語の比較（初心者の方へ）

- Python (オブジェクト指向): `InMemoryItemDataAccess`のようなクラスが、`ItemDataAccessInterface`という契約（インターフェース）を実装します。
- C言語 (手続き型): `save_item_to_file(item)`や`find_item_from_file(slot_id)`のような、特定のデータソースに密結合した関数群として実装されることが多いでしょう。データソースを変更するには、これらの関数をすべて書き直す必要があり、変更の影響が大きくなりがちです。

## 🛡️ このクラス群の鉄則

このファイル内のすべてのクラスは、同じ鉄則に従います。

> データを変換し、黙って実行せよ。 (Translate and execute, but do not think.)
> 
- これらのクラスはビジネスルールについて思考しません。`UseCase`からの指示を、技術的な作業として忠実に実行するだけです。
- このファイルのおかげで、`UseCase`はデータがメモリ上にあるのか、ファイルにあるのかを一切気にせずに済みます。将来、在庫管理を外部のWeb APIで行うことになったとしても、影響範囲はこの`data_access.py`ファイルだけに限定されます。