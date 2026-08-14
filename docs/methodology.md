# Technical Methodology

This document provides supporting detail for the [case-study README](../README.md). It describes the engineering method generically and does not reproduce client code, schemas, thresholds, configuration, results, or proprietary business rules.

## 1. System Objective

The recurring workflow was designed to produce a transparent forecasting baseline and the analytical datasets required to review it. Forecast generation was one stage of a broader process:

```text
prepared sources
    -> standardized analytical data
    -> demand profiles and candidate eligibility
    -> chronological model evaluation
    -> selected aggregate forecast
    -> business post-processing and allocation
    -> review-ready outputs with model context
```

I approached the work as a forecasting system, not only as a model-building exercise. Improving the algorithm alone would not have resolved repeated data preparation, inconsistent comparison views, limited model traceability, or unclear ownership of business adjustments.

## 2. Input and Automation Boundary

The workflow began from business-prepared files representing historical demand, hierarchy mappings, planning information, comparison forecasts, and lifecycle context. Source-file assembly remained human-owned because approved automated interfaces to the upstream systems were outside the implemented operating model.

Once those files were available, the pipeline:

- standardized identifiers, types, and monthly dates
- mapped records to a consistent business hierarchy
- calculated the agreed demand representation
- identified missing or inconsistent mappings
- removed records outside the modeling scope
- completed monthly histories where required
- produced aggregate datasets for modeling
- published cleaned comparison and reporting datasets for analyst use

The pipeline was triggered manually in Databricks for each required cycle. I keep that operating boundary explicit because the implemented workflow was user-triggered monthly execution rather than scheduled orchestration or continuous retraining.

The observable modeling target was an internal sales-volume measure used as a proxy for demand, not total market demand. Some external flows and future business events were outside the historical target. This boundary limits what a history-driven forecast can infer and makes structured business context important for interpretation.

## 3. Modeling Grain and Hierarchical Outputs

Demand planning required detailed outputs, but many histories at the lowest reporting level were too sparse or unstable for meaningful model competition. The workflow modeled a more stable aggregate grain and recovered the required detail through maintained allocation shares.

This decision had three consequences:

- aggregation improved signal density and validation stability
- top-down allocation preserved the required reporting hierarchy
- detailed quality depended on both the aggregate model and the allocation inputs

The system should therefore diagnose aggregate-model error separately from allocation error. A reasonable parent forecast can still produce weak detailed outputs when shares are missing, outdated, or inconsistent.

For every parent series and forecast period, a hardened implementation should assert:

```text
sum(published detailed forecasts) == adjusted parent forecast
```

Allocation was intended to preserve the adjusted parent total when a complete and valid set of shares was available. Missing shares, filters, fills, and downstream business rules can alter that relationship, so reconciliation should be an explicit automated assertion rather than an assumed property of the allocation logic.

## 4. Demand Profiling

### 4.1 Operational Purpose

Profiling was a model-pool filtering step. It reduced unnecessary computation and prevented methods from competing where the observed demand structure did not support them. The profile did not choose the winning model.

The profiling design followed the established supply-chain forecasting framing for intermittent demand. At a high level, it considered:

- **Average demand interval:** how frequently non-zero demand occurred.
- **Squared coefficient of variation of non-zero demand:** how variable demand quantities were when demand occurred.

These dimensions separated regular, erratic, intermittent, and lumpy demand behavior. History length, the share of zero periods, recent activity, and time since last demand provided additional operational routes for short and apparently inactive histories. I keep the public explanation at this level and omit client-specific thresholds and implementation parameters.

Supporting diagnostics included trend strength, seasonality strength, autocorrelation, growth, structural-change indicators, outlier frequency, and spectral forecastability. These features supported analysis and interpretation; they did not all determine the active candidate pool.

### 4.2 Profile Lifecycle

A profile was a current-state description, not a permanent product attribute. It could change as new history arrived or recent activity changed. Recomputing profiles for each forecast cycle was therefore intentional. Profile migration should be monitored because it can change candidate eligibility and the expected level of analyst review.

### 4.3 Full-history Profiling and Evaluation Interpretation

Profiles were recalculated from the latest complete history to identify suitable model families for the next forecasting cycle. Their operational purpose was to reduce unnecessary computation, align candidate methods with current demand behavior, and make model selection easier to explain.

Forecast models were trained on pre-holdout history and evaluated against a separate chronological holdout. Profiling determined which methods were eligible for evaluation; it did not fit the models or select the final method by itself.

In the retrospective study, profiling and candidate eligibility were reconstructed at each historical forecast origin from the inputs available at that cutoff. This reproduced the workflow's point-in-time routing decision instead of applying the latest profile retrospectively to every origin.

## 5. Candidate Model Families

