(ns geology.governor-test
  (:require [clojure.test :refer [deftest is testing]]
            [geology.governor :as governor]
            [geology.store :as store]))

(defn- fresh-store []
  (let [st (store/mem-store)]
    (store/register-site! st {:site-id "site-1" :name "Alpine Basin" :coordinates [45.5 -110.5]})
    (store/register-data! st {:data-id "data-1" :site-id "site-1" :type :seismic :description "Seismic survey"})
    st))

(deftest rejects-unregistered-site-hard
  (let [st (fresh-store)
        request {:site-id "no-site"}
        proposal {:op :analyze-survey-data :effect :propose :confidence 0.9}
        verdict (governor/check request {} proposal st)]
    (is (:hard? verdict))
    (is (not (:ok? verdict)))
    (is (seq (:violations verdict)))
    (is (some #(= :no-site (:rule %)) (:violations verdict)))))

(deftest rejects-non-propose-effect-hard
  (let [st (fresh-store)
        request {:site-id "site-1"}
        proposal {:op :analyze-survey-data :effect :commit :confidence 0.9}
        verdict (governor/check request {} proposal st)]
    (is (:hard? verdict))
    (is (not (:ok? verdict)))
    (is (some #(= :no-actuation (:rule %)) (:violations verdict)))))

(deftest rejects-missing-data-for-analyze-hard
  (let [st (fresh-store)
        request {:site-id "site-1" :data-id "no-data"}
        proposal {:op :analyze-survey-data :effect :propose :confidence 0.9}
        verdict (governor/check request {} proposal st)]
    (is (:hard? verdict))
    (is (not (:ok? verdict)))
    (is (some #(= :no-data (:rule %)) (:violations verdict)))))

(deftest rejects-finalized-report-claim-hard
  (let [st (fresh-store)
        request {:site-id "site-1"}
        proposal {:op :draft-report :effect :propose :confidence 0.9 :finalized? true}
        verdict (governor/check request {} proposal st)]
    (is (:hard? verdict))
    (is (not (:ok? verdict)))
    (is (some #(= :no-finalized-reports (:rule %)) (:violations verdict)))))

(deftest escalates-flag-anomalous-formation
  (let [st (fresh-store)
        request {:site-id "site-1"}
        proposal {:op :flag-anomalous-formation :effect :propose :confidence 0.95}
        verdict (governor/check request {} proposal st)]
    (is (not (:hard? verdict)))
    (is (:escalate? verdict))
    (is (not (:ok? verdict)))))

(deftest escalates-draft-report-with-hazard-finding
  (let [st (fresh-store)
        request {:site-id "site-1"}
        proposal {:op :draft-report :effect :propose :confidence 0.9 :hazard-finding? true}
        verdict (governor/check request {} proposal st)]
    (is (not (:hard? verdict)))
    (is (:escalate? verdict))
    (is (not (:ok? verdict)))))

(deftest escalates-low-confidence
  (let [st (fresh-store)
        request {:site-id "site-1" :data-id "data-1"}
        proposal {:op :analyze-survey-data :effect :propose :confidence 0.4}
        verdict (governor/check request {} proposal st)]
    (is (not (:hard? verdict)))
    (is (:escalate? verdict))
    (is (not (:ok? verdict)))))

(deftest approves-clean-low-stake-request
  (let [st (fresh-store)
        request {:site-id "site-1" :data-id "data-1"}
        proposal {:op :analyze-survey-data :effect :propose :confidence 0.95 :stake :low}
        verdict (governor/check request {} proposal st)]
    (is (not (:hard? verdict)))
    (is (not (:escalate? verdict)))
    (is (:ok? verdict))))

(deftest approves-draft-report-without-hazard-finding
  (let [st (fresh-store)
        request {:site-id "site-1"}
        proposal {:op :draft-report :effect :propose :confidence 0.85 :hazard-finding? false}
        verdict (governor/check request {} proposal st)]
    (is (not (:hard? verdict)))
    (is (not (:escalate? verdict)))
    (is (:ok? verdict))))
