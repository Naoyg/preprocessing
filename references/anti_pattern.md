# [4 Pandas Anti-Patterns to Avoid and How to Fix Them](https://www.aidancooper.co.uk/pandas-anti-patterns/)
pandasのアンチパターンと良い書き方を、例を挙げて丁寧に解説した記事

## 1. 1つのdfへの変更は、1コマンドで  
1コマンドでやらないと、
- グローバル変数を作成することでメモリを消費する
- コードが煩雑で読みにくい
- バグが発生しやすい
- `SettingWithCopyWarning`を引き起こす
```
df = pd.read_csv("titles.csv")

# Mutation - DON'T DO THIS
df_bad = df.query("runtime > 30 & type == 'SHOW'")
df_bad["score"] = df_bad[["imdb_score", "tmdb_score"]].sum(axis=1)
df_bad = df_bad[["seasons", "score"]]
df_bad = df_bad.groupby("seasons").agg(["count", "mean"])
df_bad = df_bad.droplevel(axis=1, level=0)
df_bad = df_bad.query("count > 10")

# Chaining - DO THIS
df_good = (df
    .query("runtime > 30 & type == 'SHOW'")
    .assign(score=lambda df_: df_[["imdb_score", "tmdb_score"]].sum(axis=1))
    [["seasons", "score"]]
    .groupby("seasons")
    .agg(["count", "mean"])
    .droplevel(axis=1, level=0)
    .query("count > 10")
)

# returns:
#          count       mean
# seasons
# 1.0        835  13.064671
# 2.0        189  14.109524
# 3.0         83  14.618072
# 4.0         41  14.887805
# 5.0         38  15.242105
# 6.0         16  15.962500
```
### pandasの`.pipe`メソッドを使う
複雑な処理が必要な場合、pandasの`.pipe`メソッドを使うことで処理を抽象化することができる。

以下の例では、国コードのリストを含む文字列の列を、最初の3つの個別の国コード用の3つのカラムに変換している。
```
df = pd.read_csv("titles.csv")

def split_prod_countries(df_):
    # split `production_countries` column (containing lists of country
    # strings) into three individual columns of single country strings
    dfc = pd.DataFrame(df_["production_countries"].apply(eval).to_list())
    dfc = dfc.iloc[:, :3]
    dfc.columns = ["prod_country1", "prod_country2", "prod_country3"]
    return df_.drop("production_countries", axis=1).join(dfc)

df_pipe = df.pipe(split_prod_countries)

print(df["production_countries"].sample(5, random_state=14))
# returns:
# 3052    ['CA', 'JP', 'US']
# 1962                ['US']
# 2229                ['GB']
# 2151          ['KH', 'US']
# 3623                ['ES']

print(df_pipe.sample(5, random_state=14).iloc[:, -3:])
# returns:
#      prod_country1 prod_country2 prod_country3
# 3052            CA            JP            US
# 1962            US          None          None
# 2229            GB          None          None
# 2151            KH            US          None
# 3623            ES          None          None
```
## 2. `.apply`を活用し、forループは避ける
pandasでforループを回避すべき理由は
- 冗長で連鎖的な書き方と互換性がない
- 行を個別にループするのは速度が遅い

`prod_country1`列の国が出現回数でtop3/10/20に入っているかどうかを表す別の列を作成する。
```
df = pd.read_csv("titles.csv").pipe(split_production_countries)

# obtain country ranks
vcs = df["prod_country1"].value_counts()
top3 = vcs.index[:3]
top10 = vcs.index[:10]
top20 = vcs.index[:20]

# Looping - DON'T DO THIS
vals = []
for ind, row in df.iterrows():
    country = row["prod_country1"]
    if country in top3:
        vals.append("top3")
    elif country in top10:
        vals.append("top10")
    elif country in top20:
        vals.append("top20")
    else:
        vals.append("other")
df["prod_country_rank"] = vals

# df[col].apply() - DO THIS
def get_prod_country_rank(country):
    if country in top3:
        return "top3"
    elif country in top10:
        return "top10"
    elif country in top20:
        return "top20"
    else:
        return "other"

df["prod_country_rank"] = df["prod_country1"].apply(get_prod_country_rank)
print(df.sample(5, random_state=14).iloc[:, -4:])
# returns:
#      prod_country1 prod_country2 prod_country3 prod_country_rank
# 3052            CA            JP            US             top10
# 1962            US          None          None              top3
# 2229            GB          None          None              top3
# 2151            KH            US          None             other
# 3623            ES          None          None             top10
```

## 3. `.apply`を使いすぎず、`np.select`, `np.where`, `.isin`を使う
大容量のデータセットや計算量の多い場合、`.apply`よりも効率的なベクトル化ワークフローが速いことがある。
### `np.select`と`.isin`の組み合わせ
国ごとのランク列を追加する例
```
df = pd.read_csv("titles.csv").pipe(split_production_countries)

def get_prod_country_rank(df_):
    vcs = df_["prod_country1"].value_counts()
    return np.select(
        condlist=(
            df_["prod_country1"].isin(vcs.index[:3]),
            df_["prod_country1"].isin(vcs.index[:10]),
            df_["prod_country1"].isin(vcs.index[:20]),
        ),
        choicelist=("top3", "top10", "top20"),
        default="other"
    )

df = df.assign(prod_country_rank=lambda df_: get_prod_country_rank(df_))
```

### `np.where`
選択肢が2つで場合分けしたいときには`np.where`を使う。

例えば、IMDBのスコアにバグがあることを知っていて、2016年以降にリリースされた番組/映画のスコアについて1を引く必要があるとする。
```
df = df.assign(
    adjusted_score=lambda df_: np.where(
        df_["release_year"] > 2016, df_["imdb_score"] - 1, df_["imdb_score"]
    )
)
```

## 4. `.astype("category")`でデータ型を正確に
カラムのデータ型を正確にすることでパフォーマンスとメモリ使用量を改善することができる。  
特に、カテゴリ型を使わずに文字列を使っていることは大きな問題になる。
```
df = pd.read_csv("titles.csv")

df = df.assign(
    age_certification=lambda df_: df_["age_certification"].astype("category")
)
```