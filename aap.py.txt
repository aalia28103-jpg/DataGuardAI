import streamlit as st
import pandas as pd
import numpy as np
from io import BytesIO

# ---------------------------------------------------------
# PAGE CONFIGURATION
# ---------------------------------------------------------

st.set_page_config(
    page_title="DataGuard AI",
    page_icon="🤖",
    layout="wide"
)

# ---------------------------------------------------------
# TITLE
# ---------------------------------------------------------

st.title("🤖 DataGuard AI")
st.subheader("AI-Powered Data Quality & Cleaning Assistant")
st.caption("Developed by Aliya Banu A | MBA – Business Analytics")
st.write(
    "Upload a CSV or Excel dataset to detect data-quality problems, "
    "clean the data, analyze it and generate recommendations."
)

# ---------------------------------------------------------
# FILE UPLOAD
# ---------------------------------------------------------

uploaded_file = st.file_uploader(
    "📂 Upload your dataset",
    type=["csv", "xlsx"]
)

if uploaded_file:

    # -----------------------------------------------------
    # LOAD DATASET
    # -----------------------------------------------------

    try:

        if uploaded_file.name.lower().endswith(".csv"):
            df = pd.read_csv(uploaded_file)
        else:
            df = pd.read_excel(uploaded_file)

    except Exception as e:

        st.error(f"❌ Unable to read the file: {e}")
        st.stop()

    st.success("✅ Dataset uploaded successfully!")

    # -----------------------------------------------------
    # DATASET OVERVIEW
    # -----------------------------------------------------

    st.header("📊 Dataset Overview")

    col1, col2, col3, col4 = st.columns(4)

    with col1:
        st.metric("Rows", df.shape[0])

    with col2:
        st.metric("Columns", df.shape[1])

    with col3:
        st.metric(
            "Total Cells",
            df.shape[0] * df.shape[1]
        )

    with col4:
        st.metric(
            "Memory",
            f"{df.memory_usage(deep=True).sum() / 1024:.1f} KB"
        )

    # -----------------------------------------------------
    # DATA PREVIEW
    # -----------------------------------------------------

    st.header("👀 Data Preview")

    st.dataframe(
        df.head(100),
        use_container_width=True
    )

    # -----------------------------------------------------
    # MISSING VALUES
    # -----------------------------------------------------

    st.header("🔍 Missing Values Detection")

    missing_values = df.isnull().sum()
    total_missing = int(missing_values.sum())

    missing_table = pd.DataFrame({
        "Column": missing_values.index,
        "Missing Values": missing_values.values,
        "Missing %": (
            missing_values.values / len(df) * 100
        ).round(2)
    })

    missing_table = missing_table[
        missing_table["Missing Values"] > 0
    ]

    if missing_table.empty:

        st.success("✅ No missing values found!")

    else:

        st.warning(
            f"⚠️ {total_missing} missing values found."
        )

        st.dataframe(
            missing_table,
            use_container_width=True
        )

    # -----------------------------------------------------
    # DUPLICATES
    # -----------------------------------------------------

    st.header("🔄 Duplicate Rows Detection")

    duplicate_count = int(df.duplicated().sum())

    if duplicate_count == 0:

        st.success("✅ No duplicate rows found!")

    else:

        st.warning(
            f"⚠️ {duplicate_count} duplicate rows found."
        )

        duplicate_rows = df[
            df.duplicated(keep=False)
        ]

        st.dataframe(
            duplicate_rows.head(100),
            use_container_width=True
        )

    # -----------------------------------------------------
    # DATA TYPES
    # -----------------------------------------------------

    st.header("🔎 Data Type Detection")

    datatype_table = pd.DataFrame({
        "Column": df.columns,
        "Data Type": df.dtypes.astype(str).values,
        "Unique Values": [
            df[column].nunique(dropna=True)
            for column in df.columns
        ]
    })

    st.dataframe(
        datatype_table,
        use_container_width=True
    )

    # -----------------------------------------------------
    # NEGATIVE VALUES
    # -----------------------------------------------------

    st.header("➖ Negative Values Detection")

    numeric_columns = df.select_dtypes(
        include="number"
    ).columns

    negative_data = []
    total_negative = 0

    for column in numeric_columns:

        negative_count = int(
            (df[column] < 0).sum()
        )

        if negative_count > 0:

            total_negative += negative_count

            negative_data.append({
                "Column": column,
                "Negative Values": negative_count
            })

    if len(negative_data) == 0:

        st.success("✅ No negative values found!")

    else:

        negative_table = pd.DataFrame(
            negative_data
        )

        st.warning(
            f"⚠️ {total_negative} negative values found."
        )

        st.dataframe(
            negative_table,
            use_container_width=True
        )

    # -----------------------------------------------------
    # OUTLIER DETECTION
    # -----------------------------------------------------

    st.header("📈 Outlier Detection")

    outlier_data = []
    outlier_limits = {}
    total_outliers = 0

    for column in numeric_columns:

        valid_values = df[column].dropna()

        if len(valid_values) >= 4:

            q1 = valid_values.quantile(0.25)
            q3 = valid_values.quantile(0.75)

            iqr = q3 - q1

            lower_limit = q1 - (1.5 * iqr)
            upper_limit = q3 + (1.5 * iqr)

            outlier_limits[column] = (
                lower_limit,
                upper_limit
            )

            outlier_count = int(
                (
                    (df[column] < lower_limit) |
                    (df[column] > upper_limit)
                ).sum()
            )

            if outlier_count > 0:

                total_outliers += outlier_count

                outlier_data.append({
                    "Column": column,
                    "Outliers": outlier_count,
                    "Lower Limit": round(
                        lower_limit, 2
                    ),
                    "Upper Limit": round(
                        upper_limit, 2
                    )
                })

    if len(outlier_data) == 0:

        st.success(
            "✅ No significant outliers found!"
        )

    else:

        outlier_table = pd.DataFrame(
            outlier_data
        )

        st.warning(
            f"⚠️ {total_outliers} potential outliers found."
        )

        st.dataframe(
            outlier_table,
            use_container_width=True
        )

    # -----------------------------------------------------
    # CONSTANT COLUMNS
    # -----------------------------------------------------

    st.header("📌 Constant Columns Detection")

    constant_columns = []

    for column in df.columns:

        if df[column].nunique(
            dropna=False
        ) <= 1:

            constant_columns.append(column)

    if len(constant_columns) == 0:

        st.success(
            "✅ No constant columns found!"
        )

    else:

        st.warning(
            f"⚠️ {len(constant_columns)} constant column(s) found."
        )

        st.dataframe(
            pd.DataFrame({
                "Constant Columns":
                    constant_columns
            }),
            use_container_width=True
        )

    # -----------------------------------------------------
    # INCONSISTENT TEXT DETECTION
    # -----------------------------------------------------

    st.header("🔤 Inconsistent Text Detection")

    text_columns = df.select_dtypes(
        include=["object", "string", "category"]
    ).columns

    inconsistent_data = []

    for column in text_columns:

        values = (
            df[column]
            .dropna()
            .astype(str)
            .str.strip()
        )

        if len(values) > 0:

            unique_original = values.nunique()

            unique_lower = (
                values.str.lower().nunique()
            )

            if unique_original > unique_lower:

                inconsistent_data.append({
                    "Column": column,
                    "Original Unique Values":
                        unique_original,
                    "After Standardizing Case":
                        unique_lower,
                    "Potential Inconsistency":
                        unique_original -
                        unique_lower
                })

    if len(inconsistent_data) == 0:

        st.success(
            "✅ No obvious text inconsistencies found!"
        )

    else:

        st.warning(
            "⚠️ Possible inconsistent text values detected."
        )

        st.dataframe(
            pd.DataFrame(inconsistent_data),
            use_container_width=True
        )

    # -----------------------------------------------------
    # DATA QUALITY SCORE
    # -----------------------------------------------------

    st.header("⭐ Data Quality Score")

    total_cells = df.shape[0] * df.shape[1]

    if total_cells > 0 and df.shape[0] > 0:

        missing_rate = (
            total_missing / total_cells
        )

        duplicate_rate = (
            duplicate_count / df.shape[0]
        )

        missing_penalty = min(
            missing_rate * 40,
            40
        )

        duplicate_penalty = min(
            duplicate_rate * 20,
            20
        )

        negative_penalty = min(
            (total_negative / total_cells) * 20,
            20
        )

        outlier_penalty = min(
            (total_outliers / total_cells) * 15,
            15
        )

        constant_penalty = min(
            (len(constant_columns) /
             max(df.shape[1], 1)) * 5,
            5
        )

        score = 100 - (
            missing_penalty
            + duplicate_penalty
            + negative_penalty
            + outlier_penalty
            + constant_penalty
        )

        score = max(
            0,
            min(100, score)
        )

        st.metric(
            "Overall Data Quality Score",
            f"{score:.1f}/100"
        )

        if score >= 90:

            st.success(
                "🟢 Excellent Data Quality"
            )

        elif score >= 75:

            st.info(
                "🟡 Good Data Quality"
            )

        elif score >= 50:

            st.warning(
                "🟠 Moderate Data Quality"
            )

        else:

            st.error(
                "🔴 Poor Data Quality"
            )

    else:

        score = 0

    # -----------------------------------------------------
    # AI RECOMMENDATIONS
    # -----------------------------------------------------

    st.header("🤖 AI Recommendations")

    recommendations = []

    if total_missing > 0:

        recommendations.append(
            f"🔍 Missing Values: {total_missing} "
            "missing values were detected. "
            "Numeric values can usually be filled "
            "using the median, while categorical "
            "values can use the most frequent value."
        )

    else:

        recommendations.append(
            "✅ Missing Values: No missing values detected."
        )

    if duplicate_count > 0:

        recommendations.append(
            f"🔄 Duplicate Rows: {duplicate_count} "
            "duplicate rows were detected. "
            "Review them and remove them if they "
            "represent repeated records."
        )

    else:

        recommendations.append(
            "✅ Duplicate Rows: No duplicate rows detected."
        )

    if total_negative > 0:

        recommendations.append(
            f"➖ Negative Values: {total_negative} "
            "negative values were detected. "
            "Check whether negative values are "
            "valid for the relevant business fields."
        )

    else:

        recommendations.append(
            "✅ Negative Values: No negative values detected."
        )

    if total_outliers > 0:

        recommendations.append(
            f"📈 Outliers: {total_outliers} "
            "potential outliers were detected. "
            "Investigate them before deleting because "
            "they may represent genuine observations."
        )

    else:

        recommendations.append(
            "✅ Outliers: No significant outliers detected."
        )

    if len(constant_columns) > 0:

        recommendations.append(
            f"📌 Constant Columns: "
            f"{len(constant_columns)} constant "
            "column(s) were found. "
            "Consider removing them if they do not "
            "provide useful analytical information."
        )

    else:

        recommendations.append(
            "✅ Constant Columns: No constant columns detected."
        )

    if len(inconsistent_data) > 0:

        recommendations.append(
            "🔤 Text Consistency: Some text columns "
            "contain values with inconsistent capitalization "
            "or spacing. Standardization is recommended."
        )

    else:

        recommendations.append(
            "✅ Text Consistency: No obvious text inconsistencies found."
        )

    if score >= 90:

        recommendations.append(
            "⭐ Overall: Excellent data quality. "
            "The dataset is ready for analysis with "
            "minimal cleaning."
        )

    elif score >= 75:

        recommendations.append(
            "⭐ Overall: Good data quality. "
            "Address the detected issues before analysis."
        )

    elif score >= 50:

        recommendations.append(
            "⭐ Overall: Moderate data quality. "
            "Major data-quality issues should be addressed."
        )

    else:

        recommendations.append(
            "⭐ Overall: Poor data quality. "
            "Significant cleaning is recommended before analysis."
        )

    for recommendation in recommendations:

        st.write(recommendation)

    # -----------------------------------------------------
    # OUTLIER TREATMENT
    # -----------------------------------------------------

    st.header("🛠️ Outlier Treatment")

    st.write(
        "Choose how DataGuard AI should handle detected outliers."
    )

    outlier_option = st.selectbox(
        "Select Outlier Treatment",
        [
            "Keep Outliers",
            "Remove Outliers",
            "Cap Outliers"
        ]
    )

    # -----------------------------------------------------
    # TEXT CLEANING OPTION
    # -----------------------------------------------------

    text_clean_option = st.checkbox(
        "🔤 Standardize text values "
        "(remove extra spaces and standardize capitalization)"
    )

    # -----------------------------------------------------
    # CONSTANT COLUMN OPTION
    # -----------------------------------------------------

    remove_constant_option = st.checkbox(
        "📌 Remove constant columns"
    )

    # -----------------------------------------------------
    # EDA
    # -----------------------------------------------------

    st.header("📊 Exploratory Data Analysis")

    if len(numeric_columns) > 0:

        chart_column = st.selectbox(
            "Select a numeric column",
            list(numeric_columns)
        )

        chart_type = st.selectbox(
            "Select chart type",
            [
                "Histogram",
                "Box Plot",
                "Line Chart"
            ]
        )

        if chart_type == "Histogram":

            st.bar_chart(
                df[chart_column].value_counts(
                    bins=10
                ).sort_index()
            )

        elif chart_type == "Box Plot":

            st.line_chart(
                df[[chart_column]].reset_index(
                    drop=True
                )
            )

            st.write(
                "Box plots are represented using the "
                "selected numeric values for quick inspection."
            )

        else:

            st.line_chart(
                df[chart_column].reset_index(
                    drop=True
                )
            )

    else:

        st.info(
            "ℹ️ No numeric columns are available for charts."
        )

    # -----------------------------------------------------
    # DATASET Q&A
    # -----------------------------------------------------

    st.header("💬 Ask DataGuard AI")

    question = st.text_input(
        "Ask a question about your dataset",
        placeholder=(
            "Example: How many rows are there?"
        )
    )

    if question:

        q = question.lower()

        if "row" in q:

            st.info(
                f"📊 Your dataset contains "
                f"{df.shape[0]} rows."
            )

        elif "column" in q:

            st.info(
                f"📊 Your dataset contains "
                f"{df.shape[1]} columns."
            )

        elif "missing" in q:

            st.info(
                f"🔍 Your dataset contains "
                f"{total_missing} missing values."
            )

        elif "duplicate" in q:

            st.info(
                f"🔄 Your dataset contains "
                f"{duplicate_count} duplicate rows."
            )

        elif "outlier" in q:

            st.info(
                f"📈 Your dataset contains "
                f"{total_outliers} potential outliers."
            )

        elif "negative" in q:

            st.info(
                f"➖ Your dataset contains "
                f"{total_negative} negative values."
            )

        elif "quality" in q or "score" in q:

            st.info(
                f"⭐ Your current data quality score is "
                f"{score:.1f}/100."
            )

        elif "numeric" in q:

            st.info(
                f"🔢 There are "
                f"{len(numeric_columns)} numeric columns."
            )

        elif "text" in q:

            st.info(
                f"🔤 There are "
                f"{len(text_columns)} text columns."
            )

        elif "constant" in q:

            st.info(
                f"📌 There are "
                f"{len(constant_columns)} constant columns."
            )

        else:

            st.info(
                "🤖 I can currently answer questions about "
                "rows, columns, missing values, duplicates, "
                "outliers, negative values, quality score, "
                "numeric columns, text columns and constant columns."
            )

    # -----------------------------------------------------
    # AUTOMATIC CLEANING
    # -----------------------------------------------------

    st.header("🧹 Data Cleaning")

    st.write(
        "Apply the selected cleaning options to create a cleaned dataset."
    )

    if st.button("🧹 Clean Dataset"):

        cleaned_df = df.copy()

        original_rows = cleaned_df.shape[0]
        original_columns = cleaned_df.shape[1]

        # -----------------------------------------------
        # REMOVE DUPLICATES
        # -----------------------------------------------

        cleaned_df = cleaned_df.drop_duplicates()

        duplicates_removed = (
            original_rows -
            cleaned_df.shape[0]
        )

        # -----------------------------------------------
        # FILL MISSING VALUES
        # -----------------------------------------------

        numeric_cols = cleaned_df.select_dtypes(
            include="number"
        ).columns

        text_cols = cleaned_df.select_dtypes(
            exclude="number"
        ).columns

        numeric_filled = 0
        text_filled = 0

        for column in numeric_cols:

            missing_before = int(
                cleaned_df[column].isnull().sum()
            )

            if missing_before > 0:

                median_value = (
                    cleaned_df[column].median()
                )

                if pd.notna(median_value):

                    cleaned_df[column] = (
                        cleaned_df[column]
                        .fillna(median_value)
                    )

                    numeric_filled += missing_before

        for column in text_cols:

            missing_before = int(
                cleaned_df[column].isnull().sum()
            )

            if missing_before > 0:

                mode_values = (
                    cleaned_df[column].mode()
                )

                if not mode_values.empty:

                    cleaned_df[column] = (
                        cleaned_df[column]
                        .fillna(
                            mode_values.iloc[0]
                        )
                    )

                    text_filled += missing_before

        # -----------------------------------------------
        # TEXT STANDARDIZATION
        # -----------------------------------------------

        text_values_changed = 0

        if text_clean_option:

            for column in cleaned_df.select_dtypes(
                include=["object", "string"]
            ).columns:

                before = (
                    cleaned_df[column]
                    .astype(str)
                    .copy()
                )

                after = (
                    cleaned_df[column]
                    .astype(str)
                    .str.strip()
                    .str.lower()
                )

                text_values_changed += int(
                    (before != after).sum()
                )

                cleaned_df[column] = after

        # -----------------------------------------------
        # CONSTANT COLUMN REMOVAL
        # -----------------------------------------------

        constant_removed = 0

        if remove_constant_option:

            columns_to_remove = []

            for column in cleaned_df.columns:

                if cleaned_df[column].nunique(
                    dropna=False
                ) <= 1:

                    columns_to_remove.append(column)

            if columns_to_remove:

                cleaned_df = cleaned_df.drop(
                    columns=columns_to_remove
                )

                constant_removed = len(
                    columns_to_remove
                )

        # -----------------------------------------------
        # OUTLIER TREATMENT
        # -----------------------------------------------

        outliers_removed = 0
        outliers_capped = 0

        if outlier_option != "Keep Outliers":

            for column in cleaned_df.select_dtypes(
                include="number"
            ).columns:

                valid_values = (
                    cleaned_df[column].dropna()
                )

                if len(valid_values) >= 4:

                    q1 = valid_values.quantile(0.25)
                    q3 = valid_values.quantile(0.75)

                    iqr = q3 - q1

                    lower_limit = (
                        q1 - 1.5 * iqr
                    )

                    upper_limit = (
                        q3 + 1.5 * iqr
                    )

                    if outlier_option == "Remove Outliers":

                        mask = (
                            (cleaned_df[column] < lower_limit) |
                            (cleaned_df[column] > upper_limit)
                        )

                        outliers_removed += int(
                            mask.sum()
                        )

                        cleaned_df = cleaned_df[
                            ~mask
                        ]

                    elif outlier_option == "Cap Outliers":

                        lower_mask = (
                            cleaned_df[column]
                            < lower_limit
                        )

                        upper_mask = (
                            cleaned_df[column]
                            > upper_limit
                        )

                        outliers_capped += int(
                            lower_mask.sum()
                            + upper_mask.sum()
                        )

                        cleaned_df[column] = (
                            cleaned_df[column]
                            .clip(
                                lower=lower_limit,
                                upper=upper_limit
                            )
                        )

        # -----------------------------------------------
        # CLEANING RESULTS
        # -----------------------------------------------

        st.success(
            "✅ Dataset cleaned successfully!"
        )

        result1, result2, result3 = st.columns(3)

        with result1:

            st.metric(
                "Original Rows",
                original_rows
            )

        with result2:

            st.metric(
                "Cleaned Rows",
                cleaned_df.shape[0]
            )

        with result3:

            st.metric(
                "Cleaned Columns",
                cleaned_df.shape[1]
            )

        st.write(
            f"🔄 Duplicate rows removed: "
            f"{duplicates_removed}"
        )

        st.write(
            f"🔢 Missing numeric values filled: "
            f"{numeric_filled}"
        )

        st.write(
            f"🔤 Missing text values filled: "
            f"{text_filled}"
        )

        st.write(
            f"✏️ Text values standardized: "
            f"{text_values_changed}"
        )

        st.write(
            f"📌 Constant columns removed: "
            f"{constant_removed}"
        )

        st.write(
            f"📈 Outliers removed: "
            f"{outliers_removed}"
        )

        st.write(
            f"📊 Outliers capped: "
            f"{outliers_capped}"
        )

        # -----------------------------------------------
        # CLEANED DATA PREVIEW
        # -----------------------------------------------

        st.header("✨ Cleaned Dataset Preview")

        st.dataframe(
            cleaned_df.head(100),
            use_container_width=True
        )

        # -----------------------------------------------
        # DOWNLOAD CLEANED EXCEL
        # -----------------------------------------------

        output = BytesIO()

        with pd.ExcelWriter(
            output,
            engine="openpyxl"
        ) as writer:

            cleaned_df.to_excel(
                writer,
                index=False,
                sheet_name="Cleaned Data"
            )

        st.download_button(
            label="📥 Download Cleaned Dataset",
            data=output.getvalue(),
            file_name="DataGuard_AI_Cleaned.xlsx",
            mime=(
                "application/vnd.openxmlformats-officedocument."
                "spreadsheetml.sheet"
            )
        )

        # -----------------------------------------------
        # DATA QUALITY REPORT
        # -----------------------------------------------

        st.header("📄 Data Quality Report")

        report = f"""
DATAGUARD AI - DATA QUALITY REPORT
===================================

Dataset:
{uploaded_file.name}

DATASET OVERVIEW
----------------
Rows: {df.shape[0]}
Columns: {df.shape[1]}
Total Cells: {total_cells}

DATA QUALITY
------------
Quality Score: {score:.1f}/100

Missing Values: {total_missing}
Duplicate Rows: {duplicate_count}
Negative Values: {total_negative}
Potential Outliers: {total_outliers}
Constant Columns: {len(constant_columns)}
Text Inconsistency Columns: {len(inconsistent_data)}

CLEANING RESULTS
----------------
Duplicate Rows Removed: {duplicates_removed}
Missing Numeric Values Filled: {numeric_filled}
Missing Text Values Filled: {text_filled}
Text Values Standardized: {text_values_changed}
Constant Columns Removed: {constant_removed}
Outliers Removed: {outliers_removed}
Outliers Capped: {outliers_capped}

RECOMMENDATIONS
---------------
"""

        for recommendation in recommendations:

            report += (
                "- "
                + recommendation
                + "\n"
            )

        st.text_area(
            "Report Preview",
            report,
            height=350
        )

        st.download_button(
            label="📄 Download Data Quality Report",
            data=report,
            file_name="DataGuard_AI_Report.txt",
            mime="text/plain"
        )

else:

    st.info(
        "👆 Please upload a CSV or Excel file to begin."
    )