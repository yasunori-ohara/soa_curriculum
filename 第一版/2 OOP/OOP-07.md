# OOP-07

# オプション

## GoFデザインパターンの適用

### GoFデザインパターンとは？

これは、オブジェクト指向設計で頻繁に現れる特定の問題に対して、先人たちが見つけ出した**「成功事例のカタログ」あるいは「設計の定石集」**です。

SOLID原則が個々のクラスをどう作るかという「ミクロな原則」だとすれば、デザインパターンは、それらのクラスをどう組み合わせて、どう連携させるかという「メソな（中規模の）戦術」と言えます。

### 今のコードに適用できるデザインパターンの例

現在のコードは非常に良い状態ですが、デザインパターンを適用することで、さらに柔軟で保守性の高い構造に進化させることができます。

オブジェクトの生成を専門家に任せる「Factory Method パターン」
現状の問題: `main.py`の中で、`PhysicalProduct(...)`や`DigitalProduct(...)`といった具体的なクラス名を使って、オブジェクトを直接生成しています。もし将来、新しい商品クラス`SubscriptionProduct`が追加されたら、`main.py`を修正する必要があります。

解決策: 商品を生成する専門の「工場（Factory）」クラスを作ります。

```
# Factoryクラスの例
class ProductFactory:
    @staticmethod
    def create_product(product_type: str, **kwargs):
        if product_type == "physical":
            return PhysicalProduct(**kwargs)
        if product_type == "digital":
            return DigitalProduct(**kwargs)
        # 新しい商品もここに追加するだけ
        # if product_type == "subscription":
        #     return SubscriptionProduct(**kwargs)
        raise ValueError("未知の商品種別です")

# main.pyでの使い方
# main.pyは具体的なクラス名を知らなくて済む
mouse = ProductFactory.create_product("physical", product_id="p-001", ...)
ebook = ProductFactory.create_product("digital", product_id="d-001", ...)

```

このパターンを使うと、オブジェクトの生成ロジックを一箇所に集約でき、`main.py`のような利用側のコードを、具体的なクラス名の変更から守ることができます。