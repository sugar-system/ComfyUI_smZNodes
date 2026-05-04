# ComfyUI V3 変更点まとめ — smZNodes 修正リファレンス

> **対象バージョン**: ComfyUI 0.10.0 以降（V3 Node API 導入後）  
> **対象ファイル**: `ComfyUI_smZNodes/nodes.py`  
> **症状**: `with_SDXL=True` で `AttributeError: 'CLIPTextEncodeSDXL' object has no attribute 'encode'`

---

## 1. V1 vs V3 ノード API の違い

### 1-1. 構造の比較

| 項目 | V1 (旧) | V3 (新) |
|------|---------|---------|
| 実行メソッド名 | `FUNCTION = "encode"` などで任意指定 | **`execute` に固定** |
| メソッド種別 | インスタンスメソッド `def encode(self, ...)` | **classmethod** `def execute(cls, ...)` |
| スキーマ定義 | `INPUT_TYPES()` + クラス属性 | `define_schema()` + `io.ComfyNode` 継承 |
| 戻り値 | `(output,)` タプル | `io.NodeOutput(output)` |
| 登録方法 | `NODE_CLASS_MAPPINGS = {...}` | `ComfyExtension` + `comfy_entrypoint()` |

### 1-2. コード例

```python
# ── V1 (Legacy) ──────────────────────────────────────
class CLIPTextEncodeSDXL:
    @classmethod
    def INPUT_TYPES(cls):
        return {"required": {"clip": ("CLIP",), "text_g": ("STRING", ...), ...}}

    RETURN_TYPES = ("CONDITIONING",)
    FUNCTION = "encode"          # ← ここで指定したメソッド名を呼ぶ
    CATEGORY = "advanced/conditioning"

    def encode(self, clip, width, height, crop_w, crop_h,
               target_width, target_height, text_g, text_l):
        tokens = clip.tokenize(text_g)
        tokens["l"] = clip.tokenize(text_l)["l"]
        # ... 長さ揃え処理 ...
        cond, pooled = clip.encode_from_tokens(tokens, return_pooled=True)
        return ([[cond, {"pooled_output": pooled, "width": width, ...}]], )
        #        ↑ タプルで返す

# ── V3 (Modern) ──────────────────────────────────────
from comfy_api.latest import io, ComfyNode

class CLIPTextEncodeSDXL(ComfyNode):
    @classmethod
    def define_schema(cls) -> io.Schema:
        return io.Schema(
            node_id="CLIPTextEncodeSDXL",
            inputs=[io.Clip.Input("clip"), io.String.Input("text_g"), ...],
            outputs=[io.Conditioning.Output()],
        )

    @classmethod
    def execute(cls, clip, width, height, crop_w, crop_h,    # ← 常に "execute"
                target_width, target_height, text_g, text_l) -> io.NodeOutput:
        tokens = clip.tokenize(text_g)
        tokens["l"] = clip.tokenize(text_l)["l"]
        # ... 長さ揃え処理 ...
        cond, pooled = clip.encode_from_tokens(tokens, return_pooled=True)
        return io.NodeOutput([[cond, {"pooled_output": pooled, "width": width, ...}]])
        #       ↑ NodeOutput で返す
```

---

## 2. 問題の根本原因

`comfy_extras/nodes_clip_sdxl.py` の `CLIPTextEncodeSDXL` が V3 に移行されたことで、
**`encode` メソッドが消えて `execute` に変わった**。

smZNodes の `nodes.py` は他のノードクラスのメソッドを**直接呼ぶ**設計だった：

```python
# smZNodes/nodes.py（旧コード）
from comfy_extras.nodes_clip_sdxl import CLIPTextEncodeSDXL

# 2箇所でこのパターンを使っている
return CLIPTextEncodeSDXL().encode(clip, ...)            # → AttributeError!
schedules = CLIPTextEncodeSDXL().encode(clip, ...)[0]    # → AttributeError!
```

これはそもそも「外部ノードクラスのメソッドを直接たたく」という壊れやすい設計で、
API 変更に弱い。**根本的な解決策はノードクラスを経由しない**こと。

---

## 3. `encode_from_tokens` の戻り値（重要）

### 3-1. 基本的な使い方（現在も有効）

```python
cond, pooled = clip.encode_from_tokens(tokens, return_pooled=True)
# cond   : torch.Tensor  — conditioning tensor
# pooled : torch.Tensor  — pooled output (SDXL で必要)
```

