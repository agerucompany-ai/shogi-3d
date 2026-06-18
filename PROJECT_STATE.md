# 地形将棋3D — プロジェクト状態（別PC引き継ぎ用ハンドオフ）

最終更新: 2026-06-18 / このファイルは Google Drive (マイドライブ) 同期なので別PCでも開ける。
別PCで作業を続けるときは **まずこのファイルを読む**。Claude を使う場合は末尾「Claudeで再開する手順」を参照。

---

## 1. これは何か
- 3D版の地形将棋（対CPU＋オンライン2人対戦）。元の地形将棋(neoshogi)のルール・構成をそのまま3D化したもの。牧場ゲーム(bokujo)とは別プロジェクト。
- **単一HTML**: `shogi-3d.html`（約5.2MB / three.js r128・駒/地形画像WebP・peerjs を全インライン＝オフラインでもダブルクリックで動く）。
- 駒＝OpenAI画像生成のリアル三國志武将立ち絵をスプライト（ビルボード）で3D地形上に配置。

## 2. 公開URL / リポジトリ（=別PCでの入手元）
- **公開ページ**: https://agerucompany-ai.github.io/shogi-3d/
- **クライアント repo**: `agerucompany-ai/shogi-3d`（public・GitHub Pages）。配信ファイル＝`index.html`。
- **オンラインサーバ repo**: `agerucompany-ai/terrain-shogi`（Render・WebSocket権威サーバー）。
  - 本番WS: `wss://terrain-shogi-server.onrender.com`（無料枠：初回接続はスリープ復帰に最大1分）
  - Render は push で自動デプロイ（Blueprint `render.yaml`・healthCheck `/health`）。
