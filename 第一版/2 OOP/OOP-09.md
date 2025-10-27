# OOP-09

## 適用できるその他のパターン

Factory Method パターンは適用できるものの一つにすぎません。
現在のオブジェクト指向設計は非常に良い状態ですが、これを土台として、さらに多くのデザインパターンを適用することで、さまざまなビジネス要件の変化に、よりエレガントに対応できるようになります。

ここでは、特に親和性が高く、学習効果の高い代表的なパターンを3つ紹介します。

---

### 1\. Strategy パターン

- **目的**: アルゴリズム（戦略）をカプセル化し、クライアントから独立して動的に切り替えられるようにする。
- **適用シナリオ**: **割引戦略の適用**
    - 現在の価格は固定ですが、「全品10% OFFセール」「会員限定500円引き」といった割引戦略を追加したくなったとします。`if/else`を`Product`クラスに追加していくと、コードが複雑化します。
- **実装イメージ**:
    1. `PricingStrategy`というインターフェース（割引計算のルール）を定義します。
    2. `RegularPriceStrategy`（通常価格）、`PercentageDiscountStrategy`（10% OFF）といった具体的な戦略クラスを実装します。
    3. `Product`クラスが`PricingStrategy`を持つようにし、価格計算をその戦略に委譲します。

<!-- end list -->

```python
# --- Strategyパターンのイメージ ---
class PricingStrategy(ABC):
    @abstractabstractmethod
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

- **アナロジー**: **カーナビのルート検索**。目的地（結果）は同じでも、「最短ルート」「高速優先」「一般道優先」といった戦略（アルゴリズム）を自由に切り替えられるのと同じです。

---

### 2\. Decorator パターン

- **目的**: オブジェクトのコア機能はそのままに、**動的に**新しい機能（装飾）を追加する。
- **適用シナリオ**: **ギフトラッピングや延長保証の追加**
    - 物理商品に「ギフトラッピング (+300円)」や「延長保証 (+1000円)」といったオプションを追加したい場合に有効です。継承で対応しようとすると「ラッピング付きマウス」「保証付きマウス」「ラッピングと保証付きマウス」…とクラスが爆発的に増えてしまいます。
- **実装イメージ**:
    1. `Product`インターフェースを実装した`ProductDecorator`という抽象クラスを作ります。
    2. `GiftWrapDecorator`や`ExtendedWarrantyDecorator`といった具体的な装飾クラスを作ります。これらのクラスは、内側に本物の`Product`オブジェクトを保持します。
    3. 価格を尋ねられたら、内側のオブジェクトの価格に、自身のオプション料金を加算して返します。

<!-- end list -->

```python
# --- Decoratorパターンのイメージ ---
# ギフトラッピングで商品を「装飾」する
mouse = PhysicalProduct(...)
wrapped_mouse = GiftWrapDecorator(mouse)

# 装飾されたオブジェクトの価格は、自動的にオプション料金が加算される
print(wrapped_mouse.get_price()) # -> 4300

```

- **アナロジー**: **アイスクリームのトッピング**。バニラアイス（コア機能）に、チョコレートソースやナッツを後から自由に追加（装飾）していくのと同じです。

---

### 3\. Observer パターン

- **目的**: あるオブジェクト（出版社のSubject）の**状態が変化した**ときに、それに依存するすべてのオブジェクト（購読者のObserver）に自動的に**通知**して更新する。
- **適用シナリオ**: **注文成功後の連携処理**
    - `Store`クラスで注文が成功した際に、「在庫管理システムへの通知」「配送部門への通知」「顧客へのメール送信」といった複数の連携処理が必要になった場合に有効です。`Store`クラスがこれらの連携先のクラスをすべて知っていると、密結合になってしまいます。
- **実装イメージ**:
    1. `Observer`インターフェース（`update`メソッドを持つ）を定義します。
    2. `InventorySystem`, `ShippingDepartment`といった具体的なObserverクラスを実装します。
    3. `Store`クラス（Subject）に、Observerを登録・解除するメソッドと、変更を通知するメソッドを追加します。
    4. `process_order`が成功した最後に、登録されているすべてのObserverに「注文がありました」と通知します。

<!-- end list -->

```python
# --- Observerパターンのイメージ ---
# 監視対象であるStoreに、監視者（Observer）を登録
inventory_system = InventorySystem()
shipping_dept = ShippingDepartment()

store = Store(...)
store.add_observer(inventory_system)
store.add_observer(shipping_dept)

# 注文が成功すると、storeは登録されているすべてのObserverに自動で通知する
store.process_order(...)

```

- **アナロジー**: **新聞や雑誌の定期購読**。出版社（Subject）は、新しい号が出たら、誰が購読しているかを意識することなく、すべての購読者（Observer）に新聞を送り届けます。

---

### まとめ

| パターン名 | 一言でいうと | 解決する課題 |
| --- | --- | --- |
| **Strategy** | アルゴリズムの着せ替え | `if/else`だらけのロジック分岐 |
| **Decorator** | 機能の後付けトッピング | 継承では対応できない、動的な機能追加 |
| **Observer** | 変化の自動お知らせ | 状態変化に伴う、多方面への連携処理 |

これらのパターンは、SOLID原則を土台として、より複雑な要求に柔軟に対応するための強力なツールです。Factory Method パターンだけでなく、これらのパターンも知っておくと、設計の引き出しが格段に増えます。

## 次のステージへ

これで、私たちのプログラムはSOLID原則を完全に満たし、Factory Method パターンによって生成ロジックも分離され、**「非常に優れたオブジェクト指向設計」**と呼べるレベルに到達しました。

しかし、それでもまだ解決できていない、より大きな課題が存在します。

`Store`クラスの`process_order`メソッドは、ビジネスロジック（在庫チェックなど）と、アプリケーションのフロー制御（エラーメッセージを`print`する、など）が混在しています。

もしデータベースに注文を保存したくなったら、`Store`クラスにデータベース接続のコードを追加することになるでしょうか？

このように、ビジネスの核心となるルールと、それをどうやってユーザーに届け、どうやってデータを永続化するかという技術的詳細が、まだ同じクラスの中に混在する危険性をはらんでいます。

この、より大きなレベルでの「関心事の分離」を実現するため、いよいよ次のステージであるクリーンアーキテクチャに進む準備が整いました。