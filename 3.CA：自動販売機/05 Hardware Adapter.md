# 05 Hardware Adapter

# 🤖 Hardware Adapter : adapters/hardware.py

今回の題材で最も特徴的なアダプター、`Hardware Adapter`を解説します。

`DataAccess`がデータベースとの接続を担当したように、この`HardwareAdapter`は物理的なハードウェアとの接続を担当します。`Presenter`の前にこちらを解説することで、`UseCase`が依存する部品がすべて揃い、UI層の理解がしやすくなります。

## 🎯 このファイルの役割

このファイルは、`boundaries.py`で定義された`HardwareInterface`を、具体的な技術（今回はコンソールへのシミュレーション出力）で実装するアダプターです。

`UseCase`からの「スロットA1の商品を排出しろ」「30円のお釣りを返せ」といった抽象的なハードウェアへの指示を、コンソールの`print`文を使って「（ガコンッ！『お茶』を排出しました）」のように、物理的な動作をシミュレートする処理に変換します。

`UseCase`という「頭脳」と、物理的な「手足」の間に立つ、「神経系」のような役割を果たします。

![クリーンアーキテクチャ](../クリーンアーキテクチャ.png)

## ✅ このクラスにあるべき処理

⭕️ 含めるべき処理の例:

- `HardwareInterface`で定義されたメソッドの具体的な実装。
- 物理デバイスを制御するための低レベルなコード（例：モーターを制御するライブラリの呼び出し、シリアル通信など）。
- 今回の例では、ハードウェアの動作をコンソールに表示するシミュレーションコード。

❌ 含めてはいけない処理の例:

- ビジネスロジック（`UseCase`の責務）。
    - 例：「この商品に排出指示を出して良いか？」といった判断は行いません。ただ「排出しろ」という指示を忠実に実行するだけです。

## 💻 ソースコードの詳細解説

```python
# adapters/hardware.py

# 内側の世界の「境界」にのみ依存する
from application.boundaries import HardwareInterface

# -----------------------------------------------------------------------------
# Hardware Adapter
# - クラス図の位置: Hardware (DataAccessなどと同じ外部実装)
# - 同心円図の位置: Adapters (外側の円)
# -----------------------------------------------------------------------------
class ConsoleHardwareAdapter(HardwareInterface):
    """
    HardwareInterfaceの「コンソールシミュレーション」版実装。
    物理デバイスの動作をprint文で模倣する。
    """

    def dispense_item(self, slot_id: str):
        """
        [インターフェースの実装]
        'dispense_item'という抽象的な要求を、
        コンソールに排出メッセージを表示するという具体的な処理で実現する。
        """
        print(f"（ガコンッ！スロット {slot_id} の商品を排出しました）")

    def return_change(self, amount: int):
        """
        [インターフェースの実装]
        'return_change'という抽象的な要求を、
        コンソールにお釣りメッセージを表示するという具体的な処理で実現する。
        """
        print(f"（チャリン！お釣り {amount} 円を返却しました）")

```

- `class ConsoleHardwareAdapter(HardwareInterface):`
この一行で、「私は`HardwareInterface`という契約を守る実装クラスです」と宣言しています。
- `dispense_item`, `return_change`: `boundaries.py`で定義された抽象メソッドを、`print`文でシミュレーションとして実装しています。もしこれが本物の組み込みシステムであれば、この中にはモータードライバのライブラリを呼び出すコードなどが書かれます。

## 💡 ユニットテストでAdapterの正しさを証明する

アダプターのテストでは、それが正しく外部の世界（今回はコンソール）と対話するかを検証します。`print`文の出力をキャプチャして、期待通りのメッセージが表示されるかを確認します。

```python
# tests/adapters/test_hardware.py の例
import io
from contextlib import redirect_stdout

def test_dispense_itemは正しいメッセージをコンソールに出力する():
    # 1. Arrange (準備)
    hardware = ConsoleHardwareAdapter()
    # printの出力先を、通常のコンソールから一時的なテキストバッファに変更
    f = io.StringIO()

    # 2. Act (実行)
    with redirect_stdout(f):
        hardware.dispense_item("A1")

    # 3. Assert (検証)
    # バッファに書き込まれた内容を取得して検証
    output = f.getvalue()
    assert "スロット A1 の商品を排出しました" in output

```

## 🐍 PythonとC言語の比較（初心者の方へ）

- Python (オブジェクト指向): このように、ハードウェア制御のコードを`ConsoleHardwareAdapter`というクラスにカプセル化し、`HardwareInterface`という抽象を通じて利用します。
- C言語 (手続き型): `dispense_item_motor_driver.c`のようなファイルに関数をまとめることはできますが、インターフェースの強制力がないため、ビジネスロジックから直接`PORTB |= (1 << PB3);`のような低レベルなコードを呼び出してしまいがちです。これにより、ハードウェアとソフトウェアが密結合になります。

## 🛡️ このクラスの鉄則

このクラスは、忠実な「手足」に徹します。

> 指示に従い、物理的に動け。 (Follow instructions and act physically.)
> 
- このクラスはビジネスルールについて思考しません。`UseCase`という頭脳からの指示を、物理的な作業として忠実に実行するだけです。
- この`HardwareInterface`と`ConsoleHardwareAdapter`があるおかげで、高価な実機がなくても`UseCase`のロジックをテストできます。また、将来ハードウェアの仕様が変更になったとしても、修正が必要なのはこの`hardware.py`ファイルだけであり、ビジネスロジックには一切影響がありません。これが、組み込み開発においてクリーンアーキテクチャが非常に強力である理由です。

---

## ❓ Q\&A

### *Q. `Presenter`と`Hardware Adapter`のどちらを先に学習したほうがいいでしょうか？

**結論から言うと、`Presenter`の前に`Hardware Adapter`を解説する方が、より学習効果が高いでしょう。**

「`Presenter`の前にこちらを解説することで、`Use Case`が依存する部品がすべて揃い、理解がしやすくなります」という理由は、まさにその通りで、教育的な観点から非常に優れた判断です。

### `Hardware Adapter`を先に見せる理由

1. **`UseCase`の全体像が完成する 🧩**`SelectItemUseCase`は、`ItemDataAccessInterface`と`HardwareInterface`という2つの主要な「道具」を使って仕事をします。UI層（`Presenter`など）を解説する前に、`UseCase`が直接依存するこれらの道具（アダプター）をすべて揃えておくことで、`UseCase`の役割である「オーケストレーション」の全体像がより明確になります。
2. **組み込み題材の「主役」を先に紹介できる 🤖**
この第三巡の最もユニークで重要な要素は、間違いなく`HardwareInterface`とその実装です。この「主役」を先に登場させることで、学習者は「UIはこれまでの応用だが、ハードウェアの扱い方が今回の新しい学びだ」と強く意識でき、学習のメリハリがつきます。
3. **学習の流れがスムーズになる ➡️**
「内側から外側へ」という流れで解説を進めるのが、クリーンアーキテクチャの学習では非常に効果的です。
`Entity` → `UseCase` → **`UseCase`が使うAdapter (`DataAccess`, `Hardware`)** → `UseCase`を使うAdapter (`Controller`, `View`, `Presenter`) という順番は、依存関係の方向と一致しており、非常に自然で理解しやすい流れです。