# data-integrity-scenario
import os
import re
import pandas as pd


def clean_supplier_data(file_path: str, output_path: str):
    """Loads, validates, and cleans a 5000-row supplier CSV file for system import."""
    # 1. LOAD FILE WITH SAFETY CHECKS
    if not os.path.exists(file_path):
        raise FileNotFoundError(f"Input file not found at: {file_path}")

    # Use UTF-8 encoding to prevent character corruption
    df = pd.read_csv(file_path, encoding="utf-8")
    print(f"Initial Load: {len(df)} rows found.")

    # 2. STRIP WHITESPACE & STANDARDIZE COLUMN NAMES
    # Remove leading/trailing spaces from column headers and lower/snake_case them
    df.columns = (
        df.columns.str.strip().str.lower().str.replace(" ", "_").str.replace(r"[^\w]", "", regex=True)
    )

    # Required columns mapping (adjust these strings to match your supplier's expected headers)
    required_cols = ["sku", "product_name", "price", "quantity"]
    missing_cols = [col for col in required_cols if col not in df.columns]
    if missing_cols:
        raise ValueError(f"Missing critical columns in CSV: {missing_cols}")

    # Strip whitespace from all text columns to clean up formatting
    text_cols = df.select_dtypes(include=["object"]).columns
    for col in text_cols:
        df[col] = df[col].astype(str).str.strip()

    # 3. DATA INTEGRITY CHECKS
    # Drop rows where critical identifiers are missing
    initial_count = len(df)
    df = df.dropna(subset=["sku", "product_name"])
    print(f"Dropped {initial_count - len(df)} rows due to missing SKU or Product Name.")

    # Remove duplicate SKUs (keep the first occurrence)
    pre_dup_count = len(df)
    df = df.drop_duplicates(subset=["sku"], keep="first")
    print(f"Removed {pre_dup_count - len(df)} duplicate SKU rows.")

    # 4. VALUE CLEANING & STANDARDIZATION
    # Clean pricing: Remove currency symbols, commas, and force float data type
    df["price"] = df["price"].astype(str).str.replace(r"[^\d.]", "", regex=True)
    df["price"] = pd.to_numeric(df["price"], errors="coerce")

    # Clean quantities: Force integer values, replace missing/invalid data with 0
    df["quantity"] = pd.to_numeric(df["quantity"], errors="coerce").fillna(0).astype(int)

    # 5. LOGICAL LOGIC VALIDATION
    # Flag or remove negative values
    invalid_prices = df[df["price"] < 0]
    if not invalid_prices.empty:
        print(f"Warning: Found {len(invalid_prices)} items with negative prices. Setting to 0.00.")
        df.loc[df["price"] < 0, "price"] = 0.00

    invalid_qty = df[df["quantity"] < 0]
    if not invalid_qty.empty:
        print(f"⚠️ Warning: Found {len(invalid_qty)} items with negative stock. Setting to 0.")
        df.loc[df["quantity"] < 0, "quantity"] = 0

    # 6. SYSTEM COMPATIBILITY (Example: Character Limits)
    # Truncate product names to 150 characters to match database limits
    df["product_name"] = df["product_name"].str.slice(0, 150)

    # Strip dangerous HTML elements from descriptions if a description field exists
    if "description" in df.columns:
        df["description"] = df["description"].apply(lambda x: re.sub(r"<[^>]*>", "", str(x)))

    # 7. EXPORT CLEANED FILE
    df.to_csv(output_path, index=False, encoding="utf-8")
    print(f"\nSuccess! Cleaned file saved with {len(df)} rows to: {output_path}")


# --- RUN SCRIPT ---
if __name__ == "__main__":
    # Update these file paths with your actual filenames
    INPUT_CSV = "supplier_export.csv"
    OUTPUT_CSV = "cleaned_import_ready.csv"

    try:
        clean_supplier_data(INPUT_CSV, OUTPUT_CSV)
    except Exception as e:
        print(f"Script Failed: {e}")
