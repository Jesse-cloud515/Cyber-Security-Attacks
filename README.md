# Cybersecurity Threat Intelligence Dashboard

An end-to-end analytics project on 40,000 security events (January 2020 to October 2023): cleaning and exploratory analysis in Python, statistical testing, and a 7-page interactive Power BI dashboard.

![Executive Overview page](images/1_dashboard_executive_overview.png)

## The short version

Attack types, severities, protocols and response actions each split almost exactly into thirds, and none of the relationships I tested is statistically significant. The one result worth acting on is that **response does not follow risk**. High-severity events were blocked 33.9% of the time and Low-severity events 34.0%. Of the 13,382 High-severity events, 4,460 were ignored.

The evenness, plus every IP address appearing exactly once, points to a synthetic dataset. I treat the project as a demonstration of method, not a description of a real network.

## Repository structure

```
├── cybersecurity_attacks_cleaned.csv        # cleaned dataset (40,000 rows × 33 columns)
├── original data/
│   └── cybersecurity_attacks.csv            # raw source file
├── Cybersecurity_Attacks.ipynb              # data cleaning
├── cybersecurity_attacks-eda.ipynb          # exploratory data analysis
├── Visualisation and Presentation.ipynb     # presentation charts + findings table
└── Cybersecurity_PowerBI_Project/
    ├── PowerBI_Project/PBIP_Source/         # Power BI Project (.pbip, .Report, .SemanticModel)
    ├── Power_Query/Transformations.m        # all Power Query (M) code
    ├── Documentation/                       # overview, DAX measures, data model, user guide (PDF + MD)
    └── Validation/                          # KPI_Validation.dax, validate_kpis.py, expected_kpis.json
```

## Questions

1. Which attack types, severities, protocols and traffic types dominate?
2. Are some attack types riskier than others?
3. Do anomaly scores separate high-risk events from low-risk ones?
4. Is the response (Blocked, Logged, Ignored) matched to risk?
5. Are there time, device or geographic patterns?

## The data

Each row is one security event. The 33 columns cover time (timestamp plus derived year, month, weekday, hour and time period), network (IPs, ports, protocol, packet length, traffic type, segment, proxy), threat (attack type, signature, severity, anomaly score, malware indicators), detection and response (alerts, IDS/IPS alerts, firewall logs, log source, action taken), user and device (user name, browser user-agent, "City, State" location) and a free-text payload.

Five indicator columns are about half empty (49.6% to 50.2%), stored as empty strings. An empty value only means the indicator wasn't recorded.

## Process

1. **Cleaning** (`Cybersecurity_Attacks.ipynb`) produced the cleaned CSV and added derived time columns.
2. **EDA** (`cybersecurity_attacks-eda.ipynb`) covered univariate, categorical, numerical, bivariate and multivariate analysis, plus a correlation matrix.
3. **Presentation charts** (`Visualisation and Presentation.ipynb`) produced about 15 charts and a findings table.
4. **Statistical tests** in SciPy: chi-square tests of independence and a Kruskal–Wallis test on anomaly scores.
5. **Power BI** build: Power Query transformations, a star-schema model, 52 DAX measures and seven report pages, saved as a Power BI Project (PBIP) so every part is a readable text file.
6. **Validation**: every KPI was recalculated in pandas and compared against the dashboard.

## Power BI dashboard

![Data model](images/3_data_model.png)

One fact table of events is filtered by three lookup tables: Date, Severity and Response Action. All relationships are one-to-many and single direction. Auto date/time is off, since the model has its own date table. Measures live in a separate `_Measures` table, grouped into seven folders.

| Page | What it shows |
| --- | --- |
| Executive Overview | KPI cards, monthly attack trend, severity mix, top attack types, response mix, key findings |
| Attack Analysis | Attack type share, weekday and year → month frequency (drill-down), type × severity, signatures, protocol mix, IPs ranked by anomaly score |
| Network Traffic | Traffic type, protocol, packet-size bands, packet type, port ranges, traffic trend |
| Threat & Severity | Severity, anomaly bands, anomaly score by severity, IoC flags, indicators by attack type, device OS, states |
| Response & Operations | Blocked / Logged / Ignored counts, response by severity and attack type, trend, operational-gap table |
| Investigation | Row-level event log with IP and user search (drill-through target) |
| Key Insights | Findings, recommendations, limitations |

Every page has synced slicers (date, attack type, severity, protocol, traffic type, action taken), a reset button, cross-filtering and tooltips. Severity and response always use the same colours: red, orange and teal for High, Medium and Low; green, orange and red for Blocked, Logged and Ignored.

Power Query does the shaping, and the original CSV is never changed:

- Load with proper quote handling, because the payload text contains line breaks.
- Type dates with the en-GB culture (the file is day/month/year).
- Turn the empty indicator strings into real blanks.
- Rename columns and add an Event ID.
- Add readable flag columns (Malware Flag, Alert Status, IDS/IPS Status and others).
- Split location into City and State, and parse device OS and browser from the user-agent.
- Add bands for anomaly score, packet size and port range.

