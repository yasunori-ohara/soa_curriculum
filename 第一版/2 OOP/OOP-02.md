# OOP-02 : SOLID原則による設計評価

`OOP-01` で構築したオブジェクト指向のコードは、手続き型プログラミングの問題点を鮮やかに解決しました。しかし、このコードは「変更に強い、良い設計」と言えるでしょうか？

この章では、オブジェクト指向設計の健全性を測るための有名な指針である **SOLID原則** を使って、`OOP-01` のコードを評価（健康診断）してみましょう。

## 🎯 この章のゴール

  * SOLID原則（S, O, L, I, D）の5つの概要を理解する。
  * 現状のコード（`OOP-01`）が、各原則をどの程度満たしているかを評価する。
  * 次のステップで取り組むべき設計上の課題（OCP違反、DIP違反など）を明確にする。

-----

## 📗 S: 単一責任の原則 (Single Responsibility Principle)

> 「クラスを変更する理由は、一つだけであるべき」

### 評価： ⚠️ 違反の可能性あり

`Product` クラスは「商品そのもののルール（在庫管理）」にのみ責任を持っており、これは原則を満たしています。

しかし、`Store` クラスの `process_order` メソッドを見てみましょう。

```python:logic.py (Storeクラス抜粋)
    def process_order(self, product_id: str, quantity: int):
        # ... (中略) ...

        # 責任1: 在庫チェックと削減の「指示」
        product = self._find_product(product_id)
        if not product.check_stock(quantity):
            # ...
            return
        product.reduce_stock(quantity)

        # 責任2: 「注文記録（OrderRecord）の作成」
        order_record = {
            "product_name": product.name,
            "quantity": quantity,
            "total_price": product.price * quantity,
            "order_date": datetime.datetime.now().isoformat()
        }
        
        # 責任3: 「注文記録の保存」
        self._orders.append(order_record)

        print(f"注文成功: ...")
```

このメソッドは、注文のフローを実行する責任（責任1）だけでなく、「注文記録をどのような形式で作成するか」（責任2）、「注文記録をどこに保存するか（今はリスト）」（責任3）という、**複数の責任**を持ってしまっています。

もし将来、「注文記録に顧客IDも追加したい」や「注文記録をリストではなくデータベースに保存したい」という変更要求が来た場合、`Store` クラスを修正する必要があります。これは「注文処理フロー」の変更ではないため、単一責任の原則に違反している兆候です。

-----

## 📖 O: オープン・クローズドの原則 (Open/Closed Principle)

> 「クラスは拡張に対して開いており、修正に対して閉じているべき」

### 評価： ❌ 違反している

この原則は「機能追加（拡張）の際に、既存のコードを修正しなくても良い」ことを意味します。

`Store` クラスは、新しい**商品インスタンス**を追加（拡張）する際に `add_product` を呼ぶだけで、`Store` クラスのコードを修正する必要はありません。この点では原則を満たしています（開いている）。

しかし、`Product` クラスはどうでしょうか。
もし「**ダウンロード商品**」（在庫の概念がない）のような、新しい**商品の種類**を追加したい場合、`Product` クラスの `check_stock` や `reduce_stock` メソッドを以下のように修正する必要があります。

```python:logic.py (Productクラスの修正イメージ)
    # 「もしダウンロード商品なら...」というif文が必要になってしまう
    def check_stock(self, quantity: int) -> bool:
        if self.product_type == "download": # 既存コードの修正
            return True
        return self._stock >= quantity

    def reduce_stock(self, quantity: int):
        if self.product_type == "download": # 既存コードの修正
            return # 在庫を減らさない
        if self.check_stock(quantity):
            self._stock -= quantity
            # ...
```

このように、既存のコードに `if` 文を追加するような「**修正**」が必要になるため、`Product` クラスは修正に対して「**閉じられていない**」と言えます。これは `OOP-04` で解決すべき大きな課題です。

-----

## 🔄 L: リスコフの置換原則 (Liskov Substitution Principle)

> 「親クラスのオブジェクトを、その子クラスのオブジェクトで置き換えても、プログラムの動作が変わらないべき」

### 評価： ➖ 評価対象外

この原則は、**継承関係**（親クラスと子クラス）がある場合に意味を持ちます。
現在のコード（`OOP-01`）には継承が使われていないため、この原則は（違反されてはいませんが）まだ本格的に問われていません。

`OOP-04` でOCP違反（ダウンロード商品）を解決するために「継承」を導入する際、この原則を守ることが極めて重要になります。

-----

## 🖇️ I: インターフェース分離の原則 (Interface Segregation Principle)

> 「クライアント（クラスの利用者）に、利用しないインターフェース（メソッド）への依存を強制すべきではない」

### 評価： ✅ 適用できている

この原則は、「クラスが持ちすぎている不要なメソッドを、利用者に強制してはいけない」ことを意味します。

`Product` クラスが持つメソッド（`check_stock`, `reduce_stock` など）は、商品として扱われる上で凝集度が高く、`Store` クラスはそれらをすべて必要としています。不要なメソッドはありません。
`Store` クラスも同様に、店舗としての役割に特化しており、`main.py` に不要なメソッドを強制していません。

いわゆる「太ったインターフェース（fat interface）」にはなっておらず、この原則は満たされています。

-----

## 🔗 D: 依存性逆転の原則 (Dependency Inversion Principle)

> 「上位モジュールは下位モジュールに依存すべきではない。両方とも抽象に依存すべき」

### 評価： ⚠️ 部分的に違反している

  * **上位モジュール**: `Store` クラス（`Product` を利用する側）
  * **下位モジュール**: `Product` クラス（`Store` に利用される側）

`main.py` が `Product` インスタンスを作成し、`tokyo_store.add_product(...)` で注入（DI）している点は評価できます。`Store` クラス自身が `Product(...)` を `new` していないため、依存は一方的です。

しかし、`Store` クラスは `Product` という**具体的なクラス**に依存しています。

```python:logic.py (Storeクラス抜粋)
    # 型ヒントが具象クラス「Product」になっている
    def add_product(self, product: Product):
        self._products[product.product_id] = product

    # 戻り値も具象クラス「Product」
    def _find_product(self, product_id: str) -> Product | None:
        return self._products.get(product_id)

    def process_order(self, product_id: str, quantity: int):
        # 具象クラス「Product」のメソッドを直接呼び出している
        product = self._find_product(product_id)
        if product:
            product.check_stock(quantity)
            product.reduce_stock(quantity)
```

「抽象に依存すべき」という原則に照らすと、`Store` は `Product` という具象クラスではなく、`IProduct` のような「抽象的な商品の概念（インターフェースや抽象クラス）」に依存すべきです。

この問題は、`OOP-03` でリファクタリングする主要なテーマとなります。

-----

## 📋 評価まとめ

`OOP-01` のコードは、手続き型と比べて大きな進歩を遂げましたが、SOLID原則による評価で以下の課題が明らかになりました。

1.  **S違反の疑い**: `Store` が注文記録の作成・保存ロジックまで持っている。
2.  **O違反 (明確)**: 新しい種類の商品（ダウンロード商品）に対応できない。
3.  **D違反 (明確)**: `Store` が具象クラス `Product` に依存している。

次の章（`OOP-03`）では、まず **D（依存性逆転の原則）** の違反を解消するリファクタリングに取り組みます。