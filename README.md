# PNG なのに HDR で光る? PQ + ICC で部分グロー画像を作る仕組み

> 8bit の普通の PNG を HDR ディスプレイで光らせる仕掛けを解剖する。PQ 符号化、ICC プロファイル埋め込み、そして X(Twitter) で `iCCP` が落ちないようにする工夫まで。
>
> Tags: HDR, PQ, PNG, Canvas, ICC, Twitter

「X (Twitter) のタイムラインで一部だけ眩しく光るアイコン」を見たことがあるだろうか。一見ただの PNG なのに、HDR 対応ディスプレイだと sRGB の白を遥かに超えて発光する。あれを自前のスクリプトで再現しつつ、**任意の場所だけ部分的に光らせる**ようにするまでの仕組みをまとめる。

ベースは「ひかるやつ X用ピカピカアイコンメーカー」のフロントエンドコードを読み解いた結果で、そこに「円・矩形・回転で部分マスク」「Per-pixel な輝度補間」を足している。

同梱の `partial_glow.html` をブラウザで開けば動かせる (HDR 表示には HDR 対応ディスプレイ + Safari/Chrome が必要)。

## TL;DR

- 8bit RGBA PNG のピクセル値を **PQ (SMPTE ST 2084)** で再符号化する
- 「これは sRGB ではなく BT.2100 PQ です」というメタデータを **`iCCP` チャンク** に埋め込む
- HDR 対応ブラウザはピクセル値を **0–10000 nits** の絶対輝度として解釈し、SDR の白(~203 nits)を超えて発光させる
- 部分グローはピクセルごとに「目標 nits」を変えるだけで実装できる (PQ 自体はピクセル独立な変換)
- X (Twitter) で光らせるには **出力を 400×400 ぴったりにする + PQ で書き出す** ことが必須。HLG だと無視され、サイズが少しでも違うとサーバ側で再エンコされて `iCCP` が消える
- Web Share API でアプリへ直接渡すと再エンコの確率を更に下げられる

---

## 1. なぜ普通の PNG は HDR で光らないのか

普通の sRGB PNG は「ピクセル値 255 = ディスプレイの白 (~80–203 nits)」という暗黙の約束で動いている。HDR ディスプレイは 1000 nits 以上を出せるのだが、画像側から「ここはもっと明るく光らせて」と指示する手段がないため、自動でブーストされたりはしない。

HDR ディスプレイに「これは HDR 映像で、ピクセル値は絶対輝度を表してるよ」と伝えるためには、

1. **転送関数 (TF)** を sRGB ガンマから **PQ** か **HLG** に切り替える
2. **色域** を sRGB から **BT.2020 / DCI-P3** に広げる
3. それをファイルのメタデータで明示する

の 3 つが必要になる。今回は 1 と 3 で勝負する。

---

## 2. 仕掛けの全体像

ざっくり 3 ステップ:

```
[元の sRGB ピクセル]
    ↓ sRGB ガンマを外して linear に
[linear 0–1]
    ↓ "ターゲット nits" を掛ける
[輝度 0–10000 nits]
    ↓ PQ EOTF⁻¹ で符号化
[PQ 符号 0–1] → 8bit に量子化して PNG に書く

[PNG]
    ↓ iCCP チャンクに "Rec. ITU-R BT.2100 PQ" の ICC プロファイルを差し込む
[HDR PNG]
```

ピクセル値そのものは 0–255 のままなので、HDR 非対応の環境では「ちょっと色がおかしい sRGB 画像」として表示される (許容できる)。HDR 対応環境では iCCP を見て「PQ だ」と気付き、ピクセル値を絶対輝度に展開して描画してくれる。

---

## 3. PQ (SMPTE ST 2084) を実装する

PQ の EOTF⁻¹ (linear nits → 符号値) は次の式 (`Y = L / 10000`、L は nits):

```
V = ((c1 + c2 * Y^m1) / (1 + c3 * Y^m1))^m2
```

定数は仕様で固定されている:

