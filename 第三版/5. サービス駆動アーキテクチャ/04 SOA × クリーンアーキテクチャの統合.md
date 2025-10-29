## 第4章：SOA × クリーンアーキテクチャの統合

この章では、これまでの知識をすべて統合します。
すなわち、

* **サービスを分ける（SOA）**
* **内部を整える（クリーンアーキテクチャ）**
  の両方を組み合わせて、**実務で使える全体構成**を理解します。

## 🎯 この章の目的

* 各サービスを内部的にクリーンアーキテクチャで設計する
* サービス間通信をOpenAPI契約で接続する
* 全体を統合したシステム構成を理解する
* チーム開発・テスト・デプロイの観点から整理する

---

## 1. 全体像のイメージ

**構造：**

```
[Reservation Service] ── REST/gRPC ── [Member Service]
         │                                  │
     (内部構造: Clean Arch)          (内部構造: Clean Arch)
         │                                  │
      Database                          Database
```

**説明：**

* 各サービスは「独立した小さなクリーンアーキテクチャ」
* 外部との通信はOpenAPI（REST）またはgRPCで実現
* 内部構造と外部契約が明確に分離されている

---

## 2. サービス内構造（クリーンアーキテクチャ）

各サービスの中身は、DDDとクリーンアーキテクチャを組み合わせて設計します。

**層構成：**

```
1. Entity（ドメインオブジェクト）
2. UseCase（アプリケーションロジック）
3. Interface Adapter（Controller / Presenter / Gateway）
4. Infrastructure（DB / Web Framework）
5. Main（Composition Root）
```

**特徴：**

* 内側（Entity）は業務ルール
* 外側（Infra）は技術依存部
* 中間層（Interface Adapter）が翻訳の役割を果たす

**まとめ：**

> サービスの内部は「変更に強い構造」にする。
> サービス間は「契約でつなぐ」。
> この2つを組み合わせることで、柔軟かつ安全なシステムになる。

---

## 3. 例：Reservation Service の構造

```
Reservation Service
 ├─ entity/
 │   └─ reservation.py
 ├─ usecase/
 │   └─ create_reservation.py
 ├─ interface/
 │   ├─ controller/
 │   │   └─ reservation_controller.py
 │   ├─ presenter/
 │   │   └─ reservation_presenter.py
 │   └─ gateway/
 │       └─ reservation_gateway.py
 ├─ infrastructure/
 │   ├─ db/
 │   │   └─ sqlite_repository.py
 │   └─ web/
 │       └─ fastapi_app.py
 └─ main.py
```

**ポイント：**

* 内部構造は完全にクリーンアーキテクチャで分離。
* 外部API（OpenAPI）はFastAPIで公開。
* 他サービスとの通信はgateway内で行う。

---

## 4. サービス間通信の責務分担

| 層            | 責務        | 通信の役割                  |
| ------------ | --------- | ---------------------- |
| **UseCase層** | 業務の中心ロジック | 他サービスの呼び出し要求を出す（依存逆転）  |
| **Gateway層** | 通信処理の実装   | HTTP/gRPC呼び出しを実際に行う    |
| **Infra層**   | 技術的詳細     | APIクライアント、リトライ、ログなど    |
| **契約ファイル**   | 仕様        | OpenAPIや.protoで定義された契約 |

**説明：**

> UseCaseは「呼び出したい」という意図だけを記述し、
> 実際の通信はGateway（Adapter）で行う。

---

### 例：Reservation Service → Member Service 呼び出し

**(1) UseCase側**

```python
class CreateReservationUseCase:
    def __init__(self, member_gateway, room_gateway, reservation_repo):
        self.member_gateway = member_gateway
        self.room_gateway = room_gateway
        self.reservation_repo = reservation_repo

    def execute(self, request):
        member = self.member_gateway.get_member(request.member_id)
        if not member["is_active"]:
            raise Exception("会員が無効です")
```

**(2) Gateway側**

```python
import requests

class MemberGateway:
    def get_member(self, member_id):
        url = f"http://member-service:8000/members/{member_id}"
        response = requests.get(url)
        return response.json()
```

**(3) Contract（契約定義）**

```yaml
GET /members/{id}
→ 200 OK
{
  "id": 1,
  "name": "Taro",
  "email": "taro@example.com",
  "is_active": true
}
```

**まとめ：**

