# 技術設計書

- 文書更新日: **2026-11-08**
- 実装基準: `bcf2bec81820a2b899712541fd20d0b8503baa1c`（2026-10-19）
- 対象: eBay US向け、日本からの越境販売の利益試算アプリ
- 関連文書: [機能仕様書](requirements.md) / [README](../README.md)

本書は上記コミットのコード確認に基づく現状設計であり、将来計画を実装済みとは扱わない。税・関税・手数料はユーザーが確定した試算モデルとして記述し、法令への適合、実際の徴収額や還付資格・還付額を保証しない。

## 1. 全体構成と責務

| 層・ファイル | 現在の責務 |
| --- | --- |
| `client/src/main.tsx` | React起動、TanStack Query / tRPC Provider、SuperJSON付きHTTPバッチ通信 |
| `client/src/App.tsx` | `/`にHome、`/404`と未定義パスにNotFound。ErrorBoundary、ライトテーマ、Tooltip、Toaster |
| `client/src/pages/Home.tsx` | 入力状態、リアルタイム結果、TTM取得・保存、ライバル2件比較、履歴、CSV |
| `client/src/lib/calculator.ts` | 定数、送料表、入出力型、副作用を持たない`calculate`、表示フォーマット |
| `client/src/lib/ttm-cache.ts` | 保存済みTTMの構造・USD・保存経過時間の検査 |
| `server/routers.ts` | 公開tRPCクエリ、MURCのHTML取得、eBay OAuth / Browse API呼び出し |
| `drizzle/schema.ts` | MySQL向け認証ユーザースキーマのみ。計算用テーブルなし |

主要技術はReact 19、TypeScript 5.9、Vite 7、Tailwind CSS 4、Express 4、tRPC 11、TanStack Query 5、Zod 4、Drizzle ORM、Vitest 2。依存パッケージの存在だけでは、個別の業務機能が実装されていることを意味しない。

```text
画面入力 → HomeのCalculatorInputs → useMemo(calculate) → 結果・比較表示
                          ↑
MURC HTML ← サーバーGET ← exchangeRate.getTtm ← 初回取得 / 再取得
                          ↓ 取得成功時保存 / 失敗時だけ復元
                  ブラウザlocalStorage（TTM）
eBay OAuth / Browse API ← ebay.getItemInfo ← ライバルitem number
                          ↓
                   ライバル価格・送料・タイトル
現在結果 → 明示的な保存 → localStorage（最大10件の要約履歴）
現在入力・結果 → CSV生成 → Blob / ブラウザダウンロード
```

利益計算はブラウザで完結し、計算APIやDBへの履歴送信はない。外部情報取得だけをサーバーへ委譲する。

## 2. UIと状態管理

既存レイアウトは`grid-cols-1 lg:grid-cols-2`。デスクトップは左に入力、右に結果、狭い画面では縦積みとし、ヘッダーは上部固定である。

左カラムは必須価格、買い手送料の自動・手動切替、広告・保険込送料・仕入区分・5通貨TTM、説明用の詳細パラメータ、多通貨表示、ライバル比較を持つ。詳細パラメータは固定モデルの説明であり、FVF等の編集フォームではない。送料早見表はモーダル表示する。

右カラムは還付込み利益のサマリー、粗利・利益率ゲージ、フロア警告、コスト内訳バー、売上、手数料詳細、利益内訳、履歴・CSV操作を持つ。ゲージの表示幅は制限するが、数値の利益率そのものは制限しない。

`inputs`はReact state、結果は`useMemo(() => calculate(inputs), [inputs])`で都度更新する。結果表示・保存・CSVの有効条件は商品価格と仕入価格がともに正数であることのみ。リセットは入力を`DEFAULT_INPUTS`へ戻し、履歴・ライバル・TTMキャッシュは消さず、TTM再取得も実行しない。

数値入力は主にテキスト入力で、通常の`NumInput`は`parseInt`、小数指定時は`parseFloat`を使用する。現画面の商品価格・広告料率・手動送料等は整数解釈だが、計算関数は小数を扱える。為替専用入力は`parseFloat`で、小数2桁の表示を行う。`step`や`min`の宣言は通常の数値入力の強制検証として機能せず、計算関数にも包括的な入力検証はない。

