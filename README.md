# DAX patterns

Reusable DAX patterns I use on Power BI projects. Each one is small, readable and written to be copied into a model and adapted.

All examples assume a star schema with a fact table `Sales` (column `Amount`) and a date table `'Date'` that is marked as a date table. Rename the tables and columns to match your model. The sample names are illustrative, not from any real client or employer.

## Contents

1. [Safe division](#1-safe-division)
2. [Time intelligence: YTD, prior year, year on year](#2-time-intelligence)
3. [Running total](#3-running-total)
4. [Reconciliation checks between two sources](#4-reconciliation-checks)
5. [Ranking and Top N](#5-ranking-and-top-n)
6. [Row-level security by user](#6-row-level-security)
7. [Dynamic titles](#7-dynamic-titles)

## 1. Safe division

Use `DIVIDE` instead of the `/` operator. It returns blank (or an alternate result) when the denominator is zero or blank, so visuals do not show errors.

```dax
Margin % = DIVIDE ( [Margin], [Revenue] )
```

## 2. Time intelligence

Base measure first, then build on it.

```dax
Revenue = SUM ( Sales[Amount] )

Revenue YTD = TOTALYTD ( [Revenue], 'Date'[Date] )

Revenue PY = CALCULATE ( [Revenue], SAMEPERIODLASTYEAR ( 'Date'[Date] ) )

Revenue YoY % = DIVIDE ( [Revenue] - [Revenue PY], [Revenue PY] )
```

Notes: these need a continuous date table. If the fiscal year does not end in December, pass the year-end date as the third argument of `TOTALYTD`, for example `"31/3"`.

## 3. Running total

```dax
Running Total =
CALCULATE (
    [Revenue],
    FILTER ( ALL ( 'Date'[Date] ), 'Date'[Date] <= MAX ( 'Date'[Date] ) )
)
```

## 4. Reconciliation checks

Compare two sources of the same figures, for example a ledger extract and a collections report that share a receipt ID. The point is that a model should tell you when it does not agree with the source.

```dax
Ledger Amount = SUM ( Ledger[Amount] )

Collections Amount = SUM ( Collections[Amount] )

Variance = [Ledger Amount] - [Collections Amount]

Reconciliation Status =
IF ( ROUND ( [Variance], 2 ) = 0, "Reconciled", "Exception" )

Unmatched Receipts =
COUNTROWS (
    EXCEPT (
        VALUES ( Ledger[ReceiptID] ),
        VALUES ( Collections[ReceiptID] )
    )
)
```

`Unmatched Receipts` counts receipt IDs that exist in the ledger but not in collections. Put the receipt ID in a table visual and filter it with this measure to list the exceptions. Swap the two arguments of `EXCEPT` to find the reverse case.

## 5. Ranking and Top N

```dax
Product Rank =
RANKX ( ALLSELECTED ( Product[Name] ), [Revenue], , DESC, Dense )

Revenue Top 5 =
IF ( [Product Rank] <= 5, [Revenue] )
```

Use `Revenue Top 5` in a visual that lists products. Rows ranked below 5 return blank and drop out of the table.

## 6. Row-level security

Create a table `UserAccess` with one row per user and region, then add a role on the `Region` table with this filter:

```dax
Region[RegionName]
    IN SELECTCOLUMNS (
        FILTER ( UserAccess, UserAccess[Email] = USERPRINCIPALNAME () ),
        "Region", UserAccess[RegionName]
    )
```

Test every role with View as in Power BI Desktop, and again in the service, before you publish.

## 7. Dynamic titles

```dax
Report Title =
"Revenue for " & SELECTEDVALUE ( 'Date'[Year], "all years" )
```

Bind it to a visual title with the fx button, using the measure as the field value.

## Licence and use

Free to use and adapt. No warranty: test every measure on your own data before relying on it.

---

Nadeera Madushanka. Power BI, Microsoft Fabric and web apps. [nadeera.officesolutionscube.com](https://nadeera.officesolutionscube.com)
