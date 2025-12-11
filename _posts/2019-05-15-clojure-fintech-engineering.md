---
layout: post
title: "Engineering at our Fintech: A Deep Dive into Our Clojure Stack"
date: 2019-05-15
categories: clojure engineering fintech
---

Welcome to our engineering blog series! At our company, we rely heavily on Clojure to build robust, high-performance financial systems. In this series, we're pulling back the curtain on three key areas of our codebase: how we validate data, how we process large datasets, and how we handle complex transaction states using XTDB.

---

## Part 1: Guardrails for Business Data with `clojure.spec`

Financial data allows for zero margin of error. To ensure the integrity of the data flowing through our systems—from user inputs to API payloads—we leverage `clojure.spec.alpha`.

In `src/biz/spec/core.clj`, we define a set of reusable, composable specs that act as the first line of defense.

### Regex-Based Validation
For standard formats like emails and phone numbers, we combine strict regular expressions with spec's predicate logic. This allows us to fail fast when data is malformed.

```clojure
;; Leveraging robust regex patterns for validation
(def email-regex #"[A-Za-z0-9!#$%&'*+/=?^_`{|}~-]+(?:\.[A-Za-z0-9!#$%&'*+/=?^_`{|}~-]+)*@(?:[a-z0-9](?:[a-z0-9-]*[a-z0-9])?\.)+[a-z0-9](?:[a-z0-9-]*[a-z0-9])?")

;; Defining a spec that checks both type and format
(s/def ::email-type (s/and string? #(re-matches email-regex %)))
```

### Collection Constraints
Handling collections often requires more than just checking types. We need to ensure uniqueness and minimum counts. We make heavy use of `s/coll-of` to enforce these constraints declaratively.

```clojure
;; Ensuring a vector of unique UUIDs with at least one element
(s/def ::uuid-vec (s/coll-of uuid? :min-count 1 :distinct true :into []))
```

### Composite Rules
Some fields require complex validation logic. For example, a description field might need to be a string, non-empty, *and* within a certain character limit. `s/and` lets us compose these predicates elegantly.

```clojure
(s/def ::description
  (s/and string?
         #(not (clojure.string/blank? %))
         #(<= (count %) 2000)))
```

By centralizing these definitions, we utilize them across API boundaries and internal logic, ensuring a consistent understanding of "valid data" throughout the application.

---

## Part 2: High-Performance Data Wrangler with `tech.v3.dataset`

When dealing with large volumes of transaction data for analytics (likely targeting OLAP stores like ClickHouse), standard Clojure sequences can sometimes be too slow or memory-intensive. We utilize the `tech.v3.dataset` (TMD) library to handle data in efficient, columnar structures.

In `src/app/chdb/dataset.clj`, we've built a pipeline to ingest, clean, and prepare data.

### Robust Type Coercion
Data from external providers often comes in messier formats than we'd like—strings that should be integers, or booleans represented as numbers. We define custom coercion functions to handle these edge cases gracefully.

```clojure
(defn coerce-double-to-int [n & [opts]]
  (cond-> n
    (string? n) (#(if (string/blank? %) 0 (try (Double/parseDouble %) (catch NumberFormatException _ 0))))
    (:cents? opts) (* 100)
    :always (#(if (:bigint? opts) (bigint %) (int %)))))
```

### Declarative Schema
We define our target schema explicitly using a parser function `get-parser-fn`. This maps input keys to their desired types and coercion logic, effectively compiling a raw stream of maps into a strictly typed dataset.

```clojure
(-> {"amount" [:int64 (fn [d] (coerce-double-to-int d {:cents? true :bigint? true}))]
     "card_last_four" [:uint16 coerce-to-int]
     "response_code" [:uint16 coerce-to-int]}
    ...)
```

### The Prep Pipeline
The `prep-docs` function represents our ETL pipeline. It takes raw documents, parses them into a dataset, merges them with a rigorous schema (likely loaded from `ckh.edn`), handles missing values, and sanitizes strings—all in a functional chain.

```clojure
(defn prep-docs [tk docs]
  (ds/bind-> (ds/->dataset docs {:parser-fn (get-parser-fn tk)}) d
    (ds/remove-columns (get-rm-column-names tk))
    (ds/update d #(merge (get-ckh-ds tk) %))
    ;; Mass update missing values for specific types
    (ds/update (get-uuid-column-names tk) ds/replace-missing-value "00000000-0000-0000-0000-000000000000")
    (ds/update (get-datetime-column-names tk) ds/replace-missing-value "1970-01-01T00:00Z")))
```

This approach allows us to process high-throughput streams of financial events with the performance of Java primitives while keeping the code readable and idiomatic to Clojure.

---

## Part 3: Event-Driven State Machines with XTDB Transaction Functions

Perhaps the most critical part of our system is how we maintain the state of financial transactions. We use XTDB (formerly Crux) not just as a store, but as a transactional logic engine.

In `src/app/comb/node.clj`, we implement "In-Memory Event Replay" capabilities by using **Transaction Functions** (`:xt/fn`).

### Logic Inside the Transaction
Instead of reading data, calculating a change, and writing it back (which is prone to race conditions), we submit the *logic* to the database. The database executes this logic serially, ensuring strong consistency.

We define a main dispatcher function `:provider-to-app-tx` (Provider to Application Transaction) that inspects an incoming event and decides how it affects the ledger.

```clojure
;; The main entry point for event processing
:xt/fn '(fn [ctx {:keys [token type state ...] :as event} vt il]
          (let [db (xtdb.api/db ctx)
                ;; Look up related entities INSIDE the transaction
                entity (cond-> {}
                         (seq preceding_related_transaction_token)
                         (merge (xtdb.api/entity db preceding_related_transaction_token)))]
            (case type
              "authorization" ... ; Dispatch to create
              "authorization.clearing" ... ; Dispatch to update
              "authorization.reversal" ... ; Dispatch to reverse
              )))
```

### Stateful Updates
The update logic (`:provider-to-app-tx-update`) is a pure function that takes the current database context and the event. It can handle complex business rules like "incremental authorization" where amounts are added to an existing total, or reversals where amounts are deducted.

```clojure
:xt/fn '(fn [ctx entity {:keys [token type state amount] :as event} vt il]
          (when (seq entity)
            (cond
              :normal
              [[::xt/put (assoc entity
                                :amount (+ (:amount entity) amount) ;; Increment logic
                                :provider-status state)]]
              ;; Handling "Corner" cases like partial settlements
              :corner 
              (cond
                 (< amount (:amount entity))
                 [[::xt/fn :provider-to-app-tx-refund ...]]))))
```

### Infinite Replayability
Because our logic is stored and versioned as functions, and our inputs are immutable events, we can technically replay the entire history of events to reach the current state. If we need to fix a bug in how a "reversal" is calculated, we can correct the transaction function and replay the event stream to repair the ledger state deterministically.

This architecture gives us the consistency of an ACID database with the flexibility and auditability of an event-sourced system.
