---
title: "旅行動画ハイライトを自動生成する——PySceneDetect + 顔検出 + xfadeクロスフェード"
emoji: "🎬"
type: "tech"
topics: ["python", "opencv", "ffmpeg", "scenedetect", "cli"]
published: true
---

![header](/images/articles/video-highlight-detection-pipeline/header.png)

## はじめに

旅行動画を自動で「ハイライト動画」にまとめるCLIツールを作りました。

```bash
uv run highlight --input /path/to/videos --duration 180
```

動画フォルダを渡すと、スコアの高いシーンを自動選択して、クロスフェード付きで結合したMP4を出力します。

この記事では以下の3つの実装ポイントを解説します。

1. **PySceneDetect** でシーン分割
2. **OpenCV Haar Cascade** で顔検出スコアリング
3. **FFmpeg xfade/acrossfade** で複数シーンのスムーズなクロスフェード結合（タイムベース統一が肝）

## 構成

```
highlight-maker/
├── src/highlight_maker/
│   ├── cli.py      # Clickベース CLIエントリポイント
│   ├── detector.py # シーン検出・スコアリング
│   ├── editor.py   # FFmpeg呼び出し・クロスフェード結合
│   └── utils.py    # SDカード検出など
├── tests/
│   └── test_detector.py
└── pyproject.toml
```

## シーン検出とスコアリング（detector.py）

### PySceneDetect でシーン分割

```python
from scenedetect import detect, ContentDetector

def split_scenes(video_path: str, threshold: float = 27.0) -> list[tuple]:
    return detect(video_path, ContentDetector(threshold=threshold))
```

`ContentDetector` はフレーム間の輝度・色変化を検出します。旅行動画は手持ち撮影のブレが多いので、`threshold` を高めにしても誤検出が出ます。長さフィルタ（1秒〜30秒）で短すぎ・長すぎを除外します。

### 顔検出でスコアを付ける

「顔が映っているシーン = 良いシーン」という単純な仮説で実装しました。OpenCV の Haar Cascade は古典的な手法ですが、**Raspberry Pi 5（ARM64）でも動作する**のが選んだ理由です。

```python
import cv2

_FACE_CASCADE_PATH = cv2.data.haarcascades + "haarcascade_frontalface_default.xml"

def _detect_face(frames: list) -> bool:
    cascade = cv2.CascadeClassifier(_FACE_CASCADE_PATH)
    for frame in frames:
        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        faces = cascade.detectMultiScale(
            gray, scaleFactor=1.1, minNeighbors=5, minSize=(30, 30)
        )
        if len(faces) > 0:
            return True
    return False
```

シーン内から5フレームをサンプリングして、1枚でも顔が検出されれば `+2点`。

### フレーム差分でモーション量を測る

静止しすぎ（風景固定）も動きすぎ（激しいブレ）も除外します。

```python
def _calc_motion_level(frames: list) -> float:
    if len(frames) < 2:
        return 0.0
    diffs = []
    for i in range(1, len(frames)):
        diff = cv2.absdiff(frames[i], frames[i - 1])
        diffs.append(float(diff.mean()))
    return sum(diffs) / len(diffs)

MOTION_THRESHOLD_LOW  = 5.0   # 静止しすぎ
MOTION_THRESHOLD_HIGH = 80.0  # 動きすぎ（ブレ）
```

適切な動き量のシーンに `+1点`。

### シーン選択ロジック

```python
def select_scenes(scored_scenes, target_duration: float) -> list:
    # スコア降順でソート
    sorted_scenes = sorted(scored_scenes, key=lambda x: x["score"], reverse=True)

    selected = []
    total = 0.0
    for scene in sorted_scenes:
        dur = scene["duration"]
        if total + dur <= target_duration:
            selected.append(scene)
            total += dur

    # 元の時系列順に並べ直す
    selected.sort(key=lambda x: x["start"])
    return selected
```

目標時間（デフォルト180秒）に収まるよう貪欲選択して、最後に時系列順へ戻します。

## クロスフェード結合（editor.py）

ここが一番詰まったところです。

### 2クリップの場合はシンプル

```python
def _build_xfade_filter(n: int, durations: list[float], fade_sec: float) -> str:
    if n == 2:
        offset = durations[0] - fade_sec
        return (
            f"[0:v][1:v]xfade=transition=fade:duration={fade_sec}:offset={offset}[vout];"
            f"[0:a][1:a]acrossfade=d={fade_sec}:c1=tri:c2=tri[aout]"
        )
```

