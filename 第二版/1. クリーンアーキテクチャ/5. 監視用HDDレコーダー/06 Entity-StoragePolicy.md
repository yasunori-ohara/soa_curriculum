## `domain/entities.py` - StoragePolicy

Entityの最後を飾る、最も重要で頭脳的な`StoragePolicy` Entityを実装します。

-----

### このクラスの役割 🧠

`StoragePolicy`**は、HDD全体の**「運用戦略」と「ルール」を表現する、非常に重要なEntityです。

個々の`RecordingSegment`を管理する上位の存在として、「次に上書きすべきセグメントはどれか？」という判断や、「緊急録画領域はまだ空いているか？」といった、**システム全体のストレージ戦略**に責任を持ちます。

DDD（ドメイン駆動設計）の用語では、`RecordingSegment`の集合体を管理する集約ルート（Aggregate Root）に近い役割を担います。

-----

### このクラスにあるべき処理

⭕️ **含めるべき処理の例:**

  * ストレージ運用方針を定義するプロパティ（緊急録画領域の割合など）。
  * **上書き対象のセグメントを特定するロジック** (`find_segment_to_overwrite()`メソッド)。
  * 緊急録画領域に空きがあるかを確認するロジック (`can_record_emergency()`メソッド)。

❌ **含めてはいけない処理の例:**

  * HDDから物理的にファイルを削除する処理（それは`Hardware`か`DataAccess`の責務です）。このEntityは、あくまで「どのセグメントを消すべきか」を**特定する**だけです。
  * UIに関する知識。

-----

### ソースコードの詳細解説

このEntityは、他のEntity（`RecordingSegment`のリスト）を操作して、高度なビジネス判断を行います。

```python
# domain/entities.py (続き)

import datetime
from enum import Enum
from dataclasses import dataclass
from typing import List, Optional

# --- 以前定義したクラス ---
# class RecordingSegment: ...
# (中略)

# --- 新しく定義するEnum ---
class EmergencyAreaFullPolicy(Enum):
    """緊急録画領域が満杯になった際の振る舞い"""
    STOP_RECORDING = "新規録画を停止"
    # OVERWRITE_OLDEST_EMERGENCY = "最も古い緊急録画を上書き" # 仕様ではA案なのでこちらは使わない

# ----------------------------------

class StoragePolicy:
    """ストレージポリシーエンティティ"""
    def __init__(self,
                 emergency_area_percentage: int,
                 full_policy: EmergencyAreaFullPolicy = EmergencyAreaFullPolicy.STOP_RECORDING):
        """
        [データ]
        HDD全体の運用ルールを定義する。
        """
        if emergency_area_percentage not in [0, 10, 20]:
            raise ValueError("緊急録画領域の割合は0, 10, 20%のいずれかでなければなりません。")
        
        self.emergency_area_percentage = emergency_area_percentage
        self.full_policy = full_policy

    def find_segment_to_overwrite(self, segments: List[RecordingSegment]) -> Optional[RecordingSegment]:
        """
        [ビジネスルール]
        上書きすべき通常録画セグメントを特定する。
        """
        # 保護されていない（通常またはアラーム）セグメントのみを抽出する
        overwritable_segments = [s for s in segments if not s.is_protected()]

        # 上書きできるセグメントがなければ、Noneを返す
        if not overwritable_segments:
            return None

        # 開始時刻（start_time）が最も古いセグメントを見つけて返す
        oldest_segment = min(overwritable_segments, key=lambda s: s.start_time)
        return oldest_segment

    def can_record_emergency(self, segments: List[RecordingSegment], total_hdd_capacity_gb: int) -> bool:
        """
        [ビジネスルール]
        新しい緊急録画が可能かどうかを判断する。
        """
        if self.emergency_area_percentage == 0:
            return False
        
        # --- ここではセグメントのサイズを仮に「時間」で代用して計算します ---
        # 全セグメントの合計録画時間を計算する
        # 緊急録画セグメントの合計録画時間を計算する
        # HDD全体の許容される緊急録画時間を計算する
        #   (total_duration * (emergency_area_percentage / 100))
        # 緊急録画の合計時間が、許容時間を超えていないかをチェックしてTrue/Falseを返す
        
        # 簡易的な実装例（セグメント数で代用）
        emergency_segments_count = len([s for s in segments if s.is_protected()])
        total_segments_capacity = total_hdd_capacity_gb * 1000 # 仮に1GBあたり1000セグメントと仮定
        allowed_emergency_count = total_segments_capacity * (self.emergency_area_percentage / 100)

        return emergency_segments_count < allowed_emergency_count


    def __repr__(self):
        return (f"StoragePolicy(emergency_area={self.emergency_area_percentage}%)")

```

  * `EmergencyAreaFullPolicy(Enum)`: 仕様でA案と決まった「満杯時は新規録画を停止する」というルールを、コードで明確に表現しています。
  * `find_segment_to_overwrite()`メソッド：このEntityの核となるビジネスルールの一つです。
    1.  `RecordingSegment`のリストを引数として受け取ります。
    2.  リストの中から、`is_protected()`が`False`を返すセグメント（通常・アラーム録画）だけを絞り込みます。
    3.  絞り込んだセグメントの中で、`start_time`が最も古いものを探し出して返します。この一連の判断ロジックが、このEntityの内部にカプセル化されています。
  * `can_record_emergency()`メソッド：こちらも重要なビジネスルールです。緊急録画領域の仕様（今回はセグメント数で簡易的に代用）に基づき、新しい緊急録画が可能かどうかを`True`/`False`で答えます。`UseCase`は、このメソッドの結果を見て、録画処理を進めるか、ユーザーにエラーを通知するかを判断します。

