# 偽直排字型 (Pseudo-vertical Fonts) 技術標準與處理指引

本文件詳細記載 CrossPoint Reader 偽直排（Pseudo-vertical）字型之技術原理、跨 Up-EM 縮放對齊協定（Scale-alignment Protocol）、直排標點符號轉換標準與自動化工具指引。

---

## 1. 偽直排原理 (Pseudo-vertical Layout)

在尚未提供原生直排排版引擎的閱讀系統或韌體環境下，社群發展出「偽直排」技術以達成傳統直排閱讀體驗：

1. **字形逆時針旋轉**：將字型中的所有 CJK 統一漢字、假名及全形字元逆時針旋轉 90 度（CCW 90°）。
2. **標點與符號置換**：橫排標點符號（如逗號、句號、直角引號「」『』、破折號、省略號、括弧）在旋轉後位置會錯置，必須置換為直排專用的字形形式（例如居中、偏右上或直排專屬對齊）。
3. **實體旋轉閱讀**：閱讀時將電子紙實體設備順時針旋轉 90 度（CW 90°）。此時橫向排版的每行文字在視覺上自然轉變為縱向直排，達到直排閱讀效果。

---

## 2. 跨 Up-EM 縮放對齊協定 (Scale-alignment Protocol)

當自特定字型（來源字型，例如 `H-MingLan-Regular-v2d.ttf`）提取偽直排專用符號，並替換或合併至目標字型（例如 `LXGWWenKai-*.ttf`）時，兩者設計單位（UnitsPerEm / upem）通常不一致，**必須嚴格進行幾何與度量縮放對齊**：

### 2.1 縮放因子計算
$$scale\_factor = \frac{target\_upem}{source\_upem}$$

* 例：目標字型 `LXGWWenKai` 的 `unitsPerEm = 1000`，來源字型 `H-MingLan` 的 `unitsPerEm = 256`：
  $$scale\_factor = \frac{1000}{256} = 3.90625$$

### 2.2 輪廓座標縮放與整數取整
TrueType 輪廓格式要求點座標必須為整數：
1. **外廓點縮放**：對於 `numberOfContours > 0` 的字形，所有頂點座標需乘上 `scale_factor`，並使用四捨五入取整：
   ```python
   new_x = int(round(x * scale_factor))
   new_y = int(round(y * scale_factor))
   ```
2. **重算邊界框 (Bounding Box)**：取整後必須重新計算字形的邊界框：
   ```python
   glyph.recalcBounds(glyphSet)
   ```

### 2.3 橫向度量 (`hmtx`) 縮放
水平 advance 寬度（`advanceWidth`）與左側軸承（`leftSideBearing`）亦須等比例放大並四捨五入取整：
```python
advance_width = int(round(orig_advance_width * scale_factor))
lsb = int(round(orig_lsb * scale_factor))
```

### 2.4 補全缺失字元映射 (`cmap`)
若目標字型缺少該符號的 Unicode 編碼：
1. 將新字形名稱註冊至目標字型的 Glyph Order。
2. 同步更新 `cmap` 表中對應的 Unicode 碼位映射。

### 2.5 垂直度量 (`vmtx`) 表維護
若目標字型包含 `'vmtx'` 表，新增直排符號時必須補齊垂直度量資訊：
* **垂直前進量 (`advanceHeight`)**：標準設為 `target_upem`（通常為 `1000`）。
* **頂部軸承 (`topSideBearing`) 計算公式**：
  $$topSideBearing = sTypoAscender (880) - yMax$$
  *(若為空白字元或無外廓，則設為 0)*。

---

## 3. Python (fontTools) 處理範例

以下為使用 `fontTools.ttLib` 進行符號提取、度量縮放與整數取整的標準處理核心：

