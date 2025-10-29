
## 第3章：マイクロサービスのテスト戦略とCI/CD

## 🎯 この章の目的

* マイクロサービス特有のテスト構造を理解する
* 契約テストを含めた4層のテストピラミッドを学ぶ
* CI/CDでテストを自動化する流れを理解する
* 独立デプロイでも品質を保つ方法を学ぶ

---

## 1. テスト構造の全体像

テストは、目的に応じて4層に分けます。

```
E2Eテスト        ← 全体の流れを確認（ユーザー視点）
統合テスト        ← サービス間通信の確認
契約テスト        ← APIやイベント契約の整合性確認
ユニットテスト    ← ロジック単体の確認
```

* 下に行くほど範囲は狭く、実行が速い
* 上に行くほど範囲は広く、実行コストが高い

---

## 2. 各テスト層の概要

| 層       | 対象       | 目的        | 実施場所                |
| ------- | -------- | --------- | ------------------- |
| ユニットテスト | クラス／関数   | 内部ロジックの確認 | 各サービス内              |
| 契約テスト   | API／イベント | 仕様の一致確認   | Consumer / Provider |
| 統合テスト   | 複数サービス   | 実際の通信確認   | ステージング環境            |
| E2Eテスト  | 全体       | ユーザー操作の確認 | QA環境など              |

---

## 3. ユニットテストの例

```python
def test_create_reservation_success():
    repo = FakeReservationRepo()
    uc = CreateReservationUseCase(repo)
    result = uc.execute({"member_id": 1, "room_id": 3})
    assert result.status == "OK"
```

ポイント：

* DBアクセスや外部APIはモック化（擬似化）する
* 小さく速く、たくさん実行する

---

## 4. 契約テスト（Contract Test）

目的：

> サービス間の「約束（契約）」が実装と一致しているかを確認する。

```python
from openapi_spec_validator import validate_spec
import yaml, requests

spec = yaml.safe_load(open("member.yaml"))
response = requests.get("http://localhost:8000/members/1")

assert response.status_code == 200
validate_spec(spec)
```

* **Consumer側**（呼び出し側）：期待するレスポンスを確認
* **Provider側**（提供側）：OpenAPIやスキーマ通りに実装されているか検証

---

契約変更時の流れ：

```
1. Provider（例：Member Service）がAPI仕様を変更
2. Contract TestがConsumer側で自動的に検出
3. 修正が必要なサービスが特定される
```

→ サービス間の破壊的変更を早期に防ぐ

---

## 5. 統合テスト（Integration Test）

目的：

> 実際に複数サービスを起動して、通信経路が正しく機能するかを確認する。

構成イメージ：

```
Reservation → Member → Room
        ↑
   すべてDocker上で起動
```

実行例：

```bash
docker compose up -d
pytest tests/integration/
```

確認する内容：

* API呼び出しが成功するか
* 非同期イベントが正しく受信されるか
* タイムアウトやリトライが動作するか

---

## 6. E2Eテスト（End-to-End）

目的：

> 実際のユーザー操作の流れを確認する。

```python
def test_reservation_flow(browser):
    browser.goto("http://localhost:8080")
    browser.fill("#member_id", "1")
    browser.fill("#room_id", "3")
    browser.click("#reserve_button")
    assert "予約完了" in browser.text("#message")
```

* UIを含めた全体テスト
* 実行コストが高いため、デイリーテストや本番前に限定して行う

---

## 7. CI/CDパイプライン構成

CI/CD（継続的インテグレーション／継続的デリバリ）では、
各サービスのテストを自動化し、合格後にビルド・デプロイを実行します。

パイプラインの流れ：

```
Commit / Pull Request
   ↓
Unit Test（最速で実行）
   ↓
Contract Test（API整合性確認）
   ↓
Integration Test（通信検証）
   ↓
Build & Deploy（Docker/K8s）
```

---

GitHub Actions例：

```yaml
name: CI
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt
      - run: pytest tests/unit
      - run: pytest tests/contract
  deploy:
    needs: [test]
    steps:
      - run: docker build -t myapp:${{ github.sha }} .
      - run: kubectl rollout restart deployment myapp
```

---

## 8. 共通基盤で支える自動化

マイクロサービスが多い場合、共通チームがCI/CDや契約テストを管理します。

共通化の対象：

* OpenAPI仕様の管理（全サービスの契約一覧）
* テスト結果ダッシュボード（Green/Redの可視化）
* セキュリティスキャン（Dockerイメージ検査）
* 通知連携（Slack / Teams連携）

---

## 9. チーム運用のポイント

* 各サービスは**自分のテストとデプロイを自前で持つ**
* テスト仕様は**全サービス共通フォーマット**で統一
* テスト結果は**全員が見える状態**にする
* 手動手順は極力なくす
* 「テストが通ったものだけ」自動デプロイする

---

## 10. まとめ

| 要点    | 内容                                         |
| ----- | ------------------------------------------ |
| テスト構造 | Unit → Contract → Integration → E2E の4段階構成 |
| 契約テスト | マイクロサービス連携の信頼性を守る仕組み                       |
| CI/CD | テストとデプロイを自動化して品質と速度を両立                     |
| 運用    | 共通基盤とトレーサビリティを持つことで大規模化に対応                 |
| 成果    | 「壊れない変更」「止まらないリリース」を実現する                   |

---

## 🔜 第4章予告：「運用・監視・トレーシング — 可観測性（Observability）の設計）」

次章では、運用段階での**可観測性（Observability）**を扱います。
ログ・メトリクス・トレーシング・アラートの設計を通じて、
「どのサービスで何が起きたか」をリアルタイムで把握できる
マイクロサービス監視の基本構成を学びます。

---

次はこのフォーマットで
**第4章「運用・監視・トレーシング — 可観測性（Observability）の設計」**
を作成してよろしいですか？
