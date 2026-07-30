# AAV ddPCR Analytical Review Assistant - Product Spec

Version: 0 prototype  
Status: Draft for scientific and product review  
Data policy: Synthetic data only  
Validation status: Not validated for GMP, clinical, release, stability, or regulatory use

## 1. Target User

Primary users are analytical development scientists, QC assay scientists, and technical reviewers who review AAV ddPCR run outputs and need a faster way to check calculations, acceptance rules, trends, and supporting evidence.

Secondary users are assay owners, method developers, and prototype evaluators who need to inspect whether deterministic review logic is transparent enough to become the basis for a future validated workflow.

This Version 0 prototype is not intended for GMP analysts performing official batch disposition, final release testing, regulatory submissions, or validated electronic record review.

## 2. User Problem

ddPCR analytical review often requires manual checks across well-level data, sample replicates, controls, assay rules, historical run behavior, and reviewer-ready documentation. Manual spreadsheet workflows can be slow, inconsistent, and hard to audit, especially when reviewers need to trace each flag back to specific wells and calculated evidence.

The prototype should show whether a deterministic Python review engine can:

- ingest a CSV export of synthetic ddPCR well-level data;
- validate the file before calculations are trusted;
- compute consistent well, sample, and run metrics;
- apply predefined rule logic without language-model involvement;
- compare the current synthetic run against synthetic historical runs;
- present flags with traceable evidence; and
- produce a draft report suitable for human review.

## 3. Product Principles

- Deterministic first: All calculations, flags, and pass/fail/needs-review decisions are made by versioned Python functions.
- AI is optional and bounded: The language model may draft narrative review text only from structured deterministic outputs. It must not calculate values, infer missing data, override flags, or determine acceptance.
- Evidence is inspectable: Every flag must link to the rule, metric, threshold, affected entity, and source rows.
- Prototype honesty: The UI and report must clearly state that the system uses synthetic data only and is not a validated GMP system.
- Reproducibility: Given the same CSV, historical baseline, configuration, and code version, the app must produce the same results.

## 4. In-Scope Features

- CSV upload for a single ddPCR run containing well-level synthetic data.
- Schema validation for required columns, data types, categorical values, uniqueness rules, and numeric ranges.
- Well-level metric calculations from droplet counts.
- Replicate-level and sample-level titer metrics using the configured replicate structure.
- Run-level summary metrics and deterministic run status.
- Predefined acceptance rules with versioned thresholds.
- Synthetic historical comparison using a bundled or uploaded synthetic historical dataset.
- Flag display with evidence, severity, rule ID, metric value, threshold, and affected wells/samples.
- Optional AI-written draft review summary generated only from structured deterministic result JSON.
- Reviewer-ready report export containing calculations, flags, evidence, historical comparison, disclaimers, and review placeholders.
- Test fixtures using synthetic CSV data only.

## 5. Out-of-Scope Features

- GMP validation, Part 11 compliance, audit trails, validated electronic signatures, or controlled record retention.
- Use with real patient, donor, batch, clinical, manufacturing, or proprietary assay data.
- Final batch disposition, product release, stability decisions, or regulatory conclusions.
- Language-model calculation, rule interpretation, acceptance determination, or scientific adjudication.
- Direct integration with ddPCR instruments, LIMS, ELN, eQMS, data historians, or identity providers.
- Automated threshold setting from amplitude clusters.
- Raw droplet amplitude reanalysis.
- Standard curve or standards-based quantitation; standards are not present in the Version 0 ddPCR workflow.
- Assay method validation, LOD/LOQ establishment, precision studies, or system suitability qualification.
- Multi-run campaign management beyond synthetic historical comparison.

## 6. Data Schema

The Version 0 input is one CSV representing one current synthetic ddPCR run. Each row represents one well for the transgene target. Only transgene-targeted AAV titer is in scope for Version 0.

### 6.1 Required Columns

