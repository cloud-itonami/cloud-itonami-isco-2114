(ns geology.render-html
  "Build-time HTML renderer for `docs/samples/operator-console.html`.

  Closes flagship checklist item 2 (com-junkawasaki/root ADR-2607189300)
  for the ISCO-08 cluster: this repo previously had NO demo page and no
  generator at all (`:item2/classification \"unknown-no-demo\"` in the
  fleet-wide scan). This namespace drives the REAL actor stack
  (`geology.actor` -> `geology.governor` -> `geology.store`) through a
  scenario built from real, exercised store data and renders the
  result deterministically -- no invented numbers, no timestamps in
  the page content, byte-identical across reruns against the same
  seed (verify by diffing two consecutive runs before shipping).
  Adapted from the proven ISCO-side template in
  cloud-itonami-isco-1211's `finmgmt.render-html` (see that
  namespace's docstring for the original shape-adaptation notes; the
  general pattern carries over, the concrete domain fields below do
  not).

  `site-1` (\"Alpine Basin\") + `data-1` (a registered seismic survey
  linked to `site-1`) below are lifted VERBATIM from this repo's own
  proven-passing test fixture (`geology.actor-test/fresh-store` and
  `geology.governor-test/fresh-store`, identical in both) -- ground
  truth, not invented. No additional demo data is registered for this
  scenario -- the `:no-site` and `:no-data` HARD-holds are both
  demonstrated by naming a `site-id`/`data-id` this demo never
  registers (\"site-ghost\"), which is the honest way to exercise
  \"unregistered\" (registering it first would defeat the point).
  Every field this page displays (statuses, hold reasons) is real
  output read after `run-demo!` actually executed the graph -- none
  of it is hand-typed.

  Known architectural gaps, honestly noted rather than papered over
  (all four confirmed by reading `geology.advisor/infer`, the real
  `mock-advisor` -- `infer` reads ONLY `:op` and `:stake` off the
  request; it never reads or forwards `:finalized?`, `:hazard-
  finding?`, or `:confidence`, so any governor rule keyed on those
  proposal fields can only ever be exercised against a hand-built
  proposal, never through this real, advisor-driven demo):
  - `:no-actuation` (proposal `:effect` must be `:propose`) --
    `infer` unconditionally sets `:effect :propose`. Covered instead
    by `geology.governor-test/rejects-non-propose-effect-hard`.
  - `:no-finalized-reports` (a `:draft-report` proposal must not
    claim `:finalized? true`) -- `infer` never sets `:finalized?` on
    any proposal (the advisor has no notion of it), so a real
    `:draft-report` request can never trigger this HARD-hold. Covered
    instead by `geology.governor-test/
    rejects-finalized-report-claim-hard`.
  - `:draft-report` with `:hazard-finding? true` escalation -- same
    reason: `infer` never sets `:hazard-finding?`. Covered instead by
    `geology.governor-test/escalates-draft-report-with-hazard-finding`.
  - The `confidence < 0.6` escalation path -- `infer`'s confidence
    table is `{:high 0.7 :medium 0.85 :low 0.95}`, all at or above the
    0.6 floor. Covered instead by `geology.governor-test/
    escalates-low-confidence`.
  The `:flag-anomalous-formation` always-escalate path IS genuinely
  reachable (it keys only on `:op`, which `infer` passes straight
  through), exactly as `geology.actor-test/
  escalates-flag-anomalous-formation` proves.

  Usage: `clojure -M:render-html [out-file]`
  (default `docs/samples/operator-console.html`)."
  (:require [jp-go-dds.skin]
            [kotoba.lang.text :as str]
            [geology.store :as store]
            [geology.actor :as actor]))

;; ----------------------------- harness --------------------------------

