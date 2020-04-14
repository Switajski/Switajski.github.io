#  understanding setState in cljs

https://funcool.github.io/clojurescript-unraveled/#state-management and there observation:

	(def a (atom))

	(add-watch a :logger (fn [key the-atom old-value new-value]
	                       (println "Key:" key "Old:" old-value "New:" new-value)))

	(reset! a 42)
	;; Key: :logger Old: nil New: 42
	;; => 42

	(swap! a inc)
	;; Key: :logger Old: 42 New: 43
	;; => 43

	(remove-watch a :logger)

