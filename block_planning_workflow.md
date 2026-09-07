# AI-Powered Automatic Block Planning — Indian Railways
## Full Project Workflow

---

## 1. Problem Summary

Indian Railways needs to allocate "blocks" (track possession windows) to maintenance
departments (Engineering, Signal & Telecom, Electrical) for track, bridge, signal, and
OHE maintenance — while keeping maximum track availability for train operations.

This is a constrained optimization problem: balance maintenance demand against train
traffic disruption, subject to safety, resource, and non-overlap constraints.

**Scope:** Fixed infrastructure only — track, bridges, signals, OHE.
(Rolling stock maintenance — coach/wheel/AC servicing — is out of scope; that is
handled by depot-based systems, not track blocks.)

---

## 2. Input Datasets

### Dataset 1 — Timetable / Train Schedule (`timetable_master`)
- train_id, train_number, train_name
- train_type (passenger / mail-express / freight / suburban)
- priority_level (high / medium / low)
- section_id
- scheduled_departure_time, scheduled_arrival_time
- frequency (daily / weekly / specific days)
- direction (up / down line)
- avg_speed, halt_duration_at_stations

**Purpose:** Operations-side demand — which sections are busy at which times.

### Dataset 2 — Asset Health (`asset_health_records`)
- asset_id, asset_type (track / bridge / signal / OHE)
- section_id, location_km_marker
- last_inspection_date, next_due_inspection_date
- condition_score (0–100 or Good/Fair/Poor/Critical)
- degradation_rate
- inspection_type (ultrasonic, track geometry car, visual, etc.)
- statutory_max_interval_days (safety-critical hard deadline)
- failure_risk_flag

**Purpose:** Predictive maintenance signal — when an asset actually needs work.

### Dataset 3 — Block Requests (`block_requests`)
- request_id, requesting_department
- asset_id (linked to Dataset 2)
- section_id, km_range
- requested_start_time, requested_end_time, requested_duration
- job_type (renewal / repair / inspection / servicing)
- urgency_level (declared by requester)
- linked_asset_condition_score
- request_status (pending / approved / declined / modified)
- request_date_submitted

**Purpose:** Raw demand from departments — also used as training data (request vs. grant gap).

### Dataset 4 — Block Allocation / Execution History (`block_allocations`)
- allocation_id, linked_request_id
- section_id, actual_granted_start_time, actual_granted_end_time
- actual_duration_vs_requested_duration (delta)
- execution_status (completed / partially completed / cancelled / overrun)
- traffic_impact_recorded (delay minutes, trains affected)
- reason_for_modification

**Purpose:** Ground truth — what actually happened. Used to validate/benchmark AI output.

### Dataset 5 — Resource / Crew & Machine Availability (`resource_availability`)
- resource_id, resource_type (crew / tamper / ballast cleaner / OHE van, etc.)
- department_owner
- available_date, available_time_window
- assigned_section (if already booked)
- capacity/count
- skill_type

**Purpose:** Hard execution constraint — without this, plans can be theoretically valid
but practically impossible to execute.

---

## 3. User Input (New Block Request Submission)

When a planner/engineer submits a new maintenance request via the UI:

- Department name
- Asset ID / section / km-range (from asset master)
- Job type & description
- Requested duration (min–max range, not fixed — flexibility helps the optimizer)
- Urgency justification (auto-suggested from asset health score)
- Preferred time window (optional, soft preference not a hard constraint)
- Required resources (crew type, machine — auto-checked against Dataset 5)

This creates a new row in `block_requests`, which enters the prediction + optimization pipeline.

---

## 4. Core Workflow Pipeline

### Stage 1 — Data Layer to Prediction
```
5 input datasets  →  Model training (offline)  →  Live prediction
```
- Model training happens offline, on historical data.
- Two models are trained:
  - **Asset risk / urgency model** (from Dataset 2) — predicts when maintenance is
    actually needed, even before a formal request arrives.
  - **Disruption-cost model** (from Datasets 1 + 4) — predicts how much traffic
    impact a given block window would cause, learned from historical patterns.