### 3-2. 新しい dict 形式（ComfyUI 0.8+ で追加）

```python
output = clip.encode_from_tokens(tokens, return_pooled=True, return_dict=True)
# output["cond"]           : conditioning tensor
# output["pooled_output"]  : pooled tensor
```

`return_pooled=True` のみの形式は**現在でも動く**（tuple で返る）。

### 3-3. 黒画像になる場合の確認事項

`cond, pooled = clip.encode_from_tokens(tokens, return_pooled=True)` を実行したあと：

```python
# デバッグ確認
print(type(cond))          # torch.Tensor であること
print(cond.shape)          # 何らかのテンソル形状
print(torch.isnan(cond).any())   # NaN が混入していないか
print(torch.all(cond == 0))      # ゼロ埋めになっていないか
print(type(pooled))        # torch.Tensor であること
```

`cond` が `dict` になっていた場合は V3 移行後の変化。その場合は：

```python
output = clip.encode_from_tokens(tokens, return_pooled=True, return_dict=True)
cond   = output["cond"]
pooled = output["pooled_output"]
```

---

## 4. 修正パターン

### 4-1. 第1の呼び出し（return 型）

```python
# ── 修正前 ──
return CLIPTextEncodeSDXL().encode(
    clip, width, height, crop_w, crop_h,
    target_width, target_height, text_g, text_l
)

# ── 修正後 ──
tokens = clip.tokenize(text_g)
tokens["l"] = clip.tokenize(text_l)["l"]
if len(tokens["l"]) != len(tokens["g"]):
    empty = clip.tokenize("")
    while len(tokens["l"]) < len(tokens["g"]):
        tokens["l"] += empty["l"]
    while len(tokens["g"]) < len(tokens["l"]):
        tokens["g"] += empty["g"]
cond, pooled = clip.encode_from_tokens(tokens, return_pooled=True)
return ([[cond, {
    "pooled_output": pooled,
    "width": width, "height": height,
    "crop_w": crop_w, "crop_h": crop_h,
    "target_width": target_width, "target_height": target_height,
}]], )
```

> **注意**: 末尾の `,` を忘れずに — タプルとして返す必要がある。

### 4-2. 第2の呼び出し（schedules 代入型）

```python
# ── 修正前 ──
schedules = CLIPTextEncodeSDXL().encode(
    clip, width, height, crop_w, crop_h,
    target_width, target_height, [text_g], [text_l]
)[0]

# ── 修正後 ──
tokens = clip.tokenize(text_g)
tokens["l"] = clip.tokenize(text_l)["l"]
if len(tokens["l"]) != len(tokens["g"]):
    empty = clip.tokenize("")
    while len(tokens["l"]) < len(tokens["g"]):
        tokens["l"] += empty["l"]
    while len(tokens["g"]) < len(tokens["l"]):
        tokens["g"] += empty["g"]
cond, pooled = clip.encode_from_tokens(tokens, return_pooled=True)
schedules = [[cond, {
    "pooled_output": pooled,
    "width": width, "height": height,
    "crop_w": crop_w, "crop_h": crop_h,
    "target_width": target_width, "target_height": target_height,
}]]
```

> **旧コードとの違い**:  
> - `[text_g]`, `[text_l]` → `text_g`, `text_l`（リストで包む必要はない）  
> - `[0]` が消える → タプルを剥がす必要がなく conditioning リストを直接作る

---

## 5. 黒画像問題の分析

エラーは消えたが黒画像になる場合、`schedules` の中身が想定と違う構造になっている可能性がある。

### 5-1. 該当箇所の前後コンテキストを確認

`schedules` を代入した後、次にどう使われているかを確認する：

```python
# パターン A: そのまま conditioning として使われる
#   → 修正後コードで OK のはず

# パターン B: smZNodes の schedule 変換パイプラインに通される
#   schedules = convert_schedules_to_comfy(schedules, steps)
#   → この場合 schedules は [cond, meta] のリストではなく
#     prompt_parser が返す Schedule オブジェクトのリストでなければならない
```

### 5-2. `with_SDXL=True` + プロンプト編集の処理フロー

smZNodes で `with_SDXL=True` のコードパスは大きく 2 種類ある：

```
A. プロンプト編集なし（シンプルなケース）
   → clip.tokenize → encode_from_tokens → conditioning 直接返す

B. プロンプト編集あり（"[word1:word2:0.5]" 構文）
   → prompt_parser でスケジュール生成
   → 各ステップごとに encoding
   → convert_schedules_to_comfy で ComfyUI 形式に変換
```

