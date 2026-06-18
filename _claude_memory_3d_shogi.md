---
name: project_3d_shogi
description: 地形将棋3D（対CPU）プロジェクト。場所・仕様・ビルド/検証手順と、ユーザの強い要望
metadata: 
  node_type: memory
  type: project
  originSessionId: 80ded60b-0ddd-4dd7-9a34-6dd2b700f5a1
---

3D版の地形将棋（対CPUのみ）。元の地形将棋(neoshogi)の **ルールと構成をそのまま**3D化したもの。牧場ゲーム(bokujo)とは**別プロジェクト**。

- **公開URL**: https://agerucompany-ai.github.io/shogi-3d/ （GitHub Pages・repo `agerucompany-ai/shogi-3d` public）。`index.html`単一ファイル(約5.4MB)。
- **場所**: `H:\マイドライブ\3d-shogi\shogi-3d.html`（単一HTML・three.js r128 をインライン・**駒画像もbase64で全埋め込み**＝オフライン/ダブルクリックでも動く）。牧場(bokujo)とは分離。
- **元ルール/AI**は neoshogi の `gameLogic.js`/`aiBrain.js` を移植（同一）。
- **駒＝OpenAI画像生成(gpt-image-1, 透過PNG)のリアル三國志武将立ち絵を、3D地形上にスプライト(ビルボード)で配置**（視点固定なのでフィギュアに見える）。procedural駒は廃止。
  - 名将割当: 飛=張飛, 角=趙雲, 馬(成角)=関羽, 竜(成飛)=呂布, 王=劉備 / 敵王=曹操, 桂=騎兵, 香=太い槍の戦車, 成駒=精鋭兵(金)。あなた=青系/敵=赤系。
  - 生成プロンプト/スクリプトは `C:\Users\ageru\AppData\Local\Temp\shogi_shot\gen_all.js`。鍵は `H:\マイドライブ\work_kurosaki\.env` の OPENAI_API_KEY。画像は `H:\マイドライブ\3d-shogi\pieces\*.png`(640pxに sharp で縮小, 計3.5MB)。
  - 駒を差し替えたい時: pieces/ の該当PNGを再生成→640px化→`template.html`+three+rules+builders+PIECE_IMG(base64) を node で再アセンブル→shogi-3d.html。テンプレ等の素材は temp/shogi_shot/。
