(ns geology.governor
  "GeologyGovernor — the independent safety/traceability layer for
  the ISCO-08 2114 survey support actor (geologists and geophysicists).
  Wired as its own `:govern` node in `geology.actor`'s StateGraph,
  downstream of `:advise` — the Advisor has no notion of site
  provenance or geological hazard risk, so this MUST be a separate
  system able to reject a proposal (itonami actor pattern, per
  ADR-2607011000 / CLAUDE.md Actors section).

  `check` is a pure function of (request, context, proposal, store) ->
  verdict; it never mutates the store. The StateGraph's `:decide` node
  routes on the verdict:
    :hard? true                → :hold  (irreversible, no write)
    :escalate? true            → :request-approval (interrupt-before)
    otherwise                  → :commit

  HARD invariants (:hard? true, ALWAYS :hold, never overridable):
    1. survey-site provenance  — the request's site must be registered.
    2. data verification       — :analyze-survey-data ops must reference
                                 registered survey data.
    3. no-actuation            — proposal :effect must be :propose.
    4. no-finalized-reports    — :draft-report proposals can never claim
                                 a result as finalized/ready-for-publication
                                 (draft is draft-for-review only).

  ESCALATION invariants (:escalate? true, ALWAYS human sign-off):
    5. :flag-anomalous-formation — always escalates (geological hazard
                                   safeguard, never silently dismissed).
    6. :draft-report with :hazard-finding? true — hazard findings require
                                                  human review before
                                                  proceeding.
    7. low confidence (< `confidence-floor`)."
  (:require [geology.store :as store]))

(def confidence-floor 0.6)
(def ^:private escalating-ops #{:flag-anomalous-formation :draft-report})

(defn- hard-violations [{:keys [proposal request]} site-record data-record]
  (cond-> []
    (nil? site-record)
    (conj {:rule :no-site :detail "未登録 survey site"})

    (and (= :analyze-survey-data (:op proposal))
         (nil? data-record))
    (conj {:rule :no-data :detail "analyze-survey-data 前に survey data は要登録"})

    (not= :propose (:effect proposal))
    (conj {:rule :no-actuation :detail "effect は :propose のみ許可（直接書込禁止）"})

    (and (= :draft-report (:op proposal))
         (:finalized? proposal))
    (conj {:rule :no-finalized-reports :detail "report 最終化は draft 提案では不可（draft は査読用のみ）"})))

(defn check
  "Assess a proposal against `request`/`context`/`proposal` and a
  `store` implementing `geology.store/Store`. Returns
  `{:ok? bool :violations [...] :confidence n :hard? bool :escalate? bool}`."
  [request context proposal store]
  (let [site-record (store/survey-site store (:site-id request))
        data-record (when (:data-id request) (store/survey-data store (:data-id request)))
        hard (hard-violations {:proposal proposal :request request} site-record data-record)
        hard? (boolean (seq hard))
        conf (or (:confidence proposal) 0.0)
        low? (< conf confidence-floor)
        is-flag? (= :flag-anomalous-formation (:op proposal))
        is-draft-hazard? (and (= :draft-report (:op proposal)) (:hazard-finding? proposal))
        risky-op? (and (contains? escalating-ops (:op proposal))
                       (or is-flag? is-draft-hazard?))]
    {:ok? (and (not hard?) (not low?) (not risky-op?))
     :violations hard
     :confidence conf
     :hard? hard?
     :escalate? (and (not hard?) (or low? risky-op?))}))