| Column | Type | Validation | Purpose |
| --- | --- | --- | --- |
| `run_id` | string | non-empty, same value for all rows in uploaded run | Identifies the synthetic run |
| `plate_id` | string | non-empty | Identifies the plate |
| `well_id` | string | 96-well format such as `A01` through `H12`; unique within `plate_id` | Locates source well |
| `sample_id` | string | non-empty | Groups replicate wells |
| `sample_type` | enum | one of `test_sample`, `positive_control`, `negative_control`, `ntc`, `system_control` | Drives rule logic |
| `assay_name` | string | non-empty | Identifies the assay configuration |
| `target_name` | string | non-empty; must match the configured transgene target in the ruleset | Identifies the transgene target |
| `replicate_id` | string | for test samples, one of `R1`, `R2`, or `R3`; each `sample_id` plus `replicate_id` should have three wells | Groups technical replicate wells |
| `accepted_droplets` | integer | `>= 0` | Total accepted droplets for the well |
| `positive_droplets` | integer | `>= 0` and `<= accepted_droplets` | Positive droplets for the target |
| `dilution_factor` | number | `> 0` | Well-specific sample dilution used to convert concentration to titer |
| `template_volume_uL` | number | `> 0` | Template volume added to reaction |
| `reaction_volume_uL` | number | `> 0` | Total reaction volume basis for concentration normalization |

### 6.2 Optional Columns

| Column | Type | Validation | Purpose |
| --- | --- | --- | --- |
| `negative_droplets` | integer | if present, must equal `accepted_droplets - positive_droplets` | Cross-check exported count |
| `positive_control_lower_titer_vg_per_mL` | number | required for positive-control rows if the value is not supplied by the ruleset | Inclusive lower titer bound for the positive control |
| `positive_control_upper_titer_vg_per_mL` | number | required for positive-control rows if the value is not supplied by the ruleset; must be `>= positive_control_lower_titer_vg_per_mL` | Inclusive upper titer bound for the positive control |
| `threshold_value` | number | optional, non-negative | Documents manual or software threshold |
| `threshold_method` | string | optional | Documents thresholding approach |
| `positive_amplitude_mean` | number | optional | Evidence only; no raw cluster reanalysis in V0 |
| `negative_amplitude_mean` | number | optional | Evidence only; no raw cluster reanalysis in V0 |
| `operator_id` | string | synthetic only | Synthetic metadata for report |
| `instrument_id` | string | synthetic only | Synthetic metadata for report |
| `reagent_lot_id` | string | synthetic only | Synthetic metadata for report |
| `notes` | string | optional | Reviewer context; not used in deterministic calculations |

### 6.3 Historical Dataset Schema

The synthetic historical dataset should use the same calculated result schema produced by the deterministic engine, not raw proprietary records. At minimum it should include:

- `historical_run_id`
- `assay_name`
- `target_name`
- `sample_type`
- `metric_name`
- `metric_value`
- `run_date`
- `ruleset_version`

Historical comparisons are flag-generating only in Version 0. They must not change sample or run acceptance status.

## 7. Calculations

All calculations must be implemented as deterministic Python functions with unit tests and frozen expected outputs for known fixtures. Droplet counts are used directly for deterministic calculations. Vendor-reported concentration values, if ever included, may be displayed only as evidence or cross-check fields and must not replace the deterministic droplet-count calculation.

### 7.1 Well-Level Calculations

For each row:

- `negative_droplets_calculated = accepted_droplets - positive_droplets`
- `fraction_positive = positive_droplets / accepted_droplets`
- `fraction_negative = negative_droplets_calculated / accepted_droplets`
- `lambda_per_droplet = -ln(fraction_negative)`
- `copies_per_uL_reaction = lambda_per_droplet / droplet_volume_uL`
- `copies_per_uL_sample = copies_per_uL_reaction * reaction_volume_uL * dilution_factor / template_volume_uL`
- `well_titer_vg_per_mL = copies_per_uL_sample * 1000`

Default prototype assumption:

- `droplet_volume_uL = 0.00085`
- one transgene copy is treated as one vector genome for Version 0 titer reporting

Special cases:

- If `accepted_droplets == 0`, concentration and titer calculations are invalid and the well receives a critical calculation flag.
- If `positive_droplets == 0`, `lambda_per_droplet = 0` and titer is reported as 0 with an interpretive flag if the sample is expected to be positive.
- If `positive_droplets == accepted_droplets`, `fraction_negative = 0`; the well is treated as saturated or non-quantifiable and titer is not reported as finite.
- If optional `negative_droplets` is present and does not match the calculated value, the row fails validation.

### 7.2 Sample-Level Calculations

For each `sample_id`, calculate titer through a two-stage replicate structure:

