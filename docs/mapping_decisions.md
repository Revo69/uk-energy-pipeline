# Mapping Decisions & Known Limitations

This document records assumptions, edge cases, and authoritative source choices
made during data mapping. Undocumented mappings are technical debt.

---

## BM Unit → Power Plant mapping

**Sources used (in priority order):**
1. OSUKED Power Station Dictionary (`seeds/bm_unit_fuel_type_map.csv`)
2. Elexon reference API (`/reference/bmunits/all`)
3. National Grid TEC Register

**Known issues:**
- Some BM Units are retired but still appear in historical data — flagged with `is_active = false`
- One physical station can have multiple BM Units (e.g. Drax has 6 units: `T_DRAXX-1` through `T_DRAXX-6`)
- Non-BM participants (STOR, demand response) use different ID prefixes — not all map to a power plant

---

## Settlement periods

- Standard day: 48 periods × 30 minutes
- Clock change to BST (spring forward): **46 periods**
- Clock change to GMT (fall back): **50 periods**
- Edge case: period 0 and period 49+ should be treated as anomalies, not data errors

---

## BOALF vs BOD discrepancies

BOALF (accepted actions) and BOD (submitted offers) will not always align. Known reasons:
- Late submissions: BOD submitted after gate closure is not reflected in BOALF
- Settlement reruns: historical BOALF may be revised; BOD is immutable
- Non-BM actions: some BOALF entries have no corresponding BOD

**Decision:** discrepancies are logged to `int_discrepancy_log`, not silently dropped.
Downstream users should be aware that ~2–5% of records may show no BOD match.

---

## ETYS Zone mapping

Substations are mapped to ETYS zones via the TEC Register.
Some substations appear in multiple zones during network reconfiguration periods.
**Decision:** use the zone active at the settlement date.

---

## Data quality thresholds

| Check | Threshold | Action |
|---|---|---|
| Price spike (BOD offer) | > £9,999/MWh | Flag, do not drop |
| Negative MW volume | Any | Flag as potential meter error |
| Missing settlement periods | > 3 consecutive | Alert |
| Duplicate timestamps | Any | Deduplicate, keep latest |