- Output: urgency scores + predicted disruption costs for each candidate job/window.

### Stage 2 — Optimization to Approval
```
Optimization engine (MILP/CP-SAT)
        ↓
Simulation & scoring (tests candidate plans)
        ↓
Recommended plan (ranked block windows)
        ↓
Explainable AI + human approval (planner reviews, can override)
        ↓
↻ outcome feeds back into the 5 datasets
```
- **Optimization engine:** Takes predicted scores + hard constraints (resource
  availability, non-overlap, safety deadlines) and produces candidate feasible schedules.
- **Simulation & scoring:** Runs top candidate plans through a network-impact
  simulation (delay propagation, congestion check) to validate before final selection.
- **Recommended plan:** Best-scoring candidate presented with ranked alternatives.
- **Explainable AI + approval:** Every recommendation comes with a human-readable
  reason (e.g. "chosen because failure risk is high and traffic in this window is
  historically lowest"). Final approval stays with a human (human-in-the-loop).
- **Feedback loop:** Whatever is approved/rejected/modified becomes new data in
  `block_allocations`, closing the loop so the model keeps improving.

---

## 5. Key Constraints Modeled

- **Resource exclusivity:** Same section/asset cannot have two overlapping blocks,
  even from different departments.
- **Non-overlap / capacity:** Two blocks cannot double-book the same time-slot on
  the same or conflicting section — this reduces available traffic-running windows.
- **Corridor-level conflict graph:** Adjacent sections sharing a junction/interlocking
  can conflict even if not directly blocked — model this as a conflict-neighbor set
  per section, not just same-section overlap.
- **Safety-critical deadlines:** Hard constraints, never violated regardless of
  optimization pressure (`statutory_max_interval_days`).
- **Resource availability:** Crew/machine capacity limits from Dataset 5.

---

## 6. Tech Stack

| Layer | Tools |
|---|---|
| Data & Backend | Python, PostgreSQL, FastAPI |
| Optimization | Google OR-Tools (CP-SAT) or PuLP + CBC solver |
| ML / Forecasting | scikit-learn, XGBoost/LightGBM, Prophet |
| Frontend | React + TypeScript, Tailwind CSS, Recharts/D3.js |
| Explainability | SHAP (for ML parts), rule-based explanation (for optimization parts) |
| Deployment | Docker, Render/Railway.app, Vercel, Postgres (Supabase/Neon) |

---

## 7. Build Phases

0. **Domain research & scoping** — understand block/possession terminology, define
   exact network scope (recommend: 1 division, 15–25 sections).
1. **Data layer** — build synthetic dataset generator for all 5 datasets.
2. **Core optimization engine** — MILP/CP-SAT model, start small (1 section, 1 day),
   scale to multi-section rolling horizon.
3. **Predictive/demand forecasting layer** — asset risk model + disruption-cost model.
4. **Dynamic / rolling re-planning** — fast re-optimization on disruption events.
5. **Backend API** — FastAPI endpoints for schedule, requests, re-optimization, what-if.
6. **Frontend dashboard** — Gantt-chart timeline, what-if slider, approve/override UI.
7. **Explainability layer** — human-readable reasoning per decision.
8. **Testing & validation** — constraint unit tests, scenario stress tests, domain expert review.
9. **Deployment** — containerize and deploy for demo/portfolio.

---

## 8. Known Gaps to Watch For

- Real IR operational data is restricted — plan for realistic synthetic data,
  disclose this transparently.
- Optimization must genuinely enforce hard constraints (safety, resources) —
  a pure ML wrapper without constraint-satisfaction will not hold up to scrutiny.
- System should support dynamic re-planning, not just a static one-shot solve,
  to handle real-world disruptions.