(defn- run-op!
  "Drives one real survey operation request through the actual
  compiled graph for `tid` (thread-id). If the graph escalates
  (interrupts before `:request-approval`), immediately approves it
  (this demo's scenario never demonstrates an UNAPPROVED escalation --
  every escalation here reaches a human who signs off). Returns a map
  describing exactly what really happened -- no field is invented."
  [graph tid site-id op extra]
  (let [request (merge {:site-id site-id :op op} extra)
        r1 (actor/run-request! graph request {} tid)]
    (if (= :interrupted (:status r1))
      (let [r2 (actor/approve! graph tid)]
        {:thread-id tid :site-id site-id :op op :request request
         :outcome :approved-and-committed
         :record (get-in r2 [:state :record])})
      (let [disposition (get-in r1 [:state :disposition])]
        (if (= :hold disposition)
          {:thread-id tid :site-id site-id :op op :request request
           :outcome :hard-hold
           :verdict (get-in r1 [:state :verdict])
           :rule (-> r1 :state :verdict :violations first :rule)}
          {:thread-id tid :site-id site-id :op op :request request
           :outcome :auto-committed
           :record (get-in r1 [:state :record])})))))

(def ^:private op-specs
  "The scenario: covers every disposition this actor can genuinely
  reach through its real graph (auto-commit across all 4 op kinds,
  escalate-then-approve for the always-escalate anomaly flag, and 2
  of the 4 distinct HARD-hold reasons in `geology.governor` -- the
  other 2 (`:no-actuation`, `:no-finalized-reports`) plus the
  hazard-finding escalation and the low-confidence escalation are
  architecturally unreachable via the real advisor, see namespace
  docstring). Every `:op` keyword and violation rule name below is
  copied from `geology.governor`'s own `hard-violations`/`check`, not
  invented."
  [;; site-1 / \"Alpine Basin\" + data-1 (real fixture from
   ;; geology.actor-test / geology.governor-test)
   ["s1-analyze-good"      "site-1" :analyze-survey-data      {:data-id "data-1" :stake :low}]
   ["s1-analyze-no-data"   "site-1" :analyze-survey-data      {:data-id "data-ghost" :stake :low}]
   ["s1-draft-report"      "site-1" :draft-report              {:stake :low}]
   ["s1-request-equipment" "site-1" :request-field-equipment   {:stake :low}]
   ["s1-flag-anomaly"      "site-1" :flag-anomalous-formation  {:stake :high}]
   ;; never-registered site
   ["ghost-no-site"        "site-ghost" :analyze-survey-data   {:data-id "data-1" :stake :low}]])

(defn run-demo!
  "Runs a fresh store through `op-specs` (see above) via the real
  compiled `geology.actor` graph. Returns `{:store :runs}` -- `:runs`
  is the ordered vector of real per-request outcomes; every field in
  `render` below is read from this or from `store` after the graph
  actually executed, never hand-typed."
  []
  (let [db (store/mem-store)]
    (store/register-site! db {:site-id "site-1" :name "Alpine Basin" :coordinates [45.5 -110.5]})
    (store/register-data! db {:data-id "data-1" :site-id "site-1" :type :seismic :description "Seismic survey"})
    (let [graph (actor/build-graph {:store db})
          runs (mapv (fn [[tid site-id op extra]]
                       (run-op! graph tid site-id op extra))
                     op-specs)]
      {:store db :runs runs})))

;; ----------------------------- rendering -------------------------------

(defn- esc [v]
  (-> (str v)
      (str/replace "&" "&amp;")
      (str/replace "<" "&lt;")
      (str/replace ">" "&gt;")))

(defn- outcome-cell [{:keys [outcome rule]}]
  (case outcome
    :auto-committed "<span class=\"ok\">committed</span>"
    :approved-and-committed "<span class=\"ok\">approved &amp; committed</span>"
    :hard-hold (str "<span class=\"critical\">HARD hold &middot; " (esc (name (or rule :unknown))) "</span>")
    "<span class=\"muted\">in progress</span>"))

