Things that have been removed, or notes to review about the [track-]

# Extra Cypher not needed on page:

```cypher
// ============================================================
// VERIFICATION QUERIES (run after loading)
// ============================================================

// 1. Node counts — expect:
//    Batch=20, ProcessOrder=10, PurchaseOrder=3,
//    DeliveryLine=5, Delivery=5
MATCH (n) RETURN labels(n) AS label, count(n) AS count
ORDER BY count DESC;

// 2. Forward trace from suspect batch — expect 5 deliveries
MATCH path = (s:Batch {batchNr:'1100_1101475_2899901'})
  -[:TOTAL_ISSUED|TOTAL_RECEIVED|TOTAL_BATCH_TRANSFER|
     TOTAL_PLANT_TRANSFER|TOTAL_MAPPED_DELIVERY|
     TOTAL_ISSUE_DELIVERY*1..100]->
  (d:Delivery)
WITH d, length(path) AS pathLength,
     [r IN relationships(path) | r.partInReceived] AS parts
RETURN d.deliveryNr          AS deliveryNr,
       d.soldToCustomer      AS customer,
       d.shipToCustomer      AS shipTo,
       pathLength,
       round(reduce(p=1.0, x IN parts | p*x) * 10000) / 100
         AS contaminationPct
ORDER BY contaminationPct DESC;

// 3. Backward trace from DEL-2024-001 — expect 6 leaf batches:
//    B01(API), B05(MCC), B07(HPC), B10(Coating), B15(Foil), B18(Cartons)
MATCH path = (d:Delivery {deliveryNr:'DEL-2024-001'})
  <-[:TOTAL_ISSUED|TOTAL_RECEIVED|TOTAL_BATCH_TRANSFER|
      TOTAL_PLANT_TRANSFER|TOTAL_MAPPED_DELIVERY|
      TOTAL_ISSUE_DELIVERY*1..100]-
  (raw:Batch)
WHERE NOT (raw)<-[:TOTAL_RECEIVED]-()
RETURN DISTINCT raw.batchNr AS rawBatch, raw.materialDesc AS description,
       raw.plant AS plant, length(path) AS depth
ORDER BY depth;

// 4. Regulatory KPI summary — expect:
//    totalDeliveries=5, uniqueCustomers=4, maxLevel=16
MATCH path = (s:Batch {batchNr:'1100_1101475_2899901'})
  -[:TOTAL_ISSUED|TOTAL_RECEIVED|TOTAL_BATCH_TRANSFER|
     TOTAL_PLANT_TRANSFER|TOTAL_MAPPED_DELIVERY|
     TOTAL_ISSUE_DELIVERY*1..100]->(n)
WITH n, length(path) AS depth, labels(n) AS lbl
RETURN
  count(CASE WHEN 'Delivery' IN lbl THEN 1 END) AS totalDeliveries,
  count(DISTINCT CASE WHEN 'Delivery' IN lbl THEN n.soldToCustomer END)
    AS uniqueCustomers,
  count(DISTINCT CASE WHEN 'Batch' IN lbl THEN n.batchNr END)
    AS deliveryBatches,
  max(depth) AS maxLevel;
```





== Cypher Queries

Example queries:

* *Path of a serialized unit*: Start from a `SerializedUnit` by `serial_number`, traverse `SHIPPED_IN` to `Shipment`, then `FROM`/`TO` to `Location`, and optionally `OPERATED_BY` to `Party`, ordered by time.
* *All locations that received a batch*: Start from `Batch` by `batch_number`, traverse to `Shipment` and then `TO`->`Location`, return distinct locations (and optionally parties).
* *Full chain of custody for a unit*: Same path as the first query, returning a chronological list of locations and parties.

One important design decision to note: `partInReceived = 1.0` is set on all `TOTAL_RECEIVED` relationships. This is necessary because the forward trace in Query 1 uses an unfiltered `reduce()` that would propagate `null` if any relationship in the path lacks the property. The packaging steps (blistering, cartoning) also use `partInReceived = 1.0` for the tablet batch since contamination is tracked as a drug content fraction, not a mass fraction. The packaging material relationships carry `0.0` to reflect that foil and cartons contribute no suspect API.