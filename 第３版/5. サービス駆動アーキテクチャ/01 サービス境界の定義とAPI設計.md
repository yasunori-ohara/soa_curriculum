## 第1章：サービス境界の定義とAPI設計（図表簡易版）

### 1. サービス分割（Reservation / Member / Room）

**構造：**

```
Reservation Service
 ├─ Domain（予約のビジネスルール）
 ├─ UseCase（予約の作成・取消）
 ├─ Repository（予約データの保存）
 └─ API（外部公開: /reservations）

Member Service
 ├─ Domain（会員のビジネスルール）
 ├─ UseCase（登録・照会・有効化）
 └─ API（外部公開: /members）

Room Service
 ├─ Domain（部屋のルール・定員・空き時間）
 ├─ UseCase（登録・空き確認）
 └─ API（外部公開: /rooms）
```

**関係：**

* Reservation → Member に問い合わせ（会員が有効か）
* Reservation → Room に問い合わせ（部屋が空いているか）

---

### 2. サービス境界を決める考え方

**分ける基準（YESなら別サービス）**

* 異なる言葉を使っている（ユビキタス言語が違う）
* データの所有者が異なる
* チームが異なる
* 更新タイミングが独立している

**説明：**

> DDDで言う「境界づけられたコンテキスト（Bounded Context）」が
> そのままSOAの「サービス境界」になります。

---

### 3. サービス間依存の整理

**通信方向（依存関係）：**

```
Reservation Service
 ├─ calls → Member Service  (会員の有効性を確認)
 └─ calls → Room Service    (空き状況を確認)
```

**説明：**

> Reservation Service は他サービスのデータを直接参照しません。
> 代わりに「API（契約）」を通して結果だけを受け取ります。

---

### 4. OpenAPIによる契約定義の概要

**Member Service のAPI**

```
GET /members/{id}
Response:
{
  "id": 1,
  "name": "Taro",
  "email": "taro@example.com",
  "is_active": true
}
```

**Room Service のAPI**

```
GET /rooms/{id}/availability?start=...&end=...
Response:
{ "available": true }
```

**Reservation Service のAPI**

```
POST /reservations
Request:
{
  "member_id": 1,
  "room_id": 2,
  "start": "2025-05-01T10:00:00",
  "end": "2025-05-01T11:00:00",
  "attendees": 3
}
Response:
{
  "id": 99,
  "status": "created"
}
```

**説明：**

> 各サービスはこのように「リクエスト」「レスポンス」を契約として定義します。
> OpenAPIではこれをYAML形式で明示します。

---

### 5. DDDとの対応関係

| DDDの概念          | SOAでの位置づけ             | OpenAPI上の要素                |
| --------------- | --------------------- | -------------------------- |
| Bounded Context | Service               | OpenAPIファイル単位              |
| Aggregate       | Resource              | 各エンドポイント                   |
| Value Object    | JSON Schema部品         | `components/schemas`       |
| UseCase         | Operation（GET/POSTなど） | `paths`                    |
| Repository契約    | APIのリクエスト・レスポンス       | `requestBody` / `response` |

---

### 6. サービス全体像（文章で表現）

* **Reservation Service**

  * 自サービス内で予約作成ロジックを持つ
  * 会員有効性と部屋空き状況をAPI経由で問い合わせ
* **Member Service**

  * 会員登録・照会・有効フラグ管理
  * 他サービスには「GET /members/{id}」のみ公開
* **Room Service**

  * 会議室の登録・空き時間確認
  * 「GET /rooms/{id}/availability」で公開

---

### 7. まとめ（この章の理解ポイント）

| 観点           | 内容                                |
| ------------ | --------------------------------- |
| サービス境界       | DDDのBounded ContextをSOAのサービスに変換する |
| 契約（Contract） | OpenAPIで仕様を明示する                   |
| 責務分離         | 各サービスは自分のデータ・ロジックを管理              |
| 通信方法         | REST API（JSON）でゆるく接続する            |
| 開発効果         | サービス単位で独立リリース可能になる                |

---

このように、**日本語の図を使わずに構造を箇条書き＋コード形式で表す**形に統一すれば、
環境依存せずにスライド・PDF・Markdown・Web教材のどこでも安定して読めます。