Candidate eligibility depended on the operational profile.

| Demand situation | Representative eligible methods | Rationale |
| --- | --- | --- |
| Regular demand | Classical methods, baselines, and eligible global ML | More consistent signal supports broader competition |
| Intermittent history | Croston-family methods, ADIDA, TSB, and baselines | Purpose-built treatment for stop-start demand |
| Lumpy history | ADIDA, IMAPA, TSB, and baselines | Sparse and variable demand requires restricted competition |
| Short history | Baselines | Avoid unsupported model complexity |
| Apparently inactive | Separate operational route | Lifecycle handling should not rely on ordinary extrapolation alone |

The implemented candidate library included:

- historical-average, naïve, seasonal-naïve, and drift baselines
- Theta, ETS, CES, ARIMA, and seasonal ARIMA methods
- Croston variants, ADIDA, IMAPA, and TSB for sparse demand
- a global LightGBM candidate trained across eligible series

The global model used lagged demand, rolling and expanding summaries, exponentially weighted features, seasonal transformations, and calendar variables. MLForecast provided the cross-series forecasting framework, LightGBM provided the estimator, and Optuna tuned feature and model configurations.

LightGBM was intentionally excluded from intermittent and lumpy final competition. That decision reduced computation and avoided selecting a flexible global model on weak, zero-heavy evidence where specialized methods were more defensible.

The earlier exploratory phase used a smaller and different pool, including Prophet, ARIMA/SARIMA, and lag-based linear regression. It also permitted manual method selection or blending. The recurring workflow instead selected one traceable method per series.

## 6. Chronological Validation and Selection

### 6.1 Rolling-origin Shortlisting

Statistical candidates were evaluated through rolling-origin cross-validation on the training history. This preserved temporal order and tested performance at several forecast origins rather than relying on a random split.

The global LightGBM configuration was tuned across eligible series using time-based folds. Hyperparameter and feature search occurred within the training-stage competition rather than being optimized once on the final recent holdout.

The strongest statistical candidates were shortlisted for each series.

### 6.2 Recent-period Selection

Shortlisted statistical candidates—and LightGBM where eligible—were trained on the pre-holdout history and evaluated against a recent chronological period. One method was selected per series using an objective that combined absolute forecast error with the magnitude of forecast bias:

```text
selection score = MAE + abs(Bias)
```

This discouraged solutions that achieved a reasonable average error while persistently over- or under-forecasting.

### 6.3 Refit and Forecast

After recent-period ranking, candidate methods were refit on the available history and used to generate the planning horizon. Refitting allowed the future forecast to use the latest observations after model-selection evidence had been established.

### 6.4 Selection Holdout and Broader Validation

The recent holdout supported candidate comparison and method selection. Its results were not treated as guarantees of future accuracy.

Broader performance was evaluated separately through the structured retrospective portfolio validation described in Section 8.

### 6.5 Validation Boundary for Profile Routing

The implementation trained candidate models on pre-holdout history, but its operational demand profile was calculated from the complete history available at the run date. The profile did not fit or select the final model, but it did restrict which model families were eligible to enter the holdout comparison. The recent holdout was therefore chronologically separate for model fitting but not completely independent of candidate routing.

A hardened selection design would calculate the routing profile from training-only history, perform holdout selection, and then recompute the operational profile on all available history before final refitting. This would preserve the operational purpose of current-state profiling while keeping the model-selection holdout fully isolated from eligibility decisions.

## 7. Metrics and Their Roles

The workflow used several metrics for different purposes rather than treating them as interchangeable.

| Metric | Role | Important interpretation |
| --- | --- | --- |
| MAE | Original-unit forecast error | Directly interpretable but scale-dependent |
| Bias | Mean directional error | Reveals systematic over- or under-forecasting |
| MAE + \|Bias\| | Statistical shortlisting and final per-series selection | Balances magnitude and direction in original units |
| sMAPE and MAPE | Supporting diagnostics | Percentage metrics require caution for sparse demand |
| WMAPE | Later portfolio-level retrospective analysis | Weights error by business volume; not the original selection rule |
| AIC | ARIMA configuration in the pilot | In-sample information criterion used during exploration, not the final selection objective |

The statistical and global-ML tuning paths therefore did not use identical objective scaling. Final series-level selection returned to the absolute-unit error-and-bias score. This implementation choice should be explicit when interpreting how candidates reached the holdout stage.

## 8. Retrospective Portfolio Validation

### 8.1 Reconstructed Historical Origins

After the recurring workflow had been completed, the full pipeline was rerun retrospectively at successive historical cutoffs. For each origin, the prepared datasets contained only the historical and business inputs that would have been available at that point. Profiling, candidate eligibility, chronological evaluation, selection, refitting, post-processing, allocation, and output publication were rerun, and each origin's outputs were stored for later analysis.