- one test sample has three replicate groups, expected as `R1`, `R2`, and `R3`;
- each replicate group has three wells;
- each well is converted to `well_titer_vg_per_mL` using its row-specific `dilution_factor`;
- each replicate group receives a `replicate_titer_vg_per_mL`, calculated from the three valid well titers;
- the sample-level reported titer is calculated from the three replicate titers;
- sample-level standard deviation is calculated across the three replicate titers;
- sample-level `cv_percent = 100 * sd / mean` is calculated across the three replicate titers when mean titer is greater than 0;
- minimum and maximum replicate titer are calculated across `R1`, `R2`, and `R3`;
- replicate fold spread is calculated as `max_replicate_titer / min_replicate_titer` when minimum replicate titer is greater than 0;
- total wells, valid wells, invalid wells, and wells failing droplet or saturation rules are retained as evidence.

Version 0 assumes `replicate_titer_vg_per_mL` is the arithmetic mean of the three valid well titers in that replicate. This averaging approach requires scientific review.

### 7.3 Run-Level Calculations

For the current run:

- total rows and total wells;
- total samples by `sample_type`;
- percentage of valid wells;
- number of samples passing, failing, and requiring review;
- control pass/fail status by control type and target;
- run-level mean accepted droplets;
- run-level median accepted droplets;
- distribution summaries for titer and replicate-level CV for the transgene target;
- count of critical, major, and informational flags;
- deterministic run status.

### 7.4 Historical Comparison Calculations

For each configured run-level or control metric:

- current value;
- historical count `n`;
- historical mean;
- historical standard deviation;
- historical median;
- historical median absolute deviation;
- percentile rank of current value against synthetic history;
- z-score when historical standard deviation is greater than 0;
- robust z-score when historical median absolute deviation is greater than 0.

If historical variance is zero or historical count is below the configured minimum, the app must not force an outlier conclusion. It should report insufficient historical basis. Historical outliers are flags only in Version 0 and must not determine run acceptance.

## 8. Acceptance Rules

Rules must be defined in a versioned ruleset and executed deterministically. Each rule result must include:

- `rule_id`
- `rule_version`
- `entity_type` such as `well`, `sample`, `control`, or `run`
- `entity_id`
- `severity`
- `metric_name`
- `metric_value`
- `operator`
- `threshold`
- `status`
- `evidence_rows`
- `message_template`

### 8.1 Severity Levels

- `critical`: affects deterministic run or sample acceptance.
- `major`: requires reviewer attention but may not automatically fail the run unless configured.
- `info`: contextual evidence only.

### 8.2 Prototype Default Rule Set

These are proposed Version 0 defaults for synthetic-data demonstration. They require scientific review before any non-prototype use.

| Rule ID | Entity | Severity | Prototype Logic |
| --- | --- | --- | --- |
| `SCHEMA_REQUIRED_COLUMNS` | upload | critical | All required columns must be present |
| `SCHEMA_TYPES` | upload | critical | Required columns must parse to expected types |
| `SCHEMA_DROPLET_COUNTS` | well | critical | `positive_droplets <= accepted_droplets` |
| `WELL_MIN_DROPLETS` | well | critical | `accepted_droplets >= 10000` |
| `WELL_NOT_SATURATED` | well | critical | `positive_droplets < accepted_droplets` |
| `WELL_QUANTIFIABLE_RANGE_FLAG` | well | info | fraction-positive or lambda values may be flagged if configured, but no default acceptance criterion is set in Version 0 |
| `SAMPLE_REPLICATE_STRUCTURE` | sample | critical | each test sample must have `R1`, `R2`, and `R3`, with three wells per replicate |
| `SAMPLE_REPLICATE_CV` | sample | critical | `cv_percent <= 20`, calculated across the three replicate-level titers |
| `SAMPLE_REPLICATE_SPREAD` | sample | major | replicate titer fold spread must be within configured limit if enabled |
| `NTC_FALSE_POSITIVE_LIMIT` | control | critical | NTC positive droplets must be `<= 20` for the transgene target |
| `NEGATIVE_CONTROL_LIMIT` | control | critical | negative-control titer must be below configured limit |
| `POSITIVE_CONTROL_TITER_RANGE` | control | critical | positive-control titer must be `>= positive_control_lower_titer_vg_per_mL` and `<= positive_control_upper_titer_vg_per_mL` |
| `HISTORICAL_CONTROL_OUTLIER_FLAG` | run | info | configured control metric may be flagged when absolute z-score is greater than configured limit and historical basis is sufficient |
| `HISTORICAL_RUN_OUTLIER_FLAG` | run | info | configured run metric may be flagged when outside configured synthetic historical bounds |

