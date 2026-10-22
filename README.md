# YieldAnalytics-Template

JPX全上場銘柄（約3,700銘柄）の配当・財務データを取得し、**累進配当の候補を「ウォッチリスト（約100件）」に絞り込み、その中から「今月の買い時（10件）」を選ぶ**ツールです。
GitHub Actionsから**手動で**実行します。定期実行（schedule）は設定していません。

選んだ銘柄は、NotebookLMで分析するためのプロンプトまで自動で作ります。

> **ご注意**
> このツールの出力は、銘柄を探すための参考情報です。投資の勧誘・助言ではありません。
> 投資の判断は、ご自身の責任で行ってください。

---

## できること

| | 目的 | 判定に使うもの |
|---|---|---|
| ウォッチリスト（約100件） | 長く持ちたい銘柄の母集団を選ぶ | 配当の実績・財務・収益性、過去2年の平均利回り |
| 今月の買い時（10件） | 今月買う銘柄を選ぶ | 現在の利回りが、過去2年の水準と比べて割安か（買いシグナル） |

買いシグナルが「×」の銘柄も、ウォッチリストには残ります。今は買い時でないだけで、時期が来れば買い時に上がってきます。

## 全体の流れ

```
input/jpx_all.csv（銘柄コード一覧）
      │
      ▼
① データ取得        run-fetch.yml
      │  output/output_financial_dividend_merged.csv
      ▼
② スコアリングと絞り込み   run-scoring.yml
         output/weekly_scored_output.csv    … 全銘柄のスコア
         output/weekly_filtered_output.csv  … ウォッチリスト（MonthlyPick=Y が今月の買い時）
         output/weekly_prompt_output.txt    … NotebookLM用プロンプト
```

## 使い方

### 1. 自分のリポジトリを作る
このページ右上の **Use this template** → **Create a new repository** で、自分のアカウントにコピーします。

- 公開（Public）でも非公開（Private）でも動きます
- 非公開の場合、GitHub Actionsの無料枠（月2,000分）を使います。①は実行時間が長いため、残りの時間に注意してください

### 2. ① データ取得を実行する
1. リポジトリの **Actions** タブを開きます
2. 左の一覧から **① データ取得（手動実行）** を選びます
3. **Run workflow** → **Run workflow** を押します
4. 全銘柄を順番に取得するため、完了までに時間がかかります。完了したら、`output/output_financial_dividend_merged.csv` が作られます

### 3. ② スコアリングと絞り込みを実行する
1. **Actions** タブで **② スコアリングと絞り込み（手動実行）** を選びます
2. **Run workflow** を押します。必要なら、入力欄で条件を変えられます

| 入力欄 | 意味 | 初期値 |
|---|---|---|
| score_min | 総合スコアの下限（0〜100） | 75 |
| growth_min | 増配率枠の平均増配率の下限（0.25 = 25%） | 0.25 |
| watch_max | ウォッチリストの上限件数 | 100 |
| pick_n | 今月の買い時の件数 | 10 |

3. 完了後、`output/` に結果が保存されます。実行画面の下に、ウォッチリストと今月の買い時の件数が表示されます

### 4. 結果を見る
- `output/weekly_filtered_output.csv` を開き、`MonthlyPick` が `Y` の行が「今月の買い時」です
- `WatchType` は、ウォッチリストに入った理由です（高利回り枠／増配率枠）
- `BuySignal` は、現在の利回りの買い時の判定です（☆ ◎ 〇 △ ×）

### 5. NotebookLMで分析する（任意）
`output/weekly_prompt_output.txt` に、NotebookLM用のプロンプトと、参照するURLの一覧が入っています。内容に従って、NotebookLMにURLをソースとして登録し、プロンプトを貼り付けてください。

---

## 選び方のルール

### スコアリング（100点満点）
- 配当：40点（連続して配当を出している年数、増配・減配の傾向など）
- 財務：30点
- 収益性：30点

配当のスコアが38点以上の銘柄に「累進配当」のフラグが付きます。

### ウォッチリスト
次の条件をすべて満たす銘柄から選びます。
1. 累進配当のフラグがある
2. 次の問題がない：配当性向100%以上、営業CFの合計がマイナス、自己資本比率が低い、EPSが低い
3. 総合スコアが下限（初期値75点）以上

そのうえで、次のどちらかを満たす銘柄を、2年平均利回りの高い順に上限件数まで残します。
- 高利回り枠：2年平均利回りが、規模別の基準以上（超大型3.0%、大型3.5%、中型4.0%、小型4.5%、超小型5.0%）
- 増配率枠：2年平均利回りが3%以上で、平均増配率が25%以上

### 今月の買い時
ウォッチリストのうち、買いシグナルが☆◎〇の銘柄を、☆、◎、〇の順に並べます。同じ順位なら、割安度（現在利回り÷2年平均利回り）が高い順に、10件選びます。

---

## ファイル構成

```
.
├── .github/workflows/
│   ├── run-fetch.yml          # ① データ取得（手動）
│   └── run-scoring.yml        # ② スコアリングと絞り込み（手動）
├── input/
│   └── jpx_all.csv            # 銘柄コード一覧
├── output/                    # 結果の保存先
├── utils/sector_map.py        # セクター名の対応表
├── jpx_screening_fetch.py     # データ取得
├── jpx_scoring_engine.py      # スコアリング
├── jpx_yield_filter.py        # ウォッチリスト・今月の買い時の選定
├── makeprompt.py              # NotebookLM用プロンプト生成
└── requirements.txt
```

## 知っておいてほしいこと

- データの取得元は [yfinance](https://github.com/ranaroussi/yfinance)（Yahoo Financeのデータ）です。利用条件はYahoo側の規約に従います。ご自身の責任で、個人利用の範囲で使ってください
- データの取得に失敗した銘柄は、黙ってウォッチリストから外れることがあります
- 特別配当の補正はスコアリングだけで、利回りの計算には入っていません
- 取得元のデータの形式が変わると、動かなくなることがあります
- 結果が0件の場合は、プロンプトは作られません（`weekly_prompt_output.txt` は「No data.」になります）

## ローカルで動かす場合

```bash
pip install -r requirements.txt

# ① データ取得（動作確認だけなら、銘柄の少ないCSVで）
python jpx_screening_fetch.py input/jpx_all.csv

# ② スコアリング（カレントディレクトリの output_financial_dividend_merged.csv を読みます）
cp output/output_financial_dividend_merged.csv .   # ①の出力を移していない場合は不要
python jpx_scoring_engine.py

# ③ 絞り込み（引数：スコア下限 増配率下限 ウォッチリスト上限 買い時の件数）
python jpx_yield_filter.py 75 0.25 100 10

# ④ プロンプト生成
python makeprompt.py yield_filtered_output.csv
```

## 免責

- このツールの出力は参考情報であり、投資の勧誘・助言ではありません
- 出力の正確性・完全性は保証しません
- このツールを使って生じた損失について、作者は責任を負いません
