(ns geology.actor-test
  (:require [clojure.test :refer [deftest is testing]]
            [geology.actor :as actor]
            [geology.store :as store]
            [geology.advisor :as advisor]))

(defn- fresh-store []
  (let [st (store/mem-store)]
    (store/register-site! st {:site-id "site-1" :name "Alpine Basin" :coordinates [45.5 -110.5]})
    (store/register-data! st {:data-id "data-1" :site-id "site-1" :type :seismic :description "Seismic survey"})
    st))

(deftest commits-approved-request
  (let [st (fresh-store)
        g (actor/build-graph {:store st})
        request {:site-id "site-1" :data-id "data-1" :op :analyze-survey-data :stake :low}
        result (actor/run-request! g request {} "thread-1")]
    (is (= :done (:status result)))
    (is (= :commit (get-in result [:state :disposition])))))

(deftest holds-unregistered-site
  (let [st (fresh-store)
        g (actor/build-graph {:store st})
        request {:site-id "no-site" :op :analyze-survey-data}
        result (actor/run-request! g request {} "thread-2")]
    (is (= :done (:status result)))
    (is (= :hold (get-in result [:state :disposition])))))

(deftest escalates-flag-anomalous-formation
  (let [st (fresh-store)
        g (actor/build-graph {:store st})
        request {:site-id "site-1" :op :flag-anomalous-formation :stake :high}
        result (actor/run-request! g request {} "thread-3")]
    (is (= :interrupted (:status result)))
    (is (= :request-approval (get-in result [:state :disposition])))))

(deftest approve-escalated-request
  (let [st (fresh-store)
        g (actor/build-graph {:store st})
        request {:site-id "site-1" :op :flag-anomalous-formation :stake :high}
        result (actor/run-request! g request {} "thread-4")
        _ (is (= :interrupted (:status result)))
        approval-result (actor/approve! g "thread-4")]
    (is (= :done (:status approval-result)))
    ;; disposition is the governor's decision and remains unchanged;
    ;; graph has progressed through :commit node (a finish point)
    (is (= :request-approval (get-in approval-result [:state :disposition])))
    (is (seq (get-in approval-result [:state :record])))))