- 別PCでは: `gh repo clone agerucompany-ai/shogi-3d` / `... terrain-shogi`。Drive の `H:\マイドライブ\3d-shogi\` も同期で入手可。

## 3. ファイルの場所（重要）
- **正本(編集対象)**: `H:\マイドライブ\3d-shogi\shogi-3d.html`（Drive直下・gitではない）。
- `index.html` は `shogi-3d.html` のコピー＝デプロイ用。**編集後は必ず `cp shogi-3d.html index.html`**。
- **デプロイ用クローン**: `C:\Users\ageru\repos\shogi-3d\`（Driveだとgit破損しやすいのでcloneはCドライブ推奨）。
- サーバ: `C:\Users\ageru\repos\terrain-shogi\server\index.js`, `gameLogic.js`。
- 画像素材: `H:\マイドライブ\3d-shogi\pieces\*.png`。再アセンブル素材(template/three/rules/builders/worker)は `C:\Users\ageru\AppData\Local\Temp\shogi_shot\`。
  - ※ Temp の `rules.txt`(6/15) は古く攻城戦/山を含まない。**現運用は shogi-3d.html を直接編集**している（assemble.js は当面使わない）。

## 4. コード構造（shogi-3d.html 内）
- ルールコードは **2重化**: `<script id="aiwsrc" type="text/js-worker">`（AI探索Worker・145〜787行付近）と本体（743行〜）。
  - 地形生成 `genTerrain` と AI布陣 `pickPlacementForTerrain/pickMountainPlacement/pickSiegePlacement` は本体側のみ実機で使用。
  - 移動生成 `genMoves` 等は両方にあるが forts/fences は共通仕様。
- **AIエンジン(Worker)**: 反復深化+αβ(PVS)+LMR+静止探索(quies)+Zobrist置換表。leaf評価＝`evalFast`（628行付近、ここが実機のAI評価本体）。`searchBest`(768行付近)→postMessage。
- マップ種別 `mapType`: `'plain'`(平地戦) / `'siege'`(攻城戦) / `'mountain'`(山)。

## 5. マップ別ルール（現状の確定仕様）
### 平地戦 plain
- 王=自陣の城。陣地冊(柵)3つ=最深3行。砦1つ=自陣。川/森は密度で生成。
### 攻城戦 siege
- 守備(side1)=城2つを「城枡」(preset柵)で囲う。王は城の上。柵は配置済み。
- 攻撃(side0)=砦2つ・王は砦の上。**砦は前方含め盤上どこでも可**(城/柵/川以外)。駒は自陣5行。
- オンライン対応済み（サーバ genTerrain/validatePlacement に siege 分岐、preset柵を room.fences に seed）。
### 山 mountain
- 森が全体の**約7割**(`Math.round(N*N*0.7)`=85マス)。川は6行目(MID)に密度準拠。**城なし**。
- **王は最深3行(1〜3列=fenceRows)に置く**（砦に乗せる必要なし。森マスなら自動で伏兵化 hidden=forest）。
- **砦3つ・柵3つは自陣(homeRows)どこでも自由配置**。
- 森7割＝駒/王が自然に伏兵化＝「森に隠れる」のが基本戦略。
- AI布陣 `pickMountainPlacement`: 王・砦3・柵3・大駒すべて**森を最優先**で配置（王の位置がバレない）。
- AI対局 `evalFast`: 山判定(`TER.castles`両側0)で**大駒(飛角)・王が平地に出ると強ペナルティ**／森に温存で加点（狙撃回避）。
- オンライン対応済み。

## 6. オンライン対戦（権威サーバー方式）
- クライアント→サーバ送信: create{name,density,mapType,hostSide,handLimit}/join{code,name}/resume/placement{king,pieces,fences,forts}/move/drop/resign/takeback/reroll/rematch/chat。
- サーバ→受信: created/joined/resumed/state{publicState:マスク済board("?"=伏兵)}/info/error/placed/chat。
- 地形はサーバ・クライアント双方が同seed+density+mapTypeで `genTerrain` 再生成し**完全一致**（伏兵マスキング整合の要）。**genTerrain を変更したら client↔server の RNG一致を必ず検証**。
- 部屋コードは6桁数字。token再接続あり。

## 7. ユーザの強い要望（厳守）
1. 構成は元の地形将棋と同一（駒選び合計Lv20→自陣に配置→対局の本来フロー）。
2. 視点は固定寄り（fitCamera自動。手動回転/ズームは可だが基本は俯瞰一望）。
3. 地形は一体化（浮いた盤＋枠は禁止。地面/川/森/城が連続した一枚）。
4. 駒は全種キャラ化・頭上に駒名ラベル。
5. **スマホ最優先**（タップ操作・縦画面で盤全体が収まる）。
- 報告は簡潔に（完了報告のみ）。「やって」発言後は push/pull 含め完了まで承認不要。

## 8. ビルド/検証（必須）
- 構文: HTMLはそのまま `node --check` 不可→puppeteerで実機ロード。サーバは `node --check`。
- **必ず puppeteer-core + システムChrome** (`--use-angle=swiftshader --enable-unsafe-swiftshader`) でヘッドレス検証。
  - Chrome: `C:/Program Files/Google/Chrome/Application/chrome.exe`
  - デバッグフック: `window.__dbg() / __autoplace() / __cam() / __pieceScreen() / __ol`。
- オンライン検証: ローカルで `cd server && PORT=8787 node index.js`、puppeteerで `WebSocket` を localhost に書換える(evaluateOnNewDocument)。**攻城戦の検証は __autoplace が前方布陣で不適→ canPlace* 準拠の正当布陣ヘルパで配置すること**。
- 地形パリティ検証: server `gameLogic.js` の `genTerrain` を dynamic import し、クライアント `genTerrain` (puppeteer evaluate) と grid/castles を全seed比較。
- テストスクリプト群は `C:\Users\ageru\AppData\Local\Temp\shogi_shot\`（comp_test.js / mountain_test*.js / online_*.js / parity.js 等）。Tempなので別PCには無い→必要なら作り直す（このdocの方針通り）。

## 9. デプロイ手順
```
# クライアント
cp "H:/マイドライブ/3d-shogi/shogi-3d.html" "H:/マイドライブ/3d-shogi/index.html"   # Drive正本を同期
cd C:/Users/ageru/repos/shogi-3d && cp "H:/マイドライブ/3d-shogi/shogi-3d.html" index.html
git add index.html && git commit -m "..." && git push origin main                      # Pages自動反映(数十秒)
# サーバ
cd C:/Users/ageru/repos/terrain-shogi && git commit -am "..." && git push origin main   # Render自動デプロイ(数分)
```
- gh 認証: `agerucompany-ai`（keyring・repoスコープ）。コミット末尾は Co-Authored-By を付ける運用。

## 10. 現在の到達点（2026-06-18 時点・全て本番反映＆E2E検証済み）
- 平地戦/攻城戦/山 の3マップ、対CPU＋オンライン2人対戦すべて稼働。
- 直近実装: 山モード追加→山オンライン化→(王は砦不要・1〜3列・砦自由 へルール変更)＋再戦バグ修正＋山AIの森隠れ＋大駒温存評価＋攻城戦オンライン対応。
- 検証済み: 地形パリティ(全seed一致)・オフライン回帰(平地/攻城)・本番オンラインE2E(攻城/山/平地でplay到達＆着手同期＆エラー0)。

## 11. 既知の注意/未対応
- 山AIの駒構成は香ロケット流用(`pickArmyForTerrain`)。山専用の最適編成は未調整。
- 対称NAT等は旧P2P経路では不可だがサーバー方式なので基本問題なし。
- Render無料枠スリープ＝初回接続に最大1分。

---

## Claudeで別PCから再開する手順
1. このファイル（PROJECT_STATE.md）と同フォルダの `_claude_memory_3d_shogi.md` を読み込ませる（＝旧メモリの全文バックアップ）。
2. 「地形将棋3Dの続き。PROJECT_STATE.md と _claude_memory_3d_shogi.md を読んで」と指示。
3. コードは GitHub から clone するか、Drive の `shogi-3d.html` を直接編集（編集→index.htmlへcp→両repoへpush）。