- `m1 = 2610/16384 ≈ 0.1593`
- `m2 = (2523/4096)*128 ≈ 78.84`
- `c1 = 3424/4096 ≈ 0.8359`
- `c2 = (2413/4096)*32 ≈ 18.852`
- `c3 = (2392/4096)*32 ≈ 18.687`

JavaScript で書くとそのまま 1:1 で写せる:

```js
const PQ_M1 = 2610 / 16384;
const PQ_M2 = (2523 / 4096) * 128;
const PQ_C1 = 3424 / 4096;
const PQ_C2 = (2413 / 4096) * 32;
const PQ_C3 = (2392 / 4096) * 32;

function srgbToLinear(v) {
  const t = v / 255;
  return t <= 0.04045 ? t / 12.92 : Math.pow((t + 0.055) / 1.055, 2.4);
}

function linearToPQ(nits) {
  const t = Math.pow(Math.max(0, Math.min(10000, nits)) / 10000, PQ_M1);
  return Math.pow((PQ_C1 + PQ_C2 * t) / (1 + PQ_C3 * t), PQ_M2);
}
```

ピクセル単位のエンコードはこんな感じになる:

```js
function encodePixel(r, g, b, nits) {
  const lr = srgbToLinear(r) * nits;
  const lg = srgbToLinear(g) * nits;
  const lb = srgbToLinear(b) * nits;
  return [
    Math.round(linearToPQ(lr) * 255),
    Math.round(linearToPQ(lg) * 255),
    Math.round(linearToPQ(lb) * 255),
  ];
}
```

> **Tip:** `nits` を SDR 基準白の **203 nits** (BT.2408 推奨) に設定すれば sRGB 白がそのまま「SDR 白」として再現できる。**1000–4000 nits** にすれば HDR で光って見える。

---

## 4. PNG に PQ プロファイルを埋め込む

PQ なピクセルを書くだけだと、ビューア側は「これは sRGB だな」と思って普通にデコードしてしまう。「いやこれは PQ です」と伝えるための仕掛けが **`iCCP` チャンク** だ。

### 4.1 ICC プロファイル

Apple が macOS / iOS に同梱している `Rec. ITU-R BT.2100 PQ.icc` を使うのが楽。中身を覗くと:

- `acsp` signature (ICC profile)
- CMM = `appl` (Apple)
- ICC v4.0
- Device class = `mntr` (Display)
- 8 個のタグ: `desc`, `cprt`, `wtpt`, `A2B0`, `B2A0`, `chad`, `cicp`, `lumi`

`cicp` タグが重要で、ここに「Primaries = 9 (BT.2020), Transfer = 16 (PQ), Matrix = 0 (Identity), Range = Full」が記録されている。ブラウザはこれを見て即座に HDR と認識する。

### 4.2 PNG のチャンク形式

PNG は `[長さ(4)] [タイプ(4)] [データ] [CRC32(4)]` のチャンクの並び。`iCCP` チャンクのペイロードは:

```
"ICC Profile" \0 \x00 deflate(icc_bytes)
```

(プロファイル名、null 終端、圧縮方式バイト 0=deflate、deflate 圧縮された ICC データ)

実装はこう:

```js
// PNG 用 CRC32 テーブル
const CRC_TABLE = (() => {
  const t = new Uint32Array(256);
  for (let n = 0; n < 256; n++) {
    let c = n;
    for (let k = 0; k < 8; k++) c = (c & 1) ? (0xedb88320 ^ (c >>> 1)) : (c >>> 1);
    t[n] = c >>> 0;
  }
  return t;
})();

function crc32(bytes) {
  let c = 0xffffffff;
  for (let i = 0; i < bytes.length; i++) {
    c = (c >>> 8) ^ CRC_TABLE[(c ^ bytes[i]) & 0xff];
  }
  return (c ^ 0xffffffff) >>> 0;
}

function makeChunk(type, data) {
  const n = data.length;
  const out = new Uint8Array(8 + n + 4);
  const dv = new DataView(out.buffer);
  dv.setUint32(0, n);
  for (let i = 0; i < 4; i++) out[4 + i] = type.charCodeAt(i);
  out.set(data, 8);
  // CRC は type + data に対して計算する
  const crcSrc = new Uint8Array(4 + n);
  crcSrc.set(out.slice(4, 8), 0);
  crcSrc.set(data, 4);
  dv.setUint32(8 + n, crc32(crcSrc));
  return out;
}

async function makeICCPChunk(iccBytes) {
  const name = new TextEncoder().encode("ICC Profile");
  const cs = new CompressionStream("deflate");
  const w = cs.writable.getWriter();
  w.write(iccBytes); w.close();
  const compressed = new Uint8Array(await new Response(cs.readable).arrayBuffer());

  const payload = new Uint8Array(name.length + 2 + compressed.length);
  payload.set(name, 0);
  payload[name.length] = 0;       // null terminator
  payload[name.length + 1] = 0;   // compression method (0 = deflate)
  payload.set(compressed, name.length + 2);
  return makeChunk("iCCP", payload);
}
```