**line 117 付近のコードは B のパスにいる可能性**が高い。その場合は
`schedules` に代入する値が ComfyUI conditioning 形式ではなく、
smZNodes の内部 Schedule オブジェクトでなければならない。

---

## 6. 調査手順（次のステップ）

### Step 1: line 117 の前後 20 行を確認

```python
# nodes.py の該当箇所：どのような制御フローの中にあるか
# - if 文の条件は何か（on_sdxl? prompt_editing?）
# - schedules の次の行で何をしているか（return? 変換処理?）
```

### Step 2: 小さいデバッグを挟む

```python
cond, pooled = clip.encode_from_tokens(tokens, return_pooled=True)
schedules = [[cond, {...}]]

# デバッグ追加
import logging
logging.warning(f"[smZFix] schedules type: {type(schedules)}")
logging.warning(f"[smZFix] schedules[0][0] shape: {schedules[0][0].shape if hasattr(schedules[0][0], 'shape') else 'no shape'}")
```

### Step 3: 旧バージョンの CLIPTextEncodeSDXL.encode を手元で再現

旧 `comfy_extras/nodes_clip_sdxl.py` の `encode` は実質的にこれだけ：

```python
def _encode_sdxl(clip, width, height, crop_w, crop_h,
                 target_width, target_height, text_g, text_l):
    """旧 CLIPTextEncodeSDXL.encode の等価実装"""
    tokens = clip.tokenize(text_g)
    tokens["l"] = clip.tokenize(text_l)["l"]
    if len(tokens["l"]) != len(tokens["g"]):
        empty = clip.tokenize("")
        while len(tokens["l"]) < len(tokens["g"]):
            tokens["l"] += empty["l"]
        while len(tokens["g"]) < len(tokens["l"]):
            tokens["g"] += empty["g"]
    cond, pooled = clip.encode_from_tokens(tokens, return_pooled=True)
    return ([[cond, {
        "pooled_output": pooled,
        "width": width, "height": height,
        "crop_w": crop_w, "crop_h": crop_h,
        "target_width": target_width, "target_height": target_height,
    }]], )
```

これを `nodes.py` の上部に定義して、2 箇所の呼び出しを置き換えると
`CLIPTextEncodeSDXL` への依存を完全に切り離せる：

```python
# 第1の箇所
return _encode_sdxl(clip, width, height, crop_w, crop_h,
                    target_width, target_height, text_g, text_l)

# 第2の箇所
schedules = _encode_sdxl(clip, width, height, crop_w, crop_h,
                         target_width, target_height, text_g, text_l)[0]
```

---

## 7. 参考：V3 ノードクラスの `execute` を安全に呼ぶ方法（別解）

どうしてもノードクラス経由で呼びたい場合の安全な書き方：

```python
node_cls = CLIPTextEncodeSDXL
# V1/V3 どちらでも動く汎用呼び出し
if hasattr(node_cls, 'FUNCTION'):
    # V1: FUNCTION 属性があればその名前のメソッドを使う
    instance = node_cls()
    result = getattr(instance, node_cls.FUNCTION)(clip, ...)
else:
    # V3: execute classmethod
    result = node_cls.execute(clip, ...)

# 戻り値の正規化（V1=tuple, V3=NodeOutput）
if hasattr(result, '__iter__') and not isinstance(result, (list, tuple)):
    # NodeOutput の場合
    conditioning = result.outputs[0]  # NodeOutput の取り出し方（要確認）
else:
    conditioning = result[0]          # tuple の場合
```

> ただしこの方法は V3 の `NodeOutput` の内部構造に依存するため、
> **セクション 4 の直接実装のほうが推奨**。

---

## まとめ

| 問題 | 原因 | 対策 |
|------|------|------|
| `AttributeError: 'CLIPTextEncodeSDXL' has no attribute 'encode'` | V3 移行でメソッド名が `encode` → `execute` に変更 | ノードクラスを経由せず CLIP API を直接呼ぶ |
| 黒画像 | `schedules` の構造が downstream コードの期待と不一致 | line 117 前後のコンテキストを確認してデータ構造を合わせる |
| 将来の互換性 | 他のノードクラスのメソッドを直接呼ぶ設計は壊れやすい | `_encode_sdxl()` のようなヘルパー関数に切り出す |