## 3. 計算の入出力型

### 3.1 `CalculatorInputs`

| フィールド | 型・意味 / 初期値 |
| --- | --- |
| `itemPriceUsd`, `purchasePriceJpy` | `number`、商品価格USD / 税込仕入JPY。いずれも0 |
| `promotedListingRate` | `number`、手動広告料率%。2 |
| `useAutoPromotedListingRate` | `boolean`、3%試算後の自動判定。false |
| `taxRefundEligibility` | `"qualified" \| "nonQualified"`、仕入区分。`nonQualified` |
| `exchangeRate` | `number`、USD/JPY。155 |
| `exchangeRateGbp`, `exchangeRateEur` | `number`、GBP/JPY=200、EUR/JPY=165 |
| `exchangeRateCad`, `exchangeRateAud` | `number`、CAD/JPY=110、AUD/JPY=100 |
| `useAutoShipping`, `manualShippingUsd` | `boolean` / `number`、買い手請求送料。true / 0 |
| `useAutoDomesticShipping`, `manualDomesticShippingJpy` | `boolean` / `number`、保険込発送費。true / 0 |

### 3.2 `CalculatorResult`

数値は原則JavaScriptの`number`で保持し、表示前の一律丸めは行わない。広告の適用結果、仕入区分・還付率、利益フロアは型の一部として返す。

| グループ | 実フィールド |
| --- | --- |
| 買い手送料・売上 | `shippingFeeUsd`, `totalSalePriceUsd`, `buyerShippingCostJpy`, `customsShippingJpy`, `revenueJpy` |
| eBay手数料 | `feeBasisUsd`, `fvfUsd`, `internationalFeeUsd`, `promotedListingFeeUsd`, `fixedOrderFeeUsd`, `ebayFeeTaxUsd`, `ebayTotalFeeUsd` |
| 広告判定 | `appliedPromotedListingRate`（数値）、`isAutoPromotedListingRate`, `wasPromotedListingDisabledByMargin`（boolean） |
| 受取・合計手数料 | `netReceiptUsd`, `payoneerFeeUsd`, `totalFeeJpy` |
| 発送費 | `autoDomesticShippingJpy`, `domesticShippingJpy`, `totalShippingJpy` |
| 粗利・フロア | `grossProfitJpy`, `grossProfitRate`, `profitFloorJpy`, `isBelowProfitFloor`（boolean） |
| 還付・還付後利益 | `taxRefundEligibility`（仕入区分）、`taxRefundRate`, `taxRefundJpy`, `netProfitJpy`, `netProfitRate` |
| 他通貨・方式 | `priceGbp`, `priceEur`, `priceCad`, `priceAud`, `isDdp`（boolean） |

`customsShippingJpy`は旧UI互換名で、`buyerShippingCostJpy`と同額。`taxRefundRate`は100や70ではなく、1または0.7である。

## 4. 計算順序と境界

1. 自動買い手送料は静的な`SHIPPING_TABLE`を上から検索し、商品価格が各段の`maxPrice`以下なら採用する。$100は$15、$100.50は$22、$2,499.01〜$2,499.99は最終帯$375。$2,500以上は$0。手動モードは入力送料を使用する。
2. 販売価格合計は商品価格＋買い手送料、売上は販売価格合計×USD/JPY。買い手送料の円換算額を、同額の費用としても計上する。
3. 手数料ベースは販売価格合計×1.08。Sales Tax 8%は手数料算定用に加えるが、売上や独立したセラー費用としては加算しない。
4. FVFは手数料ベースの$2,500まで12.35%、超過分2.35%。商品価格で判定するDDP/DDU境界とは別である。国際手数料はベース×1.35%、広告料はベース×適用広告率。
5. 注文固定費は販売価格合計が$10.01以上なら$0.40、未満なら$0。FVF・国際手数料・広告料・固定費の合計へ10%を加え、eBay合計手数料とする。
6. ネット受取は販売価格合計−eBay合計手数料。Payoneerはネット受取×2%、合計手数料JPYは両手数料の合計×USD/JPY。
7. 保険込送料は商品価格$800未満なら¥4,000、以上なら`Math.round(4000 + max(2500, 商品価格 × USD/JPY × 0.012))`。手動時は`Math.round(max(0, 入力送料))`。自動送料の候補値も結果に残す。
8. 粗利は売上−仕入−合計手数料−保険込送料−買い手送料円換算。送料合計は後者2項の合計。粗利率の分母は売上JPYで、売上が正数でなければ0。

