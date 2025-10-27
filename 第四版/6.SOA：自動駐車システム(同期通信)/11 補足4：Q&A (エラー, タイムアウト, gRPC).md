# 11 補足4

# 補足4：Q&A (エラー, タイムアウト, gRPC)

このアーキテクチャを実運用する上で重要となる、いくつかの補足的なQ&Aをまとめます。

## ❓ Q1: REST APIのエラーハンドリングはどうするか？

**A1: HTTPステータスコードを活用します。**

REST APIでは、処理が成功したか失敗したか、失敗した理由は何かを、HTTPステータスコードで表現するのが標準です。

- **2xx (成功):**
    - `200 OK`: リクエスト成功（`GET` や `PUT` の成功）。
- **4xx (クライアントエラー):**
    - `400 Bad Request`: リクエストの形式が不正（例：必須パラメータ欠如）。
    - `404 Not Found`: 要求されたリソース（例：`/users/999`）が存在しない。
    - `401 Unauthorized` / `403 Forbidden`: 認証・認可エラー。
- **5xx (サーバーエラー):**
    - `500 Internal Server Error`: サーバー内部で予期せぬエラーが発生。
    - `503 Service Unavailable`: （今回ステップ3で使った）サーバーが一時的に過負荷やメンテナンス中、または依存先（認識サービスなど）からのエラーで処理を完了できない。

クライアント側 ( `requests` ライブラリ) は、`response.raise_for_status()` を呼び出すことで、4xxや5xxのステータスコードを自動的に例外として検知できます。サーバー側 (FastAPI) は、`HTTPException` を `raise` することで、適切なステータスコードとエラー詳細をクライアントに返すことができます。

---

## ❓ Q2: 同期呼び出しのタイムアウトはどう管理するか？

**A2: クライアント側で明示的に設定します。**

同期呼び出しの最大の弱点は、呼び出し先が応答を返さない（または非常に遅い）場合、呼び出し元が永遠に（または非常に長く）待機してしまうことです。

これを防ぐため、クライアント側（`requests` ライブラリを使用する側）は、リクエスト時に必ず**タイムアウト**を設定すべきです。

```python
# 制御サービス (planning_client.py) の例
try:
    response = requests.get(
        f"{PLANNING_SERVICE_URL}/parking_plan",

        # CA: Infrastructure
        # 処理内容: タイムアウトを設定。
        # (接続に5秒、応答読み取りに10秒 = 合計最大15秒待つ設定)
        timeout=(5.0, 10.0)
        # または (合計10秒)
        # timeout=10.0
    )
    response.raise_for_status()
    return ParkingPlan.model_validate(response.json())

# CA: Infrastructure
# 処理内容: タイムアウトが発生するとこの例外が捕捉される
except requests.Timeout:
    print("Error: Request to Planning Service timed out.")
    return None
except requests.RequestException as e:
    print(f"Error fetching ParkingPlan: {e}")
    return None

```

このタイムアウト値（何秒まで待つか）は、システムの要件（「車両制御は最低でも10秒以内には応答が欲しい」など）に基づいて慎重に決定する必要があります。

---

## ❓ Q3: REST vs gRPC の比較は？

**A3: RESTは「汎用性」、gRPCは「パフォーマンス」に優れます。**

`gRPC` は、Googleが開発した別の同期通信（RPC: Remote Procedure Call）フレームワークです。RESTとしばしば比較されます。

| 観点 | REST (今回の方式) | gRPC |
| --- | --- | --- |
| **通信プロトコル** | HTTP 1.1 / HTTP 2 | HTTP/2 (必須) |
| **データ形式** | **JSON** (テキスト) | **Protocol Buffers** (バイナリ) |
| **パフォーマンス** | 普通 (JSONのパースが重い) | **非常に高速** (バイナリで効率的) |
| **契約定義** | OpenAPI (任意だが推奨) | `.proto` ファイル (必須) |
| **ブラウザ互換性** | **高い** (ブラウザから直接呼べる) | 低い (専用プロキシが必要) |
| **ストリーミング** | 限定的 (HTTPチャンクなど) | **標準サポート** (双方向通信が得意) |
| **主な用途** | Web API, 公開API, シンプルなサービス間通信 | **マイクロサービス内部通信**, モバイルアプリ |

**なぜ今回はRESTを選んだか？**
RESTとJSONは、人間が読みやすく、デバッグが容易（`curl` コマンドやブラウザで簡単にテストできる）であり、Webの標準技術として学習コストが低いため、最初の同期SOAを学ぶ上で最適です。

**gRPCが適しているケース**
もし、サービス間の通信が1秒間に何千回も発生し、0.1ミリ秒単位の遅延を削減したい、かつ通信が内部ネットワークで完結する（ブラウザから直接呼ばない）ような場合は、gRPCがRESTよりも優れた選択肢となります。