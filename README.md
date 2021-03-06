# konserve

Simple durability, made easy.

A key-value store protocol defined with [core.async](https://github.com/clojure/core.async) semantics to allow Clojuresque collection operations on associative key-value stores, both from Clojure and ClojureScript for different backends. Data is generally serialized with [edn](https://github.com/edn-format/edn) semantics or, if supported, as native binary blobs and can be accessed similar to `clojure.core` functions `get-in`,`assoc-in` and `update-in`. `update-in` especially allows to run functions atomically and returns old and new value. Each operation is run atomically and must be consistent (in fact ACID), but further consistency is not supported (Riak, CouchDB and many scalable solutions don't have transactions over keys for that reason). This is meant to be a building block for more sophisticated storage solutions (Datomic also builds on kv-stores). It is not necessarily fast depending on the usage pattern. The general idea is to write most values once (e.g. in form of index fragments) and only update one place once all data is written, similar to Clojure's persistent datastructures. To store values under non-conflicting keys, have a look at [hasch](https://github.com/ghubber/hasch).

This was initially implemented as an elementary storage protocol for [geschichte](https://github.com/ghubber/geschichte).

## Supported Backends

A beta-quality file-system store in Clojure and [IndexedDB](https://developer.mozilla.org/en-US/docs/IndexedDB) for ClojureScript are provided as elementary reference implementations for the two most important platforms. No setup and no additional dependencies are needed.

### fs-store
The file-system store currently uses [fressian](https://github.com/clojure/data.fressian) and is quite efficient. It also allows to access values as a normal file-system file, e.g. to open it with a native database like HDF5 in Java.

## IndexedDB
For IndexedDB there is no internal JSON-representation of the underlying store like [transit](https://github.com/cognitect/transit-clj) yet, hence it is fairly slow for edn still. There is a JSON store protocol implemented for IndexedDB in case interoperability with a JavaScript application is wanted. Be careful not to confuse values with edn values, they are stored in separate locations and cannot clash.

### External Backends

The protocol is implemented for CouchDB in a separate project [konserve-couch](https://github.com/ghubber/konserve-couch)

New storage backends, e.g. Riak, MongoDB, Redis, JDBC, WebSQL, Local-Storage are welcome.

## Usage

Add to your leiningen dependencies:

~~~clojure
[net.polyc0l0r/konserve "0.2.3"]
~~~

For simple purposes a memory store wrapping an Atom is implemented as well:
~~~clojure
(ns test-db
  (:require [konserve.store :refer [new-mem-store]]))

(go (def my-db (<! (new-mem-store)))) ;; or
(go (def my-db (<! (new-mem-store (atom {:foo 42})))))
~~~

From a Clojure REPL run:
~~~clojure
(ns test-db
    (:require [konserve.filestore :refer [new-fs-store]]
              [clojure.core.async :as async :refer [<!!]]))

(def store (<!! (new-fs-store "/tmp/store" (atom {}))))

(<!! (-assoc-in store ["foo" :bar] {:foo "baz"}))
(<!! (-get-in store ["foo"]))
(<!! (-exists? store "foo"))

(<!! (-assoc-in store [:bar] 42))
(<!! (-update-in store [:bar] inc))
(<!! (-get-in store [:bar]))

(let [ba (byte-array (* 10 1024 1024) (byte 42))]
  (time (<!! (-bassoc store "banana" ba))))
(<!! (-bget store "banana" :input-stream))
~~~


In ClojureScript from a browser (you need IndexedDB available in your js env):
~~~clojure
(ns test-db
  (:require [konserve.indexeddb :refer [new-indexeddb-store]])
  (:require-macros [cljs.core.async.macros :refer [go go-loop]]))

(go (def my-db (<! (new-indexeddb-store "konserve"))))

(go (println "get:" (<! (-get-in my-db ["test" :a]))))

(go (doseq [i (range 10)]
       (<! (-assoc-in my-db [i] i))))

;; prints 0 to 9 each on a line
(go (doseq [i (range 10)]
      (println (<! (-get-in my-db [i])))))

(go (println (<! (-assoc-in my-db ["test"] {:a 1 :b 4.2}))))

(go (println (<! (-update-in my-db ["test" :a] inc))))
;; => "test" contains {:a 2 :b 4.2}
~~~

## Tagged Literals

You can read and write custom records with edn, but you have to supply the proper reader-functions through a store-wide tag-table atom, in the format of `{'namespace.Symbol (fn [val] ...realize proper object...)}` (this might differ depending on the backend atm.), which can be bound to a runtime wide atom in case you don't have different data schemas and code-versions to deal with. You can omit it, if you don't use tagged literals besides standard records in your edn.

## TODO
- factor serialisation protocol
- implement generic cached store(s) to wrap durable ones
- depend on hasch and use uuid hash as key/filename for file-store (and others)
- allow to iterate keys (model a cursor? or just return a snapshot of keys?)
- move repl examples to tests
- calculate deltas and store base-value and edn patches, to allow fast small nested updates

## Changelog

### 0.2.3
- filestore: flush output streams, fsync on fs operations
- filestore can be considered beta quality
- couchdb: add -exists?
- couchdb: move to new project
- remove logging and return ex-info exceptions in go channel

### 0.2.2
- filestore: locking around java strings is a bad idea, use proper lock objects
- filestore: do io inside async/thread (like async's pipeline) to not block the async threadpool
- filestore: implement a naive cache (flushes once > 1000 values)
- filestore, indexeddb: allow to safely custom deserialize file-inputstream in transaction/lock
- filestore, indexeddb, memstore: implement -exists?

### 0.2.1
- filestore: fix fressian collection types for clojure, expose read-handlers/write-handlers
- filestore: fix -update-in behaviour for nested values
- filestore: fix rollback renaming order

### 0.2.0
- experimental native ACID file-store for Clojure
- native binary blob support for file-store, IndexedDB and mem-store

## License

Copyright © 2014-2015 Christian Weilbach & Konrad Kühne

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