手動広告の既定値は2%であり、低粗利でも自動変更しない。任意の自動広告モードだけが3%で試算し、その時点の未丸め粗利率が10%未満なら0%で一度再計算する。10%ちょうどは3%維持で、再計算後に10%以上になっても3%へ戻す反復処理はない。PL保険の別費用は計上しない。

還付モデルは`仕入価格 × 10/110 × 還付率`。適格仕入は100%、非適格仕入は固定70%、初期区分は非適格とする。送料・保険は対象外。日付から率を切り替える機能や資格を審査する機能はなく、「2026年10月以降」というUI表記はユーザー確定モデルの説明である。

利益フロアは`max(5000, Math.round(仕入価格 × 0.03))`。**未丸めの最終粗利**がフロア未満なら警告し、還付は判定に含めない。等値は達成扱い。還付後利益は粗利＋還付額であり、フロア判定は販売や保存を禁止しない。

他通貨の商品価格は`商品価格USD × USD/JPY ÷ 対象通貨/JPY`、対象レートが正数でない場合は0。GBP/EUR/CAD/AUDは画面上DDUとして商品価格のみを表示し、各国の税・手数料を再計算する設計ではない。`isDdp`は商品価格$2,500未満だけで決まり、手動送料を入れても変わらない。

## 5. tRPC APIと外部取得

クライアント共通経路は`/api/trpc`、`httpBatchLink`＋SuperJSON、Cookie送信あり。TTMはtRPC React Queryを使うが、ライバル取得はHomeからバッチ形式のURLを直接`fetch`し、`json[0].result.data.json`を読む。為替・商品取得はいずれも`publicProcedure`で、画面にログイン必須のガードはない。

| API | 入力 | 成功レスポンス | 業務取得失敗レスポンス |
| --- | --- | --- | --- |
| `exchangeRate.getTtm`（query） | なし | `{ success: true, rates, date, source }` | `{ success: false, rates: null, date: null, source: null, error }` |
| `ebay.getItemInfo`（query） | `{ itemNumber: string }` | `{ success: true, title, itemPriceUsd, shippingUsd, itemUrl, currency }` | `{ success: false, title: null, itemPriceUsd: null, shippingUsd: null, itemUrl: null, currency: null, error }` |

`rates`は`Record<string, number>`で、USD以外は欠け得る。`date`は`YYYY/MM/DD`、`source`は「三菱UFJ銀行 TTM」。取得時の例外は上表の`success: false`へ変換するが、tRPC通信障害や入力検証エラーは別のエラー経路になる。

商品番号はZodで1〜20文字の数字のみを要求し、違反はクエリ本体の`try/catch`より前で拒否する。画面の直接fetchはtRPCエラー詳細の専用解析を行わず、取得失敗と表示する場合がある。`auth.me`は認証済みコンテキストのユーザーまたはnull、`auth.logout`はセッションCookieを消去して`{ success: true }`を返す。認証情報・識別値は本書の対象外とする。

### 5.1 MURCのTTM取得

サーバーは`https://www.murc-kawasesouba.jp/fx/past/index.php?id=YYMMDD`へGETする（POSTではない）。JSTの日付を基準に**前日から最大10暦日**を順番に遡り、土日はリクエストせずスキップする。10営業日を検索するわけではなく、当日分も対象外である。

