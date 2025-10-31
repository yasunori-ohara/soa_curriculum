# 30 補足：Data Structures と Boundaries の関係

## 💡 Data Structures と Boundaries の関係

### ✅ 役割の違い

- **Data Structures (`data_structures.py`)**
    - レイヤー間でデータを運ぶ「箱」。
    - 例：`TodoInputData` は Controller → Use Case の入力データ。
    - 例：`TodoOutputData` は Use Case → Presenter の出力データ。
    - **責務**：データを安全に運ぶことだけ。ロジック禁止。
- **Boundaries (`boundaries.py`)**
    - レイヤー間の「契約書」（インターフェース）。
    - 例：`TodoInputBoundary` は Controller が Use Case を呼び出すための契約。
    - 例：`TodoOutputBoundary` は Use Case が Presenter に結果を渡すための契約。
    - **責務**：メソッドの署名だけを定義。実装は持たない。

---

### ✅ なぜ分けるべきか？

1. **責務の違いが明確**
    - Data Structures → データの形を保証。
    - Boundaries → 振る舞い（メソッド契約）を保証。
    → 両者を分けることで「何を探すか」が明確になる。
2. **変更頻度の違い**
    - Data Structures → UIや表示仕様の変更で変わりやすい。
    - Boundaries → Use Caseの契約なので比較的安定。
    → 分けることで、変更の影響範囲を限定できる。
3. **テスト容易性**
    - Data Structures → 単体テストで型や不変性を確認。
    - Boundaries → モックを作るために利用。
    → 別ファイルにすることでテストコードの見通しが良くなる。

---

### ✅ 両者の関係を図で表すと？

```
+-------------+        +----------------------+        +-------------+
| Controller  | -----> | TodoInputBoundary    | -----> |  Use Case   |
|             |        +----------------------+        | Interactor  |
|   (UI層)    |             ↑       |                  |             |
+-------------+             |       |                  +-------------+
                            |       |
                            |       v
                      +----------------------+
                      |  TodoInputData       |
                      |   (Data Structure)   |
                      +----------------------+

+-------------+        +-----------------------+       +-------------+
| Presenter   | <----- | TodoOutputBoundary    | <---- |  Use Case   |
|             |        +-----------------------+       | Interactor  |
|   (UI層)    |             ↑       |                  |             |
+-------------+             |       |                  +-------------+
                            |       v
                      +----------------------+
                      | TodoOutputData       |
                      |   (Data Structure)   |
                      +----------------------+

```

- *契約（Boundaries）**は「どんなメソッドを呼べるか」を定義。
- *データ構造（Data Structures）**は「どんな形のデータを渡すか」を定義。

---

### ✅ 学習者へのメッセージ

- **Boundaries は「会話の言語」**、Data Structures は「会話の内容」。
- 両方が揃って初めて、レイヤー間のやり取りが安全で明確になる。

---

### 🔍 補足章の構成案

1. **なぜ分けるのか？（責務・変更頻度・テスト容易性）**
2. **両者の関係を図で示す**
3. **実務でのメリット（柔軟性・テスト容易性・保守性）**
4. **よくある間違い（DTOにロジックを入れる、Boundaryに具体実装を書く）**

## 🔍 Interface、Boundary、Contract などの用語の違いは？

- **Interface** と **Contract** は本質的に同じ概念です。
    
    → 「このメソッドを呼ぶなら、この引数と戻り値で」という約束を定義するもの。
    
    → 言語仕様では「Interface」という言葉を使うことが多いですが、概念的には「Contract（契約）」と同義です。
    
- **Boundary** はクリーンアーキテクチャ特有の言葉で、
    
    → 「レイヤー間の境界」を強調するために使われます。
    
    → Input Boundary（Controller→Use Case）、Output Boundary（Use Case→Presenter）など、**どの方向の境界か**を明示するのが特徴。
    

とても鋭い質問です！この違いは、**クリーンアーキテクチャの文脈での用語の使い分け**に関係します。

---

## 💡 なぜ Input Boundary と Output Boundary は「Boundary」で、Data Access Interface は「Interface」なのか？

![クリーンアーキテクチャ](../クリーンアーキテクチャ・クラス図.png)

### 1. **Boundary は「レイヤー間の境界」を強調する言葉**

- Input Boundary → **Controller → Use Case** の境界
- Output Boundary → **Use Case → Presenter** の境界
- これらは「アプリケーションの内側（Use Case）と外側（UI層）を分ける壁」なので、**Boundaryという名前で「境界」を明示**します。