Those stored results were then compared with demand observed after each origin across several horizons. I used this as a retrospectively reconstructed rolling-origin evaluation of the complete workflow, recreating the point-in-time decisions that the workflow would have made at each historical cutoff.

### 8.2 Output Perspectives, Matching, and Coverage

The study assessed three output perspectives:

- the aggregate forecasting-engine output before lower-level allocation
- the final business-facing output after allocation and post-processing
- an existing business forecast used as a comparison reference

Source-level scorecards used the evaluable rows available for each source. Direct source-to-source comparisons used only matched observations with the same historical origin, series, and target month. Forecast-actual pairs that were missing were excluded rather than imputed, and percentage metrics were calculated only where their denominators were valid.

The evaluation recorded source and horizon coverage alongside accuracy. Series coverage and evaluated actual-volume coverage were kept separate: one describes how broadly the criterion was met across individual series, while the other describes how much realized business volume those series represented. Nearer horizons naturally had broader origin coverage. Longer-horizon findings were treated as directional because fewer historical origins had corresponding realized actuals.

The evaluation used complementary lenses rather than relying on one portfolio score:

| Evaluation lens | Purpose |
| --- | --- |
| Portfolio-weighted error | Represent the effect of forecast error on higher-volume series |
| Directional bias | Identify persistent over- or under-forecasting |
| Series-level accuracy coverage | Show how broadly an accuracy criterion was met across individual series |
| Evaluated actual-volume coverage | Show how much realized portfolio volume was represented by series meeting the same criterion |
| Aggregate versus final output | Quantify the net difference associated with allocation and post-processing on matched observations |
| Demand-profile and segment views | Identify where the workflow provided stronger evidence and where more review was required |
| Horizon views | Distinguish near-term evidence from longer planning extrapolation |
| Forecast revision stability | Measure how forecasts for the same future target changed across successive origins |
| High-impact exception analysis | Prioritize review using absolute miss, volume, bias, downstream gaps, instability, and business context |

I treated aggregate-versus-final output gaps as diagnostic signals rather than causal decomposition. A net difference between them could reflect allocation shares, record coverage, filters, maintained business rules, or other post-processing, not allocation quality alone. The final-output comparison was therefore evidence about the complete published system, not an independently isolated estimate of allocation lift.

### 8.3 Stability, Exceptions, and Cause Routing

Planning stability was assessed by comparing forecasts for the same future target across successive origins. This measured forecast revision behavior rather than realized forecast accuracy.

High-impact review was not based on percentage error alone. The study combined absolute miss, directional bias, downstream output gaps, forecast-revision instability, and commercial materiality into a compound review register. These signals prioritized investigation but did not establish root cause.

The final business-facing forecast was a system output, not only a raw model prediction. Review therefore needed to route exceptions across upstream data quality or freshness, model fit, business rules and allocation inputs, missing business context, and commercial materiality. This supported lower-touch baseline use where evidence was strongest and focused analyst review for sparse, unstable, recent, event-driven, allocation-sensitive, or commercially important cases.

This was a completed validation study that I ran separately from the user-triggered monthly workflow. The same framework could be converted into recurring monitoring, but in this engagement I used it as a post-run analytical evaluation. All client-specific values, thresholds, counts, segment findings, and comparative results remain private.

## 9. Operational Validity and Business Context

### 9.1 Negative Forecast Behavior

Some statistical candidates could extrapolate below zero over a long horizon. The workflow checked the top-ranked candidate against a configured validity tolerance. If it failed, the system searched the recent-period ranking for the next acceptable candidate and recorded that a substitution occurred. Final business-facing quantities were constrained to valid non-negative values.

This preserved a model-backed alternative rather than relying only on pointwise clipping. Any fallback that changes the published method creates a lineage requirement: the method identifier, evaluation metrics, and substitution indicator should be verified as one consistent published record.

### 9.2 Inactive Histories

Apparently inactive series were excluded from normal model competition and routed to separate handling. In the public case study, I describe this at the routing level rather than presenting every inactive-series publishing outcome as a uniform zero-forecast rule.

Authoritative lifecycle status is preferable to inferring inactivity only from trailing zero demand.

### 9.3 Maintained Future Context

Selected lifecycle information represented future changes that historical forecasting models could not infer. It was applied after model generation and before detailed publication. Exact business-event types, parameters, curves, and thresholds are omitted.

## 10. Transparency and Output Contracts

Transparency was implemented through data products, not only documentation. The workflow generated a portfolio-level summary containing the operational profile, published model and family, recent-period evaluation context, and substitution indicator for each modeled series. A hardened output contract should also verify that every reported metric belongs to the method actually published.

This supported four analyst questions:

