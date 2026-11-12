# DICOM MPR Viewer（医用画像処理実験 最終課題）

Axial / Coronal / Sagittal の3断面を表示する自作 Viewer です。DICOM と連続TIFF に対応し、
読み込んだ画像から肺の3Dモデル（ボリュームレンダリング）も表示できます。
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
| `viewer.py` | メインアプリ（Viewer・メニュー・スライダー・十字線・ヘルプ） |
| `core.py` | DICOM/TIFF 読込み、ウィンドウ処理、断面切出し、肺領域抽出 |
| `lung3d.py` | 3D 表示（VTK ボリュームレンダリング） |
| `requirements.txt` | 使用パッケージ |

## 操作方法
1. **ファイル → DICOM フォルダを開く…**（`Ctrl+O`）で DICOM 画像のフォルダを選択（サブフォルダも探索）。
   **ファイル → DICOM ファイルを開く…** で個別ファイルの複数選択も可能。
2. **ファイル → TIFF 画像（連続TIFF）を開く…**（`Ctrl+T`）で連続TIFF（例: 第5回の `hai.tif`）も読み込めます。
   読込み直後に **画素間隔(スライスサイズ)の設定ウィンドウ** が開くので、X・Y(mm/px)と Z(スライス間隔 mm)を設定してください
   （不明なら 1 のままで可）。あとから **ファイル → 画素間隔(スライスサイズ)の設定…** でも変更できます。
3. 読込み後、中央のスライスが3断面に表示されます。各断面の下のスライダーで操作します。
   - **Slice**: スライス位置（Axial=Z、Coronal=Y、Sagittal=X）
   - **WL / WW**: Window Level / Window Width（断面ごとに独立）
   - **WL/WW リセット**ボタン: その断面の WL/WW を読込み時の値に戻します。
4. 各画像上の **緑の線** は他の2断面の位置を表し、スライダーに連動して動きます。
   画像を左クリック／ドラッグしても、他2断面のスライダーが移動します。
5. **3D → 現在の画像から肺を3D表示**
   現在開いている画像から3Dモデルを作って表示します（画像の選択は不要）。
   - DICOM を開いている場合: HU 値から肺領域を簡易抽出（HU < -400 の空気領域 → 体外の空気を除去 → 上位2成分）して表示。
   - TIFF を開いている場合: 白い部分（画素値の最大・最小の中間値より大きい領域）を表示。第5回の肺マスク TIFF を想定しています。
   - 3D 画面: 左ドラッグ=回転 / 右ドラッグ=拡大縮小 / 中ドラッグ=平行移動。
     **XY / YZ / ZX ボタン**で視点を切替（XY=体軸断面の向き、YZ=矢状断の向き、ZX=冠状断(正面)の向き）。
     上下が逆に見える場合は「上下(Z)反転」を切り替えてください。
6. **ヘルプ → 操作方法**（`F1`）で操作説明を表示できます。

## 実装の概要
- **DICOM 読込み**: `pydicom` で読込み、`ImagePositionPatient` と `ImageOrientationPatient` からスライス順を決定。
  `RescaleSlope/Intercept` で HU 値に変換し (Z,Y,X) のボリュームにする。
- **TIFF 読込み**: `Pillow` でマルチページTIFFを (Z,Y,X) に読込み。WL/WW の初期値・スライダー範囲は画素値の範囲から決定。
- **MPR**: ボリュームから各断面を切り出し、画素間隔から物理サイズの縦横比を保って表示（Coronal/Sagittal は頭側が上）。
- **WL/WW**: `(値 - (WL - WW/2)) / WW * 255` を 0–255 にクリップして表示。
- **3D**: 領域マスクを（XY を 1/2 に縮小して）ガウス平滑化し、不透明度・カラー伝達関数を設定して
  VTK の `vtkFixedPointVolumeRayCastMapper`（CPU レイキャスト）で描画。

## 注意
- DICOM データと TIFF は本リポジトリに含まれていません。評価用データを「ファイル」メニューから選択してください。
- 3D 表示は CPU レンダリングのため、スライス枚数が多い場合は数秒かかります。
- DICOM からの肺抽出は閾値＋連結成分による簡易版です。精度が必要な場合は、第5回の結果（肺マスクTIFF）を
  TIFF として開いてから 3D 表示してください。
- 3D 表示で VTK がエラーになる環境でも、MPR Viewer 部分は単体で動作します
  （`lung3d.py` は 3D メニューを選んだときにのみ読み込まれます）。