-----

### このクラスの鉄則

このEntityは、システムの戦略家です。

> **集合を統治せよ、ただしハードウェアには触れるな。 (Govern the collection, but do not touch the hardware.)**

  * この`StoragePolicy`は、個々の`RecordingSegment`オブジェクトの**集合全体**に対するルールを定義し、統治します。
  * `UseCase`は、「新しい通常録画を始めたいんだけど、どのセグメントを消せばいい？」といった**戦略的な判断**を、この`StoragePolicy`に問い合わせます。
  * これにより、最も複雑なビジネスルールの一つであるストレージ管理ロジックが、システムの中核であるこのEntityに集約され、見通しが良く、テストしやすい構造が保たれます。

-----

以上で、このシステムの核となる4つの`Entity`すべての洗い出しと実装が終わりました。

----
## 質問：「Entityの中にあるクラスはそれぞれが依存しない」と思っていたのですが、そういうわけではないんですね。

はい、その点はEntity同士の関係性を理解する上で非常に重要で、少し補足が必要です。「Entityはそれぞれが依存しない」という直感は、半分正しく、半分はより深い概念につながっています。

---
### 基本原則：「疎結合」であること

まず、考えの根幹にある**「Entity同士は疎結合であるべき」というのは完全に正しい**です。

`Book` Entityが、内部のプロパティとして `Member` Entityのインスタンスを保持する、といった**強い依存関係（密結合）を持つべきではありません**。なぜなら、それは「本」という概念が「会員」という概念なしでは成立しない、ということになり、Entityの独立性が損なわれるからです。

---
### 「集約」という関係性：`StoragePolicy` と `RecordingSegment`

では、なぜ`StoragePolicy`は`RecordingSegment`を知っているのでしょうか。
それは、この2つのEntityの関係が**「集約（Aggregation）」**と呼ばれる特別な関係だからです。

#### **例えるなら「学級委員」と「生徒たち」** * **`RecordingSegment`**: 一人一人の「生徒」です。それぞれの生徒は、自分の名前や成績（プロパティ）を知っていますが、クラス全体のことは知りません。
* **`StoragePolicy`**: 生徒たちをまとめる「学級委員」です。学級委員は、クラスの名簿（`List[RecordingSegment]`）を持っており、「このクラスで一番背が低い生徒は誰か？（一番古いセグメントはどれか？）」といった、**集合全体に対する問い**に答えることができます。

