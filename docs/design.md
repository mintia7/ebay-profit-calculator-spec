# 海外販売利益計算ツール 技術設計書

文書更新日：**2026-11-08**。実装基準：**2026-11-08の2モード対応版**。

Manusチェックポイント・アプリ実装コミット：**チェックポイント`1769d3ff`／実装コミット`1769d3ff626dfd9e9ad715a75ae506aa6269731e`**。今回の検証結果：**全80テスト・型検査・本番ビルド・ブラウザ確認成功**。文書更新前のGitHubコミット：`7993a64`。

対象はeBay US / CPASSと海外直接取引 / DHL直接契約による日本からの越境販売。関連文書：[機能仕様書](requirements.md) / [README](../README.md) / [変更履歴](../CHANGELOG.md)。本書は現行コードの設計で、将来計画を実装済みと扱わない。定数・式はユーザー確定モデルで、法令の現況、契約条件、実際の徴収額や還付資格を保証しない。

## 1. 全体構成と責務

| 層・ファイル | 現在の責務 |
|---|---|
| `client/src/main.tsx` | React起動、TanStack Query / tRPC Provider、SuperJSON付きHTTPバッチ通信 |
| `client/src/App.tsx` | `/`にHome、NotFound、ErrorBoundary、ライトテーマ、Tooltip、Toaster |
| `client/src/pages/Home.tsx` | 販売モード・入力状態、リアルタイム結果、TTM、eBayライバル比較、履歴操作・CSVダウンロード |
| `client/src/lib/calculator.ts` | 型・定数・送料表、純粋な`calculate`、直接決済率・自動発送費、表示フォーマット |
| `client/src/lib/calculation-records.ts` | 履歴型・保存キー・最大件数、履歴作成・モードラベル、CSV生成の共通helper |
| `client/src/lib/ttm-cache.ts` | 保存済みTTMの構造・USD・保存経過時間の検査 |
| `server/routers.ts` | 既存の公開tRPC、MURC取得、eBay OAuth / Browse API |
| `drizzle/schema.ts` | MySQL向け認証ユーザースキーマのみ。計算用テーブルなし |

主要技術はReact 19、TypeScript 5.9、Vite 7、Tailwind CSS 4、Express 4、tRPC 11、TanStack Query 5、Zod 4、Drizzle ORM、Vitest 2。依存パッケージの存在を機能実装の根拠にしない。

```text
販売モード・入力 → HomeのCalculatorInputs → useMemo(calculate) → モード別結果・換算
MURC HTML ← サーバーGET ← exchangeRate.getTtm ← 初回取得 / 再取得
                          ↓ 成功時保存 / 失敗時だけ復元
                  localStorage（TTM v1、保存から7日以内）
eBay OAuth / Browse API ← ebay.getItemInfo ← eBayモードのライバルitem number
現在結果 → createCalculationHistoryEntry → localStorage（履歴v1、最大10件）
履歴 → getHistoryModeLabel → 保存時の結果を表示（再計算しない）
現在入力・結果 → buildCalculationCsv → Blob / モード名付きダウンロード
```

利益計算はブラウザ内で完結し、計算API・DBへの履歴送信はない。外部情報取得だけをサーバーへ委譲する。直接決済は手数料のモデル計算であり、Wise／Revolut／PayPalのAPI接続や送金処理を実行しない。

## 2. UIと状態管理

`grid-cols-1 lg:grid-cols-2`でPCは左入力・右結果、モバイルは縦積み。ヘッダー上部固定。画面名称・`document.title`は「海外販売利益計算ツール」。

モード選択を上段に置き、eBayはCPASS・DDP送料・広告・ライバル比較、直接は決済方法・DHL配送先・任意の請求送料を表示する。直接モードではDDP送料表・eBay広告入力・ライバル比較を隠す。詳細パラメータは説明用で、FVF等の編集フォームではない。

右側は還付込み利益のサマリー、販売モード・配送／決済条件、DDP/DDU、粗利と両利益率、フロア警告、費目・手数料明細、履歴・CSVを持つ。サマリー色は還付込み利益の正負、警告は粗利で別判定。ゲージ幅は制限するが利益率数値は制限しない。

`inputs`はReact state、結果は`useMemo(() => calculate(inputs), [inputs])`。結果・保存・CSVの有効条件は商品価格と仕入価格がともに正数。リセットは`DEFAULT_INPUTS`へ戻すだけで、履歴・ライバル・TTMキャッシュを削除せずTTM再取得もしない。

