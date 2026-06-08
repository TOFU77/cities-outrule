# Cities: OutRuler

ブラウザで動作するスタンドアロンの都市経営シミュレーションゲームです。  
サーバー不要。`cities-outrule.html` を開くだけで起動します。

🎮 **プレイ**: https://tofu77.github.io/cities-outrule/

---

## ゲーム概要

- タイルマップ（PC: 20×15、スマートフォン: 7×12）に施設を配置し、50年間の都市運営を目指す
- 12種のリソースが毎ターン連動して変化する
- 都市プリセットごとに異なる「Signature目標」を達成することで幸福度の上限が解放される

---

## 技術仕様

### 実装

| 項目 | 内容 |
|---|---|
| 言語 | HTML + Vanilla JavaScript（ES5〜ES6、モジュール・フレームワーク不使用） |
| 描画 | Canvas 2D API（タイルマップ） |
| 永続化 | JSON ファイルのダウンロード／アップロード（localStorage 不使用） |
| レスポンシブ | `@media (max-width: 768px)` でモバイルレイアウト切替、ボトムタブ5分割 |

---

### データ定義層（静的定数）

```
FACILITY_DEF          施設マスターデータ
│  14種 (FARM/AGRO/FACT/POWER/PORT/HOUS/WASTE/POLICE/EPA/UNIV/MUSE/COMM/FOREST/RIVER)
│  各施設: icon, sector, io_base{out, in}, build_cost, maint_cost, env_sensitivity
│
TECH_TREE             固定テクノロジー定義（T_IRR / T_GRID / T_EDU 等）
│  family: PROD / GREEN / CIVIC、tier: 1〜4
│  effect: [{target, type:"mult"|"add", key, value}]
│
CITY_PRESETS[]        都市プリセット 8種
│  delta_units: 初期施設配置の差分、policies, signature（Signature目標）
│
DEVIATION_ARCHETYPES  自律変化イベント 4種
│  INDU_SURGE / CIVIC_BLOOM / BACK_TO_LAND / AUTH_ORDER
│  bias（発生条件）、delta_units（施設変化）、instant（即時ストック変化）
│
EVENTS[]              ランダムイベント定義（旱魃・騒乱・技術革新 等）
│  prob / prob_fn、effect(state)
│
COEFF                 全シミュレーション係数（単一オブジェクト）
│  需要・維持費・KON係数・イベント確率・危機閾値 等 20+項目
│
EPIGRAPHS[]           年次エピグラフ文字列（10年ごとに表示）
NPC_LINES             OTOMO発言文字列プール（状態別 6カテゴリ）
```

---

### 状態管理層（ランタイム変数）

```
state                 ゲーム全体の状態オブジェクト
│  stocks{}           12リソースの現在値
│  │  POP / FOOD / MAT / NRG / G / SCI / CUL
│  │  ENV / INF / HAP / RES / KON
│  sectors{}          4セクター（RURAL/INDU/ARTS/NATU）の知識・インフラ水準
│  tech{}             研究状態（done, name, effect 等）
│  stance{}           各セクターの稼働スタンス（active/normal/light）
│  signature          プリセット別の達成目標条件
│  crisisTurns        危機継続ターン数（イベント確率加速に使用）
│  deviationCooldowns 各Deviationのクールダウン残ターン
│  pendingDeviation   プレイヤー応答待ちのDeviation
│
map[][]               タイルデータ（facility, sector）2次元配列
│  MAP_W × MAP_H（let変数、起動時に画面サイズで決定）
│
year                  現在年（1〜50）
logs[]                ログエントリ（msg, type）
prevStocks            前ターンのstocksスナップショット（差分表示用）
techEffects{}         研究済みテクノロジーの効果を累積したキャッシュ
selectedFacility      現在プレイヤーが選択中の施設キー（null or string）
```

---

### シミュレーション層（関数）

```
computeDeltas(stk, sec, tech_state, counts, rUNIV, rMUSE)
│  1ターン分の全リソース変化量を計算して返す（純粋関数）
│  simulateTurn() と calcTurnDeltas()（プレビュー）が共用
│
│  主要な計算ステップ:
│  1. 施設ごとに産出・消費・維持費を集計（スタンス係数・施設効率を乗算）
│  2. FOOD需要を計算し、不足時は港数に応じてFOOD輸入（G消費）
│  3. 港があればMAT→G常時輸出（40MAT/港/ターン、2MAT→1G）
│  4. MAT上限キャップ（300 + POP×0.4 + 総施設数×20）を適用
│  5. KON: 施設排出 − 文化・森林・警察による吸収 − 自然減衰
│  6. Signature達成率（sigSat）を計算し、HAP上限 = 0.30 + sigSat×0.70 を設定
│  7. 飢餓ペナルティ: FOOD=0 で POP −3%/ターン
│  8. 全12変数の新値・差分を返す
│
simulateTurn()
│  computeDeltas() を呼び出してstate.stocksを更新
│  → 危機判定（crisisTurns更新）
│  → MAT輸出ログ出力
│  → ランダムイベント抽選（危機加速係数を乗算）
│  → Deviation発生チェック・モーダル表示
│  → 年次レビューモーダル（10年ごと）
│  → 終了条件チェック
│
countFacilities()     map[][]を走査して施設種別カウントを返す
getFacilityEff()      施設効率 = INF × ENV感度 × techEffects を計算
calcTechCost()        EXP_V1スケーリング式でテク研究コストを返す
                      SCI = ceil(max(40, base) × 1.80^(t-1) × 1.20^nf × 1.10^nt)
genTechs(seed)        都市IDシードから手続き的テクノロジーを生成（GEN_TECH）
```