💡 **ポイント**

Boundaryは「どの方向の境界か」を名前で示す（Input/Output）。

→ クリーンアーキテクチャの特徴的な命名。

---

### 2. **Data Access Interface は「境界」ではなく、外部サービスとの接続契約**

- Use Case → Repository（データアクセス）
ここは「レイヤー間の境界」というより、**外部システムとの接続窓口**です。
- そのため、一般的な「Interface」という名前で十分。
→ 「契約」ではあるが、Input/Outputのような方向性は不要。

💡 **ポイント**

Repositoryは外部システム（DB）とのやり取りなので、**Boundaryというより汎用的なInterface**。

---

### ✅ 命名の切り分けルール（クリーンアーキテクチャ流）

- **Boundary**
    - レイヤー間の境界を表す
    - Input/Outputなど、方向性を名前で示す
    - 主にUI層とUse Case層の間で使う
- **Interface**
    - 外部サービスや技術的詳細との接続契約
    - DB、API、外部ライブラリなど
    - 名前は「〇〇Interface」でOK

---

### ✅ 学習者へのまとめ

- **Boundary = レイヤー間の境界を明示する契約**
→ Input/Outputで方向性を示す
- **Interface = 外部システムとの接続契約**
→ DBやAPIなど技術的詳細に依存しない抽象化

## 💡 クラス図の二重線と各層の意味

- **右上：Enterprise Business Rules**
    - ドメインモデル（Entity）を含む最も内側の層。
    - ビジネスの普遍的なルールを定義。
- **その外側：Application Business Rules**
    - Use Case層（Interactor）。
    - アプリケーション固有のビジネスロジックを定義。
- **右側：Interface Adapters**
    - UI層（Controller、Presenter）やDBアクセス層（Repository）。
    - 内側のルールを外側の世界に適応させる役割。
- **下側：Framework & Drivers**
    - Webフレームワーク、DB、外部サービスなど技術的詳細。

---

### ✅ BoundaryとInterfaceの違い（命名の理由）

- **Input Boundary / Output Boundary**
    - **Interface Adapters層とApplication Business Rules層の境界**を表す。
    - Controller → Use Case（Input）、Use Case → Presenter（Output）。
    - 「レイヤー間の境界」を強調するために**Boundary**という名前。
- **Data Access Interface**
    - **Application Business Rules層とFramework & Drivers層の境界**を表す。
    - Use Case → Repository（DBアクセス）。
    - 技術的詳細との接続契約なので、一般的な**Interface**という名前。

---

### ✅ 命名ルールまとめ

- **Boundary** → レイヤー間の境界（UIとUse Caseなど）。
- **Interface** → 外部技術との接続契約（DB、APIなど）。

---

💡 **学習者へのメッセージ**

- Boundaryは「UIとUse Caseの境界」なので方向性を名前で示す。
- Interfaceは「技術的詳細との契約」なので汎用的な名前でOK。

---

## 💡 なぜ同心円図に Input Boundary が書かれていないのか？

### 1. **同心円図は「依存関係のルール」を示す図**

- 同心円図は、**どの層がどの層に依存してよいか**という原則を示すための抽象的な図です。
- 内側から外側へ向かう依存はNG、外側から内側への依存はOKという「依存性の方向」を強調します。
- そのため、**具体的なクラスやインターフェースの名前は載せない**のが基本。

---

### 2. **Input Boundary は「クラス図」で具体化される要素**

- クラス図は、同心円図の原則を実際のコード構造に落とし込んだもの。
- Input Boundary は Use Case層に属し、Controller（UI層）との境界を定義する契約。
- 同心円図では「Application Business Rules」層に含まれる概念ですが、名前は載せず、**「この層は外側とやり取りするためにインターフェースを持つ」**という抽象的な意味だけ示しています。

---

### ✅ 所属の確認

- **Input Boundary / Output Boundary** → **Use Case層（Application Business Rules）**
- **Data Access Interface** → **Use Case層（Application Business Rules）**
    - ただし、外側の Framework & Drivers と接続するための契約。

---

### ✅ 学習者へのメッセージ

- **同心円図は抽象的な依存ルールを示す** → 名前は載せない。
- **クラス図は具体的な構造を示す** → Input Boundaryなどが登場する。
- 所属は「Use Case層」で間違いない。