### 4.3 チャンクを差し替えて出力する

`canvas.toBlob('image/png')` で得た PNG は、ブラウザが勝手に `sRGB` / `gAMA` / `cHRM` / 場合によっては `iCCP` を入れてくる。これらは PQ プロファイルと矛盾するので**全部剥がして**から自前の `iCCP` を `IHDR` 直後に差し込む。

```js
async function injectICC(pngBlob, iccBytes) {
  const buf = new Uint8Array(await pngBlob.arrayBuffer());
  const iccChunk = await makeICCPChunk(iccBytes);
  const strip = new Set(["iCCP", "sRGB", "gAMA", "cHRM"]);

  const out = [buf.slice(0, 8)]; // PNG signature
  const dv = new DataView(buf.buffer, buf.byteOffset, buf.byteLength);
  let pos = 8, inserted = false;
  while (pos < buf.length) {
    const len = dv.getUint32(pos);
    const type = String.fromCharCode(buf[pos+4], buf[pos+5], buf[pos+6], buf[pos+7]);
    const end = pos + 8 + len + 4;
    if (!strip.has(type)) out.push(buf.slice(pos, end));
    if (type === "IHDR" && !inserted) { out.push(iccChunk); inserted = true; }
    pos = end;
    if (type === "IEND") break;
  }
  const total = out.reduce((s, a) => s + a.length, 0);
  const merged = new Uint8Array(total);
  let off = 0;
  for (const a of out) { merged.set(a, off); off += a.length; }
  return new Blob([merged], { type: "image/png" });
}
```

これで「中身は普通の PNG、メタデータだけ PQ」な HDR PNG が完成する。

---

## 5. 部分的に光らせる: マスクの導入

ここからが拡張パート。元のアイコンメーカーは画像全体に同じ「輝度倍率 `t`」を掛けていたが、

> **ピクセルごとに `t` を変える**

だけで部分グローになる。PQ 自体はピクセル独立な変換なので、ピクセル間に何ら干渉はない。

### 5.1 マスクの設計

UI 側で **円 (楕円) / 矩形** の形を任意個置き、各形に `brightness`(nits) と `feather`(px、エッジのぼかし幅) を持たせる。各ピクセル位置 `(px, py)` で:

1. すべての形について「マスク値 `m ∈ [0,1]`」を計算
2. 形 i に対して **目標輝度** を `bg + (brightness_i − bg) × m_i` で補間
3. 複数形が重なっているピクセルでは**最大値**を採用

```js
let nits = bgNits;
for (const s of shapes) {
  const m = maskValue(s, px, py);
  if (m > 0) {
    const t = bgNits + (s.brightness - bgNits) * m;
    if (t > nits) nits = t;
  }
}
```

### 5.2 楕円マスク (フェザー付き)

楕円は中心 `(cx, cy)`、半径 `(rx, ry)`、ぼかし `f` で表す。正規化距離 `d = √((u² + v²))` を使えば、`d < 1` が内部、`d ≥ 1` が外部:

```js
function ellipseMask(s, px, py) {
  const u = (px - s.cx) / s.rx;
  const v = (py - s.cy) / s.ry;
  const d = Math.sqrt(u * u + v * v);
  if (d >= 1) return 0;
  // フェザー幅をピクセルから正規化距離へ
  const rMin = Math.min(s.rx, s.ry);
  const fn = s.f / rMin;
  return fn <= 0 ? 1 : Math.min(1, (1 - d) / fn);
}
```