### 8.3 Deterministic Status Logic

- Upload status is `rejected` if schema-critical validation fails.
- Well status is `valid` only if all critical well rules pass.
- Sample status is `pass` if all critical sample and applicable well rules pass.
- Sample status is `fail` if any critical sample rule fails.
- Sample status is `needs_review` if no critical rule fails but one or more major rules are triggered.
- Run status is `pass` if all required controls pass, no critical run rule fails, and all required test samples pass.
- Run status is `fail` if any critical control or run rule fails.
- Run status is `needs_review` if no critical rule fails but major flags are present.
- Historical comparison flags and fraction-positive or lambda flags do not change run status in Version 0 unless a future reviewed ruleset explicitly changes their severity.

The language model must not set or revise these statuses.

## 9. User Interface Requirements

### 9.1 Required Screens

- Upload screen with clear synthetic-data-only warning.
- Schema validation results screen showing missing columns, type errors, row-level issues, and whether the file can be processed.
- Run overview dashboard showing deterministic run status, control status, sample status counts, and flag counts.
- Well table with sortable/filterable columns for source data, calculated metrics, validity, and flags.
- Sample table with replicate-level titer metrics, sample titer, titer-based CV, and status.
- Flags and evidence view grouped by severity, rule, sample, target, and well.
- Historical comparison view showing current metric, synthetic historical distribution summary, z-score or robust z-score, and interpretation.
- Report preview and export screen.
- Optional AI draft summary panel, clearly labeled as AI-generated draft text.

### 9.2 Interaction Requirements

- Users can upload a CSV and see validation results before calculations are accepted.
- Users can filter flags by severity, rule, sample, target, and control type.
- Users can click any flag to inspect metric value, threshold, source wells, and rule definition.
- Users can toggle between well, sample, run, and historical evidence views.
- Users can export the deterministic result JSON for debugging and test reproducibility.
- Users can generate an AI draft summary only after deterministic calculations complete.
- Users can regenerate the AI draft without changing deterministic results.

### 9.3 UI Copy Requirements

The following warning must appear on upload, report preview, and exported report:

`Version 0 prototype for synthetic data only. Not validated for GMP, clinical, release, stability, regulatory, or patient-impacting decisions. Deterministic calculations and rules require scientific review before operational use.`

The AI panel must state:

`AI-generated draft narrative only. The model did not calculate results, apply acceptance rules, or determine run status. Review and edit before use.`

## 10. AI Draft Summary Requirements

The AI summary feature is optional and must be disabled unless deterministic results exist.

The model input must be a structured JSON summary produced by deterministic Python functions. It may include:

- run status;
- ruleset version;
- sample and control statuses;
- calculated metric summaries;
- flags and evidence messages;
- synthetic historical comparison summaries;
- report disclaimers.

The model input must not include:

- raw CSV rows unless already transformed into deterministic evidence records;
- hidden acceptance criteria;
- instructions allowing the model to override deterministic status;
- real GMP, patient, donor, batch, or proprietary data.

The AI output must be stored separately from deterministic results and labeled as draft narrative. The app must preserve the deterministic structured result as the source of truth.

## 11. Report Requirements

The export should be reviewer-ready but clearly non-validated and prototype-only. Preferred V0 formats are HTML and PDF.

The report must include:

- title, run ID, export timestamp, application version, ruleset version, and input file name/hash;
- required prototype disclaimer;
- deterministic run status;
- upload validation summary;
- run-level metric summary;
- control results and control flags;
- sample-level results table;
- well-level appendix table;
- flag and evidence table with rule IDs, thresholds, values, severity, and affected wells;
- synthetic historical comparison summary;
- optional AI-written draft review summary, if generated, labeled as draft;
- reviewer notes section;
- reviewer name/date placeholders with no claim of validated electronic signature;
- appendix describing calculation formulas and acceptance rules.

The report must not:

- claim GMP validation;
- imply final batch disposition;
- hide failed rules behind AI narrative;
- omit critical validation failures;
- present synthetic historical data as real manufacturing history.

## 12. Test Requirements

### 12.1 Unit Tests

Unit tests are required for:

