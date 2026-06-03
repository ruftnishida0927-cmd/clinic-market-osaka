
import os
from pathlib import Path

import pandas as pd
import streamlit as st

st.set_page_config(
    page_title="clinic_market_tool 大阪DB",
    page_icon="🏥",
    layout="wide",
)

DEFAULT_BASE = Path("/content/gdrive/MyDrive/clinic_market_tool")

OSAKA_READY_PATH = os.environ.get(
    "OSAKA_MARKET_READY_CSV",
    str(DEFAULT_BASE / "features" / "osaka" / "clinic_market_analysis_ready_osaka.csv")
)

LOCAL_FALLBACK = Path(__file__).resolve().parent / "data" / "clinic_market_analysis_ready_osaka.csv"

@st.cache_data(show_spinner=True)
def load_data(path: str):
    p = Path(path)

    if p.exists():
        return pd.read_csv(p)

    if LOCAL_FALLBACK.exists():
        return pd.read_csv(LOCAL_FALLBACK)

    raise FileNotFoundError(
        f"CSVが見つかりません: {path}\n"
        f"fallbackも見つかりません: {LOCAL_FALLBACK}"
    )

def safe_unique(df, col):
    if col not in df.columns:
        return []
    vals = df[col].dropna().astype(str).unique().tolist()
    return sorted([v for v in vals if v.strip()])

def as_num(s):
    return pd.to_numeric(s, errors="coerce")

def pick_cols(df, cols):
    return [c for c in cols if c in df.columns]

st.title("🏥 clinic_market_tool 大阪DB")
st.caption("大阪府 医療機関マスター + 近隣競合特徴量 + 分析優先度")

try:
    df = load_data(OSAKA_READY_PATH)
except Exception as e:
    st.error("データ読み込みに失敗しました")
    st.code(str(e))
    st.stop()

for col in [
    "competitor_500m_count",
    "same_dept_500m_count",
    "competitor_1km_count",
    "same_dept_1km_count",
    "competitor_2km_count",
    "same_dept_2km_count",
    "competition_score",
    "distance_reliability_score",
    "nearest_competitor_distance_km_excluding_same_coord",
]:
    if col in df.columns:
        df[col] = as_num(df[col])

st.sidebar.header("🔎 絞り込み")

city_options = safe_unique(df, "city")
dept_options = safe_unique(df, "department_group")
priority_options = safe_unique(df, "analysis_priority")
status_options = safe_unique(df, "market_analysis_ready_status")
rank_options = safe_unique(df, "competition_density_rank_v2")

selected_city = st.sidebar.multiselect("市区町村", city_options, default=[])
selected_dept = st.sidebar.multiselect("診療科グループ", dept_options, default=[])
selected_priority = st.sidebar.multiselect("分析優先度", priority_options, default=["A_優先分析"] if "A_優先分析" in priority_options else [])
selected_status = st.sidebar.multiselect("分析ステータス", status_options, default=[])
selected_rank = st.sidebar.multiselect("競合密度ランク", rank_options, default=[])

keyword = st.sidebar.text_input("クリニック名・住所キーワード", "")

show_core_only = st.sidebar.checkbox("主分析対象のみ", value=True)
show_geo_caution = st.sidebar.checkbox("位置精度注意も含める", value=True)

filtered = df.copy()

if selected_city:
    filtered = filtered[filtered["city"].astype(str).isin(selected_city)]

if selected_dept:
    filtered = filtered[filtered["department_group"].astype(str).isin(selected_dept)]

if selected_priority:
    filtered = filtered[filtered["analysis_priority"].astype(str).isin(selected_priority)]

if selected_status:
    filtered = filtered[filtered["market_analysis_ready_status"].astype(str).isin(selected_status)]

if selected_rank:
    filtered = filtered[filtered["competition_density_rank_v2"].astype(str).isin(selected_rank)]

if keyword.strip():
    kw = keyword.strip()
    mask = pd.Series(False, index=filtered.index)
    for col in ["clinic_name", "address", "city", "department_group"]:
        if col in filtered.columns:
            mask = mask | filtered[col].fillna("").astype(str).str.contains(kw, case=False, na=False)
    filtered = filtered[mask]

if show_core_only and "is_core_market_analysis_target" in filtered.columns:
    filtered = filtered[filtered["is_core_market_analysis_target"].astype(str).isin(["True", "true", "1"])]

if not show_geo_caution and "market_analysis_ready_status" in filtered.columns:
    filtered = filtered[filtered["market_analysis_ready_status"].astype(str) != "ready_with_geo_caution"]

st.subheader("📊 サマリー")

c1, c2, c3, c4, c5 = st.columns(5)

c1.metric("総件数", f"{len(df):,}")
c2.metric("表示件数", f"{len(filtered):,}")

if "analysis_priority" in df.columns:
    c3.metric("A_優先分析", f"{(df['analysis_priority'] == 'A_優先分析').sum():,}")
else:
    c3.metric("A_優先分析", "-")

