


#Dataset

DATASET 1: Timetable / Train Schedule Data (timetable_master)

Features:

train_id, train_number, train_name
train_type (passenger / mail-express / freight / suburban)
priority_level (high/medium/low — freight vs premium passenger)
section_id (kaunse section se guzarta hai)
scheduled_departure_time, scheduled_arrival_time
frequency (daily/weekly/specific days)
direction (up/down line)
avg_speed, halt_duration_at_stations

Purpose: Operations-side demand — kis time kaunsa section "busy" hai, taaki disruption cost calculate ho sake.

DATASET 2: Asset Health Data (asset_health_records)

Features:

asset_id, asset_type (track/bridge/signal/OHE — jaisa humne decide kiya, rolling stock exclude)
section_id, location_km_marker
last_inspection_date, next_due_inspection_date
condition_score (0-100 ya categorical: Good/Fair/Poor/Critical)
degradation_rate (agar sensor/trend data ho)
inspection_type (ultrasonic, track geometry car, visual, etc.)
statutory_max_interval_days (safety-critical deadline — hard constraint)
failure_risk_flag (boolean/derived)

Purpose: Predictive maintenance layer ka core input — kab kis asset ko kaam chahiye.

DATASET 3: Block Requests Data (block_requests)

Features:

request_id, requesting_department (Engineering/S&T/Electrical)
asset_id (linked to asset_health_records)
section_id, sub-section/km_range
requested_start_time, requested_end_time, requested_duration
job_type (renewal/repair/inspection/servicing)
urgency_level (declared by requester)
linked_asset_condition_score (referenced at request time)
request_status (pending/approved/declined/modified) — historical ke liye
request_date_submitted

Purpose: Raw demand — jo departments maang rahe hain, existing process se aata hai (manual). Yeh training data bhi hai (pattern seekhne ke liye — request vs. grant gap).

DATASET 4: Block Allocation / Execution History (block_allocations)

Features:

allocation_id, linked_request_id
section_id, actual_granted_start_time, actual_granted_end_time
actual_duration_vs_requested_duration (delta)
execution_status (completed/partially completed/cancelled/overrun)
traffic_impact_recorded (delay minutes caused, trains affected — agar available ho)
reason_for_modification (agar request se different diya gaya)

Purpose: Ground truth — jo actually hua. Isse model seekhega ki historically kya trade-offs liye gaye, aur AI ke suggestions ko benchmark/validate karne ke liye use hoga.

DATASET 5: Resource / Crew & Machine Availability (resource_availability)

Features:

resource_id, resource_type (crew/tamper machine/ballast cleaner/OHE van, etc.)
department_owner
available_date, available_time_window
assigned_section (agar already booked)
capacity/count (kitne crew members, kitne machines)
skill_type (specific job types ke liye qualified)

Purpose: Hard execution constraint — bina isske AI theoretically-correct lekin practically impossible plan bana sakta hai.