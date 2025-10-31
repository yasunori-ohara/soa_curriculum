# OOP-05 : CAの「型」による組立 (3) - 依存性の注入 (DI)

`OOP-04`までに、CAの主要なレイヤー（ドメイン、ユースケース、インフラ）を実装しました。しかし、これらはまだ「部品」としてバラバラな状態です。

  * `ProcessOrderUseCase`（ユースケース）が依存する`IProductRepository`（抽象）に、どの「実装（`InMemory...`版）」を渡すのか、誰も決めていません。
  * `ProcessOrderUseCase`を「誰が」呼び出すのかが決まっていません。

この章では、これらの部品を**組み立て**、システム全体を完成させます。

## 🎯 この章のゴール

  * `main.py`（起動層）が**依存性の注入 (DI)** を行う「組立工場」の役割を担うことを理解する。
  * `main.py`で、具象クラスをインスタンス化し、インターフェースを介して依存関係を解決するコードを実装する。
  * CAの「型」にはめたアーキテクチャで、`OOP-01`のシナリオが実行できることを確認する。

-----

## 🛠️ 1\. 起動層 (main.py) での依存性の注入 (DI)

`main.py`は、CAの最も外側の層（あるいは層の外）です。
**唯一の責務は、すべての「具象クラス」をインスタンス化し、依存関係に従ってそれらを「組み立てる（注入する）」ことです。**

`main.py`だけが、`ProcessOrderUseCase`（ユースケース）と`InMemoryProductRepository`（インフラ）の**両方を知っている**、唯一の場所となります。

> **`main.py` (OOP-01からの全面改訂)**

```python
# --- インポートするものが劇的に変わる ---

# 1. ドメイン層（具象エンティティ）
from domain import PhysicalProduct, DigitalProduct

# 2. アプリケーション層（具象ユースケース）
from application import ProcessOrderUseCase

# 3. インフラ層（具象リポジトリ）
from infrastructure import InMemoryProductRepository, InMemoryOrderRepository

# (注意)
# main.py は「起動層」であるため、唯一
# *すべて*の具象クラスを知っている（インポートしている）ファイルとなる。
# application_boundaries.py (抽象) はインポートしない。

if __name__ == "__main__":

    # --- 1. 依存性の注入 (DI) ---
    # CAの「外側」から「内側」へと依存関係を組み立てていく

    # a. インフラ層（最も外側）の具象インスタンスを生成
    product_repo = InMemoryProductRepository()
    order_repo = InMemoryOrderRepository()

    # b. アプリケーション層（ユースケース）を生成
    #    -> コンストラクタ（__init__）を経由して、
    #       具象インスタンス（product_repo, order_repo）を「注入」する
    #    -> UseCaseは、自分が InMemory を使っているとは知らない
    order_use_case = ProcessOrderUseCase(
        product_repo=product_repo,
        order_repo=order_repo
    )

    # --- 2. 初期データのセットアップ ---
    # (OOP-01の main.py が担っていた処理)
    # 本来はユースケース(AddProductUseCase)経由で実行すべきだが、
    # 今回は簡略化のため、インフラ(リポジトリ)を直接操作する
    product_repo.add(
        PhysicalProduct("p-001", "高機能マウス", 4000, 10), "東京店"
    )
    product_repo.add(
        PhysicalProduct("p-002", "静音キーボード", 6000, 5), "東京店"
    )
    product_repo.add(
        DigitalProduct("d-001", "デザインソフト eBook", 8000), "東京店" # デジタル商品
    )
    # (大阪店のデータ追加は省略)


    # --- 3. アプリケーションの実行 ---
    # OOP-01 と同じシナリオを「ユースケース経由」で実行する

    # シナリオ1: 物理商品 (成功)
    # UseCaseのexecuteメソッドを直接呼び出す
    order_use_case.execute("p-001", 3, "東京店")

    # シナリオ2: 物理商品 (在庫不足)
    order_use_case.execute("p-002", 10, "東京店")

    # シナリオ3: デジタル商品 (成功)
    order_use_case.execute("d-001", 10, "東京店")

    print("\n--- 東京店 注文履歴 ---")
    # (UseCase に履歴取得機能がないため、リポジトリを直接参照)
    print(order_repo.find_all_by_store("東京店"))

```

-----

## 🛠️ 2\. リファクタリングの完了

これで、`OOP-01`の`Store`（神クラス）を、クリーンアーキテクチャ（CA）の「型」に従ってリファクタリングする作業が**完了**しました。

`OOP-01`のコードは、`logic.py`という単一のファイルにロジックが集中していましたが、リファクタリング後のコードは、以下の5つの明確な「関心事（レイヤー）」に分離されました。

  * **`domain.py`**:
      * **関心事**: ビジネスのルール（在庫管理方法など）
  * **`application_boundaries.py`**:
      * **関心事**: レイヤー間の「契約（インターフェース）」
  * **`application.py`**:
      * **関心事**: ビジネスの手順（いつ、何を呼び出すか）
  * **`infrastructure.py`**:
      * **関心事**: データの保存方法（どうやって保存するか）
  * **`main.py`**:
      * **関心事**: すべての「部品（具象クラス）」を組み立てる（DI）

`main.py`を除くすべてのファイル（`application.py`など）は、`InMemory...`といった**具体的な実装**を一切知らず、`I...Repository`という\*\*抽象的なインターフェース（境界）\*\*にのみ依存する、非常に柔軟な構造になりました。

-----

## 🚧 次の課題（謎解き）

さて、**私たちのリファクタリングは完了しました**。
CAの「型」に従って「関心の分離」と「依存性のルール」を守ったことで、`OOP-01`よりもはるかに見通しが良く、変更に強い（例：DBへの変更が容易な）コードが完成しました。

-----

ところで、`OOP-02`のSOLID原則評価のことを覚えているでしょうか？
あの時、`OOP-01`のコードは\*\*SRP（単一責任の原則）**と**DIP（依存性逆転の原則）\*\*に甚だしく違反している、と評価しました。

**このCAリファクタリング（`OOP-03`〜`OOP-05`）は、これらの違反を解決できたのでしょうか？**

次の章（`OOP-06`）で、このリファクタリング結果を、もう一度**SOLID原則**という「ものさし」で評価し直し、その「答え合わせ」をしてみましょう。