---

### イベント・Deviation層

```
calcDeviationProb(stk)     ストック状態からDeviation発生確率を計算
selectArchetype(stk)       bias条件でアーキタイプを重み付き選択
applyDeviation(key, accepted)
│  accepted=true:  delta_units・remove_units をmapに反映、instant変化を適用
│  accepted=false: 経済ペナルティ（G・HAP低下）を適用
showDeviationModal()       プレイヤーに受け入れ/抵抗を問うモーダル表示
```

---

### UI層（関数）

```
initCanvas()
│  Canvas初期化・マウス/タッチイベント登録
│  └── resizeCanvas()         キャンバスサイズをラッパー要素に合わせ再描画
│
drawMap()                    全タイルを描画（施設アイコン・選択ハイライト・ホバー表示）
onCanvasClick()              タイルクリック → 施設建設 or 撤去
│
buildResourcesUI()           リソースバー初期生成（ホバーでtooltip）
updateResourcesUI()          バー幅・数値・差分テキストを毎ターン更新
buildFacilityUI()            セクタータブ + スタンスボタン + 施設ボタンリストを描画
buildTechUI()                研究可能テクノロジーリストを描画
updateUI()                   全UI更新を一括呼び出し
│
showTooltip() / hideTooltip()
│  リソース名ホバー時に詳細パネルを表示
│  現在値・主要収支源・計算式を動的に組み立て
│
showYearReviewModal()        10年ごとのエピグラフ + 年次レポートをまとめて表示
showDeviationModal()         Deviation承認/拒否ダイアログ
buildPresetUI()              ゲーム開始時のプリセット選択モーダル
│
npcReact()                   ターン終了後にOTOMOコメントを更新（状態スコアで分岐）
addLog()                     ログパネルにエントリを追加（type: good/warn/event/crisis）
```

---

### モバイル対応

```
applyMapDimensions()
│  window.innerWidth ≤ 768 → MAP_W=7, MAP_H=12
│  それ以外                → MAP_W=20, MAP_H=15
│  initState() 呼び出し前に実行
│
モバイルタブ切替（IIFE内）
│  ボトムタブ5分割: 資源 / マップ / 施設 / テク / ログ
│  right-panel内を #right-facility-section / #right-tech-section / #right-log-section に分割
│  タブクリック → 対応パネルに mob-active / mob-sec-active クラスを付与
│
#mob-fac-indicator
│  マップタブ表示中に選択施設名をオーバーレイ表示
│  タップで施設タブへ遷移
│  updateMobFacIndicator(facKey) でJS側から更新
```

---

### リソース一覧

| キー | 名称 | 主な産出源 | 主な消費源 |
|---|---|---|---|
| POP | 人口 | 食料余剰・HAP上昇 | 食料不足・KON |
| FOOD | 食料 | 農場・農工場・河川・輸入 | POP×0.5/ターン |
| MAT | 資材 | 工場・森林・廃棄処理 | 施設維持費・港輸出 |
| NRG | エネルギー | 発電所・河川 | 工場・大学・美術館 |
| G | 財政 | 工場・美術館・港・MAT輸出 | 施設維持費・食料輸入 |
| SCI | 科学力 | 大学 | テクノロジー研究コスト |
| CUL | 文化力 | 美術館・コミュニティ・森林 | KON吸収に変換 |
| ENV | 環境 | 自然回復・テク | 工場・農業・KON |
| INF | 社会基盤 | G>500で上昇 | G不足で低下 |
| HAP | 幸福度 | 食料充足・文化 | KON・Signature未達成 |
| RES | 回復力 | ENV高・テク | KON高・イベント |
| KON | 困難度 | 工場・発電所・住宅 | 警察・文化・森林・自然減衰 |

---

### ファイル構成

```
cities-outrule.html               メインファイル（ゲーム本体、約1,750行）
OLC-cities-outrule-prompt09.txt   AIプロンプト版仕様書（v09）
cities-outrule-thread.txt         制作スレッド記録
cities-outrule-YYYYMMDD.html      旧バージョンのバックアップ
```
