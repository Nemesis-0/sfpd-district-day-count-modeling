# San Francisco Police Incident Analysis: PySpark and Count Modeling

This project analyzes San Francisco Police Department (SFPD) incident reports using **PySpark, Spark SQL, and statistical count modeling**. The analysis emphasizes reproducible data processing, careful construction of analytical units, temporal and geographic patterns, model diagnostics, and robustness assessment.

## Project Overview

The raw DataSF dataset contains more than one million rows, with multiple rows potentially associated with the same police report because a report may contain multiple incident categories. The analysis therefore distinguishes **row-level, report-level, and report-category-level** records before constructing the primary analytical datasets.

The main analysis focuses on `Initial`, `Coplogic Initial`, and `Vehicle Initial` reports from **January 1, 2018 through December 31, 2025**. Supplement reports are excluded so that later updates to existing reports are not treated as new report occurrences.

Temporal summaries and district-day counts are indexed by the recorded **Incident Date** rather than the report-filing date.

## Data

**Source:** DataSF, *Police Department Incident Reports: 2018 to Present*  
**Snapshot date:** July 22, 2026  
**Primary analysis period:** January 1, 2018 – December 31, 2025

The raw snapshot contains:

- **1,048,600 rows**
- **869,759 unique report IDs**
- **709,510 Initial-report IDs** in the primary analysis population

The raw CSV is not included in this repository because of its size. To reproduce the analysis, download the DataSF dataset and save the snapshot locally as:

```text
sfpd_incidents_2018_present_2026-07-22.csv
```

## Analytical Workflow

The project includes:

- PySpark-based data ingestion and schema normalization
- Missingness and data-type validation
- Record-level consistency checks across report IDs
- Construction of report-level and report-category-level analytical datasets
- Spark SQL analysis of temporal, categorical, and geographic patterns
- Construction of a complete police-district-by-day count panel
- Poisson regression as a baseline count model
- Negative Binomial (NB2) modeling for overdispersed counts
- Interpretation through expected count ratios
- Residual and extreme-observation diagnostics
- Within-district residual autocorrelation analysis
- Negative Binomial GEE with an AR(1) working correlation as a temporal-dependence sensitivity analysis

## Descriptive Findings

### Long-Term Report Volume

Mean daily Initial-report volume declines from approximately **289 reports per day in 2018** to **219 in 2020**, partially recovers during 2021–2023, and then falls to approximately **209 in 2024** and **180 in 2025**.

The magnitude of these long-term changes is substantially larger than the pooled calendar-month differences.

### Incident Composition

**Larceny Theft** is the most prevalent category, appearing in approximately **38% of Initial reports** over the full analysis period.

Category composition also changes over time. Malicious Mischief, Motor Vehicle Theft, and Burglary all increase in relative prevalence around 2020, although the persistence of these changes differs across categories. Motor Vehicle Theft remains relatively elevated through 2024, while the increases in Malicious Mischief and Burglary are less persistent.

Because a report may contain multiple incident categories, category prevalence is non-exclusive and does not sum to 100%.

### Geographic Distribution

Among in-city Initial reports, **Central** accounts for the largest cumulative share, followed by **Northern, Mission, and Southern**.

District composition changes over time. Central represents a smaller share of reports in later years, while districts including Southern and Tenderloin become relatively more prominent.

These results describe report volume rather than population- or exposure-adjusted crime risk.

### Calendar Patterns

Across the pooled 2018–2025 period:

- **April** has the lowest mean daily Initial-report volume.
- **August** has the highest mean daily Initial-report volume.
- **Friday** has the highest mean daily volume across weekdays.
- **Sunday** has the lowest.

Recorded incident times also exhibit substantial heaping at round clock times, particularly minutes `00` and `30`, so hourly patterns are interpreted cautiously.

## Count Modeling

The modeled outcome is the number of Initial-report IDs associated with each incident date and police district.

For district $d$ and date $t$, the primary model is:

```math
Y_{dt} \sim \mathrm{NB2}(\mu_{dt}, \alpha)
```

with

```math
\log(\mu_{dt})
=
\beta_0
+
\beta_{\mathrm{district}(d)}
+
\beta_{\mathrm{year}(t)}
+
\beta_{\mathrm{month}(t)}
+
\beta_{\mathrm{weekday}(t)}
```

and conditional variance

```math
\mathrm{Var}(Y_{dt}\mid X)
=
\mu_{dt}+\alpha\mu_{dt}^{2}.
```

The reference categories are:

- Central police district
- 2018
- January
- Monday

Because the model does not include a population or exposure offset, exponentiated coefficients are interpreted as **expected count ratios**, not population-standardized incidence rate ratios.