if "is_core_market_analysis_target" in df.columns:
    c4.metric("主分析対象", f"{df['is_core_market_analysis_target'].astype(str).isin(['True','true','1']).sum():,}")
else:
    c4.metric("主分析対象", "-")

if "market_analysis_ready_status" in df.columns:
    c5.metric("低精度位置", f"{(df['market_analysis_ready_status'] == 'reference_low_geo_precision').sum():,}")
else:
    c5.metric("低精度位置", "-")

st.subheader("🔥 分析優先候補")

if "competition_score" in filtered.columns:
    filtered_view = filtered.sort_values(
        ["analysis_priority", "competition_score", "competitor_1km_count"],
        ascending=[True, False, False],
        na_position="last",
    )
else:
    filtered_view = filtered.copy()

display_cols = pick_cols(filtered_view, [
    "clinic_id",
    "clinic_name",
    "city",
    "address",
    "department_group",
    "analysis_priority",
    "market_analysis_ready_status",
    "competition_density_rank_v2",
    "competitor_1km_count",
    "same_dept_1km_count",
    "nearest_competitor_name_excluding_same_coord",
    "nearest_competitor_distance_km_excluding_same_coord",
    "distance_reliability",
    "distance_warning_flag",
    "action_hint",
])

st.dataframe(
    filtered_view[display_cols].head(500),
    use_container_width=True,
    height=520,
)

st.subheader("🏥 個別クリニック確認")

if len(filtered_view) > 0 and "clinic_name" in filtered_view.columns:
    name_options = (
        filtered_view["clinic_name"].fillna("").astype(str)
        + "｜"
        + filtered_view.get("city", pd.Series([""] * len(filtered_view))).fillna("").astype(str)
        + "｜"
        + filtered_view.get("clinic_id", pd.Series([""] * len(filtered_view))).fillna("").astype(str)
    ).tolist()

    selected = st.selectbox("確認するクリニック", name_options)

    if selected:
        cid = selected.split("｜")[-1]
        row_df = filtered_view[filtered_view["clinic_id"].astype(str) == cid]

        if len(row_df) > 0:
            row = row_df.iloc[0]

            left, right = st.columns([1, 1])

            with left:
                st.markdown("### 基本情報")
                st.write("**医療機関名**:", row.get("clinic_name", ""))
                st.write("**住所**:", row.get("address", ""))
                st.write("**診療科グループ**:", row.get("department_group", ""))
                st.write("**分析優先度**:", row.get("analysis_priority", ""))
                st.write("**分析ステータス**:", row.get("market_analysis_ready_status", ""))
                st.write("**距離信頼度**:", row.get("distance_reliability", ""))
                st.write("**注意フラグ**:", row.get("distance_warning_flag", ""))

            with right:
                st.markdown("### 競合情報")
                st.write("**競合密度ランク**:", row.get("competition_density_rank_v2", ""))
                st.write("**1km競合数**:", row.get("competitor_1km_count", ""))
                st.write("**1km同系統競合数**:", row.get("same_dept_1km_count", ""))
                st.write("**2km競合数**:", row.get("competitor_2km_count", ""))
                st.write("**最寄り競合**:", row.get("nearest_competitor_name_excluding_same_coord", ""))
                st.write("**最寄り競合距離km**:", row.get("nearest_competitor_distance_km_excluding_same_coord", ""))

            st.markdown("### コメント")
            st.info(row.get("action_hint", ""))

            if "competition_comment_v2" in row.index:
                st.write(row.get("competition_comment_v2", ""))

st.subheader("📍 集計")

tab1, tab2, tab3 = st.tabs(["市区町村別", "診療科別", "ステータス別"])

with tab1:
    if "city" in filtered.columns:
        city_summary = (
            filtered.groupby("city", dropna=False)
            .agg(
                件数=("clinic_id", "count"),
                平均1km競合=("competitor_1km_count", "mean"),
                平均同系統1km=("same_dept_1km_count", "mean"),
            )
            .reset_index()
            .sort_values("件数", ascending=False)
        )
        st.dataframe(city_summary, use_container_width=True, height=400)

with tab2:
    if "department_group" in filtered.columns:
        dept_summary = (
            filtered.groupby("department_group", dropna=False)
            .agg(
                件数=("clinic_id", "count"),
                平均1km競合=("competitor_1km_count", "mean"),
                平均同系統1km=("same_dept_1km_count", "mean"),
            )
            .reset_index()
            .sort_values("件数", ascending=False)
        )
        st.dataframe(dept_summary, use_container_width=True, height=400)

with tab3:
    for col in ["market_analysis_ready_status", "analysis_priority", "competition_density_rank_v2", "distance_reliability"]:
        if col in filtered.columns:
            st.markdown(f"#### {col}")
            st.dataframe(
                filtered[col].value_counts(dropna=False).reset_index(),
                use_container_width=True,
                height=220,
            )

st.caption("注意：緯度経度は町丁目重心を含むため、個別提案時は必要に応じて精密ジオコードで再確認してください。")
