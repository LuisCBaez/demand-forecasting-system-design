# Architecture and Workflows

These Mermaid definitions are the canonical diagrams supporting the [case-study README](README.md). They are independently created, implementation-informed abstractions. They do not reproduce client architecture, systems, identifiers, schemas, thresholds, data, or results. The README uses synchronized PNG presentation views; their deterministic HTML and CSS sources are kept in [`diagrams/render/`](diagrams/render/).

## 1. Evolution from Pilot to Monthly Decision Support

The first phase established the forecasting baseline and exposed the operating constraints. The second phase converted those findings into a transparent, staged workflow.

```mermaid
flowchart LR
    A["Exploratory pilot<br/>Consolidate fragmented data<br/>EDA and initial models<br/>Manual forecast review"]
    B["Engineering findings<br/>Sparse and varied demand<br/>Repeated data preparation<br/>Selection needed traceability<br/>Review needed shared context"]
    C["Recurring forecasting engine<br/>Profile-aware candidate pools<br/>Chronological evaluation<br/>Full-history refit<br/>MLflow tracking"]
    D["Monthly decision support<br/>User-triggered Databricks run<br/>Standardized analytical outputs<br/>Power BI and analyst review"]

    A -->|Learn| B
    B -->|Redesign| C
    C -->|Publish| D
```

This progression is more important than a simple comparison of algorithms. The exploratory work identified data, evaluation, transparency, and operating-model requirements that were then addressed explicitly in the recurring design.

## 2. End-to-End System Architecture

The architecture makes the human-owned boundaries, automated forecasting stages, and one-directional review contract explicit.

```mermaid
flowchart TD
    subgraph HumanInput[Human-owned source preparation]
        A[Gather monthly business sources]
        B[Check completeness and readiness]
    end

    subgraph Engine[User-triggered Databricks workflow]
        C[Validate, map, and standardize]
        D[Publish reusable analytical datasets]
        E[Build stable aggregate histories]
        F[Profile demand and route candidate pools]
        G[Create chronological training and holdout datasets]
        H[Rolling validation and global ML tuning]
        I[Shortlist and recent-period selection]
        J[Refit on available history]
        K[Operational validity and business context]
        L[Allocate to reporting levels]
        M[Publish forecast, comparison, monthly,<br/>annual, and BI-ready datasets]
    end

    subgraph Evidence[Experiment evidence]
        N[MLflow runs, parameters,<br/>metrics, and artifacts]
    end

    subgraph Review[Human-owned review and planning]
        O[Power BI-ready datasets<br/>and analyst exports]
        P[Planning interpretation and decisions]
        Q[Streamlit review prototypes<br/>separate from operating handoff]
    end

    A --> B --> C
    C --> D --> E
    E --> F
    E --> G
    F --> H
    G --> H
    H --> I --> J --> K --> L --> M
    H -.-> N
    I -.-> N
    J -.-> N
    M --> O --> P
    M -. prototype path .-> Q
```

Automation began only after the source files were prepared. The pipeline then generated both modeling datasets and reusable analyst-facing information. Power BI and exports remained the adopted handoff; Streamlit represented implemented review prototypes rather than a deployed planning application.

## 3. Transparent Model-Selection Flow

The profile filtered eligible methods. Chronological evidence selected the winner. The published context allowed analysts to understand the result without exposing proprietary implementation details.

```mermaid
flowchart TD
    A[Latest complete series history] --> B[Demand characteristics]
    B --> C{Operational profile}

    C -->|Regular| D[Classical methods<br/>baselines<br/>eligible global ML]
    C -->|Intermittent| E[Croston-family methods<br/>ADIDA and TSB<br/>baselines]
    C -->|Lumpy| F[ADIDA, IMAPA, and TSB<br/>baselines]
    C -->|Short history| G[Restricted baselines]
    C -->|Apparently inactive| H[Separate operational handling]

    D --> I[Rolling-origin validation]
    E --> I
    F --> I
    G --> I

    I --> J[Candidate shortlist]
    J --> K[Recent chronological evaluation]
    K --> L[Rank by MAE + absolute Bias]
    L --> M[Refit candidates on available history]
    M --> N{Top candidate passes<br/>operational validity checks?}
    N -->|Yes| O[Use top-ranked method]
    N -->|No| P[Use next acceptable ranked method]
    O --> Q[Publish forecast and selection context]
    P --> Q

    Q --> R[Demand profile]
    Q --> S[Published model and family]
    Q --> T[Selection metrics<br/>must be realigned after substitution]
    Q --> U[Substitution indicator]
```

The operational profile used the latest complete history because its purpose was to route methods for the next forecast cycle. Training and recent-holdout datasets remained chronologically separated for model fitting and comparison; the holdout target was not used to fit the models. However, the full-history profile restricted candidate eligibility, so the holdout was not completely independent of routing. A hardened selection flow would build the routing profile from training-only history and recompute the operational profile after selection.

In the retrospective study, I reran the full pipeline—including profiling and candidate eligibility—at each historical origin using only inputs available at that cutoff. This reconstructed the completed workflow's decisions at each origin for evaluation.

## 4. Retrospective Portfolio Evaluation

The retrospective study was performed after the recurring workflow had been completed. It reused the complete pipeline at reconstructed point-in-time origins, stored each run's outputs, and evaluated them only when corresponding realized sales volume was available.

```mermaid
flowchart TD
    A["Origin-specific prepared inputs<br/>Only information available at the cutoff"]
    B["Reconstruct the full pipeline<br/>Profiling through final publication"]
    C["Store origin outputs<br/>Aggregate, final, and comparison forecasts"]
    D["Match to realized actuals<br/>Record source and horizon coverage"]
    E["Evaluate complementary views<br/>Error, bias, coverage, segments,<br/>downstream gaps, and revision stability"]
    F["Build compound exception register<br/>Prioritize investigation with business context"]
    G["Translate findings into<br/>operating-model recommendations"]

    A --> B --> C --> D --> E --> F --> G
```

Source-level scorecards used the valid rows available for each source. Direct comparisons used matched origin, series, and target-month observations; missing forecast-actual pairs were excluded rather than imputed. Longer-horizon results were interpreted more cautiously because fewer historical origins had corresponding realized actuals. I used this as a completed analytical study; the same framework could later be converted into recurring monitoring.