* UseCase は「契約に基づく呼び出し」を行う
* Gateway は「契約通りの通信」を担当
* Contract Test によって整合性を保証

---

## 5. 契約と内部実装の関係

| 概念                            | クリーンアーキテクチャの位置     | SOAの位置      |
| ----------------------------- | ------------------ | ----------- |
| Entity                        | 内部ビジネスルール          | 各サービスの内側    |
| Repository Interface          | UseCase → Infra間契約 | 各サービスの内部    |
| API Contract (OpenAPI / gRPC) | 外部通信の仕様            | サービス間の境界    |
| Contract Test                 | 契約と実装の整合性保証        | 両者を橋渡しする検証層 |

**説明：**

> 内部の契約（Repository Interface）と外部の契約（API Contract）は別物だが、
> どちらも「変更の境界」を明確にする役割を持つ。

---

## 6. 全体統合構成（システムレベル）

**概念的な構造：**

```
[ Member Service ]     ← OpenAPI契約 →    [ Reservation Service ]    ← OpenAPI契約 →    [ Room Service ]
       │                           │                          │
  (クリーンアーキ構造)       (クリーンアーキ構造)         (クリーンアーキ構造)
```

**補足：**

* 各サービスは独立デプロイ可能（SOAの原則）
* 内部では依存逆転原則（DIP）を適用（クリーンアーキ原則）
* 全体では契約テストとサービスカタログで整合性を維持

---

## 7. チーム開発の分担モデル

| チーム            | 主な責務             | 成果物                 |
| -------------- | ---------------- | ------------------- |
| Reservationチーム | 予約処理ロジック、API公開   | reservation.yaml、実装 |
| Memberチーム      | 会員管理API、会員状態の提供  | member.yaml、実装      |
| Roomチーム        | 会議室API、空き情報の提供   | room.yaml、実装        |
| QAチーム          | 契約テスト・統合テスト      | Contract Test Suite |
| プラットフォームチーム    | サービスカタログ、CI/CD環境 | index.yaml、CI定義     |

**効果：**

> 各チームが独立して開発・テスト・リリースできる。
> 破壊的変更は契約テストで自動検出される。

---

## 8. デプロイとテストの流れ

1. 各チームが自分のサービスを更新
2. OpenAPIファイルを更新し、Pull Requestを作成
3. CIで契約テストを実行
4. サービスカタログを更新（バージョンアップ）
5. テスト通過後、個別にデプロイ

**説明：**

* デプロイ順序を気にせずにリリースできる
* バージョン管理と契約テストが安全網となる

---

## 9. SOA × クリーンアーキテクチャ統合のメリット

| 観点   | メリット                  |
| ---- | --------------------- |
| 柔軟性  | サービス単位で機能追加・修正が可能     |
| 再利用性 | 共通APIを他プロジェクトでも利用可能   |
| 保守性  | 内部構造が安定しているため変更影響が限定的 |
| 可観測性 | トレースIDやログを統合的に管理できる   |
| 自動化  | 契約テスト・CI/CDにより品質を継続保証 |

**まとめ：**

> SOAで「分け」、クリーンアーキで「整える」。
> この組み合わせが現代のSDVや業務システムの標準構造です。

---

## 10. まとめ

| 要点     | 内容                           |
| ------ | ---------------------------- |
| サービス構造 | 各サービスは独立したクリーンアーキ構成を持つ       |
| 接続方式   | OpenAPI / gRPC 契約を介して通信      |
| 内部契約   | Repository Interface による依存逆転 |
| 外部契約   | API Contract による明示的な仕様共有     |
| 品質保証   | 契約テスト＋CIで整合性を自動確認            |
| 開発体制   | チームごとの独立開発・独立リリースが可能         |

---

## 🎓 次のステップ

これで「クリーンアーキテクチャ → DDD → SOA」までが一通り完結しました。
次のテーマ候補としては次の3つがあります。

| 選択肢                     | 内容                              |
| ----------------------- | ------------------------------- |
| **第6部：マイクロサービスアーキテクチャ** | SOAをさらに細分化して独立デプロイ・監視・スケーリングを扱う |
| **第7部：自動テストとCI/CD**     | 契約テスト・ユニットテスト・統合テスト・自動デプロイを体系化  |
| **第8部：イベント駆動SOA（EDA）**  | 非同期メッセージングによる疎結合連携を強化する         |