`switchSalesChannel`は異なるモードへ切り替えると`useAutoDomesticShipping=true`に戻し送料表を閉じる。同一モード選択では何もしない。他の値は保持し、非対象のeBay送料・広告値を直接計算に使わない。配送先変更は単独では手動発送費を解除しない。

通常の`NumInput`はテキスト入力を`parseInt`、`decimals>0`なら`parseFloat`で解析。商品価格・仕入・広告・eBay手動送料・手動発送費は整数解釈、直接請求送料は`decimals={2}`。為替専用入力は`parseFloat`で小数2桁表示。`step`・`min`等はこのテキスト入力で強制検証されず、計算関数にも包括的な入力検証はない。

## 3. 計算の入出力型

### 3.1 `CalculatorInputs`

| フィールド | 型・意味 / 初期値 |
|---|---|
| `salesChannel` | `SalesChannel = "ebay" \| "direct"`、既定`ebay` |
| `directPaymentMethod` | `DirectPaymentMethod = "wise" \| "revolut" \| "paypal"`、既定`wise` |
| `directDestination` | `DirectDestination = "hongKong" \| "singapore" \| "other"`、既定`other` |
| `directShippingUsd` | `number`、直接請求送料。0、eBay手動送料と独立 |
| `itemPriceUsd`, `purchasePriceJpy` | `number`、送料を除く商品価格USD / 税込仕入JPY。ともに0 |
| `promotedListingRate`, `useAutoPromotedListingRate` | `number` / `boolean`、eBay手動広告2%、自動判定false |
| `taxRefundEligibility` | `"qualified" \| "nonQualified"`、既定`nonQualified` |
| `exchangeRate` | `number`、USD/JPY=155 |
| `exchangeRateEur`, `exchangeRateGbp` | `number`、EUR/JPY=165、GBP/JPY=200 |
| `exchangeRateCad`, `exchangeRateAud` | `number`、CAD/JPY=110、AUD/JPY=100 |
| `useAutoShipping`, `manualShippingUsd` | `boolean` / `number`、eBay請求送料。true / 0 |
| `useAutoDomesticShipping`, `manualDomesticShippingJpy` | `boolean` / `number`、共通の保険込発送費。true / 0 |

### 3.2 `CalculatorResult`

数値は原則`number`で未丸めのまま返す。販売モード自体は結果型に追加せず、画面・履歴・CSVは入力も併せて参照する。

| グループ | 実フィールド |
|---|---|
| 送料・売上 | `shippingFeeUsd`, `totalSalePriceUsd`, `buyerShippingCostJpy`, `customsShippingJpy`, `revenueJpy` |
| eBay手数料 | `feeBasisUsd`, `fvfUsd`, `internationalFeeUsd`, `promotedListingFeeUsd`, `fixedOrderFeeUsd`, `ebayFeeTaxUsd`, `ebayTotalFeeUsd` |
| 広告 | `appliedPromotedListingRate`, `isAutoPromotedListingRate`, `wasPromotedListingDisabledByMargin` |
| 直接決済・受取・合計 | `directPaymentFeeUsd`, `appliedDirectPaymentRate`, `netReceiptUsd`, `payoneerFeeUsd`, `totalFeeJpy` |
| 発送 | `autoDomesticShippingJpy`, `domesticShippingJpy`, `totalShippingJpy` |
| 粗利・フロア | `grossProfitJpy`, `grossProfitRate`, `profitFloorJpy`, `isBelowProfitFloor` |
| 還付・還付後利益 | `taxRefundEligibility`, `taxRefundRate`, `taxRefundJpy`, `netProfitJpy`, `netProfitRate` |
| 他通貨・方式 | `priceGbp`, `priceEur`, `priceCad`, `priceAud`, `isDdp` |

`customsShippingJpy`は旧UI互換名で`buyerShippingCostJpy`と同額、直接ではともに0。`taxRefundRate`は1または0.7。広告・フロア・DDPの判定フィールドはboolean、他通貨価格は送料を除く商品価格の換算値。

## 4. 計算順序とモード分岐

