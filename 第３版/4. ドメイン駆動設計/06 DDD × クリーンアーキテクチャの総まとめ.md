## 第6章：DDD × クリーンアーキテクチャの総まとめ

ここでは、これまで学んできたDDD（ユビキタス言語／値オブジェクト／集約／サービス／ファクトリ／リポジトリ）を、
クリーンアーキテクチャ（同心円構造）と**1つの設計思想として統合**します。

つまり、これが「**クリーンアーキテクチャをDDDの構造で運用する最終形**」です。

## 🎯 この章の目的

1. **DDDとクリーンアーキテクチャの関係**を整理する
2. **各層に何を置くか（設計マッピング）**を明確にする
3. **DDD準拠のフォルダ構成テンプレート**を理解する
4. **依存性逆転（DIP）＋ドメイン主導の構造**を一枚で理解する

---

## 🧭 1. なぜDDDとクリーンアーキは相性が良いのか

クリーンアーキテクチャは

> 「依存は内へ」

DDDは

> 「意味を中心に設計する」

両者を組み合わせると：

| クリーンアーキの目的          | DDDで実現される部分                             |
| ------------------- | --------------------------------------- |
| ビジネスルールをUIやDBから切り離す | Domain層（Entity, ValueObject, Aggregate） |
| アプリの構造を安定化させる       | 境界づけられたコンテキスト（Bounded Context）          |
| 依存方向を整理する           | Repository契約（DIP）                       |
| 構造と意味の両立            | DDD + CleanArchitecture の融合構造           |

つまり、
🧩 クリーンアーキが「**構造の骨格**」を作り、
🧠 DDDが「**中身（意味）**」を満たす。

---

## 🧱 2. DDDとクリーンアーキの対応表

| クリーンアーキの層                 | DDDでの概念                            | 主な要素                                                   |
| ------------------------- | ---------------------------------- | ------------------------------------------------------ |
| **Entity層（内核）**           | Domain層                            | Entity, ValueObject, Aggregate, DomainService, Factory |
| **UseCase層**              | Application層                       | UseCase（アプリケーションサービス）, Repository契約                    |
| **InterfaceAdapter層**     | Presentation層／Infrastructureゲートウェイ | Controller, Presenter, Gateway（Repository実装）           |
| **Frameworks & Drivers層** | Technical Infrastructure           | DB接続, Webフレームワーク, 外部API, CLI, UI                       |

> 💡 **DDDの「層」＝クリーンアーキの「円」**
> → どちらの観点からも同じ構造が見えるのが理想です。

---

## 🧩 3. 全体構造図（DDD × Clean Architecture 統合図）

```
+--------------------------------------------------------------+
|  Frameworks & Drivers 層（技術依存の外側）                  |
|  - FastAPI, DB接続, HTTP通信, CLI, UI                      |
|--------------------------------------------------------------|
|  Interface Adapter 層（入出力の翻訳）                       |
|  - Controller, Presenter, Gateway                           |
|  - Repository実装（DB/REST対応）                             |
|--------------------------------------------------------------|
|  UseCase 層（アプリケーションサービス）                     |
|  - UseCase（Interactor）                                    |
|  - Repository契約（抽象）                                   |
|  - Input/Output Boundary                                    |
|--------------------------------------------------------------|
|  Entity / Domain 層（ビジネスルールの中心）                 |
|  - Entity（Reservation, Member, Room）                      |
|  - ValueObject（TimeRange, Email, Capacity）                |
|  - Aggregate / Factory / DomainService                      |
+--------------------------------------------------------------+
```

> 🧭 依存は常に内側へ（Domainが最も安定）。
> 実行時の制御は外側から内側へ入り、結果は外へ戻る。

---

## ⚙ 4. DDD的フォルダ構成テンプレート

以下は、**Scheduling（予約管理）コンテキスト**を例にした構成です。

```
src/
└── scheduling/
    ├── domain/                      ← DDDの中核（Entity, VO, Aggregateなど）
    │   ├── entity/
    │   │   ├── reservation.py
    │   │   ├── room.py
    │   │   └── member.py
    │   ├── value_object/
    │   │   ├── time_range.py
    │   │   ├── email.py
    │   │   └── capacity.py
    │   ├── service/
    │   │   └── reservation_service.py
    │   ├── factory/
    │   │   └── reservation_factory.py
    │   └── __init__.py
    │
    ├── usecase/                     ← アプリケーション層
    │   ├── contracts/
    │   │   └── reservation_repository.py
    │   ├── create_reservation.py
    │   ├── cancel_reservation.py
    │   └── __init__.py
    │
    ├── interface/                   ← 入出力変換層
    │   ├── controller/
    │   │   └── reservation_controller.py
    │   ├── presenter/
    │   │   └── reservation_presenter.py
    │   └── gateway/
    │       └── reservation_gateway.py
    │
    ├── infrastructure/              ← 技術詳細（DB, Web, 外部連携）
    │   ├── db/
    │   │   ├── sqlite_conn.py
    │   │   └── reservation_repository_impl.py
    │   └── web/
    │       └── fastapi_app.py
    │
    ├── tests/                       ← テスト
    │   ├── domain/
    │   │   └── test_time_range.py
    │   ├── usecase/
    │   │   └── test_create_reservation.py
    │   ├── contracts/
    │   │   └── test_repository_contract.py
    │   └── __init__.py
    │
    └── main.py                      ← Composition Root（依存注入）
```

---

## 📘 5. 層の依存関係（import方向）