- schema validation;
- type parsing;
- well ID validation;
- droplet count consistency;
- Poisson concentration calculation;
- well titer calculation from concentration and well-specific dilution factor;
- zero droplet edge case;
- zero positive droplet edge case;
- saturated well edge case;
- replicate titer calculation from three well titers;
- sample mean, median, standard deviation, and CV calculated across three replicate titers;
- positive-control inclusive lower and upper titer bounds;
- historical z-score and robust z-score calculations;
- deterministic status aggregation.

### 12.2 Fixture Tests

Create synthetic fixture CSVs for:

- valid passing run;
- missing required column;
- invalid numeric type;
- positive droplets greater than accepted droplets;
- low accepted droplets;
- saturated well;
- NTC false-positive failure;
- positive-control titer below inclusive lower bound;
- positive-control titer above inclusive upper bound;
- sample replicate CV failure;
- invalid replicate structure;
- historical outlier flag with sufficient history;
- historical comparison with insufficient history;
- optional fraction-positive or lambda flag.

Each fixture must have frozen expected JSON outputs for calculated metrics, flags, and statuses.

### 12.3 AI Boundary Tests

Tests must verify that:

- AI summary generation cannot run before deterministic results exist;
- AI summary input contains only approved structured fields;
- pass/fail/needs-review statuses are identical before and after AI summary generation;
- report export uses deterministic status, not AI wording, as the source of truth.

### 12.4 UI and Report Tests

Tests should cover:

- upload validation state;
- deterministic status rendering;
- flag filtering and evidence drilldown;
- historical comparison display;
- report export creation;
- required disclaimer visibility;
- AI draft label visibility when AI text is included.

## 13. Disclaimers and Prototype Limitations

- This is a Version 0 prototype using synthetic data only.
- The prototype is not validated for GMP, clinical, release, stability, regulatory, or patient-impacting decisions.
- The prototype does not establish or justify analytical acceptance criteria.
- Prototype thresholds are placeholders until reviewed and approved by qualified assay scientists.
- The prototype does not replace assay method validation, procedural controls, QA review, or regulated recordkeeping.
- The prototype does not perform raw droplet amplitude analysis or threshold setting.
- Synthetic historical comparison is for workflow demonstration only and must not be interpreted as real process capability.
- The language model is limited to optional draft narrative generation from deterministic structured results.

## 14. Assumptions Requiring Scientific Review

The following assumptions must be reviewed by qualified ddPCR/AAV assay scientists before any expansion beyond prototype demonstration:

- Whether the required CSV columns match the intended instrument export and review workflow.
- Whether any vendor-reported concentration fields should be displayed as evidence-only cross-checks.
- Whether the assumed droplet volume of `0.00085 uL` is appropriate for the intended platform, assay, and export.
- Whether the concentration normalization formula correctly reflects sample preparation, dilution, template input, extraction, digestion, and reaction setup.
- The exact transgene target name and whether one measured transgene copy should be treated as one vector genome for Version 0 titer reporting.
- Minimum accepted droplet threshold per well.
- Whether fraction-positive or lambda values should remain informational flags only, and whether any reviewed criteria should be added later.
- How to handle deviations from the expected `R1`, `R2`, `R3` structure and three wells per replicate.
- Whether replicate titer should be calculated as the arithmetic mean of the three well titers or by another estimator.
- Whether any replicate agreement checks beyond titer-based CV `<= 20` are needed, such as fold-spread or well-level outlier flags.
- Negative-control titer limit and any additional NTC handling beyond the `<= 20` positive-droplet limit.
- Positive-control lower and upper titer bounds, where those bounds should be configured, and handling when bounds are missing.
- Whether historical outlier flags should remain informational only in Version 0.
- Minimum synthetic historical dataset size for meaningful comparison.
- Whether mean, median, robust statistics, or another estimator should drive final reported sample titer.
- How invalid wells should be excluded, displayed, and justified.
- Whether reviewer comments can alter final displayed status or only document scientific rationale.
- Required report wording for internal prototype review.
- Any additional controls required for the specific AAV ddPCR method.

## 15. Non-Normative Context References

These references are included only to ground the prototype spec in common dPCR transparency concepts. They do not validate this prototype or its thresholds.

- Digital MIQE Guidelines Update, dMIQE2020: https://doi.org/10.1093/clinchem/hvaa125
- NIST listing for the original dMIQE guidelines: https://www.nist.gov/publications/guidelines-minimum-information-publication-quantitative-digital-pcr-experiments
- Example discussion of Poisson transformation in ddPCR concentration calculations: https://pmc.ncbi.nlm.nih.gov/articles/PMC5996397/
