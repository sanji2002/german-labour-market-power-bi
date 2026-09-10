# German Labour Market & Hiring Pressure Analytics

**Power BI | Power Query | DAX | Eurostat | 2015–2025**

A Power BI decision-support dashboard examining how hiring pressure in Germany has changed since 2015, where recruitment pressure remains concentrated, and how Germany compares with European labour markets for which comparable data are available.

![Executive Overview](executive_overview.png)

## Key Findings

- Germany's Job Vacancy Rate fell from a **4.5% peak in 2022-Q4 to 2.8% in 2025-Q4**, a decline of 1.7 percentage points.
- **Administrative & Support Service Activities** recorded the highest industry JVR at **5.9%**, followed by **Construction at 4.9%**.
- Vacancy intensity and vacancy volume tell different stories: **Health & Social Work had approximately 202,000 vacancies** despite a JVR of 3.1%, while Manufacturing had approximately 134,000 vacancies with a JVR of only 1.8%.
- Hiring pressure has not declined uniformly. Accommodation & Food Services recorded the largest fall between 2022-Q4 and 2025-Q4 at **-3.8 percentage points**, while Administrative & Support fell by 3.2 points but still had the highest JVR in 2025-Q4.
- Within the available comparable country sample, Germany's **2.8% JVR** stood below the Netherlands (**3.9%**) but above Poland (**0.7%**) in 2025-Q4.

## Business Question

**How has hiring pressure across the German labour market changed since 2015, where is recruitment pressure concentrated today, and how does Germany compare with European labour markets for which comparable data are available?**

## Objective

The objective is to build a decision-support view of German hiring pressure using quarterly Eurostat data, separating three dimensions that headline vacancy rates can obscure: **vacancy intensity, the absolute scale of vacancies, and changes in pressure over time**.

The analysis identifies sectors where recruitment pressure remains elevated, distinguishes high-intensity sectors from high-volume hiring markets, and places Germany's current position in a comparable European context.

## Data & Scope

The analysis uses **Eurostat Job Vacancy Statistics (`jvs_q_nace2`)**, covering quarterly labour-market observations from **2015-Q1 to 2025-Q4**. Non-seasonally adjusted data were used to maintain a consistent basis across the analysis.

The dataset provides three principal measures:

- **Job Vacancy Rate (JVR):** vacancy intensity relative to occupied and vacant posts
- **Job Vacancies:** the absolute number of vacant positions
- **Occupied Jobs:** the number of occupied positions

Industry analysis follows **NACE Rev. 2**, using the **A-S overall-economy aggregate** for headline measures and individual NACE sections for sector analysis.

The original extract covered **Germany, Austria, France, the Netherlands, Poland, Spain and EU-27**. However, comparable A-S observations were not available for every selected geography in the extracted data. Cross-country comparisons therefore include only geographies with an available A-S aggregate rather than estimating national vacancy rates from industry-level observations.

## Data Preparation & Quality Assurance

The raw Eurostat SDMX-CSV extract required transformation before analysis. Using **Power Query**, the data were cleaned and reshaped into an analysis-ready fact table, including standardised country and industry fields, quarterly date construction, year and quarter attributes, and a classification separating the **A-S overall economy** from individual NACE industries.

### Data-quality issue identified

During validation, the initial dashboard results did not reconcile with the original Eurostat observations. Germany's Job Vacancy Rate contained decimal values such as **2.5%**, but these were appearing as whole numbers after transformation.

Tracing the discrepancy through the Power Query pipeline revealed that an automatically generated **Changed Type** step had converted the observation-value field to a **Whole Number**, causing decimal precision to be lost.

The transformation sequence was corrected so that the observation value was assigned a **Decimal Number** data type before precision could be lost. Dashboard results were then reconciled against the raw Eurostat observations before analysis continued.

The validation process followed the chain:

**Source observation → Power Query output → DAX measure → Dashboard result**

### Post-transformation validation sample

| Validation check | Source / expected | Dashboard | Status |
|---|---:|---:|:---:|
| Germany JVR, 2015-Q1 | 2.5% | 2.5% | ✓ |
| Germany JVR, 2016-Q1 | 2.4% | 2.4% | ✓ |
| Germany JVR, 2025-Q4 | 2.8% | 2.8% | ✓ |
| Germany peak JVR | 4.5%, 2022-Q4 | 4.5%, 2022-Q4 | ✓ |
| 2025-Q4 YoY change | -0.4 pp | -0.4 pp | ✓ |

## Data Model

The transformed data were organised using a **star schema**, separating labour-market observations from the dimensions used to analyse them.

The model contains:

- **`FactLabourMarket`**: quarterly observations for job vacancies, occupied jobs and job vacancy rates
- **`DimDate`**: calendar, year, quarter and year-quarter attributes supporting time analysis
- **`DimCountry`**: geographic attributes
- **`DimIndustry`**: NACE industry classifications and the distinction between overall-economy and industry observations
- **`_Measures`**: a dedicated table containing analytical DAX measures

Each dimension has a **one-to-many, single-direction relationship** with `FactLabourMarket`, keeping filtering predictable and avoiding unnecessary many-to-many or bidirectional relationships.

Technical fields and metadata required for modelling and traceability were retained but hidden from the report view where appropriate.

![Power BI Data Model](data_model.png)

### Modelling consideration: avoiding double counting

The Eurostat dataset contains both the **A-S overall-economy aggregate** and observations for individual NACE industries.

This means that summing observations across all NACE categories without controlling the industry level could double count economic activity. Headline measures therefore use the **A-S aggregate**, while sector-level visuals use individual industry observations.

## DAX Measures

Explicit DAX measures were created rather than relying on uncontrolled implicit aggregations.

### Job Vacancy Rate

```DAX
Job Vacancy Rate =
CALCULATE(
    AVERAGE(FactLabourMarket[Value]),
    FactLabourMarket[Indicator_Code] = "JVR"
)

Job Vacancies =
CALCULATE(
    SUM(FactLabourMarket[Value]),
    FactLabourMarket[Indicator_Code] = "JOBVAC"
)

Job Vacancy Rate PY =
CALCULATE(
    [Job Vacancy Rate],
    DATEADD(DimDate[Date], -1, YEAR)
)

JVR YoY Change =
VAR PreviousYearRate = [Job Vacancy Rate PY]
RETURN
    IF(
        ISBLANK(PreviousYearRate),
        BLANK(),
        [Job Vacancy Rate] - PreviousYearRate
    )