| From           | To            | 理由             |
| -------------- | ------------- | -------------- |
| Controller     | InputBoundary | UseCaseを抽象的に呼ぶ |
| UseCase        | Repository契約  | 永続化を抽象化するため    |
| Repository実装   | Domain Entity | データ復元のため       |
| Infrastructure | Interface層    | 技術的依存を内に閉じる    |

✅ **Domain層は何もimportしない**（最も安定・純粋）
→ これが「**クリーンアーキの中核**」であり、DDDの「**モデルの心臓部**」。

---

## 🧠 6. DDD × Clean Architecture の設計原則

| 原則             | 内容               | 守るための手段                            |
| -------------- | ---------------- | ---------------------------------- |
| **依存性逆転（DIP）** | 外側は内側の抽象に依存      | Repository契約、Boundary              |
| **集約境界**       | 一貫性は集約内で保つ       | Aggregate Root                     |
| **意味の分離**      | 言葉の範囲＝システムの範囲    | Bounded Context                    |
| **ルールの閉じ込め**   | 値・振る舞い・制約を型に埋め込む | ValueObject, Entity                |
| **責務の分配**      | 生成／判断／保存を分ける     | Factory, DomainService, Repository |

---

## ⚖ 7. DDD × CA における責務配置マップ

| 層                    | クラス／概念                                                 | 主な責務                     |
| -------------------- | ------------------------------------------------------ | ------------------------ |
| **Domain（内核）**       | Entity, ValueObject, Aggregate, DomainService, Factory | ビジネスルールの完全表現             |
| **UseCase**          | Application Service, Repository契約                      | ドメイン操作の手順（ユースケース）        |
| **InterfaceAdapter** | Controller, Presenter, Gateway                         | 入出力変換（DTO、ViewModel）     |
| **Infrastructure**   | Repository実装, DB, WebAPI, View                         | 技術的実装（SQL, HTTP, HTMLなど） |
| **Main**             | Composition Root                                       | 依存注入と起動（接着剤）             |

---

## 🧩 8. DDD × CA 統合構造の図解

```
外側（変わりやすい世界）
┌─────────────────────────────┐
│ Frameworks & Drivers        │
│ (FastAPI, DB, CLI, WebUI)   │
├─────────────────────────────┤
│ Interface Adapter           │
│ - Controller (入力変換)      │
│ - Presenter  (出力変換)      │
│ - Gateway    (翻訳)          │
├─────────────────────────────┤
│ UseCase                     │
│ - Application Service        │
│ - Repository契約 (抽象)     │
├─────────────────────────────┤
│ Domain (Core Model)         │
│ - Entity / ValueObject       │
│ - Aggregate / Factory / Service │
└─────────────────────────────┘
内側（変わりにくい世界）
```

> 💬 クリーンアーキで“内側に向かう依存”を守りながら、
> DDDで“意味ごとに分ける”構造を実現します。

---

## 🧪 9. 演習：自分のプロジェクトをマッピングする

| 要素                        | DDDの位置づけ           | クリーンアーキの層         | 実際のファイル例                                           |
| ------------------------- | ------------------ | ----------------- | -------------------------------------------------- |
| Reservation               | Entity             | Domain層           | `domain/entity/reservation.py`                     |
| TimeRange                 | ValueObject        | Domain層           | `domain/value_object/time_range.py`                |
| ReservationService        | DomainService      | Domain層           | `domain/service/reservation_service.py`            |
| ReservationFactory        | Factory            | Domain層           | `domain/factory/reservation_factory.py`            |
| ReservationRepository     | Repository契約       | UseCase層          | `usecase/contracts/reservation_repository.py`      |
| CreateReservationUseCase  | ApplicationService | UseCase層          | `usecase/create_reservation.py`                    |
| ReservationController     | Controller         | InterfaceAdapter層 | `interface/controller/reservation_controller.py`   |
| ReservationRepositoryImpl | Repository実装       | Infrastructure層   | `infrastructure/db/reservation_repository_impl.py` |

---

## 🧩 10. 実務的アドバイス（現場での適用）

| ステージ      | 実務的アクション                                     |
| --------- | -------------------------------------------- |
| **初期設計**  | Bounded Context を定義し、集約を洗い出す                 |
| **実装開始**  | 各コンテキストを Clean Architecture で独立構築            |
| **チーム分割** | コンテキスト単位でチームを分ける（依存が少ない）                     |
| **テスト戦略** | Domain層：単体テスト／UseCase：結合テスト／Repository：契約テスト |
| **発展**    | コンテキストごとにSOAやマイクロサービス化へ展開可能                  |

---

## ✅ 11. まとめ

| 観点            | 内容                           |
| ------------- | ---------------------------- |
| **DDDとCAの関係** | クリーンアーキは構造、DDDは意味。両者で完全な設計。  |
| **依存方向**      | 内向き依存（DIP）で安定。Domainは外を知らない。 |
| **分割単位**      | Bounded ContextごとにCA構造を持つ。   |
| **保存単位**      | Aggregate単位でRepositoryを持つ。   |
| **学習効果**      | DDDとCAをつなげることで、設計・実装・保守が一貫。  |

---

## 📘 次のステップ：SOA（サービス指向アーキテクチャ）へ

ここまでで

* 「**意味で分ける（DDD）**」
* 「**構造で守る（CA）**」
  が身につきました。

次のSOAでは、

> **それぞれのコンテキストを“サービス”として公開し、協調させる**
> というステージに入ります。

---

### 🔜 第5部：SOA（サービス指向アーキテクチャ）

- 第0章：はじめに（DDDとのつながり）
- 第1章：サービス境界の定義とAPI設計
- 第2章：サービス間通信（REST / OpenAPI）
- 第3章：契約テストとサービスカタログ
- 第4章：SOA × Clean Architecture の統合構造