- ローカルfile://だとWebGLが外部PNGをCORSで弾くので**画像はdataURIで埋め込む**こと（別ファイル参照は公開(http)でのみ可）。容量対策で画像は**WebP化してから埋め込む**（PNG埋め込みだと約20MB→WebPで約4MB）。フィールド素材(地面/木/城)もAI画像のビルボード。
- 駒/城/木はマス内に収まるサイズ＋接地影。視点は固定・低FOV(望遠)俯瞰で盤一望。マス線濃く＋陣地色(自=青/敵=赤)。色統一: 自軍=青白/白馬, 敵軍=赤黒/黒馬。
- 実装済み: タイトル画面(対CPU/オンライン選択), 待った(undoStack), 棋譜(formatMove/squareLabel/KANJIはテンプレ側で定義), 地形密度(森川 少/普/多), 上限Lv(10-40)。持ち駒打ちは元ルール通り**自陣の城/制圧城/砦の周囲のみ**(バグではない)。
- **【2026-06-18】山モード(mapType='mountain')追加**: タイトル/army画面のマップ選択に「山」ボタン(平地戦/攻城戦/山)。地形=森が全体の約7割(`Math.round(N*N*0.7)`=85マス)・川は6行目(MID)に密度準拠で従来通り・城なし。配置=城の代わりに砦`FORT_MAX(mountain)=3`(うち最深3行=fenceRows="1〜3列"に王砦2つ＋自陣homeRowsどこかに追加1つ)、王は最深3行の砦の上のみ(`mtFenceRows`/`fortsInFenceRows`で判定)、柵3つも自陣どこでも(plainは最深3行のみ→mountainはhomeRows)。森7割で駒・王が自然に伏兵化=「隠れられる」。AI布陣=`pickMountainPlacement`(王砦2つ離して森優先→王はより隠れる方/追加砦前方/柵で王前/駒前方優先)。ドロップは城なしでも砦周囲で機能(canDropHere)。編集はHTML直編集(rules.txtは6/15で古く攻城戦未収録=現運用はHTML直編集)。検証: mountain_test.js/mountain_test2.js で森70.2%・川6行目のみ・城0・砦3(最深2)・王砦上・柵3・不正配置拒否・AI実着手(depth3) 全OK、平地戦(comp_test)/攻城戦も回帰OK。
- **【2026-06-18】山モード オンライン対戦対応(権威サーバー方式)**: クライアントのオンラインパネルにマップ「山」ボタン追加(olMapMountain・setMap3択化)。**サーバ(`agerucompany-ai/terrain-shogi` の server/)も改修**: `genTerrain(seed,opts,mapType)`に第3引数追加し'mountain'分岐(森7割・川6行目・城なしをクライアントとRNG完全一致で生成=伏兵マスキング整合)、`validatePlacement`に山分岐(砦3=最深3行に王砦2＋自陣に追加1/王は最深3行の砦上/柵3は自陣どこでも)、`applyPlacement`を砦配列対応(plain/siege/mountain共通)、`publicState`に`mapType`追加、create/reroll/rematchで`room.mapType`反映。検証: ローカルサーバ+本番(Pages+Render)の双方で2クライアントE2E=部屋作成→参加→布陣受理→play同期→着手同期→**伏兵マスク(ゲストから見たホスト駒8枚が"?"、可視2枚)** 全OK。Render(terrain-shogi)はpushで自動デプロイ(Blueprint・healthCheck /health)。
- **【2026-06-18 その2】山ルール変更＋再戦バグ修正＋攻城戦オンライン＋AI森隠れ**:
  - **再戦バグ修正**: `enterArmy`が`buildStructures()`を呼んでおらず、再戦/再army時に前回の柵・砦の3Dメッシュ(structGroup)が残存していた→`enterArmy`末尾に`buildStructures()`追加で全マップ・対AI/オンライン両方解消。
  - **山ルール変更**: 王は砦に乗せる必須をやめ、**最深3行(1〜3列=fenceRows)に置けばOK**(森なら自動で伏兵化 hidden=forest)。砦3つは`fortsInFenceRows`制約を撤廃し**自陣homeRowsどこでも自由**。canPlaceFort/placeItem K/allPlaced/placementClick(_kPrio)/syncMarkers/HUD/メッセージを更新。サーバvalidatePlacement(mountain)も同調(王=fenceRows・砦3=home自由)、applyPlacementは山の王 hidden=森依存。
  - **山AI布陣 刷新** `pickMountainPlacement`: 王=最深3行の森を最優先、砦3/柵3/大駒すべて森を最優先で配置→**王の位置がバレない**(検証:AI布陣で王・砦3・柵3・大駒すべて森に乗る)。
  - **山AI対局** `evalFast`(worker leaf eval)に山判定(`TER.castles`両側0)を追加し、**大駒(飛角)・王が森から平地に出ると強ペナルティ**(王-7/大駒-(4+前進*0.7))・森に居ると加点=狙撃回避で温存。
  - **攻城戦オンライン対応**(=以前「未対応」だったのを実装): サーバ`genTerrain`に siege 分岐(城枡・preset柵をクライアントとRNG完全一致で生成。検証:全seed grid/castles完全一致)、`validatePlacement`に攻城戦分岐(守備=王は城/攻撃=砦2・王は砦上、**攻撃の砦は前方含め盤上どこでも可**=元offlineルール準拠、駒は自陣5行)、create/reroll/rematchで preset柵を`room.fences`にseed。
  - 検証: 地形パリティ(siege/mountain/plain全seed一致)、オフライン回帰(平地comp_test/攻城_siege)、ローカル&本番オンラインE2E(攻城attacker/defender・山・平地すべてplay到達&着手同期&エラー0)。**注意**: __autoplaceは攻城戦で前方布陣するためオンライン検証には不適→canPlace*準拠の正当布陣ヘルパPLACERでテストした。
- **オンライン対戦 実装済み（WebRTC P2P / PeerJS公開ブローカー・サーバ不要でPages完結）**。peerjs.min.js(1.5.4)もHTMLにインライン。タイトル→オンライン対戦→「部屋を作る」(ホスト=先手side0,コード生成 `tshg-XXXXX`)／「コードで参加」(ゲスト=side1)。host が init{seed,density} 送信→guest が同seedで terrain 再生成。両者ローカルで布陣→`{t:'place'}`交換→`startOnlineGame()`でマージ→対局。着手は `{t:'mv',m:lastMove}` を送り `applyRemote()`→`executeMove(OPP,...)`。投了`{t:'resign'}`。**信頼ベース**(各クライアントが相手の伏兵を表示で隠すだけ)。`ME`可変・`OPP=1-ME`・カメラは ME 側に反転(各自自陣が手前/自分=青)。待ったはオンライン無効。NAT越え不可な回線(対称NAT)では繋がらない場合あり(PeerJSにTURN無し)。
- **【2026-06-16】オンラインを権威サーバー方式(B)に変更**: クライアントは Render稼働中の **`wss://terrain-shogi-server.onrender.com`**（repo `agerucompany-ai/terrain-shogi`・元neoshogiのサーバー、`server/index.js`）に WebSocket接続。プロトコル: 送信=create{name,density,hostSide,handLimit}/join{code,name}/resume{code,side,token}/placement{king,pieces,fences,fort}/move{mv:{from,to,promote}}/drop{drop:{type,r,c}}/resign/takeback/reroll{density}/rematch/chat{text}。受信=created/joined/resumed/resume_failed/state{state:publicState(マスク済board,"?"=伏兵)}/info/error/placed/chat。クライアントは army/placement はローカル、play は**サーバーstateを描画**(syncSceneで type"?"はスキップ)。コードは**6桁数字**。利点: 確実接続(NAT無関係)・伏兵マスキング(不正対策)・token再接続。検証: 2クライアントを実機サーバーで接続→布陣→着手同期OK。Renderキー(`RENDER_API_KEY` in .env)は確認のみで使用、サービスは既存だった。**※キーはチャット露出したので要再発行**。無料枠はスリープ(初回接続~1分)。旧P2P(PeerJS)関数は残置(未使用)。
- **切断対策**: 着手毎に局面を `localStorage('tshogi3d')` 保存。切断(conn close)時は自動再接続(ゲストは host id `tshg-CODE` へ再connect、ホストは待受)。再接続時 `{t:'sync',s:全局面,seed,density}` をホストが送って完全復元。キープアライブ ping 4秒毎。タイトルに「前回のオンライン対局に再接続」ボタン(保存1時間以内)。検証: ゲストreload→再接続→同期→双方向着手継続OK。ホストreloadは peer id 一時unavailableで再試行する作り(やや弱い)。

