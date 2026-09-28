# Autonomous Production Scheduling & Closed-Loop Line Orchestration Engine

[![n8n](https://img.shields.io/badge/Orchestrator-n8n-EA4B71?style=flat-square&logo=n8n)](https://n8n.io/)
[![JavaScript](https://img.shields.io/badge/Core-JavaScript%20ES6+-F7DF1E?style=flat-square&logo=javascript)](https://developer.mozilla.org/)
[![LINE Messaging API](https://img.shields.io/badge/ChatOps-LINE%20Bot%20API-00C300?style=flat-square&logo=line)](https://developers.line.biz/)
[![Status](https://img.shields.io/badge/Status-Production%20Active-success?style=flat-square)]()

An event-driven, closed-loop **Manufacturing Execution System (MES)** and dynamic scheduling engine. The system automates end-to-end factory floor operations: from multi-source data ingestion, 3-dimensional multi-criteria machine selection, resource-constrained queue planning (≤4 workers), shopfloor ChatOps via LINE Webhook, to automated visual telemetry reporting.

---

## 📌 Architectural Overview

The system operates across 4 synchronized pipelines forming a real-time autonomous feedback loop:

```mermaid
flowchart TD
    subgraph DataIngestion["1. Ingestion & Multi-Criteria Selection (production-scheduler-main)"]
        A1[Google Drive: Monthly PLC Energy & Breakdown Logs] --> B1[Calculate MachineStats: Repair Time & Active kW]
        A2[Daily Ingestion: FM-DSM-001 Orders] --> B2[3D Machine Selection Algorithm]
        B1 & B2 --> C1[Constraint Scheduler: Max 4 Workers & Slot Split-Fill]
        C1 --> D1[(Master Sheet: Planning & Work Orders)]
    end

    subgraph ShiftMonitoring["2. Visual Telemetry & Shift Reporting (shift-telemetry-reporter)"]
        D1 & E1[Loss Report: Actual Output] --> F1[Progress Calculation & Shift Summary]
        F1 --> G1[Headless Browser Image Rendering Engine]
        G1 --> H1[LINE API: High-Res Gantt Push & Shift Handoff]
    end

    subgraph DynamicFeedback["3. Closed-Loop Dynamic Re-planning (dynamic-replan-engine)"]
        E1 --> I1[Nightly Audit: Actual Defect vs Target Backlog]
        I1 --> J1[Re-plan Remaining Backlog & Split-Fill Idle Capacity]
        J1 --> D1
    end

    subgraph ShopfloorChatOps["4. Interactive Shopfloor ChatOps (shopfloor-chatops)"]
        K1[LINE Webhook: HMAC-SHA256 Validated] --> L1[Command Parser & Finite State TTL Guard]
        L1 -->|Move / Delay / Adjust / Shut| M1[Scoped Machine Reschedule]
        M1 --> N1[Update Master DB & Notify Shopfloor]
    end
```

---

## ⚙️ Core Technical Highlights

### 1. Multi-Criteria Decision Analysis (3D Machine Optimization Engine)
Instead of static machine assignment, orders are evaluated across a 3-dimensional ratio-to-best matrix:
* **Defect Rate (40%):** Evaluated from real production history ($100 - \text{Good}\%$).
* **Mean Time to Repair (MTTR) (40%):** Computed dynamically from historical breakdown logs ($\Sigma \text{Hours} / \Sigma \text{Occurrences}$).
* **Active Energy Demand (20%):** Derived from PLC telemetry logs where operating demand $> 5\text{ kW}$.

$$\text{Composite Score} = 0.40 \cdot S_{\text{Defect}} + 0.40 \cdot S_{\text{Repair}} + 0.20 \cdot S_{\text{Energy}}$$

* **Banding & Load Balancing:** Machines scoring within $15\%$ of the top score ($\ge \text{Top} \times 0.85$) are grouped into a candidate band, routing the job to the machine with the lowest active workload to eliminate bottlenecks.

---

### 2. Multi-Constraint Dynamic Event-Driven Scheduling
* **Factory Workforce Constraint:** The factory operates with a strict capacity limit of **at most 4 workers active concurrently across all machines** ($\text{Total Workers} \le 4$). Single-product jobs require 1 worker; heterogeneous multi-product jobs require 2 workers.
* **Non-Blocking Queue Simulation:** When a job completes on machine $M_i$, worker capacity is immediately released back to the global pool, triggering the next eligible job in the queue.
* **Split-Fill Optimization:** If idle capacity ($\ge 24\text{ hours}$) is detected alongside a long-running batch ($\ge 24\text{ hours}$), the engine splits the order and loads it across parallel idle machines, driving labor utilization toward 100%.

---

### 3. State-Machine ChatOps with HMAC-SHA256 Verification
* **Cryptographic Security:** Every webhook request is verified using HMAC-SHA256 against `X-Line-Signature`.
* **State TTL Protection:** State changes (delay delivery, quantity adjustments, job transfers, machine emergency shutdown) utilize a memory-backed pending state with a **5-minute expiration timer (TTL)** requiring explicit user confirmation.
* **Dynamic Scoped Re-plan:** In the event of machine failure (`ย้ายเครื่อง / เครื่องเสีย`), the engine recalculates only the affected machine's downstream jobs, automatically redirecting them to available machines while keeping unaffected lines running undisturbed.

---

### 4. Automated Visual Telemetry Pipeline
* Evaluates real-time production output vs. target deadlines across the entire production week.
* Dynamically constructs custom HTML/CSS responsive Gantt charts with defect indicators, schedule slippage overlays, and shift handoff logs.
* Dispatches HTML to an internal headless rendering service, uploads artifacts to cloud storage, and pushes high-resolution images and shift briefings directly to operational staff at 16:00 daily.

---

## 📂 Workflow Directory

| Workflow File | Role | Triggers | Key Responsibilities |
| :--- | :--- | :--- | :--- |
| `production-scheduler-main.json` | Master Scheduler | Midnight Schedule / Monthly Cron / Manual | Multi-criteria machine selection, labor-constrained scheduling, and initial work order dispatch. |
| `dynamic-replan-engine.json` | Closed-Loop Dynamic Re-plan | Daily 06:00 Schedule / Manual | Nightly backlog audit against actual output, dynamic remaining-quantity replanning, and split-fill optimization. |
| `shopfloor-chatops.json` | Real-time Shopfloor ChatOps | LINE Webhook (POST) | HMAC-SHA256 verification, 5-min TTL state machine, and scoped machine failover/rescheduling. |
| `shift-telemetry-reporter.json` | Telemetry & Visual Shift Reports | Daily 16:00 Schedule / Manual | Headless HTML-to-image Gantt rendering, cloud artifact storage, and shift briefings pushed via LINE. |

---

## 🚀 Setup & Deployment

### Prerequisites
1. **n8n Instance** (Self-hosted or Cloud v1.0+)
2. **Google Workspace Service Account / OAuth2** (Drive & Sheets scope)
3. **LINE Messaging API Developer Channel**

### Import Workflows
1. Clone this repository:
   ```bash
   git clone [https://github.com/](https://github.com/)Panutle/autonomous-factory-mes.git
   ```
2. In your n8n interface, select **Workflows** > **Import from File**.
3. Import the 4 JSON files from the `workflows/` directory.
4. Link your Google Sheets and LINE API credentials within each respective node.#