1. What type of demand behavior was identified?
2. Which forecasting families were eligible?
3. Which method was selected and on what evidence?
4. Did an operational rule change the initially ranked result?

The aggregate and detailed forecasts were combined with actuals, planning information, and comparison forecasts in monthly, annual, and BI-oriented datasets. These outputs formed a one-directional, file-based handoff contract for Power BI, analyst exports, and the Streamlit prototypes.

MLflow recorded nested experiment runs, parameters, metrics, configuration snapshots, environment information, forecasts, and model artifacts. The public write-up treats this as experiment and artifact tracking for the staged workflow, while deployment as one registered end-to-end model artifact remains outside the described scope.

## 11. Review Interface and Planning Governance

The Streamlit prototypes consumed prepared outputs independently from the forecasting engine. The retained implementation supported authenticated access, filtering, comparisons, drill-down, visualization, pivot-style review, and downloads. It did not retrain models or write forecast changes back into the pipeline.

Earlier exploration considered a richer human-in-the-loop workflow in which users could record context and potentially adjust a final forecast. That direction revealed governance requirements outside the application code:

- a common definition of the planning workflow
- ownership of source preparation and master data
- rules for when an analyst may override a baseline
- approval responsibilities and audit history
- deployment, authentication, support, and maintenance ownership
- a decision about how contextual information becomes a reusable data asset

For the delivered workflow, the retained Streamlit application stayed in the review-prototype path. Power BI and Excel-compatible outputs remained the practical handoff route. The initial Power BI work was transferred to an internal analyst for continued development.

## 12. Structured Context as a Future Data Product

Forecast adjustments made in offline files preserve a number but often lose the reasoning that produced it. A more valuable future workflow would capture structured explanations for anomalies and expected events, including:

- what changed and when
- whether the explanation concerns history or the future
- the expected duration and direction of the effect
- the source, author, reviewer, and approval status
- whether the information affects the current plan only or can become a future feature

With suitable governance, this event history could support auditability, exception review, future feature engineering, and better explanations of model behavior. I treated it as a recommended next step beyond the delivered workflow.

## 13. Execution and Engineering Boundaries

The recurring engine used staged Python scripts rather than relying only on independent notebooks. Its stages covered preparation, profiling, cross-validation and tuning, recent holdout selection, full-history forecasting, post-processing, allocation, and output construction.

Configuration separated local and managed execution settings. Intermediate artifacts made long-running stages inspectable and allowed selected stages to be rerun during development and investigation.

The engagement focused on workflow implementation, structural and data-quality validation, chronological model validation, and retrospective forecast evaluation. A production hardening phase would add unit tests, integration tests, data-contract tests, CI/CD, and release controls.

I keep the public scope limited to the workflow that was implemented. I do not present it as:

- scheduled orchestration or automatic source ingestion
- online inference or real-time model serving
- a fully deployed Streamlit application
- unit or integration test coverage, or CI/CD, within the engagement scope
- an autonomous approval and planning workflow

## 14. Recommended Improvements

1. **Govern source interfaces.** Introduce monitored source-system ingestion when approved access, schemas, and ownership are available.
2. **Operationalize performance monitoring.** Convert the completed retrospective validation framework into a recurring process that updates horizon, profile, bias, downstream-effect, and exception views as new realized sales volume becomes available.
3. **Reconcile hierarchy outputs.** Fail the run when detailed results do not conserve the aggregate forecast.
4. **Align published metrics and lineage.** Ensure substitution changes propagate to every downstream model and metric field.
5. **Add operational portfolio monitoring.** Track data exceptions, profile migration, selected-model changes, substitutions, allocation gaps, and output freshness between cycles.
6. **Separate forecast horizons.** Label evidence-supported horizons separately from longer planning extrapolation.
7. **Formalize tests and releases.** Add unit, integration, and data-contract tests; define acceptance criteria, release candidates, rollback expectations, and enhancement cutoffs.
8. **Define planning governance before writeback.** Agree adjustment, approval, audit, and context-capture responsibilities before implementing an editable review application.

## 15. References

- Croston, J. D. (1972). *Forecasting and Stock Control for Intermittent Demands.*
- Syntetos, A. A., & Boylan, J. E. (2005). *The accuracy of intermittent demand estimates.*
- Syntetos, A. A., Boylan, J. E., & Croston, J. D. (2005). *On the categorization of demand patterns.*
- Teunter, R. H., Syntetos, A. A., & Babai, Z. (2011). *Intermittent demand: linking forecasting to inventory obsolescence.*
- Hyndman, R. J., & Athanasopoulos, G. *Forecasting: Principles and Practice* (3rd ed.).
- Petropoulos, F., et al. (2022). *Forecasting: theory and practice.*