祝日カレンダーは持たない。祝日・ページ未公開・HTTP失敗・解析失敗などで有効なUSDを取れなければ、さらに前日に遡る。レスポンスはShift_JISで復号し、通貨名と`td class="t_right"`に依存した正規表現でTTS/TTBを抽出、`(TTS + TTB) / 2`を小数2桁へ丸める。USDが取れれば成功とし、他4通貨の完備は必須にしない。

全候補が失敗した場合のエラーは「前営業日のTTMデータが見つかりませんでした」。HTML変更への耐性、明示的なfetchタイムアウト、サーバー側TTMキャッシュは現状ない。

### 5.2 eBay商品取得

サーバー環境変数`EBAY_CLIENT_ID`、`EBAY_CLIENT_SECRET`を用い、client credentialsでアクセストークンを取得する。値はクライアントへ渡さず、トークンはプロセスメモリに保持し、有効期限まで60秒を超えて残っている場合に再利用する。

Browse APIの`v1|itemNumber|0`形式で取得し、HTTP失敗時は`get_item_by_legacy_id`へフォールバックする。マーケットプレイスは`EBAY_US`。価格・通貨・タイトルと最初の配送候補の送料だけを抽出し、欠落価格・送料は0、欠落通貨はUSDとする。通貨換算や配送先に応じた最適送料の選択はない。

APIキー未設定、OAuth失敗、商品不在、APIエラー等はエラー文字列に変換する。画面はライバルA/Bの価格・送料・タイトルへ反映し、取得後も手動修正可能。正の「商品価格＋送料」で自分と比較し、最安値を示す。継続監視・自動改定・商品検索は行わない。

## 6. ブラウザ保存

### 6.1 TTMキャッシュ

キーは`ebay-profit-calculator.ttm-cache.v1`。成功時に`{ rates, date, source, cachedAt: Date.now() }`をJSON保存する。初回マウントと再取得ボタンでまずネットワーク取得し、取得失敗時だけ保存値の復元を試みる。クエリの自動再試行は無効である。

`getFreshTtmCache`はオブジェクト、`cachedAt`のnumber型、`rates`オブジェクト、有限で正のUSD、文字列の`date`・`source`を検査する。期限は相場日`date`ではなく**保存時刻`cachedAt`から7×24時間**で、7日ちょうどは有効、超過・未来保存時刻は無効。再利用時は`cachedAt`を更新しない。

無効な保存値は可能なら削除し、復元不能なら取得失敗と手動入力案内を表示する。復元成功時は`cached`状態で保存済み相場日と警告を表示する。JSON解析・ストレージアクセスの例外は復元不能扱い。

各通貨は値がtruthyの場合だけ適用するため、未取得通貨は現在の入力値（初期値または手動値）を保持する。他通貨の有限性・正数・日付整合性はキャッシュ検査の対象外であり、5通貨が同一日の実測値で揃った保証はない。`cachedAt`の有限性も明示検査していない。

### 6.2 計算履歴

キーは`ebay-profit-calculator.history.v1`。保存ボタンで最新の要約を先頭へ追加し、最大10件に切り詰める。保存フィールドは`id`, `createdAt`, `itemPriceUsd`, `purchasePriceJpy`, `shippingFeeUsd`, `revenueJpy`, `grossProfitJpy`, `netProfitJpy`のみ。数値は計算値のままである。

初期表示時に保存配列を読み、配列なら先頭10件を復元する。個別削除・全削除は状態更新後に保存する。表は日時・出品価格・仕入価格・粗利・還付後利益を表示する。**履歴の表示復元はできるが、入力への再読込・再計算はできない**。仕入区分、広告設定、為替、全入力・全結果、商品名は保存しない。

履歴は同一ブラウザ・オリジン内だけで、DB同期・アカウント共有はない。読込時は配列判定のみで要素の厳密検証はなく、履歴書込時のストレージ例外も処理していない。ブラウザ保存の消去・容量制限等による消失を防ぐバックアップ機能はない。

## 7. CSVと丸め

CSVは**現在の1件**を「項目,値」の2列で出力し、履歴一括出力ではない。UTF-8 BOM、CRLF、全セル二重引用符、セル内引用符の二重化を使用する。ファイル名は`ebay-profit-YYYY-MM-DD.csv`で、日付は`toISOString()`由来のUTC日付。Blob URLはクリック後に解放する。