`xfade` の `offset` はフェードが始まるタイミング（秒）です。1本目の終わりから `fade_sec` 秒前で開始します。

### 3クリップ以上：タイムベースを統一しないとエラーになる

複数の動画はフレームレートや「タイムベース（時間の刻み方）」がバラバラです。そのまま `xfade` に流すと：

```
[filtergraph] Error: Mismatched A/V sample rates
```

解決策：前処理で全クリップのタイムベースを統一します。

```python
tb = "1/90000"
pre_v = [f"[{i}:v]settb={tb},fps=30[pv{i}]" for i in range(n)]
pre_a = [f"[{i}:a]asetpts=PTS-STARTPTS[pa{i}]"  for i in range(n)]
```

- `settb=1/90000`：タイムベースを 1/90000秒（90kHz）に統一
- `fps=30`：フレームレートを統一
- `asetpts=PTS-STARTPTS`：音声の開始PTSを0にリセット

その上でクロスフェードをチェーンします。

```python
# [pv0][pv1] → xfade → [v01]
# [v01][pv2] → xfade → [v012]
# ...
cumulative = durations[0]
v_label = "pv0"

for i in range(1, n):
    out_label = "vout" if i == n - 1 else f"v{i}"
    offset = max(0.0, cumulative - i * fade_sec)
    xfade_parts.append(
        f"[{v_label}][pv{i}]xfade=transition=fade:duration={fade_sec}:offset={offset:.3f}[{out_label}]"
    )
    v_label = out_label
    cumulative += durations[i]
```

`offset = cumulative - i * fade_sec` がポイントで、複数フェードが重複しないよう累積時間から調整します。

### macOS / Linux でコーデックを自動切り替え

```python
import sys

def _get_video_codec() -> str:
    return "h264_videotoolbox" if sys.platform == "darwin" else "libx264"
```

macOS の VideoToolbox ハードウェアエンコードで10〜20倍の高速化になります。

## CLI オプション

```bash
uv run highlight \
  --input ./videos \      # 動画フォルダ（必須）
  --output out.mp4 \      # 出力パス（省略可）
  --duration 180 \        # 目標時間（秒）
  --fade 0.5 \            # クロスフェード秒数
  --min-scene 3.0 \       # 最小シーン秒数
  --frame-skip 4 \        # 解析フレームスキップ（4=5倍速）
  --sd-auto \             # SDカード自動検出
  --verbose               # 詳細ログ
```

`--frame-skip 4` はシーン解析を5フレームに1回に間引きます。精度はやや落ちますが5倍速になるので、長い動画を試し処理するときに便利です。

## ハマりポイントまとめ

### `scenedetect[opencv]` の extra はLinuxで動かない

`pyproject.toml` に `scenedetect[opencv]` と書くと、Linux（ARM64）でバイナリが解決できずにエラーになりました。`scenedetect` 単体 + `opencv-python-headless` を別途追加するのが安定します。

```toml
dependencies = [
    "scenedetect",
    "opencv-python-headless",  # Linux
    # "opencv-python",         # macOS
]
```

### 3クリップ以上の xfade は前処理必須

前処理なしで3本つなごうとすると再生時に映像がずれます。`settb` + `asetpts=PTS-STARTPTS` のセットを必ず入れてください。

### VideoToolbox はクリップが短すぎると失敗する

`h264_videotoolbox` は極端に短いクリップ（0.5秒以下）でエンコードに失敗することがあります。`--min-scene` で最小秒数を指定して回避します。

## まとめ

| ステップ | 手法 | 備考 |
|---|---|---|
| シーン分割 | PySceneDetect ContentDetector | threshold=27.0〜30.0 |
| 顔判定 | OpenCV Haar Cascade | ARM64でも動作、+2点ボーナス |
| モーション | フレーム差分MAE | 5.0〜80.0が適正範囲 |
| クロスフェード | FFmpeg xfade + settb前処理 | タイムベース統一が必須 |
| エンコード | VideoToolbox / libx264 自動選択 | macOSで10〜20倍高速 |

「顔が映っているかどうか」という単純なスコアリングでも、風景だけ・移動中のシーンを自然に除外できて、体感的にかなり使えるレベルになりました。DeepFace（笑顔スコア）や Whisper（感情語検出）を組み合わせるとさらに精度が上がりそうで、そこは次のステップです。

---

体験記・感想はNoteにも書いています。

→ [スモルが旅行動画から「良いシーン」を探すお手伝いをした話（Note）](%NOTE_URL%)
