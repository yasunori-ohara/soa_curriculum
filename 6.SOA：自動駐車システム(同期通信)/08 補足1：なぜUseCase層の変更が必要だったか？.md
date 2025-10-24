# 08 補足1：なぜUseCase層の変更が必要だったか？

# 補足1：なぜUseCase層の変更が必要だったか？

今回の改修では、これまでの巡回とは異なり、Adapter層だけでなくUseCase層のコードにも修正が加わりました。
この補足ページでは、その理由と、クリーンアーキテクチャ（CA）におけるこの判断の妥当性について解説します。

## 📥 第5巡（MQTT）のUseCase：プッシュ型 (Push)

まず、第5巡のMQTTモデルを思い出してみましょう。

- **データの流れ:** 認識サービスがWorldModelをPublishすると、MQTTブローカーを経由して、経路計算サービスのMQTT Subscriber (Adapter) がそれを受信しました。
- **UseCaseの役割:** Adapterは、受信したWorldModelデータを引数としてUseCaseの `handle` メソッドを呼び出しました。

```python
# 第5巡 (MQTT) のイメージ
class CalculateParkingPlanUseCase:
    def handle(self, world_model: WorldModel):
        # 1. データは「引数」として与えられる (Push)
        # 2. 与えられたデータを使って計算する
        plan = self._calculate(world_model)
        return plan

```

このモデルでは、UseCaseは **受動的 (Passive)** です。「データが来たら、それを使って計算する」という単一の責任を持っていました。

## 📤 第6巡（REST）のUseCase：プル型 (Pull)

次に、今回のRESTモデルを見てみます。

- **データの流れ:** 車両制御サービスが `GET /parking_plan` をリクエストすると、経路計算サービスのFastAPI (Adapter) がそれを受け付けます。
- **UseCaseの役割:** Adapterは、UseCaseの `handle` メソッドを呼び出します。しかし、この時点では計算に必要な `WorldModel` データを持っていません。

```python
# 第6巡 (REST) の実装
class CalculateParkingPlanUseCase:
    def __init__(self, recognition_client: IRecognitionClient):
        # 2. データを取得するための「道具」(Adapter) を受け取る
        self.recognition_client = recognition_client

    def handle(self) -> ParkingPlan:
        # 1. 呼び出された時点ではデータがない

        # 3. 自分のタイミングでデータを「能動的」に取りに行く (Pull)
        world_model = self.recognition_client.fetch_world_model()

        # 4. 取得したデータを使って計算する
        if world_model:
            plan = self._calculate(world_model)
            return plan
        return None

```

このモデルでは、UseCaseは **能動的 (Active)** です。「呼び出されたら、まずデータを自ら取得し、それを使って計算する」という責任に変わりました。

## ⚖️ 変更の判断：ビジネスルールの変化か？

クリーンアーキテクチャの原則では、「UseCase（Application層）は、システムのビジネスルールをカプセル化する」とされています。

今回の変更は、単なるI/O（入出力）の詳細（MQTTかRESTか）の変更でしょうか？
いいえ、これは **「アプリケーションの振る舞い（シーケンス）」そのものの変更** です。

- **変更前 (Push):** 「WorldModelが与えられたら、計画を計算する」
- **変更後 (Pull):** 「計画を要求されたら、最新のWorldModelを取得し、計画を計算する」

「最新のWorldModelを取得する」というステップは、MQTTモデルには存在しなかった、新しい **アプリケーション固有のルール（手順）** です。したがって、このロジックはUseCase層に属するのが最も適切です。

### なぜこれがCA違反ではないのか？

「しかし、UseCaseがAdapter（RecognitionClient）に依存しているではないか？」という疑問が生じます。

ここで重要なのは、CAの **依存性のルール** です。
UseCaseは `adapter.recognition_client.RecognitionClient` という具体的な実装クラスに依存するのではなく、本来は `domain.repository.IRecognitionClient` のような **抽象（インターフェース）** に依存すべきです。

```python
# 本来の理想的な形
class CalculateParkingPlanUseCase:
    # UseCaseは「ドメイン層」で定義されたインターフェースに依存
    def __init__(self, recognition_repo: IRecognitionRepository):
        self.recognition_repo = recognition_repo

    def handle(self):
        # インターフェースのメソッドを呼ぶ
        world_model = self.recognition_repo.get_latest_world_model()
        ...

```

そして、`RecognitionClient` (Adapter) が、`IRecognitionRepository` (Domain) インターフェースの **実装** となります。
この「依存性の注入（DI）」により、UseCaseは「どうやってデータを取るか（RESTか、DBか）」というインフラの詳細を知ることなく、「データを取る」というビジネス手順を実行できます。

今回の実装（`usecase` が `adapter` をimport）は簡略化のためですが、**「UseCaseが能動的にデータを取得する」というロジックの変更** こそが、UseCase層のコード修正が不可避であった本質的な理由です。