重要なのは、ここでの依存が**一方通行**である点です。
* **OK**: `StoragePolicy`（学級委員）が `RecordingSegment`（生徒）を知っている。
* **NG**: `RecordingSegment`（生徒）が `StoragePolicy`（学級委員）を知っている。

`StoragePolicy`の責務は、まさに「`RecordingSegment`の**集合全体**を統治すること」なので、`RecordingSegment`がどのようなものかを知っている必要があります。しかし、その依存は`__init__`でインスタンスを保持するような強いものではなく、メソッドの引数として一時的にリストを受け取る、という弱い形の依存になっています。

これは、DDD（ドメイン駆動設計）では**集約ルート（Aggregate Root）**と呼ばれる概念で、`StoragePolicy`が集約ルートとして、`RecordingSegment`の集合全体の一貫性を保つ責任を負っている、と考えることができます。

---
### まとめ

* Entity同士は、原則として互いに**疎結合**であるべきです。
* ただし、あるEntityが別のEntityの**集合を管理する責務**を持つ場合（集約）、そのEntityは管理対象のEntityをメソッドの引数として知ることがあります。
* この依存は常に**一方通行**であり、管理される側が管理する側を知ることはありません。

はい、「集約」の関係性は、`StoragePolicy`クラスのメソッドの**引数**に最も明確に表れています。

-----

### 集約のソースコード上の表現

`StoragePolicy`のコードをもう一度見てみましょう。

```python
# domain/entities.py (StoragePolicyクラスより抜粋)

from typing import List, Optional

class StoragePolicy:
    # ... ( __init__ は省略) ...

    def find_segment_to_overwrite(self, segments: List[RecordingSegment]) -> Optional[RecordingSegment]:
        #                                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        #                                  ここに「集約」の関係が表れている
        """
        [ビジネスルール]
        上書きすべき通常録画セグメントを特定する。
        """
        overwritable_segments = [s for s in segments if not s.is_protected()]
        # ...

    def can_record_emergency(self, segments: List[RecordingSegment], total_hdd_capacity_gb: int) -> bool:
        #                              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        #                              ここにも「集約」の関係が表れている
        """
        [ビジネスルール]
        新しい緊急録画が可能かどうかを判断する。
        """
        # ...
```

-----

### なぜこれが「集約」なのか？

#### **1. 「集合」を操作している**

`StoragePolicy`のメソッドは、単一の`RecordingSegment`を引数に取るのではなく、`RecordingSegment`の**リスト（`List[RecordingSegment]`）**、つまり\*\*「集合」\*\*を受け取っています。

そして、その集合全体に対して、「保護されていないものを絞り込む」「最も古いものを探す」「緊急録画の合計サイズを計算する」といった、**集合を統治するためのロジック**を実行しています。これが「集約」の本質です。

#### **2. 疎な関係（弱い依存）**

`StoragePolicy`は、`RecordingSegment`のリストを**永続的に所有していません**。`__init__`で`self.segments = []`のようにリストを保持するのではなく、判断が必要になったその都度、メソッドの引数として一時的に受け取っているだけです。

これは、`StoragePolicy`のインスタンスと`RecordingSegment`のインスタンスのライフサイクルが完全に独立していることを意味します。`StoragePolicy`がなくても`RecordingSegment`は存在できますし、その逆もまた然りです。このような所有関係のない（または弱い）「has-a」の関係を**集約**と呼びます。

#### **3. 一方向の依存**

`StoragePolicy`は`RecordingSegment`を知っていますが、`RecordingSegment`のコードの中には、`StoragePolicy`という言葉は一切出てきません。依存関係が**一方通行**であることも、このクリーンな関係性の特徴です。

`Use Case`層がこの2つのEntityを使うとき、

1.  まず`DataAccess`から全ての`RecordingSegment`のリストを取得します。
2.  そして、そのリストを`StoragePolicy`のメソッドに渡して、「次に消すべきセグメントはどれ？」と尋ねる、という流れになります。