1. `isDirect = salesChannel === "direct"`。直接ならS=`directShippingUsd`、eBayなら自動送料表または`manualShippingUsd`。表は上限以下の最初の帯、$2,499超〜$2,500未満は最終$375、$2,500以上は0。
2. T=P+S、R=T×X。買い手送料の同額費用KはeBayでS×X、直接は0。直接請求送料を実際のDHL費と二重控除しない。
3. BはeBayでT×1.08、直接でT。eBayのみFVF二段階（Bの$2,500まで12.35%、超過2.35%）、国際1.35%、Tが$10.01以上なら固定$0.40。
4. 直接決済率はWise・Revolut 1%、PayPal 5%。直接決済手数料H=T×率。eBayでは率・Hともに0。
5. 自動発送費はeBay CPASS基本¥5,000、直接DHL香港／シンガポール¥5,500、その他¥8,500。全方法でP<$800は保険なし、以上は`Math.round(基本額 + max(2500, P×X×0.012))`。
6. 手動発送費は`Math.round(max(0, manualDomesticShippingJpy))`でD全体を置換。自動候補も結果へ残し、送料合計=D+K。
7. 広告は直接0%、eBay自動なら3%、eBay手動なら入力率。eBay手数料EはFVF・国際・広告・固定費合計に10%を加算。直接はこれらすべて0。
8. ネット受取=T−E−H。PayoneerはeBayのみネット受取×2%、直接0。合計手数料円=(E+Payoneer+H)×X。G=R−J−合計手数料−D−K、粗利率=R>0ならG/R×100、他は0。

eBay自動広告のみ3%試算時の未丸め粗利率が10%未満なら0%で一度再計算。10%ちょうどは3%を維持し、再計算後の反復はない。直接では適用率0%、自動広告フラグ・広告停止フラグともfalse。保持中の広告入力は無効。PL保険の別費用は計上しない。

共通還付はJ×10/110×k（適格1、非適格0.7）。仕入のみが基礎で送料・保険は含めない。非適格70%は固定ユーザーモデルで、UIの年月説明を法令の現況保証としない。利益フロアは`max(5000, Math.round(J×0.03))`、未丸めG<Fなら警告、等値は達成。N=G+還付。警告は販売・保存を禁止しない。

`isDdp = !isDirect && P < 2500`。手動送料に左右されず、直接は全商品DDU。他通貨の`price*`はP×X÷対象TTM（レート非正数なら0）。直接の多通貨「販売価格」欄はHomeがT×X÷対象TTMを別計算し、請求送料を含む。eBayはUSDのみDDPでT、DDUでP、他4通貨はP換算のまま。

## 5. tRPC APIと外部取得（既存構成）

共通経路は`/api/trpc`、`httpBatchLink`＋SuperJSON、Cookie送信あり。TTMはtRPC React Query、ライバル取得はHomeからバッチURLを直接`fetch`し`json[0].result.data.json`を読む。両取得は`publicProcedure`で、画面にログイン必須ガードはない。

| API | 入力 | 成功レスポンス | 業務取得失敗レスポンス |
|---|---|---|---|
| `exchangeRate.getTtm`（query） | なし | `{ success: true, rates, date, source }` | `{ success: false, rates: null, date: null, source: null, error }` |
| `ebay.getItemInfo`（query） | `{ itemNumber: string }` | `{ success: true, title, itemPriceUsd, shippingUsd, itemUrl, currency }` | `{ success: false, title: null, itemPriceUsd: null, shippingUsd: null, itemUrl: null, currency: null, error }` |

`rates`は`Record<string, number>`でUSD以外は欠け得る。dateは`YYYY/MM/DD`、sourceは「三菱UFJ銀行 TTM」。業務取得例外とtRPC通信障害・入力検証エラーは別経路。商品番号はZodで1〜20文字の数字のみを要求し、クエリ本体より前で拒否する。画面の直接fetchはtRPCエラー詳細の専用解析をしない。

### 5.1 TTM

MURCのGET `https://www.murc-kawasesouba.jp/fx/past/index.php?id=YYMMDD`を利用。JST前日から最大10暦日を遡り、土日はスキップする（10営業日でも当日でもない）。祝日カレンダーはなく、有効USDが取れない日はさらに遡る。

Shift_JIS復号、通貨名と`td class="t_right"`に依存した抽出、TTS/TTB平均の小数2桁丸め。USD取得で成功とし他4通貨完備は必須にしない。全候補失敗は「前営業日のTTMデータが見つかりませんでした」。既存構成にサーバー共有キャッシュ・明示的fetchタイムアウト・HTML変更の保証はない。

### 5.2 eBay商品取得

サーバー専用のOAuth設定を用いてclient credentialsで取得し、トークンはプロセスメモリで期限残60秒超なら再利用する。設定の実値・認証情報は公開文書に含めない。

