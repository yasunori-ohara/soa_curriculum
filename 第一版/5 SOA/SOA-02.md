# SOA-02

```markdown
## external_services/inventory_service.py

それではSOAを適用したプログラムの解説を、開発の推奨順序に従って進めていきましょう。

SOAでは、まず連携先となる外部サービスがどのような「契約（API）」を提供しているかを理解することが起点となります。したがって、今回はまず模擬的な外部サービスである**`external_services/inventory_service.py`**から説明を始めるのが最も自然です。

---
### 1. このファイルの役割：独立した「在庫管理サービス」

この`external_services/inventory_service.py`ファイルは、私たちの「注文受付アプリケーション」とは完全に独立した、別のビジネスコンポーネントを表現しています。

これは、倉庫管理チームが開発・運用している、独立した「在庫管理サービス」だと考えてください。このサービスは、自分自身のデータベース（`_products_stock`）を持ち、「在庫を確認する」「在庫を減らす」というビジネス能力を、明確なインターフェース（メソッド）を通じて外部に公開しています。

私たちの「注文受付サービス」は、この独立したサービスを**利用者（クライアント）**として呼び出すことになります。

### 2. ソースコードの詳細解説

`InventoryService` クラス
このクラスが、外部サービスそのものを模倣しています。現実のシステムでは、これは別のサーバーで動作しているWeb APIかもしれませんし、共有のメッセージキューかもしれません。

- `__init__(self)`:
このサービスが管理する、独自の在庫データベース（`_products_stock`）を初期化します。重要なのは、このデータが「注文受付サービス」が持つ`ProductRepository`のデータとは完全に分離されている点です。

- `add_stock(...)`:
このサービスが外部に公開している「契約」の一つです。在庫を追加するためのメソッドです。

- `check_stock(...)`:
在庫が十分かを確認するための「契約」です。`True`か`False`を返します。

- `reduce_stock(...)`:
在庫を減らすための「契約」です。在庫が不足している場合は例外を発生させます。

### 3. このコンポーネントの鉄則

自己完結している: 在庫管理に関するすべてのデータとロジックは、このサービス内にカプセル化されています。

独立している: このサービスは、「注文受付サービス」の存在を一切知りません。ただ、不特定多数のクライアントからの要求に応えるだけです。

明確な契約を持つ: 外部のクライアントは、このサービスが公開しているメソッド（`check_stock`など）だけを頼りに連携します。内部の`_products_stock`が辞書なのか、巨大なデータベースなのかを知る必要はありません。

---
### external_services/inventory_service.py
``` Python
# 依存性のルール:
# このファイルは、私たちのアプリケーションとは独立した外部サービスを模倣しています。
# したがって、私たちのアプリケーションのどのファイルにも依存しません。

class InventoryService:
    """
    【外部サービス】
    独立した「在庫管理サービス」を模倣したクラス。
    
    このサービスは、自分自身の在庫データベースを持ち、
    「在庫確認」や「在庫削減」といったビジネス能力を、
    明確なインターフェース（メソッド）を通じて外部に提供する。
    """
    def __init__(self):
        # このサービスが専有する、独立した在庫データベース
        self._products_stock: dict[str, int] = {}
        print("📦 在庫管理サービスが起動しました。")

    def add_stock(self, product_id: str, quantity: int):
        """在庫を追加するためのインターフェース"""
        self._products_stock[product_id] = self._products_stock.get(product_id, 0) + quantity
        print(f"📦 在庫追加: {product_id} の在庫が {self._products_stock[product_id]} になりました。")

    def check_stock(self, product_id: str, quantity: int) -> bool:
        """在庫が十分か確認するためのインターフェース"""
        stock = self._products_stock.get(product_id, 0)
        return stock >= quantity

    def reduce_stock(self, product_id: str, quantity: int):
        """在庫を削減するためのインターフェース"""
        if not self.check_stock(product_id, quantity):
            raise ValueError(f"在庫不足: {product_id}")
        
        self._products_stock[product_id] -= quantity
        print(f"📦 在庫削減: {product_id} の在庫が {self._products_stock[product_id]} になりました。")

    def get_stock(self, product_id: str) -> int:
        """現在の在庫数を取得するためのインターフェース"""
        return self._products_stock.get(product_id, 0)
```

```