## ユーザの強い要望（厳守。これを外すと激怒した）
1. **構成を元の地形将棋と同一に**：自動配置で省略せず、「**使う駒を選ぶ(合計Lv20)→自陣に配置(王=城・冊3つ・砦)→対局**」の本来フローを実装する。
2. **視点は固定**：ドラッグ回転・ズーム禁止。斜め上の固定カメラのみ。
3. **地形を一体化**：浮いた盤＋枠は禁止。地面・川・森・城・山が**外まで連続した一枚の地形**で、その上を駒が動く（川は盤外へ流す等）。
4. 駒は全種キャラ化。桂馬の騎手と歩は**鎧なし**（軽装＋陣笠）。飛車/角/桂馬は似せず差別化。頭上に駒名ラベル。
5. **スマホ最優先**（bokujoの方針と同じ）。タップ操作・縦画面で盤全体が収まること。

## ビルド/検証手順（重要）
- 巨大なthree.jsインラインを毎回書かないため、`template.html`（プレースホルダ `/*__THREE__*/ /*__RULES__*/ /*__BUILDERS__*/`）を作り、node で差し込んで最終HTMLを組み立てた。素材は `C:\Users\ageru\AppData\Local\Temp\shogi_shot\`(rules.txt/builders.txt/three.min.js)。
- **必ず検証**：`node --check` で構文 → **puppeteer-core + システムChrome**（`--use-angle=swiftshader --enable-unsafe-swiftshader`）でヘッドレス描画・コンソールエラー0・各フェーズ動作・スクショ目視。`window.__dbg()/__click()/__auto()/__autoplace()` のデバッグフックを仕込んで検証した。
- three.min.js は CDN(cdnjs r128)から取得しインライン化（オフライン堅牢化）。
- 関連: [[reference_3d_shogi_verify]]

## ハマりどころ
- three.js r128 では `mesh.position` は読み取り専用 → `Object.assign(mesh,{position:...})` は throw。`.position.set()` を使う。
- 単一HTMLで `id` に空白を入れると getElementById が null。

## AIエンジン（対CPU）
- Web Worker化（テンプレに <script id="aiwsrc" type="text/js-worker">/*__WORKER__*/</script>、assemble.jsで rules(エラーガード除去版 wrules)+worker_search.txt を注入しBlob Worker生成）。反復深化+静止探索(quiescence)+αβ+攻撃的評価(evalStrong=敵王へ前進加点)。強さ選択 弱400ms/普通1.2s/強2.5s/最強6s(最強で深さ5+)。aiMove→postMessage→onAiResultで着手。worker無効時は searchBestMoveAdv にフォールバック。さらに強化は置換表(Zobrist)+make/unmakeが有効(未実装)。
- **再アセンブルは C:UsersageruAppDataLocalTempshogi_shotassemble.js を使う**（three/peerjs/worker/rules/builders/画像WebPを差し込み）。

## カメラ/選択(2026-06-16 修正)
- カメラは**盤四隅+駒高を画面内に収める二分探索オートフィット(fitCamera)**。さらに**視点操作**追加: ドラッグ回転(camAz/camPolar)・ピンチ/ホイール拡大(camR)・「視点」ボタンでリセット。userCamフラグ=手動操作中は再フィットしない。フェーズ変更時(_lastPhase)は自動再フィット。renderHUD末尾はrelayout()。
- **駒が選択できない不具合の真因=駒が背の高い立ち絵で、地面プレーンへのレイが駒の奥マスに当たっていた**。cellAtを**pieceGroup/featGroup(城)/structGroupのスプライト自体をレイキャスト**して最近接の駒/城マスを返すよう修正(城groupにuserData.cell付与)。
- 検証: comp_test.js で PC幅広/縦長/スマホ×密度少/普/多 の9パターンで armyFit/playFit/tapSelect 全true、回転後も選択可。