| 出力項目（実際の順序） | 値・丸め |
| --- | --- |
| 出品価格（USD）、買い手請求送料（USD）、販売価格合計（USD）、仕入価格（JPY） | 元入力または計算値をそのまま文字列化 |
| 仕入区分 | 適格請求書あり（還付100%） / 非適格仕入（還付70%） |
| USD/JPY TTM | 入力値をそのまま |
| 広告料率モード、適用広告料率（%） | 自動判定 / 手動入力、実適用率 |
| 売上（JPY） | `Math.round` |
| eBay手数料（USD）、Payoneer手数料（USD） | `toFixed(2)` |
| 保険込送料（JPY） | 計算済み整数 |
| 買い手送料の実費（JPY）、粗利（JPY） | `Math.round` |
| 粗利率（%） | `toFixed(2)` |
| 消費税還付率（%）、消費税還付額（JPY） | `taxRefundRate × 100`、`Math.round` |
| 粗利フロア（JPY）、還付後利益（JPY） | 計算済み整数、`Math.round` |

出力に他4通貨TTM、FVF等の全内訳、還付後利益率、フロア警告フラグ、全入力モード、履歴日時は含まれない。CSVからのインポートもない。画面のJPYは整数丸め、USDは小数2桁、利益率は通常小数1桁の表示であり、CSVの粗利率2桁表示とは異なる。表示丸めを広告・フロア判定へ逆流させない。

## 8. DB・起動・テスト

Drizzleには認証用`users`テーブルのみがあり、ログイン識別・プロフィール・ロール・日時を管理する。計算履歴、TTM、ライバル、商品台帳のテーブルはない。認証DBが存在しても、計算履歴の永続化先はlocalStorageである。

`package.json`のコマンドは`pnpm dev`（tsx watch）、`pnpm check`（tsc --noEmit）、`pnpm test`（vitest run）、`pnpm build`（Viteクライアント＋esbuildサーバー）、`pnpm start`（本番Node起動）、`pnpm db:push`（Drizzle生成・マイグレーション）。文書のみの更新にDB変更は不要である。

VitestはNode環境で`server/**/*.test.ts`と`server/**/*.spec.ts`を対象とし、`@`等のエイリアスでクライアント計算関数も直接検証する。`server/calculator.spec.ts`はTTM保存期限境界・USD異常値、送料表の小数境界、$800保険境界、各種手数料、手動/自動広告、仕入のみの還付、70%/100%、粗利フロア、FVF超過帯、検算例を確認する。

`server/ebay.test.ts`は実ネットワーク依存のTTM・商品API応答形状、商品番号検証、未認証`auth.me`を確認する。ただし外部取得の失敗応答も正常なテスト結果として許容するため、テスト成功は外部APIの取得成功を保証しない。成功時の5通貨検証も、本体がUSDだけで成功を返す条件より厳しい。

基準コミットの記録では全18テスト・型検査・本番ビルド成功。文書更新時の2026-11-08にも現行アプリの全18テストと型検査を再実行し成功した。ただし、この結果は外部APIの継続的な正常動作を保証しない。ブラウザE2E、履歴・CSVの自動検証、MURC休日・HTML変更・タイムアウトの網羅的モック検証は確認したテストにない。

## 9. 未実装事項と今後の差分管理

現状未実装は、在庫監視・定期最安値チェック、価格の自動改定、eBayへの出品操作、複数商品一括入力、履歴の完全再読込・一括CSV、クラウド履歴同期、税率の年月自動切替、各国別利益計算、FVF等固定料率のUI編集である。説明文や依存関係から実装済みと推定しない。

今後は計算定数・型・UI・CSV・保存形式・テストを一組の差分として更新し、丸めと閾値の境界を検算する。履歴に完全入力を追加する場合は保存キーのバージョンや旧データ移行、TTM取得を変更する場合は相場日と保存日の区別・欠落通貨の扱いを明示する。文書の基準コミットと更新日も併せて更新する。