```python
from fontTools.ttLib import TTFont
import math

def scale_and_merge_glyph(target_font_path, source_font_path, codepoint, output_path):
    target = TTFont(target_font_path)
    source = TTFont(source_font_path)

    src_upem = source['head'].unitsPerEm
    tgt_upem = target['head'].unitsPerEm
    scale = tgt_upem / src_upem

    src_cmap = source.getBestCmap()
    if codepoint not in src_cmap:
        raise ValueError(f"Codepoint {hex(codepoint)} not in source font")

    src_gname = src_cmap[codepoint]
    src_glyph = source['glyf'][src_gname]

    # 1. 複製並縮放輪廓
    if src_glyph.numberOfContours > 0:
        new_coords = []
        for x, y in src_glyph.coordinates:
            new_coords.append((int(round(x * scale)), int(round(y * scale))))
        src_glyph.coordinates = new_coords
        src_glyph.recalcBounds(source['glyf'])

    # 2. 橫向度量縮放
    src_adv, src_lsb = source['hmtx'][src_gname]
    new_adv = int(round(src_adv * scale))
    new_lsb = int(round(src_lsb * scale))

    # 3. 注入目標字型
    tgt_gname = f"uni{codepoint:04X}"
    target['glyf'][tgt_gname] = src_glyph
    target['hmtx'][tgt_gname] = (new_adv, new_lsb)
    
    # 4. 更新 cmap
    for table in target['cmap'].tables:
        if table.isUnicode():
            table.cmap[codepoint] = tgt_gname

    # 5. 更新 vmtx (若存在)
    if 'vmtx' in target:
        typo_ascender = target['OS/2'].sTypoAscender if 'OS/2' in target else 880
        tsb = typo_ascender - src_glyph.yMax if src_glyph.numberOfContours > 0 else 0
        target['vmtx'][tgt_gname] = (tgt_upem, tsb)

    target.save(output_path)
```

---

## 4. 自動化工具與社群參考資源

* **tategakifont (MyK00L)**：自動化將字型逆時針旋轉 90 度並進行標點對齊的開源工具。
  - GitHub 倉庫：[https://github.com/MyK00L/tategakifont](https://github.com/MyK00L/tategakifont)
* **Unicode Vertical Text Layout 標準**：
  - [Unicode Technical Report #50 (UTR #50)](https://www.unicode.org/reports/tr50/)
* **社群偽直排教學**：
  - [YouTube 偽直排字型製作教學](https://www.youtube.com/watch?v=YKVcFtxiwi8)
* **編譯為 CrossPoint `.cpfont`**：
  - 偽直排 TTF 產出後，使用韌體倉庫提供的工具轉換：
    ```bash
    python fontconvert_sdcard.py <PseudoVertFont.ttf> \
      --intervals latin-ext,symbols,cjk,greek \
      --sizes 8,10,12,14,16,18 \
      --name <FontName> \
      --output-dir <OutputDir>
    ```

---

## 5. 自動化可覆用流水線 (Reusable Pipeline)

專案提供一體化流水線腳本 `scripts/build_font_pipeline.py`，支援動態 UPEM 自適應（相容 1000 與 2048 設計單位）、標點符號置中校正、破折號貫穿滿格伸展、引號軸線校正與雙向 `.cpfont` 編譯。

### 5.1 核心命令

```bash
# 1. 一鍵全自動轉換（產生 -CCW.ttf 偽直排，並同時編譯橫排與直排兩套完整 cpfont）
python scripts/build_font_pipeline.py fonts/lxgw-wenkai-v1.522/LXGWWenKai-Regular.ttf --all

# 2. 僅產生偽直排 TTF 字型（含全形標點置中校正）
python scripts/build_font_pipeline.py fonts/lxgw-wenkai-v1.522/LXGWWenKai-Regular.ttf --ccw

# 3. 僅編譯指定字型為 6 尺寸 cpfont
python scripts/build_font_pipeline.py fonts/lxgw-wenkai-v1.522-CCW/LXGWWenKai-Regular-CCW.ttf --cpfont
```

### 5.2 參數規範與自動計算公式
* **幾何中心**：$CX_0 = \frac{upem}{2}$
* **光學中心**：$CY_0 = \text{round}\left(354.0 \times \frac{upem}{1000.0}\right)$
* **全形標點置中**：句號、逗號、頓號、驚嘆號、問號、冒號、分號對齊 $(CX_0, CY_0)$。
* **破折號貫穿**：水平跨度縮放鋪滿 $0 \sim upem$，垂直置中於 $CY_0$，消除斷裂齒縫。
* **引號括號軸線**：保持原始朝向（轉 90° 時自然開闔），$Y$ 軸幾何中心對齊 $CY_0$。
* **必要尺寸集合**：永遠產出 `8, 10, 12`（UI Fallback）與 `14, 16, 18`（內文閱讀）共 6 個檔案。

