# Data Cleaning in Databricks SQL – Nashville Housing

This project cleans the Nashville Housing dataset using SQL in a Databricks notebook (Spark SQL on a Delta table). It is adapted from Alex The Analyst's tutorial [Data Analyst Portfolio Project | Data Cleaning in SQL](https://www.youtube.com/watch?v=8rO7ztF4NtU), which was written for SQL Server. I rewrote each step for Databricks myself and worked out the syntax differences along the way. Those differences are listed below.

## Dataset

- **Nashville Housing** property sales data, the dataset used in the tutorial
- Stored as the table `projects.portfolioprojects.nashvillehousing` (Unity Catalog `catalog.schema.table`)
- Main columns: `UniqueID`, `ParcelID`, `LandUse`, `PropertyAddress`, `SaleDate`, `SalePrice`, `LegalReference`, `SoldAsVacant`, `OwnerName`, `OwnerAddress`, `Acreage`, `TaxDistrict`, `LandValue`, `BuildingValue`, `TotalValue`, `YearBuilt`, `Bedrooms`, `FullBath`, `HalfBath`

## Cleaning steps

1. **Standardised the sale date.** Added a `SaleDateConverted` column of type `date` and filled it with `CAST(SaleDate AS DATE)`, which removes the timestamp part.
2. **Filled missing property addresses.** Rows that share a `ParcelID` have the same property address. A self-join on `ParcelID` (excluding the row itself) finds an address for each row where it is null. A `MERGE INTO` statement then writes it back. The run updated 29 rows.
3. **Split the property address** into `PropertySplitAddress` and `PropertySplitCity` with `substring` / `charindex` / `len`.
4. **Split the owner address** into `OwnerSplitAddress`, `OwnerSplitCity` and `OwnerSplitState` with `split(OwnerAddress, ',')[n]`.
5. **Standardised `SoldAsVacant`.** Changed `Y` / `N` to `Yes` / `No` with a `CASE` expression. After the update the column contains only `Yes` (4,675) and `No` (51,802).
6. **Removed duplicates.** Used `ROW_NUMBER()` over `ParcelID, PropertyAddress, SalePrice, SaleDate, LegalReference` to flag repeated rows (`rownum > 1`), then deleted them with `MERGE INTO ... WHEN MATCHED THEN DELETE`.
7. **Dropped unused columns:** `OwnerAddress`, `PropertyAddress` and `TaxDistrict`, which are replaced by the split columns.

## SQL Server vs Databricks SQL

These are the changes I made when converting the tutorial's T-SQL to Databricks:

| Task | SQL Server (tutorial) | Databricks SQL (this notebook) |
|---|---|---|
| Table reference | `PortfolioProject.dbo.NashvilleHousing` | `projects.portfolioprojects.nashvillehousing` |
| Column name with a trailing space | `[UniqueID ]` | `` `UniqueID ` `` (backticks) |
| Convert to date | `CONVERT(Date, SaleDate)` | `CAST(SaleDate AS DATE)` |
| Add a column | `ALTER TABLE ... Add SaleDateConverted Date;` | `ALTER TABLE ... ADD COLUMN SaleDateConverted date;` |
| Replace nulls | `ISNULL(a.PropertyAddress, b.PropertyAddress)` | `coalesce(a.PropertyAddress, b.PropertyAddress)` |
| Update using a join | `UPDATE a SET ... FROM table a JOIN table b ...` | `MERGE INTO ... USING (subquery) ... WHEN MATCHED THEN UPDATE` |
| Split a comma-separated address | `PARSENAME(REPLACE(OwnerAddress, ',', '.'), 3)` | `split(OwnerAddress, ',')[0]` (zero-based array index) |
| Delete duplicates | `DELETE` from a CTE with `ROW_NUMBER()` | `MERGE INTO ... WHEN MATCHED THEN DELETE` on the duplicate IDs |

`SUBSTRING`, `CHARINDEX`, `LEN`, `CASE`, `ROW_NUMBER()` and CTEs ran in Databricks without changes.

## Tools

- Databricks notebook with `%sql` cells
- Spark SQL on a Delta table in Unity Catalog

## How to run

1. Import `sql_data_cleaning.ipynb` into a Databricks workspace (Workspace, then Import).
2. Load the Nashville Housing data as a table. Either create `projects.portfolioprojects.nashvillehousing` or change the table name in the cells.
3. Attach the notebook to a cluster or SQL warehouse and run the cells in order. The `ALTER`, `UPDATE`, `MERGE` and `DROP COLUMN` cells change the table, so run them on a copy.

Large result tables in the notebook were trimmed to their first 10 rows to keep the file small.

## Possible next steps

- `trim()` the split city and state columns, because the split values keep a leading space (for example `" GOODLETTSVILLE"`).
