
# SLO Dashboard Translation — Goal Summary

>Migrate SLOs in Dynatrace that don’t follow best practices (those tied to internal components rather than business-outcome SLIs) into Grail-backed dashboards.

## Dashboard Requirements

These dashboards should:

1. **Replicate the SLO view**
    - Show the same key info as Dynatrace’s native SLO screen:
      - Name
      - Target %
      - Warning %
      - Evaluation Period
      - Status %
      - Error Budget remaining
      - State
2. **Support multiple SLI types**
    - Availability (% successful requests)
    - Performance (p90 latency ≤ threshold)
    - Throughput (requests per minute ≥ threshold)
3. **Incorporate SRE metrics**
    - Calculate and display:
      - Error Budget used
      - Error Budget remaining
      - Burn Rate (1h & 24h)
      - Optionally project days to exhaustion
4. **Be service-flexible**
    - Work for one service or a multi-service list
    - Configurable targets/warnings via dashboard variables or a static config table
5. **Provide multiple views**
    - Table view for high-level tracking across services/SLOs
    - Trend view (timeseries) for SLI performance over time
    - Single-value tiles for quick status KPIs
6. **Use clean, reusable DQL**
    - Queries should be modular, parameterized, and easily adapted between SLI types
    - Avoid non-existent or deprecated functions

---

## Usage Note

If you start future prompts with something like:

> Continue building the Grail dashboard framework that migrates non-best-practice SLOs from Dynatrace into custom dashboards, including availability/performance/throughput SLIs, error budget and burn rate calculations, and flexible views for single or multiple services.

…then I’ll know exactly to stay within this scope and not reinvent the wheel each time.

---

## Optional Enhancement

Do you want me to also add **burn-rate alerting thresholds** (Google SRE MWMBR defaults) into that goal so they’re always included? That would make the dashboards more actionable.