Browse APIの`v1|itemNumber|0`形式、HTTP失敗時は`get_item_by_legacy_id`へフォールバック、marketplaceは`EBAY_US`。価格・通貨・タイトルと最初の配送候補送料を抽出し、欠落価格・送料は0、通貨はUSD。為替換算や最適配送先選択はない。

画面で取得後の手動修正可。比較は価格＋送料が正数の自分・A/Bの最安値と差額。直接モードでは比較UIを隠すだけで既存APIは存続する。継続監視・商品検索・自動改定・出品操作はない。既存`auth.me`／`auth.logout`は認証状態取得／Cookie消去で、計算履歴のアカウント同期とは関係しない。

## 6. ブラウザ保存と互換性

### 6.1 TTMキャッシュ

キー`ebay-profit-calculator.ttm-cache.v1`を維持。成功時に`{ rates, date, source, cachedAt: Date.now() }`をJSON保存。初回と再取得ボタンは通信を先に試し、失敗時のみ復元する。自動再試行は無効。

`getFreshTtmCache`はオブジェクト・number型cachedAt・rates・有限で正のUSD・文字列date/sourceを検査。保存時刻から7×24時間以内（ちょうど7日を含む）なら採用、超過・未来は不採用。再利用時にcachedAtを更新しない。

無効保存値は可能なら削除。JSON解析・ストレージ例外は復元不能扱い。復元成功はcached状態で相場日と警告を表示、失敗は手動入力案内。各通貨はtruthy値のみ反映し、欠落通貨は初期値／手動値を維持。他通貨の有限性・正数・日付整合性、cachedAtの有限性は明示検査せず、全通貨が同日実測値で揃う保証はない。

### 6.2 計算履歴

キー`ebay-profit-calculator.history.v1`、`MAX_HISTORY_ENTRIES=10`を共通helperへ集約。保存は`createCalculationHistoryEntry(inputs, result)`、最新を先頭に追加し10件へ切り詰める。

| フィールド | 保存仕様 |
|---|---|
| `id`, `createdAt` | 時刻＋ランダム文字列のID、保存時刻 |
| `itemPriceUsd`, `purchasePriceJpy`, `shippingFeeUsd` | 商品価格・仕入価格・計算で採用した買い手送料 |
| `revenueJpy`, `grossProfitJpy`, `netProfitJpy` | 保存時の計算値（表示丸め前） |
| `salesChannel?` | 新履歴で保存。旧履歴のため省略可能 |
| `directPaymentMethod?`, `directDestination?` | 新しい直接履歴のみ。eBayではundefined、JSON保存時は省略 |
| `domesticShippingJpy?` | 新履歴の採用保険込発送費 |
| `calculationVersion?` | 新履歴は`"2026-11-08"` |

`getHistoryModeLabel`はモードなしを「eBay（旧計算）」、新eBayを「eBay / CPASS」、直接を「直接 / 決済方法 / 配送先」とする。旧データはv1のまま読み、保存時の旧送料結果を保持し、改定後の式で再計算・上書きしない。

初期読込はJSONの配列判定のみ、先頭10件を復元。個別・全削除は状態更新後に保存。表は日時・取引配送ラベル・商品価格・仕入・粗利・還付後利益を表示。**表示復元のみで、完全入力再読み込み・再計算は未実装**。為替、仕入区分、広告モード、フロア判定、全入力・全結果、商品名は保存しない。

同一ブラウザ・オリジン限定、DB同期・クラウド共有・消失防止バックアップなし。読込要素の厳密検証と書込時のストレージ例外処理はない。追加フィールドは互換用にoptionalで、旧履歴に後付けの値を推定しない。

## 7. CSV共通helperと丸め

`buildCalculationCsv(inputs, result)`は副作用なく現在の1件を2列「項目,値」で生成。UTF-8 BOM、CRLF、全セル二重引用符、セル内引用符の二重化。HomeがBlob生成・クリック・URL解放を行う。

ファイル名は`overseas-profit-{channel}-YYYY-MM-DD.csv`、channelは`ebay`／`direct`、日付は`toISOString()`由来のUTC。履歴一括CSVやCSVインポートではない。