楕円なのでフェザー幅は短い軸基準にしておくと自然に見える。

### 5.3 矩形マスク (フェザー付き)

矩形は「エッジまでの最小距離(=inset)が `f` 未満なら線形に落とす」だけ:

```js
function rectMask(s, px, py) {
  if (px < s.x || px > s.x2 || py < s.y || py > s.y2) return 0;
  const inset = Math.min(px - s.x, s.x2 - px, py - s.y, s.y2 - py);
  return s.f <= 0 ? 1 : Math.min(1, inset / s.f);
}
```

### 5.4 回転対応

形に `rotation` (ラジアン) を持たせ、当たり判定 / マスク評価では**入力ピクセルを形のローカル空間に逆回転**してから上の関数を呼ぶ。コアはこの 2 行:

```js
function toLocal(s, px, py) {
  const dx = px - s.cx, dy = py - s.cy;
  const c = Math.cos(-s.rotation), si = Math.sin(-s.rotation);
  return [s.cx + dx * c - dy * si, s.cy + dx * si + dy * c];
}
```

UI 側のリサイズハンドルは「**ドラッグした角の対角を世界座標で固定**」する流儀にすると、回転を保ったままサイズが直感的に変えられる。中心は `(fixed + cursor) / 2` で求まる。

---

## 6. UI からエクスポートまで

サイズ・形・スライダーを操作した状態で「書き出し」を押した時のフローはこう:

```
[Canvas プレビュー (低解像度)]
   ↓ 操作した形リスト + スライダー値
[フルサイズのオフスクリーン canvas に元画像を描画]
   ↓ getImageData
[各ピクセルでマスク評価 → 目標 nits 計算 → PQ 符号化]
   ↓ putImageData → toBlob('image/png')
[普通の PNG Blob]
   ↓ injectICC()
[HDR PNG Blob]
   ↓ navigator.share({ files: [...] })
[X アプリへ直接ファイルとして渡す]
```

ループの中身は次のとおり (`d` は `ImageData.data` = Uint8ClampedArray):

```js
for (let py = 0; py < H; py++) {
  for (let px = 0; px < W; px++) {
    let nits = bgNits;
    for (const s of shapes) {
      const [lpx, lpy] = s.rotated ? toLocal(s, px, py) : [px, py];
      const m = s.type === "ellipse"
        ? ellipseMask(s, lpx, lpy)
        : rectMask(s, lpx, lpy);
      if (m > 0) {
        const t = bgNits + (s.brightness - bgNits) * m;
        if (t > nits) nits = t;
      }
    }
    const i = (py * W + px) * 4;
    if (d[i + 3] === 0) continue; // 完全透明はそのまま
    d[i]     = Math.round(linearToPQ(srgbToLinear(d[i]    ) * nits) * 255);
    d[i + 1] = Math.round(linearToPQ(srgbToLinear(d[i + 1]) * nits) * 255);
    d[i + 2] = Math.round(linearToPQ(srgbToLinear(d[i + 2]) * nits) * 255);
  }
}
```

> **Note:** ピクセルあたり呼び出されるのは `srgbToLinear × 3` + `linearToPQ × 3` 程度。`Math.pow` がボトルネックなので、本気で速くしたければ 8bit LUT (256 エントリ) で `srgbToLinear` を置き換えると数倍速くなる。

---

## 7. X (Twitter) で実際に光らせる罠

ここが今回一番ハマった所。**「ファイルとして合ってる」と「X で光る」は別問題**。

### 7.1 X の画像パイプライン

X は投稿画像をサーバ側で再エンコする。何が起きるか:

- ある程度大きい PNG は WebP / JPEG に変換される
- その過程で `iCCP` をはじめとする補助チャンクが**ごっそり剥がされる**
- 結果、HDR タグが消えて普通の sRGB 画像になる

つまり「ファイル単位では完璧な PQ PNG なのに、X 経由だと光らない」が起きる。

