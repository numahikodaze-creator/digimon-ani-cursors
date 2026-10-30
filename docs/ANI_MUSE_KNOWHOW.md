# Windows 11 アニメーションマウスカーソル（.ani）制作ノウハウ

> Muse側プロジェクトでの実証記録（2026-10-30）。ユーザーから提供された技術記録をGitHub参照用に整形したもの。**このリポジトリではMuseの実機成功例を独立再検証していない。** 規格上の仕様と実機サンプル固有の観測を区別すること。

## 1. 実機動作した構成

| 項目 | Muse側の成功例 |
|---|---|
| キャンバス | 64×64 px, RGBA透過 |
| フレーム | 無圧縮32bit BMP（DIB）を収めたCUR |
| ANI | RIFF/ACON, `anih → rate → seq  → LIST(fram)` |
| flags | 3（AF_ICON=1, AF_SEQUENCE=2） |
| rate / seq | 各フレームのjiffies / 再生順を明示 |
| ANDマスク | alpha==0の画素のみbit=1 |
| 時間 | 6 jiffies = 100ms （1 jiffy=1/60秒） |
| ホットスポット | 全フレームCUR内で同一座標 |
| 配送 | ZIPを使用 |

試作：白い矢印（固定）＋オレンジ小円の明滅、8フレーム、ループ800ms、ホットスポット(12,4)、ANI 135,876 bytes。SHA-256: `76bac96d7be0512b2d1610efcf8a77cb426e2537833be9cf7fe34972e0f35701`。

**要注意：RIFFサイズ欄**。Museの成功ファイルはサイズ欄を「ファイル全体のバイト数」としていたと報告されている。しかしRIFFの規格上、この欄は「総バイト数−8」。**非標準の実測例を規格上の正解として流用しない。** 別経路で標準値と異常値の両方を実機で切り分ける余地を残す。

## 2. バイナリ構造

### ANI / ACON

- `RIFF` + 32bit little-endian size + `ACON`
- `anih` (36 bytes): cbSize=36, cFrames=N, cSteps=N, cx/cy=64, cBitCount=32, cPlanes=1, jifRate=既定のjiffies, flags=3
- `rate`: 32bit jiffies × N
- `seq `: 32bitフレーム番号 × N
- `LIST` (`fram`): `icon`チャンクにCURファイル全体をN個格納
- RIFFチャンクは奇数バイト長の場合、末尾を1バイトでパディング

### CUR / DIB

- ICONDIR: reserved=0, type=2, count=1
- ICONDIRENTRY (16 bytes): width/height=64, colors/reserved=0, hotspotX/Y, bytesInRes, imageOffset=22
- BITMAPINFOHEADER: biSize=40, biWidth=64, biHeight=128 (実高さ×2), biPlanes=1, biBitCount=32, biCompression=0
- XOR: bottom-up BGRA 4 bytes/px（完全透明ピクセルのRGBは0に）
- AND: bottom-up 1bpp（32bit境界へ行パディング）、alpha==0ならbit=1
- 64×64ならDIBは16,936バイト、CUR全体16,958バイト

Museは既存テーマKinetik Aqua Xの実動作ファイルを解析し、上記DIBの構造と寸法が一致したと記録。

## 3. 実証経緯と未確定事項

1. **v1** PNG圧縮フレーム入りANI：生成側パースは通ったが、Windowsでは適用できなかった。
2. **v2** BMP化、rate、サイズ違い、静止CURなど複数候補：いずれも実機で失敗。静止CURまで失敗した。
3. **転送経路を疑う**：例として送信側135,796 bytesのANIと受信側エクスプローラー表示172KBが不一致。
4. **v3** 既存サンプルに構造を合わせ、ZIPで受け渡し：Windows 11実機で適用・再生成功。

**真因は未確定。** v1/v2とv3は構造も配送方法も異なるため、PNG/BMP、チャンク、RIFFサイズ、ANDマスク、配送の寄与は単独では切り分けられていない。なおエクスプローラーのKB表示とbyte数は表示単位や丸めの影響もあり得る。実際の破損確定には受渡し前後のSHA-256照合が必要。

## 4. 再現性重視の制作パイプライン

AIによる動画生成ではなく、座標とピクセルを固定できるプログラム処理を採用する。

### 1種あたりの入力

1. `base.png`: 64×64 RGBA、固定レイヤー
2. `mask.png`: 64×64グレースケール。255の部分へ効果。ハローを出す際は余白とグラデーションを持たせる
3. `config.json`: 効果、色、フレーム数、jiffies、ホットスポット

```json
{
  "name": "normal_select",
  "base": "base.png",
  "mask": "mask_glow.png",
  "effect": "glow_pulse",
  "effect_color": [255, 140, 0],
  "frames": 8,
  "jiffies": 6,
  "hotspot": [12, 4]
}
```