## Model Selection

A Poisson baseline exhibits substantial residual overdispersion:

- Raw variance-to-mean ratio: approximately **5.45**
- Poisson Pearson dispersion: approximately **2.03**
- Poisson deviance / residual degrees of freedom: approximately **1.92**

The Negative Binomial model provides a substantially improved variance specification:

- Estimated NB2 dispersion parameter: **α = 0.0382**
- NB2 Pearson dispersion: approximately **1.09**
- Substantially lower AIC and BIC than the Poisson baseline

NB2 is therefore retained as the primary count model.

## Adjusted Findings

After simultaneously adjusting for district, year, month, and weekday:

- Expected daily report volume in **2025 is approximately 62.5% of the corresponding 2018 level**.
- Geographic differences across police districts remain substantial.
- Calendar-month effects are comparatively modest relative to the annual shifts.
- August remains slightly above the January reference level, while March and April are lower.
- Friday remains approximately **10% above Monday**, while Sunday remains below the Monday reference level.

These estimates represent **adjusted associations rather than causal effects**.

## Model Diagnostics and Sensitivity Analysis

The NB2 model substantially improves the count variance specification, but its Pearson residuals retain positive within-district temporal dependence.

The average lag-1 residual autocorrelation is approximately **0.28**.

To assess whether this dependence materially changes the estimated covariate effects, the primary mean specification is re-estimated using a Negative Binomial GEE with:

- Police district as the clustering unit
- An AR(1) working correlation structure
- The NB2 dispersion estimate fixed at **α = 0.0382**

The estimated AR(1) working correlation is approximately **0.30**.

The substantive effect estimates are highly stable between NB2 and GEE:

- Median relative count-ratio difference: **0.029%**
- Mean relative count-ratio difference: **0.172%**
- Maximum relative count-ratio difference: **0.899%**

The GEE analysis is therefore used as a **coefficient-stability sensitivity analysis** rather than as a replacement for the primary NB2 specification.

## Interpretation and Limitations

### Reported Incidents versus Underlying Crime

The dataset records police incident reports rather than the complete underlying incidence of crime. Changes in report volume may reflect changes in criminal activity, reporting behavior, police recording practices, online reporting systems, or other institutional processes.

The results therefore describe **reported SFPD incidents** and should not be interpreted directly as changes in underlying crime risk.

### Geographic Exposure

The district-level models do not include population, foot traffic, employment, tourism, or other exposure measures.

District effects therefore represent differences in expected report counts rather than per-capita or exposure-adjusted crime rates.

### Observational Interpretation

The models estimate adjusted associations rather than causal effects. Year indicators summarize differences across years after adjustment for district, month, and weekday, but they do not identify the causes of those differences.

### Recorded Incident Time

The incident-time field exhibits substantial heaping at round clock times. Hourly patterns should therefore be interpreted as patterns in recorded incident times rather than precise behavioral timing.

### Temporal Dependence

NB2 substantially improves the variance specification but does not eliminate within-district temporal dependence.

The GEE sensitivity analysis indicates that the principal effect estimates are highly stable to an AR(1) working correlation structure. However, only **10 police-district clusters** are available, so GEE robust standard errors and significance tests are not emphasized.

### Additive Mean Structure

The primary model uses additive district and calendar effects and does not include district-by-year interactions or district-specific temporal trends.

Adjusted district effects should therefore be interpreted as average differences across the 2018–2025 study period rather than as time-invariant district relationships.

## Repository Files

```text
sfpd_incident_analysis.ipynb   # Complete analysis notebook
sfpd_incident_analysis.html    # Static rendered version
requirements.txt               # Python dependencies
.gitignore                     # Excludes raw data and local files
```

The raw CSV remains local and is intentionally excluded from version control.

## Requirements

- Python 3
- Java 17
- PySpark
- pandas
- NumPy
- Matplotlib
- statsmodels

Install the Python dependencies with:

```bash
pip install -r requirements.txt
```

## Reproducibility

1. Download the SFPD incident-report dataset from DataSF.
2. Save the CSV in the project directory as:

```text
sfpd_incidents_2018_present_2026-07-22.csv
```

3. Install the required Python packages:

```bash
pip install -r requirements.txt
```

4. Ensure Java 17 is available for PySpark.
5. Open and run:

```text
sfpd_incident_analysis.ipynb
```

The notebook uses `America/Los_Angeles` as the Spark SQL session time zone.

The current notebook contains a local Homebrew Java 17 path in the environment setup cell. Users on other systems may need to update `JAVA_HOME` to match their local Java installation.