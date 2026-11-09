# DICOM MPR Viewer（医用画像処理実験 最終課題）

Axial / Coronal / Sagittal の3断面を表示する自作 Viewer です。
Python + PySide6（Qt）で実装しています（画像表示は `QPainter` による自前描画）。

## 動作環境
- Python **3.11 または 3.12**（3.12.3 で作成）
- Windows / macOS / Linux（GUI が使える環境）
- 使用パッケージとバージョンは `requirements.txt` に記載

## インストール手順
```bash
# 1. 仮想環境の作成と有効化
python -m venv venv
# Windows (PowerShell):  .\venv\Scripts\Activate.ps1
# Windows (cmd):         venv\Scripts\activate.bat
# macOS / Linux:         source venv/bin/activate

# 2. パッケージのインストール
pip install -r requirements.txt

# 3. 起動
python viewer.py
```

## ファイル構成
| ファイル | 内容 |
|---|---|
| `viewer.py` | メインアプリ（Viewer・メニュー・スライダー・十字線） |
| `core.py` | DICOM 読込み、ウィンドウ処理、断面切出し、肺マスクTIFF読込み |
| `lung3d.py` | 肺領域のボリュームレンダリング（VTK, 観点7） |
| `requirements.txt` | 使用パッケージ |

## 操作方法
1. **File → DICOM フォルダを開く…**（`Ctrl+O`）で、DICOM 画像が入ったフォルダを選択
   （サブフォルダも再帰的に探索）。**File → DICOM ファイルを開く…** で個別ファイルの複数選択も可能。
2. 読込み後、中央のスライスが3断面に表示されます。
3. 各断面の下にあるスライダーで操作します。
   - **Slice**: スライス位置（Axial=Z、Coronal=Y、Sagittal=X）
   - **WL**: Window Level / **WW**: Window Width（断面ごとに独立）
4. 各画像上の **緑の線** は他の2断面の位置を表します。
   スライダーを動かすと、他の断面の線も連動して動きます。
5. （おまけ）画像を左クリック／ドラッグすると、その位置に合わせて他2断面のスライダーも移動します。
6. **3D → 肺マスク(TIFF)をボリュームレンダリング…**（観点7）
   第5回で作成した肺マスクの連続TIFF（`hai.tif`）を選択すると、3D表示します。
   - 白（輝度>127）の画素を肺領域として扱います。
   - 先に DICOM を読み込み、スライス数・画像サイズがマスクと一致する場合は、そのスペーシング(mm)を使います。
     一致しない／DICOM未読込の場合は 1mm 等方で表示します。
   - 3D ウィンドウ上部の **XY / YZ / ZX ボタン** で視点を切り替えられます（XY=体軸断面の向き、YZ=矢状断の向き、ZX=冠状断(正面)の向き）。
   - 上下が逆に見える場合は、3D ウィンドウのメニュー「上下(Z)反転」を切り替えてください。
   - 左ドラッグ:回転 / 右ドラッグ:拡大縮小 / 中ドラッグ:平行移動。

## 実装の概要
- **読込み**: `pydicom` で読込み、`ImagePositionPatient` と `ImageOrientationPatient`
  からスライス順を決定。`RescaleSlope/Intercept` で HU 値に変換し (Z,Y,X) のボリュームにする。
- **MPR**: ボリュームから各断面を切り出し、`PixelSpacing` とスライス間隔から
  物理サイズの縦横比を保って表示（Coronal/Sagittal は頭側が上）。
- **WL/WW**: `(HU - (WL - WW/2)) / WW * 255` を 0–255 にクリップして表示。
- **肺マスク読込み**: `Pillow` でマルチページTIFFを読込み、(Z,Y,X) の bool 配列にする。
- **ボリュームレンダリング**:   (1/2 にダウンサンプリングして軽くガウス平滑化した) マスクに不透明度・カラー伝達関数を設定し、VTK の `vtkFixedPointVolumeRayCastMapper`
  （CPU レイキャスト）で描画。

## 注意
- DICOM データと `hai.tif` は本リポジトリに含まれていません。評価用データを File メニューから選択してください。
- 3D 表示は CPU レンダリングのため、スライス枚数が多い場合は数秒かかります。
- 3D ウィンドウは閉じてからでも、そのまま File → 終了 でも正常に終了します。
- 3D 表示で VTK がエラーになる環境でも、観点1〜6（MPR Viewer）は単体で動作します
  （`lung3d.py` は 3D メニューを選んだときにのみ読み込まれます）。