### 7.2 実測で分かった X の制約

実際に色々試して分かったのが次の 2 つの**固い**制約:

1. **転送関数は PQ のみ。HLG は通らない。**
   HLG で書いた `iCCP` は X 側のパイプラインで剥がされ、SDR にフォールバックされる。BT.2100 PQ (`cicp` の Transfer = 16) でないと光らない。
2. **サイズは 400×400 ぴったり。**
   401×400 でも 800×800 でも、サイズが少しでもずれると再エンコが走って `iCCP` がもれなく消える。元のひかるやつアプリが律儀に 400×400 を吐いている理由はこれ。

つまり**「BT.2100 PQ で書いた 400×400 PNG」だけが X で光る**、という非常に狭い窓を通すゲームになる。

### 7.3 Web Share API の併用

400×400 PQ で吐いてもまだ落ちる時がある。さらに確率を上げるには:

- **`navigator.share({ files: [file] })` で X アプリに直接渡す** — Web の投稿フォーム経由だと変換が走りやすい
- iOS Safari → 共有シート → X アプリ、の経路を取る
- ダウンロードしてから「写真を選択」で添付すると再エンコ確率が跳ね上がるので避ける

### 7.4 対策まとめ

| 罠 | 対策 |
|---|---|
| HLG だと X で剥がれる | 必ず **PQ (BT.2100)** で書き出す |
| サイズが 400×400 でないと iCCP が消える | 出力を **400×400 ピッタリ**に固定 |
| Web 経路だと再エンコ確率が上がる | `navigator.canShare({files})` を確認して `navigator.share` でファイル渡し |
| HDR 非対応ブラウザでプレビュー | プレビューは SDR + 暖色のオーバーレイで「光る場所」だけ示す |

---

## 8. 動かしてみる

実装は 1 ファイルの `partial_glow.html` にまとまっていて、ICC プロファイルは base64 で内包してあるので `file://` でもそのまま開ける。

1. 画像を読み込む
2. 円 / 矩形ボタンでドラッグして範囲を作る
3. 四隅で拡縮、上の青丸で回転 (Shift で 15° スナップ)、内側ドラッグで移動
4. 輝度 (nits) とフェザー (px) を調整
5. **出力サイズは 400×400** を選ぶ (X に投げるなら必須)
6. **iOS Safari なら「X/共有」ボタン** → X アプリで投稿

HDR 対応の Safari / Chrome で `file://` 経由で開けばローカルでも本来の発光が確認できる。

---

## 9. もう少し踏み込みたい人へ

- **`cICP` チャンク**: PNG にはコンパクトな代替メタデータとして `cICP` チャンク (4 バイトで Primaries/Transfer/Matrix/Range を指定) がある。最近のブラウザはこれもサポートしているので、`iCCP` の代わりに使うとファイルサイズが小さくなる。ただし対応状況は `iCCP` の方がまだ広い。
- **HLG**: PQ ではなく **Hybrid Log-Gamma (BT.2100 HLG)** で符号化する手もある。HLG は SDR 互換性が高い (素直なガンマカーブの拡張) ので「HDR 非対応端末でもそこそこマシに見える」。
- **AVIF / HEIC**: HDR 対応コーデックなら 10bit 以上で素直に表現できる。8bit PNG + PQ は「どこでも開ける PNG という装い」を保てるのが強み。
- **JPEG-XL Gain Map**: 「SDR 画像 + ゲインマップ」というハイブリッド方式。Apple Photos が採用。

---

## まとめ

「8bit の普通の PNG」というガワを保ちながら、

- ピクセルを **PQ で再符号化** して「これは輝度値です」と意味を切り替え
- **iCCP に Rec.2100 PQ プロファイル** を埋め込んでブラウザに気付かせる
- 部分グローは**ピクセルごとに目標 nits を変える**だけ
- X に届けるには **400×400 ぴったり + PQ + Web Share** の 3 点セット

という 4 点が今回の核だった。HDR は「特別な画像フォーマット」ではなく、**メタデータと符号化のレイヤーで普通の PNG にも乗る**、というのが面白い。

