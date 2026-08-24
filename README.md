# 🏭 Real-Time Production Monitoring - Decision Support System

End-to-end **Data Warehouse & BI system** for a manufacturing production line: raw data from an Oracle ERP and proprietary machine log files is extracted (Python + SSIS), modeled into a SQL Server star schema across three layers, and served through **Power BI dashboards** that answer concrete operational questions, which operator generates the most waste, which article consumes the most raw material, how machine capacity is allocated.

## Context and objective

A wire/terminal-crimping production line generates two very different kinds of data: structured order data in an **Oracle ERP**, and raw **machine-generated log files** (`.SDC`/`.LDC`) tracking every sample, learn, and production cycle per article. Neither is directly usable for decision-making. This project builds the pipeline that turns both into a single, queryable data warehouse and a set of **real-time dashboards** a production manager can actually act on.

## Architecture

```
┌────────────────────┐        ┌───────────────────────────┐
│   Oracle ERP DB     │       │  Machine log files        │
│ (order requirements)│       │  (.SDC / .LDC,per machine)│
└──────────┬──────────┘       └──────────────┬────────────┘
           │ Python (cx_Oracle)              │ Python (custom parser)
           │ export to CSV                   │ + SSIS package
           ▼                                 ▼
        ┌─────────────────────────────────────────────┐
        │   Staging DB — SystemeSuiviProduction       │
        └───────────────────────┬─────────────────────┘
                                │ cleaning, filtering, type casting
                                ▼
        ┌─────────────────────────────────────────────┐
        │Transform DB—TransformSystemeSuiviProduction │
        └───────────────────────┬─────────────────────┘
                                │ dimensional modeling
                                ▼
        ┌─────────────────────────────────────────────────┐
        │   Data Warehouse — DWHSystemeSuiviProduction    │
        │   Star schema: DimJOB · DimEQPT · DimOperateur  │
        │   · DimDetailsProducti (fact)                   │
        └───────────────────────┬─────────────────────────┘
                                ▼
                     Power BI — 2-page dashboard
              (Operational overview + Consumption/Waste KPIs)
```

Three distinct SQL Server databases (**staging → transform → DWH**) rather than one monolithic schema, a deliberate layered design that keeps raw ingestion, business-rule cleaning, and the reporting-ready star schema decoupled, so each stage can be debugged or rebuilt independently.

## Data sources

1. **Oracle ERP** (`BESOIN_OF_H`, `BESOIN_OF_GROF_H`) : order/requirement data, extracted via `cx_Oracle` and exported to CSV.
2. **Machine log files** (`.SDC`, `.LDC`) : proprietary, semi-structured text logs written by production machines per job. Each file mixes multiple record types (`SampleStarted`, `LearnStarted`, `Counter`, quality/measurement blocks, …) that had to be parsed and recombined into coherent rows, a custom Python parser handles this rather than a generic CSV reader, since no off-the-shelf tool speaks this format.

## ETL pipeline deep-dive

**Extraction isn't just `pd.read_csv`.** The `.SDC`/`.LDC` parser reconstructs the production machine's identity from the folder structure itself (`.../<machine_id>/<month>/<day>/Producti.SDC`), and stitches related sections (e.g. `LearnStarted` + the `Counter` block that follows it) back into single, analyzable records, the kind of parsing work that's specific to real industrial data, not a dataset that arrives pre-cleaned.

**Transformation applies domain rules, not just generic cleaning**: irrelevant block types (`MaterialChangeDetection`, `QualityParameters`, …) are filtered out, numeric fields are coerced and null-handled, and the staging-to-transform step keeps only the record types that matter for production tracking (dropping `Started`/`Aborted` noise in favor of `Terminated` outcomes, for instance).

**Loading builds a proper star schema**: `DimJOB`, `DimEQPT` (equipment), `DimOperateur`, and a `DimDetailsProducti` fact table carrying the consumption and waste metrics (`ConsoBrute`/`ConsoNET` for wire, terminal, and seal materials) that the dashboards report on.

## Data warehouse schema

| Table | Role | Key measures / attributes |
|---|---|---|
| `DimJOB` | Dimension | Job ID, date/time, mode (insert/delete) |
| `DimEQPT` | Dimension | Equipment/machine code |
| `DimOperateur` | Dimension | Operator username |
| `DimDetailsProducti` | Fact | Wire/Terminal/Seal gross and net consumption, waste (`Dechet`), sample/learn/production counts per article |

## Power BI dashboards

**Page 1 : Operational overview**
- Jobs per machine (donut chart)
- Operator generating the most waste / highest yield (KPI cards)
- Distinct wire codes, terminals, and seals in production (KPI cards)
- Sample vs. Learn vs. Production split per article (donut + 100% stacked column by operator)

**Page 2 : Consumption & waste KPIs**
- Raw material consumption gauges: Wire, Terminal ×2, Seal ×2
- Net consumption by article, broken down by material (clustered column)
- Waste vs. gross vs. net consumption (pie chart)
- Article with the most / least waste (KPI cards)
