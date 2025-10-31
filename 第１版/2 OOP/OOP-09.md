# OOP-09 : OOPの設計力を高めるデザインパターン

`OOP-08` では、デザインパターンの一つである Factory パターンを学び、オブジェクトの「生成」の責任を分離しました。
デザインパターンは、これ以外にも多くの「設計の定石集」を提供しています。

この章では、SOLID原則を満たした現在のコードを土台として、さらに複雑なビジネス要件に対応するために役立つ、代表的なデザインパターンを3つ紹介します。

## 🎯 この章のゴール

  * Factory 以外にも強力なデザインパターンがあることを知る。
  * Strategy, Decorator, Observer パターンの基本的な目的と利用シナリオを理解する。
  * OOP設計が完了したコードが、次の「クリーンアーキテクチャ」になぜ進むべきなのか、その動機を理解する。

-----

## 📚 適用できるその他のパターン

現在のコードを土台として、さまざまなビジネス要件の変化に、よりエレガントに対応できる代表的なパターンを紹介します。

### 1\. 🎯 Strategy パターン

  * 目的: アルゴリズム（戦略）をカプセル化し、クライアントから独立して動的に切り替えられるようにする。
  * 適用シナリオ: 割引戦略の適用
      * 現在の価格は固定ですが、「全品10% OFFセール」「会員限定500円引き」といった割引戦略を追加したくなったとします。`if/else`を`Product`クラスに追加していくと、コードが複雑化します。
  * 実装イメージ:
    1.  `PricingStrategy`というインターフェース（割引計算のルール）を定義します。
    2.  `RegularPriceStrategy`（通常価格）、`PercentageDiscountStrategy`（10% OFF）といった具体的な戦略クラスを実装します。
    3.  `Product`クラスが`PricingStrategy`を持つようにし、価格計算をその戦略に委譲します。

<!-- end list -->

```python:strategyパターンのイメージ
class PricingStrategy(ABC):
    @abstractmethod
    def calculate(self, price: int) -> int: pass

class PercentageDiscountStrategy(PricingStrategy):
    def calculate(self, price: int) -> int:
        return int(price * 0.9)

# ProductクラスがStrategyを持つ
class Product(ABC):
    def __init__(self, ..., strategy: PricingStrategy):
        # ...
        self.pricing_strategy = strategy

    def get_price(self) -> int:
        # 価格計算を戦略に委譲する
        return self.pricing_strategy.calculate(self.price)
```

  * アナロジー: カーナビのルート検索。目的地（結果）は同じでも、「最短ルート」「高速優先」「一般道優先」といった戦略（アルゴリズム）を自由に切り替えられるのと同じです。

-----

### 2\. 🎁 Decorator パターン

  * 目的: オブジェクトのコア機能はそのままに、動的に新しい機能（装飾）を追加する。
  * 適用シナリオ: ギフトラッピングや延長保証の追加
      * 物理商品に「ギフトラッピング (+300円)」や「延長保証 (+1000円)」といったオプションを追加したい場合に有効です。継承で対応しようとすると「ラッピング付きマウス」「保証付きマウス」…とクラスが爆発的に増えてしまいます。
  * 実装イメージ:
    1.  `Product`インターフェースを実装した`ProductDecorator`という抽象クラスを作ります。
    2.  `GiftWrapDecorator`や`ExtendedWarrantyDecorator`といった具体的な装飾クラスを作ります。これらのクラスは、内側に本物の`Product`オブジェクトを保持します。
    3.  価格を尋ねられたら、内側のオブジェクトの価格に、自身のオプション料金を加算して返します。

<!-- end list -->

```python:decoratorパターンのイメージ
# ギフトラッピングで商品を「装飾」する
mouse = PhysicalProduct(...)
wrapped_mouse = GiftWrapDecorator(mouse)

# 装飾されたオブジェクトの価格は、自動的にオプション料金が加算される
# (get_price() メソッドが Product と Decorator で共通（LSP）である前提)
print(wrapped_mouse.get_price()) # -> 4300 (例)
```

  * アナロジー: アイスクリームのトッピング。バニラアイス（コア機能）に、チョコレートソースやナッツを後から自由に追加（装飾）していくのと同じです。

-----

### 3\. 📡 Observer パターン

  * 目的: あるオブジェクト（Subject）の状態が変化したときに、それに依存するすべてのオブジェクト（Observer）に自動的に通知して更新する。
  * 適用シナリオ: 注文成功後の連携処理
      * `Store`クラスで注文が成功した際に、「在庫管理システムへの通知」「配送部門への通知」「顧客へのメール送信」といった複数の連携処理が必要になった場合に有効です。`Store`クラスがこれらの連携先のクラスをすべて知っていると、密結合になってしまいます。
  * 実装イメージ:
    1.  `Observer`インターフェース（`update`メソッドを持つ）を定義します。
    2.  `InventorySystem`, `ShippingDepartment`といった具体的なObserverクラスを実装します。
    3.  `Store`クラス（Subject）に、Observerを登録・解除するメソッドと、変更を通知するメソッドを追加します。
    4.  `process_order`が成功した最後に、登録されているすべてのObserverに「注文がありました」と通知します。

<!-- end list -->

```python:observerパターンのイメージ
# 監視対象であるStoreに、監視者（Observer）を登録
inventory_system = InventorySystem()
shipping_dept = ShippingDepartment()

store = Store(...) # StoreはSubject(主題)
store.add_observer(inventory_system)
store.add_observer(shipping_dept)

# 注文が成功すると、storeは登録されているすべてのObserverに自動で通知する
store.process_order(...)
```

  * アナロジー: 新聞や雑誌の定期購読。出版社（Subject）は、新しい号が出たら、誰が購読しているかを意識することなく、すべての購読者（Observer）に新聞を送り届けます。

-----

### 📋 パターンまとめ

| パターン名 | 一言でいうと | 解決する課題 |
| --- | --- | --- |
| Strategy | アルゴリズムの着せ替え | `if/else`だらけのロジック分岐 |
| Decorator | 機能の後付けトッピング | 継承では対応できない、動的な機能追加 |
| Observer | 変化の自動お知らせ | 状態変化に伴う、多方面への連携処理 |

これらのパターンは、SOLID原則を土台として、より複雑な要求に柔軟に対応するための強力なツールです。Factory パターンだけでなく、これらのパターンも知っておくと、設計の引き出しが格段に増えます。

-----

## 🚀 次のステージへ

これで、私たちのプログラムはSOLID原則を完全に満たし、代表的なデザインパターンによって生成ロジックや将来の拡張性も確保され、\*\*「非常に優れたオブジェクト指向設計」\*\*と呼べるレベルに到達しました。

しかし、それでもまだ解決できていない、より大きな課題が存在します。

`Store`クラスの`process_order`メソッドは、

  * ビジネスロジック（`product.check_stock()`）
  * アプリケーションのフロー制御（`print("エラー: ...")`）

これらが混在しています。

もし、このシステムをWebアプリケーションにしたくなったら、`print` を `return "エラーメッセージ"` に書き換えるのでしょうか？
`Store` クラスは、ビジネスの核心的なルール（ドメイン）を担うはずが、`print` という「技術的詳細（UI層）」を知ってしまっている状態です。

この、より大きなレベルでの「関心事の分離」——すなわち、**「ビジネスの核心的ルール」と「技術的詳細（UI、DB、フレームワーク）」とを完全に分離する——を実現するため、いよいよ次のステージであるクリーンアーキテクチャ**に進む準備が整いました。