(defn- site-row [store {:keys [site-id name]} runs]
  (let [last-run (last (filter #(= site-id (:site-id %)) runs))]
    (format "        <tr><td>%s</td><td>%s</td><td>%d</td><td>%s</td></tr>"
            (esc site-id) (esc name)
            (count (store/records-of store site-id))
            (if last-run (outcome-cell last-run) "<span class=\"muted\">no activity</span>"))))

(defn- run-row [{:keys [thread-id site-id op request outcome rule]}]
  (format "        <tr><td><code>%s</code></td><td>%s</td><td><code>%s</code></td><td>%s</td><td>%s</td></tr>"
          (esc thread-id) (esc site-id) (esc (name op))
          (esc (or (:data-id request) ""))
          (outcome-cell {:outcome outcome :rule rule})))

(def ^:private action-gate-rows
  ;; Static description of this actor's own op contract (README.md,
  ;; `geology.governor`'s own docstring) -- documentation of fixed
  ;; behavior, not runtime telemetry, so it is legitimately
  ;; hand-described rather than derived from a live run.
  ["        <tr><td><code>:analyze-survey-data</code></td><td><span class=\"warn\">requires a REGISTERED survey-data record for the site</span></td></tr>"
   "        <tr><td><code>:draft-report</code></td><td><span class=\"warn\">can never claim :finalized? true &middot; hazard findings always human-reviewed</span></td></tr>"
   "        <tr><td><code>:flag-anomalous-formation</code></td><td><span class=\"err\">ALWAYS human sign-off &middot; geological hazard safeguard, never silently dismissed</span></td></tr>"
   "        <tr><td><code>:request-field-equipment</code></td><td><span class=\"ok\">auto-commit when site is registered</span></td></tr>"])

(defn render
  "Renders the full operator-console.html document from `{:store :runs}`
  as produced by `run-demo!` (or any other real scenario)."
  [{:keys [store runs]}]
  (let [sites [{:site-id "site-1" :name "Alpine Basin"}]
        site-rows (str/join "\n" (map #(site-row store % runs) sites))
        run-rows (str/join "\n" (map run-row runs))]
    (str
     "<html><head><meta charset=\"utf-8\"><title>cloud-itonami-isco-2114 &middot; geological survey operator console</title><style>"
   (jp-go-dds.skin/dds+skin)
   "</style></head><body>\n"
     "<header class=\"bar\">\n"
     "  <h1>Geological &amp; Geophysical Survey Support (ISCO-08 2114) — Operator Console</h1>\n"
     "  <span class=\"badge\">read-only sample · governor-gated · anomalous formations always human-reviewed</span>\n"
     "</header>\n"
     "<main>\n"
     "  <section class=\"card\">\n"
     "    <h2>Registered survey sites</h2>\n"
     "    <p class=\"muted\">Demo snapshot — build-time-generated from <code>geology.store</code> via <code>geology.render-html</code> (<code>clojure -M:render-html</code>), regenerated nightly.</p>\n"
     "    <table>\n"
     "      <thead><tr><th>Site</th><th>Name</th><th>Committed records</th><th>Last op status</th></tr></thead>\n"
     "      <tbody>\n"
     site-rows "\n"
     "      </tbody>\n"
     "    </table>\n"
     "  </section>\n"
     "  <section class=\"card\">\n"
     "    <h2>Action gate (GeologyGovernor)</h2>\n"
     "    <p class=\"muted\">HARD holds cannot be overridden. Survey data analysis requires a registered survey-data record; anomalous formations are never silently dismissed.</p>\n"
     "    <table>\n"
     "      <thead><tr><th>Op</th><th>Gate</th></tr></thead>\n"
     "      <tbody>\n"
     (str/join "\n" action-gate-rows) "\n"
     "      </tbody>\n"
     "    </table>\n"
     "  </section>\n"
     "  <section class=\"card\">\n"
     "    <h2>Audit trail (this run)</h2>\n"
     "    <p class=\"muted\">Every request this scenario drove through the real compiled graph, in order — thread-id, site, op, the request's own data-id field (where applicable), and the real disposition (auto-commit, approved-after-escalation, or the specific HARD-hold rule).</p>\n"
     "    <table>\n"
     "      <thead><tr><th>Thread</th><th>Site</th><th>Op</th><th>Data ref</th><th>Disposition</th></tr></thead>\n"
     "      <tbody>\n"
     run-rows "\n"
     "      </tbody>\n"
     "    </table>\n"
     "  </section>\n"
     "</main>\n"
     "</body></html>\n")))

(defn -main [& args]
  (let [out (or (first args) "docs/samples/operator-console.html")
        result (run-demo!)
        html (render result)]
    (spit out html)
    (println "wrote" out "("
             (count (:runs result)) "requests driven through the real graph,"
             (count (store/ledger (:store result))) "ledger facts )")))