| 出力項目（実際の順序） | 値・丸め |
|---|---|
| ツール名、取引モード、発送方法、関税方式 | 海外販売利益計算ツール、eBay／海外直接取引、CPASS／DHL直接契約、DDP／DDU |
| 配送先、決済方法、決済手数料率（%） | 直接のみ追加 |
| 商品価格（USD）、買い手請求送料（USD）、販売価格合計（USD）、仕入価格（JPY） | 元入力／計算値を文字列化 |
| 仕入区分、USD/JPY TTM | 適格100%／非適格70%、入力レート |
| 広告料率モード、適用広告料率（%） | 直接は対象外・0、eBayは自動判定／手動入力・採用率 |
| 売上（JPY） | `Math.round` |
| eBay手数料、Payoneer手数料、直接取引の決済手数料（USD） | 各`toFixed(2)`、非対象費用も0として出力 |
| 合計手数料（JPY） | `Math.round` |
| 保険込送料モード、保険込送料（JPY） | 自動／手動、計算済み整数 |
| 買い手送料の実費控除（JPY）、粗利（JPY） | `Math.round`、直接の実費控除は0 |
| 粗利率（%） | `toFixed(2)` |
| 消費税還付率（%）、消費税還付額（JPY） | `taxRefundRate×100`、`Math.round` |
| 粗利フロア（JPY）、還付後利益（JPY） | 計算済み整数、`Math.round` |

他4通貨TTM、FVF等の全内訳、還付後利益率、フロア警告フラグ、全入力モード、履歴日時はCSVに含まれない。画面JPYは整数、USDは2桁、利益率は通常1桁。費目ごとのUSD丸めは計算へ戻さず、広告・フロア判定は未丸め値で行う。

## 8. DB・起動・検証記録

Drizzleは認証用usersのみで、計算履歴・TTM・ライバル・商品台帳のテーブルはない。文書更新にDB変更は不要。

既存のアプリコマンドは`pnpm dev`、`pnpm check`（tsc --noEmit）、`pnpm test`（vitest run）、`pnpm build`（Vite＋esbuild）、`pnpm start`、`pnpm db:push`。この公開文書リポジトリ単独ではアプリを起動・テストできない。

2026-11-08の今回版で**全80テスト（5ファイル）・`pnpm check`・`pnpm build`が成功**。実装作業側の実行記録に基づく内訳はcalculator 67件、records 4件、title 2件、ebay 6件、logout 1件。

ブラウザでは直接$500・Wise・その他の発送費¥8,500、直接$800・PayPal・香港の発送費¥8,000・請求送料$20に対する決済手数料$41・粗利表示¥58,850を確認。eBayへ戻した際のCPASS保険込送料¥7,500・広告2%、手動発送費¥9,999の上書き、モード切替で自動発送費へ戻る挙動、375pxモバイル表示も確認成功。これは操作時の確認記録で、文書の固定TTM検算表とは別の入力条件である。

文書検算は現行`calculate`のtsx実行により、eBay4例と直接5例（Wise／Revolut／PayPal、香港／シンガポール／その他、$799／$800保険境界、保険料率が下限を超える例）を確認した。直接ではeBay入力が保持されても関連費用・広告フラグが混入しないことも確認した。数値は[機能仕様書](requirements.md)に掲載。

過去版`bcf2bec8`では全18テスト・型検査・本番ビルド成功、2026-11-08の前版文書反映時にも全18テスト・型検査成功の記録がある。これは今回の2モード版の総件数・成功を示すものではなく、[変更履歴](../CHANGELOG.md)の過去記録として保持する。

既存外部APIテストが失敗応答を許容する場合、テスト成功は実取得や継続正常動作の保証ではない。手動ブラウザ確認とは別に、自動ブラウザE2EやMURC休日・HTML変更・タイムアウト等の網羅的検証を実装済みとは扱わない。

## 9. 未実装事項と差分管理

未実装は在庫監視・定期最安値チェック、自動価格改定、eBay出品操作、複数商品一括入力、履歴の完全再読み込み・一括CSV、クラウド履歴同期、税率年月自動切替、各国別の税手数料計算、固定料率UI編集、ManusからGitHubへの自動同期。直接取引の試算は対応済みだが、決済サービス実行・DHLラベル発行は行わない。

今後は定数・型・UI・CSV・保存形式・テストを一組の差分としてレビューし、丸め・閾値・モード独立性・旧履歴互換を検算する。完全入力保存には旧データ移行／バージョン方針、TTM変更には相場日と保存日・欠落通貨の扱いを明示する。

アプリ保存とGitHub文書の手動同期は別操作のまま。今回の保存版のチェックポイント・実装コミットは冒頭に記録しています。archiveは変更せず履歴資料としてのみ扱い、公開文書に秘密設定の実値・認証情報を含めない。