Two requested measures weren't built because the data can't support them. There is no Critical severity level. A true detection rate is also impossible, because every row is already a labelled attack and there are no benign or missed events. `IDS IPS Detection Coverage %` is the closest valid alternative.

## Findings

![Response by severity](images/2_response_vs_severity.png)

| Area | Finding | Evidence |
| --- | --- | --- |
| Response vs severity | Not risk-based | Blocked: High 33.9%, Medium 33.6%, Low 34.0%; chi-square p = 0.49 |
| Gaps | High-risk events left without action | 4,460 High-severity events ignored (33.3%), 66.1% not blocked; 6,576 IoC-flagged events ignored |
| Attack types | Even split | DDoS 33.6%, Malware 33.3%, Intrusion 33.2% |
| Severity | Even split, no Critical level | Medium 33.6%, High 33.5%, Low 33.0% |
| Attack type vs severity | No type is riskier | Chi-square p = 0.77 |
| Anomaly scores | Uniform 0–100, unrelated to severity | Mean 50.11, median 50.35; Kruskal–Wallis p = 0.32 |
| Volume over time | Steady, no trend | About 29 events a day (range 15–49); full months 788–950 |
| Time of day / week | No real peak | Busiest hour 13:00 (1,737) vs quietest 12:00 (1,588); weekday and weekend both 29.0 a day |
| Network | Balanced | ICMP, UDP and TCP about 33% each; DNS, HTTP and FTP about 33% each |
| Numeric columns | No correlation | All Pearson \|r\| ≤ 0.005 |
| IP addresses | No repeats | All 40,000 source and destination IPs unique |
| Devices | Windows leads | Windows 44.9%, Linux 17.9%, macOS 14.5%, iOS 14.4%, Android 8.2% |

## Recommendations

1. Block High-severity and IoC-flagged events automatically.
2. Send every ignored High-severity or IoC event to a review queue with an owner.
3. Validate or recalibrate the anomaly model before using it for triage.
4. Record outcomes (true or false positive, time to contain), so "Logged" can be told apart from "resolved".
5. Add asset criticality and stable host and account keys, so repeat attackers and exposed assets can be ranked.
6. Log benign traffic as well, so a real detection rate can be measured.

## Issues I ran into

- **Day-first dates.** Without a pinned culture, dates like 05/07 load as May on some machines and July on others. I set en-GB in Power Query. pandas flags it too, so the notebooks should pass the format explicitly.
- **Line breaks inside text fields** split rows until the file was loaded with CSV quote handling.
- **Empty strings that looked like data.** Five columns were half empty strings, which counted as values until I converted them to blanks.
- **An empty findings CSV.** The notebook cell that creates an empty findings table ran after the cell that filled it, so the save cell wrote only a header. Running the notebook top to bottom fixes it.
- **Dashboard layout.** KPI cards showed their title twice, which pushed the numbers off the card, and the slicers had no room to draw their dropdowns. Both were small fixes once spotted.
- **Results that looked too even.** It took a few tests to accept that the honest finding was "no pattern", and to say the data is probably synthetic rather than talk up 33.6% vs 33.2%.

## Validation

Every headline KPI was recalculated in pandas (`Validation/validate_kpis.py`), with expected values in `Validation/expected_kpis.json`. The dashboard matches: 40,000 events, 13,382 High-severity, average anomaly score 50.11, 20,000 IoC-flagged, 33.8% block rate. Running `Validation/KPI_Validation.dax` in Power BI's DAX query view gives the full measure-by-measure comparison.

## Limitations

- The data looks synthetic, so the findings show the method, not real attack behaviour.
- Labels are as recorded. Nothing confirms an event was malicious or that a block worked.
- The tests show associations or their absence, not causes.
- 2023 stops on 11 October, so its totals aren't comparable with full years.

## How to run it

**Python notebooks.** You need Python 3 with pandas, NumPy, Matplotlib, seaborn and SciPy. Open the notebooks in Jupyter and run them top to bottom.

**Power BI dashboard.**

1. Install a current version of Power BI Desktop (it needs PBIP/PBIR project support).
2. Open `Cybersecurity_PowerBI_Project/PowerBI_Project/PBIP_Source/Cybersecurity_Threat_Intelligence_Dashboard.pbip`.
3. Go to **Home > Transform data > Edit parameters** and set `CsvFilePath` to the full path of `cybersecurity_attacks_cleaned.csv` on your machine.
4. Click **Refresh**. The first load reads the full 19 MB file and takes a minute or two.
5. Optional: **File > Save as** a `.pbix` for a single-file copy.

## Tools

Python (pandas, NumPy, Matplotlib, seaborn, SciPy), Jupyter, Power BI Desktop (Power Query M, DAX, TMDL/PBIR project format).
