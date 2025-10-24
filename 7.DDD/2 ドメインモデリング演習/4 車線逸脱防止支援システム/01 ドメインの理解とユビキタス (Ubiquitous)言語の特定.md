# 01 ドメインの理解とユビキタス (Ubiquitous)言語の特定

# ステップ１：ドメインの理解とユビキタス (Ubiquitous)言語の特定 🗣️

最初のステップは、対象となるドメイン（今回は「車線逸脱防止支援システム - LDW/LKA」）が**何をするためのものか**、**どのような言葉**が使われているかを理解することです。ここでの目標は、**ユビiquitous 言語**の候補を見つけ出すことです。このドメインは少し技術的な側面が強くなります。

*(ドライバー、車両制御エンジニア、センサー開発者などの視点を想像してみましょう。)*

## 🤔 何をするシステム？ (Core Purpose)

- カメラなどのセンサーを使って、車両が走行している**車線 (Lane)** を認識する。
- 車両が**車線**から**逸脱 (Departure)** しそうになっているか、あるいは逸脱したかを検知する。
- **逸脱の危険**がある場合、ドライバーに**警告 (Warning)** を出す (LDW - Lane Departure Warning)。
- さらに、システムが**ステアリング (Steering)** に介入し、車線内に留まるように**支援 (Assist / Intervention)** を行う (LKA - Lane Keeping Assist)。
- システムが**作動中 (Active)** か**非作動 (Inactive)** か、**一時停止中 (Suppressed)** かといった状態を持つ。
- (将来的には) カーブでの支援、ドライバーの意図（ウィンカー操作など）の考慮、道路標識の認識…などが考えられますが、まずは**基本的な機能**（直線での逸脱検知、警告、簡単な操舵支援）に焦点を当てます。

## 💬 どんな言葉が使われそう？ (Candidate Ubiquitous Language)

このシステムについて話すとき、自然に出てきそうな**名詞**や**動詞**、**形容詞**を洗い出してみましょう。

**名詞 (候補):**

- **車線** (Lane) - 白線(Lane Line/Marker)、実線(Solid Line)、破線(Dashed Line)
- **車両 / 自車** (Vehicle / Ego Vehicle)
- **車両状態** (Vehicle State) - 速度(Speed)、ヨーレート(Yaw Rate)、ステアリング角度(Steering Angle)
- **車線内の位置** (Lane Position / Lateral Position) - 車線中央からの距離
- **逸脱** (Departure / Deviation)
- **逸脱度 / 逸脱量** (Departure Degree / Deviation Amount)
- **警告** (Warning / Alert) - 音(Sound)、振動(Vibration)、表示(Display)
- **操舵支援 / 介入** (Steering Assist / Intervention / Torque Overlay)
- **システム状態** (System Status / State) - 作動中(Active)、非作動(Inactive)、待機中(Standby)、一時停止(Suppressed)、故障(Fault)
- **センサー** (Sensor) - カメラ(Camera)、(もしあれば)レーダー(Radar)
- **(もしあれば) ドライバー操作** (Driver Input) - ステアリング操作、ウィンカー(Turn Signal)
- **(もしあれば) 道路** (Road) - 直線(Straight)、カーブ(Curve)

**動詞 (候補):**

- **認識する** (Detect / Recognize) - 車線
- **検知する / 検出する** (Detect / Sense) - 逸脱
- **計算する** (Calculate) - 車線内の位置、逸脱度
- **判断する** (Judge / Determine) - 警告/介入の要否
- **警告する** (Warn / Alert)
- **介入する / 支援する** (Intervene / Assist / Apply Torque)
- **作動させる / 解除する** (Activate / Deactivate)
- **一時停止する** (Suppress)

**形容詞 / 状態 (候補):**

- **作動中 / アクティブ** (Active)
- **非作動 / インアクティブ** (Inactive)
- **待機中 / スタンバイ** (Standby)
- **一時停止中 / サプレス** (Suppressed)
- **故障中 / フォルト** (Fault / Error)
- **逸脱の可能性あり** (Potential Departure / Drifting)
- **逸脱した** (Departed)
- **車線内** (In Lane)

## 📝 用語の絞り込みと定義（仮）

特に重要そうな言葉を選び、仮の定義を与えます。このシステムは、単一の「運転支援」コンテキスト内で完結する可能性が高いと考えられます。

- **車線モデル (LaneModel)**: 認識された左右の車線（白線）の位置や種類（実線/破線）、曲率などの情報。
- **車両状態 (VehicleState)**: 自車の現在の速度、ヨーレート、ステアリング角度など、制御に必要な物理的な状態。
- **車線内相対位置 (LaneRelativePosition)**: 車線モデルと車両状態から計算される、車線の中央からの横方向のずれ（距離）や角度。
- **逸脱状態 (DepartureState)**: 車線内相対位置に基づいて判断される、現在の逸脱に関する状態（例: `IN_LANE`, `DRIFTING_LEFT`, `DEPARTED_RIGHT`）。
- **システム状態 (SystemStatus)**: LDW/LKAシステム全体の動作状態（例: `INACTIVE`, `ACTIVE_WARNING_ONLY`, `ACTIVE_ASSISTING`, `SUPPRESSED`, `FAULT`)。
- **警告要求 (WarningRequest)**: ドライバーに警告（音、振動など）を発する必要があることを示す要求。
- **操舵トルク要求 (SteeringTorqueRequest)**: ステアリングに介入するためのトルク要求値。

**ポイント:**

- `LaneModel`, `VehicleState`, `LaneRelativePosition` は、入力データや計算結果を表す**値オブジェクト**の候補です。これらは特定の瞬間の「スナップショット」であり、不変性が適しています。
- `DepartureState` や `SystemStatus` は、システムの「状態」を表し、**Enum** や状態を持つ**値オブジェクト**の候補です。
- `WarningRequest` や `SteeringTorqueRequest` は、システムからの「出力」を表す**値オブジェクト**（またはDTO）の候補です。
- このドメインの中心となる「モノ」（＝集約候補）は何でしょうか？ システム全体の状態 (`SystemStatus`) を管理し、入力 (`LaneModel`, `VehicleState`) を受け取って判断 (`DepartureState`) し、出力 (`WarningRequest`, `SteeringTorqueRequest`) を生成する、**`LaneKeepingAssistSystem`** のような集約が考えられます。

---

## ➡️ 次へ

ユビiquitous 言語の候補と、中心となりそうな集約のイメージが見えてきました。次は、この `LaneKeepingAssistSystem` 集約の境界と責務、そして内部で使われる値オブジェクトを具体的に考えてみましょう。「**(Step 2) 集約の発見と境界設定**」に進みます。