### 効果の候補

- `glow_pulse`: マスク形状をぼかし、周期的に発光量を増減
- `brightness_pulse`: マスク内のみ明るさ変更
- `reflection_sweep`: マスク内に斜めの反射帯を移動

**固定領域のハード保証**：加工後、マスク値0のピクセルを元のベースから完全復元する。ハローを含めて変化させたい領域はマスクに含める。全フレームのループ継ぎ目にも注意する。

工程：素材読込・検証 → エフェクト適用・固定領域復元 → 連番PNG → DIB CUR → ANI → 構造/画素検証 → ZIPにまとめる。

## 5. 配送

Museの実験ではANI/CUR直接受渡しで失敗したが、ZIP経由のv3で成功した。そのため暫定運用として**成果物はZIPに格納**する。ZIP化前後のSHA-256と正確なbytes値を記録し、破損検証を可能にする。

## 6. 検証項目

- ANIの再パース：`anih`, `rate`, `seq `, `LIST(fram)`, `icon`数、時間、サイズ
- CUR DIB往復デコード：元のフレームとの画素一致（透明画素RGB=0正規化）
- マスク外の全フレーム差分ゼロ、マスク内の適切な変化
- 全フレームで同一ホットスポット
- 背景透過とループ継ぎ目の確認
- 最終的に**Windows 11実機で適用・再生・クリック位置確認**。自作パーサーの成功だけで実機互換としない

## 7. 17種類への展開

各カーソル役割でベース画像・マスク・設定JSONを交換して同じパイプラインを使う。アニメーションが不要なら静止CURも選択肢。最終的なテーマ一括導入は`install.inf`の検討余地があるが、本番制作前にシステムフォルダへのコピーやレジストリ編集を自動実行しない。

対象：通常選択、ヘルプ選択、バックグラウンドで作業中、待ち状態、精密選択、テキスト選択、手書き、使用不可、上下/左右/斜め2種リサイズ、移動、代替選択、リンク選択、場所の選択、人の選択。

## 8. ハマりどころ

- 自動検証OKでもWindowsで無言の適用失敗があり得る
- 静止CURでの切り分け、正常ファイルとの構造diffが有用
- 形式候補は少数の試験ファイルに分けて実機比較する
- 転送経路検証はファイルサイズだけでなく**SHA-256**で行う
- AI動画で指定領域外のピクセル完全一致を保証しない

## 9. Museが提示した生成コード例

以下は**報告されたv3構造を再現する参考コード**で、RIFFサイズの非標準値を維持している。規格準拠実装を作る場合はRIFFサイズ欄を修正し、Windows実機で比較すること。

```python
import struct

def cur_bmp_v3(img, hotspot):
    w, h = img.size
    px = img.load()
    xor = bytearray()
    for y in range(h - 1, -1, -1):
        for x in range(w):
            r, g, b, a = px[x, y]
            if a == 0:
                r = g = b = 0
            xor += bytes((b, g, r, a))
    row_bytes = ((w + 31) // 32) * 4
    mask = bytearray()
    for y in range(h - 1, -1, -1):
        row = bytearray(row_bytes)
        for x in range(w):
            if px[x, y][3] == 0:
                row[x // 8] |= 0x80 >> (x % 8)
        mask += row
    dib = struct.pack("<IiiHHIIiiII", 40, w, 2*h, 1, 32, 0,
                      len(xor)+len(mask), 0, 0, 0, 0) + bytes(xor) + bytes(mask)
    header = struct.pack("<HHH", 0, 2, 1)
    entry = struct.pack("<BBBBHHII", w, h, 0, 0, hotspot[0], hotspot[1],
                        len(dib), 22)
    return header + entry + dib

def chunk(tag, data):
    out = tag + struct.pack("<I", len(data)) + data
    if len(data) % 2:
        out += b"\x00"
    return out

def make_ani_v3(cur_frames, size, jiffies):
    anih = struct.pack("<9I", 36, len(cur_frames), len(cur_frames),
                       size, size, 32, 1, jiffies, 3)
    rate = chunk(b"rate", b"".join(struct.pack("<I", jiffies) for _ in cur_frames))
    seq = chunk(b"seq ", b"".join(struct.pack("<I", i) for i in range(len(cur_frames))))
    fram = b"fram" + b"".join(chunk(b"icon", c) for c in cur_frames)
    body = b"ACON" + chunk(b"anih", anih) + rate + seq + chunk(b"LIST", fram)
    return b"RIFF" + struct.pack("<I", 8 + len(body)) + body
```

## 10. 適用範囲

- この記録はWindows 11での**報告済み成功例**を扱う。多解像度格納（32/48/256等）は未検証
- 生成側環境：Linux / Python 3.12 / Pillow 10.2
- 完成したANIを他のPCに配布する場合の権利・動作互換性は別途検討

