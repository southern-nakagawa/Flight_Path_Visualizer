# ドローン飛行経路ビジュアライザー 開発ブリーフ

最終更新: 2026-07-03 ／ 更新者: 自分

> 使い方：新しいスレッドの冒頭にこの全文を貼る。開発終わりに更新して出力し、§6・§7 を確認・追記・修正して保存し直す。

---

## 1. 概要
DJIドローンのSRT（GPS/高度付き）を読み込み、飛行経路を2D/3D/地形3Dで可視化するWebアプリ。フォルダ指定で複数フライトを日付・場所でフィルタ、チェック選択して比較・反省できる。公開先: flightpath.bitriseone.com。

## 2. スタック / 環境
- フロント: バニラ JS（単一HTML・2623行に全部入り）
- 2D地図: Leaflet
- 3D: Three.js（r128）
- 地形3D: 国土地理院 dem_png タイル（ズーム14、96×96頂点メッシュ）
- ホスティング: GitHub Pages（`southern-nakagawa/Published_Applications` の `flight/`）＋独自ドメインCNAME
- 開発機: Mac mini
- Git: GitHub Desktop 運用（コマンドは最終手段）

## 3. 主要機能（把握範囲・古ければ削除/修正）
- 3ビュー：2D（Leaflet）／3D（Three.js）／地形3D（国土地理院DEM）
- フォルダ指定＋日付/場所フィルタ、フライト一覧のチェック選択表示
- 既定選択は「最新日フライトのある場所グループ」（同日は現在地に近い順）
- 視点リセット固定（真南から北・俯瞰）、N/S/E/Wコンパス（回転連動）
- 録画開始（🔴明赤）／終了（⬛暗赤）マーカー（各ファイル先頭・末尾フレーム）
- 鼻向きサークル動画エクスポート（方位±sweep°＋ピッチをsin揺動、5秒、1920×1080/12Mbps）
  - Safari対策：`video/mp4;codecs=avc1`優先（SafariはWebM偽陽性）→WebM→PNG連番ZIPフォールバック
- 手動標高オフセット入力
- PDF出力（A4横、左=飛行データ／右=画面キャプチャ、日時マスク `****-**-** **:**`）
- 多言語（日/英 setLang）・About モーダル

## 4. ファイル構成マップ
ローカルフォルダ名: `Flight_Path_Visualizer`。除外: `node_modules` / `.git`。

**構造を出し直すコマンド**（ターミナルでプロジェクトルートに `cd` してから実行）:
```
find . -type d \( -name node_modules -o -name .git \) -prune -o -print | sort
```

```
index.html      ← 全部入り（HTML＋CSS＋JS 2623行、ハンドリング考慮で単一ファイル）
CNAME           ← 独自ドメイン（flightpath.bitriseone.com）
LICENCE
```
※単一HTMLなので「index.html を貼って」で全体を渡す。下の内部マップで該当関数の行を指定すれば部分修正できる。

### 4-1. index.html 内部関数マップ（行番号は目安）
- データ取込/整形：`parseSRT`(513) `sampleBySecond`(527) `addFiles`(647) `recomputeAll`(548)
- 統計/凡例/経由点：`updateStats`(591) `updateWaypoints`(604) `updateLegend`(631)
- 位置名/ソート/一覧：`fetchLocName`(700) `getSortedLocKeys`(722) `buildLocationDropdown`(745) `applyFilter`(800) `renderFlightTable`(828) `syncDisplay`(897) `renderFileList`(931)
- 空状態/エラー：`setEmptyState`(921) `showNoDataMessage`(927) `showError`(936)
- 2D（Leaflet）：`initLeaflet`(1828) `switchTileLayer`(1834) `update2D`(2030)
- オフスクリーン地図テクスチャ：`initOffscreenMap`(954) `updateOffscreenLayer`(969) `buildMapTextureAndRender3D`(983) `captureMapToTexture`(1035)
- 3D描画/カメラ：`render3D`(1189) `updateCamera3D`(1493) `applyMovieCameraPose`(1523)
- 動画export：`pickRecorderMime`(1540) `exportNoseInCircleMovie`(1556) `downloadBlob`(1691)
- 地形3D（DEM）：`lonLatToTileXY`(1876) `fetchDEMTile`(1885) `preloadDEMTiles`(1929) `sampleElevation`(1942) `buildTerrainMesh`(1954) `updateTerrainInfoPanel`(2015)
- 設定：`getSettings`(1114) `saveSettings`(1132) `loadSettings`(1152) `applySettings`(2161)
- UI/言語：`setLang`(2596) `openAbout`(2605) `closeAbout`(2608)
- ユーティリティ：`latLonToMeters`(488) `haversine`(494) `fmtDist`(499) `fmtDur`(501) `fmtSpeed`(1756) `hexToRgb`(505) `geoDistance`(692)

## 5. flightpath固有の判断デフォルト 【要確認/追記】
※全体の働き方ルールはプロフィール設定済み。ここにはこのアプリ固有の方針だけ。
- Three.jsは r128（CDN）。OrbitControls不可・CapsuleGeometry不可（r142+）。Cylinder/Sphere/自作ジオメトリで代替
- 動画export：MIME候補は mp4(avc1)→webm→PNG連番ZIP の順（Safari WebM偽陽性対策、`pickRecorderMime`）
- captureStreamにHTMLオーバーレイは映らない → クレジット等はオフスクリーン合成canvasで焼き込む
- 地形3Dの地図テクスチャは `initOffscreenMap`→`captureMapToTexture` のオフスクリーン経由
- 3Dの地面プレーンは正方形でなくメートル換算スパン（東西cos補正 `latLonToMeters`）でアスペクト比を合わせる
- PDFは日本語フォント非対応のjsPDF回避 → 地図は2D白背景キャプチャで焼き込み
- 単一HTMLの巨大ファイル。部分修正は §4-1 の関数マップで行指定して渡す
- （追記）

## 6. 現在地・次の一手
- 今どこ：2D/3D/地形3Dの3ビュー、動画export、PDF出力まで実装し flightpath.bitriseone.com で公開中（単一HTML／GitHub Pages）。
- 次やる：**Next.js化＋Vercel配布**。2623行の単一HTMLをコンポーネント分割し、Vercelへ移行するのが主要目標。§7の移行論点を参照。

## 7. 既知の問題 / 保留
- Next.js/Vercel移行の論点：
  - 単一HTML→分割の切り方（データ取込・2D・3D・地形・動画export・設定 のモジュール境界。§4-1がそのまま分割単位の候補）
  - Three.js r128前提のコードをどう持ち込むか（npm版Three最新へ上げると API差異。まずは現r128をそのまま移植し描画確認 → 後で更新）
  - Leaflet/DEMタイル取得のSSR回避（`window`依存が多い → クライアントコンポーネント化 or dynamic import ssr:false）
  - MediaRecorder等ブラウザAPIはクライアント側限定
  - ドメイン切替：現flightpath.bitriseone.com（GitHub Pages）→ Vercel。CNAME/DNSの付け替え
- GPSロスト（渓谷・岩壁＋水面）でSRTの点データ欠落 → パーサ段の対応要
- 地形3Dの初回描画が出ないことがある（display:none→block時のWebGL/リサイズ同期）。IntersectionObserver か遅延初期化で対応

---
運用ルール：開発終わりにあなた（Claude）が §6・§7 を含めた全体更新案を出す → 私（ユーザー）このファイルに追記保存。