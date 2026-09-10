(ns geology.store
  "SSoT for the ISCO-08 2114 survey support actor (geologists and
  geophysicists). Store is a protocol injected into the `geology.actor`
  StateGraph — `MemStore` is the default, deterministic, zero-dep
  backend; a Datomic/kotoba-server-backed implementation can be
  swapped in without touching the actor or governor (itonami actor
  pattern, per ADR-2607011000 / CLAUDE.md Actors section).

  Domain:

    survey-site — a registered geological/geophysical survey location
                  (:site-id, :name, :coordinates, :location)
    survey-data  — recorded seismic, core sample, or subsurface data
                   associated with a site (:data-id, :site-id, :type,
                   :description)
    equipment    — field survey equipment resources
                   (:equipment-id, :name, :type)
    record       — a committed survey operation under a site
                   (data analysis, report draft, anomaly flag,
                   equipment request) — written ONLY via
                   commit-record!, never mutated in place
    ledger       — an append-only audit trail of every proposal/verdict/
                   disposition, regardless of outcome (commit or hold)")

(defprotocol Store
  (survey-site [s site-id])
  (survey-data [s data-id])
  (equipment [s equipment-id])
  (records-of [s site-id])
  (ledger [s])
  (register-site! [s site])
  (register-data! [s data])
  (register-equipment! [s equipment])
  (commit-record! [s record])
  (append-ledger! [s fact]))

(defrecord MemStore [a]
  Store
  (survey-site [_ site-id] (get-in @a [:sites site-id]))
  (survey-data [_ data-id] (get-in @a [:data data-id]))
  (equipment [_ equipment-id] (get-in @a [:equipment equipment-id]))
  (records-of [_ site-id] (filter #(= site-id (:site-id %)) (:records @a)))
  (ledger [_] (:ledger @a))
  (register-site! [s site]
    (swap! a assoc-in [:sites (:site-id site)] site) s)
  (register-data! [s data]
    (swap! a assoc-in [:data (:data-id data)] data) s)
  (register-equipment! [s equipment]
    (swap! a assoc-in [:equipment (:equipment-id equipment)] equipment) s)
  (commit-record! [s record]
    (swap! a update :records (fnil conj []) record) s)
  (append-ledger! [s fact]
    (swap! a update :ledger (fnil conj []) fact) s))

(defn mem-store
  ([] (mem-store {}))
  ([seed] (->MemStore (atom (merge {:sites {} :data {} :equipment {} :records [] :ledger []} seed)))))
