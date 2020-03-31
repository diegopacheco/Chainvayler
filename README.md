# Chainvayler
## ~Transparent Replication and Persistence for POJOs (Plain Old Java Objects)

* [What is this?](#what-is-this)
* [Introduction](#introduction)
* [Bank sample](#bank-sample)
  * [Overview](#overview)
  * [Running the sample in Kubernetes](#running-the-sample-in-kubernetes)
  * [Running the sample locally](#running-the-sample-locally)
* [How it works?](#how-it-works)
* [Consistency](#consistency)
* [Performance](#performance)
* [Determinism](#determinism)
* [Limitations](#limitations)
  * [Garbage collection](#garbage-collection)
  * [Clean shutdown](#clean-shutdown)
* [FAQ and more](#faq-and-more)

## [What is this?](#what-is-this)

This is a _proof of concept_ library which provides [POJOs](http://en.wikipedia.org/wiki/Plain_Old_Java_Object) (Plain Old Java Objects) __replication__ and __persistence__ capabilities 
almost __transparently__.

_Chainvayler_ requires neither implementing special interfaces nor extending from special classes nor a backing relational database. 
Only some _@Annotations_ and conforming a few rules is necessary.

Either _replication_ or _persistence_ can be disabled independently. If _replication_ is disabled, you will have locally persisted POJOs. 
If _persistence_ is disabled, you will have your POJOs replicated over JVMs possibly spreading over multiple nodes. 
If both _replication_ and _persistence_ is disabled, well you will only have _Chainvayler_'s overhead ;)

I've found the idea really promising, so went ahead and made a PoC implementation.

Sounds too good to be true? Well, keep reading... ;)

## [Introduction](#introduction)

As mentioned, _Chainvayler_ only requires some _@Annotations_ and conforming a few rules.

Here is a quick sample:
```
@Chained
class Library {
   final Map<Integer, Book> books = new HashMap<Integer, Book>();
   int lastBookId = 1;
   
   @Modification
   void addBook(Book book) {
      book.setId(lastBookId++);
      books.put(book.getId(), book);
   }
}
```
Quite a _Plain Old Java Object_, isn't it? Run the __Chainvayler compiler__ after __javac__ and then to get a reference to a chained instance:

```
Library library = Chainvayler.create(Library.class);
```
or this variant to configure options:
```
Library library = Chainvayler.create(Library.class, config);
```

Now, add as many books as you want to your library, they will be _automagically_ persisted and replicated to other JVMs. 
Kill your program any time, when you restart it, the previously added books will be in your library.

Note, the call to _Chainvayler.create(..)_ is only required for the _root_ of object graph. All other objects are created in regular ways, either with the _new_ oprerator or via factories, builders whatever. As it is, _Chainvayler_ is quite flexible, other objects may be other instances of _root_ class, subclasses/superclasses of it, or instances of a completely different class hierarchy.

`@Chained` annotation marks the classes which will be managed by _Chainvayler_ and `@Modification` annotation marks the methods in _chained_ classes which modifies the data (class variables).

## [Bank sample](#bank-sample)

### [Overview](#bank-sample-overview)

_Chainvayler_ comes with a [_Bank_](bank-sample/src/main/java/raft/chainvayler/samples/bank) sample, for both demonstration and testing purposes.

Below is the class diagram of the _Bank_ sample:
![Class diagram](https://chainvayler-public.s3-us-west-2.amazonaws.com/images/bank-sample-class-diagram.png) 

Nothing much fancy here. Apparently this is a toy diagram for a real banking application, but hopefully good enough to demonstrate _Chainvayler_'s capabilities.

[_Bank_](bank-sample/src/main/java/raft/chainvayler/samples/bank/Bank.java) class is the _root_ class of this object graph. It's used to get a _chained_ instance of the object graph via _Chainvayler_. Every object reachable directly or indirectly from the root _Bank_ object will be _chained_ (persisted/replicated). Notice _Bank_ class has super and sub classes and even has a reference to another _Bank_ object.

For the sake of brevity, I've skipped the class methods in the diagram but included a few to demonstrate _Chainvayler_'s capabilities regarding some edge cases:
  * _Person_ and _RichPerson_ constructors throw an exception if _name_ has a special value
  * _SecretCustomer_ throws an exception whenever _getName_ is called
  * _SecretCustomer_ resides in a different package, I will tell in a bit what it demonstrates
  
__Note__: I tried to cover all the possible edge cases. If you notice a case which is not covered, please create an issue.

### [Running the sample in Kubernetes](#running-the-sample-in-kubernetes)

The easiest way to see _Chainvayler_ in action is to run the _Bank_ sample in _Kubernetes_ via provided _Helm_ charts.

In `Chainvayler/bank-sample` folder, run the following command:
```
helm install kube/chainvayler-bank-sample/ --name chainvayler-sample 
```

This will by default create 3 writer pods and the watcher application `peer-stats` to follow the process. Any writer or reader pods will register themselves to `peer-stats` pod via RMI and you can follow the process via `peer-stats` pod's logs:

```
kubectl logs chainvayler-peer-stats-<ID> --follow
```

The output will be similar to below (will be updated every 5 seconds):
```
created RMI registry
bound PeerManager to registry
-- started | completed | pool size  | tx count   | tx/second | own tx count | own tx/sec | read count     | reads/sec --
------------------------------------------------------------------------------------------------------------------------
registered peer, count: 1
registered peer, count: 2
registered peer, count: 3
-- started | completed | pool size  | tx count   | tx/second | own tx count | own tx/sec | read count     | reads/sec --
   false     false       -1           1            0.05        0              0.00         0                0.00
   false     false       4            1            0.05        0              0.00         0                0.00
   true      false       4            1            40.00       0              0.00         0                0.00
------------------------------------------------------------------------------------------------------------------------
-- started | completed | pool size  | tx count   | tx/second | own tx count | own tx/sec | read count     | reads/sec --
   true      false       15154        19607        3891.25     6418           1273.41      0                0.00
   true      false       15161        19624        3897.52     6427           1276.21      0                0.00
   true      false       15184        19644        3893.76     6782           1344.43      0                0.00
------------------------------------------------------------------------------------------------------------------------
-- started | completed | pool size  | tx count   | tx/second | own tx count | own tx/sec | read count     | reads/sec --
   true      false       39019        45812        4555.33     14908          1482.35      0                0.00
   true      false       39021        45811        4556.50     14922          1484.19      0                0.00
   true      false       39049        45827        4553.56     15986          1588.28      0                0.00
------------------------------------------------------------------------------------------------------------------------
-- started | completed | pool size  | tx count   | tx/second | own tx count | own tx/sec | read count     | reads/sec --
   true      false       69130        77516        5141.67     25268          1675.89      0                0.00
   true      false       69143        77547        5144.76     25216          1672.93      0                0.00
   true      false       69146        77547        5141.35     27053          1793.49      0                0.00
------------------------------------------------------------------------------------------------------------------------
-- started | completed | pool size  | tx count   | tx/second | own tx count | own tx/sec | read count     | reads/sec --
   true      true        84774        94390        5249.50     32033          1787.16      0                0.00
   true      true        84774        94390        5228.20     31971          1770.85      0                0.00
   true      true        84774        94390        5278.17     30385          1812.62      0                0.00
------------------------------------------------------------------------------------------------------------------------
stopped all readers
will compare results..
transaction counts are the same! 94390
--all banks are identical!!
pool sizes are same  84774 == 84774
pool sizes are same  84774 == 84774
all pool classes are the same
this pair looks identical
this pair looks identical
this pair looks identical
-- all pools are identical!!
average tx/second: 5251.96, own tx/second: 1790.21
```

So, what happened (with the default values) exactly is:
* We had launched 3 instances of _Bank_ sample where _replication_ is enabled and _persistence_ is disabled
* They all registered themselves to `peer-stats` application to provide some metrics
* They started to make random _write_ operations over their copy of the _Bank_ with 5 threads
* `peer-stats` application collected and printed their metrics every 5 seconds
* After all writers are finished:
  * `peer-stats` got the final copy of each one's _Bank_ object and also _Chainvayler_'s implemenation details, like transaction count and internal object pool
  * Compared each one of them against any other them
  * Checked if they are __completely identical__
* Printed the final statistics

Welcome to _Chainvayler_ world! You just witnessed a POJO graph is _automagically_ and _~transparently_ replicated!

Feel free to try different settings: more writers, some readers, some writer-readers or more write actions. See the [values.yaml](bank-sample/kube/chainvayler-bank-sample/values.yaml) file for all options.

For example, lets create additional 2 readers:
```
helm install kube/chainvayler-bank-sample/ --name chainvayler-sample --set replication.readerCount=2 --set load.actions=5000
```
Increased the action count, so we will have more time until they are completed. Kill any pod any time, when restarted, they will retrieve the initial state and catch the others.

For example, from the logs after they are killed and restarted:
```
requesting initial transactions [2 - 23582]
received all initial transactions [2 - 23582], count: 23581
```

If you enable persistence and mount external disks with the flags `--set persistence.enabled=true --set persistence.mountVolumes=true`, when pods are killed and restarted, logs will be something like:
```
requesting initial transactions [40154 - 69079]
received all initial transactions [40154 - 69079], count: 28926
```
In this case, the first 40153 transactions are loaded from the disk, next 28926 are retrieved from the network.

__Note:__ When replication is enabled, most of the time pods recover successfully after a kill/restart cycle. But still sometimes they cannot properly connect to Hazelcast cluster or some of the initial transactions get lost. Not sure if this is a misconfuguration by myself or Hazelcast is not bullet proof.

__Important__: Do not kill writer pods with `-9` switch before they are completed. This will hang the whole system. See [clean shutdown](#clean-shutdown) for details.

### [Running the sample locally](#bank-sample-run-local)

## [How it works?](#how-it-works)

### Prevayler

To explain how _Chainvayler_ works, I need to first introduce [Prevayler](http://prevayler.org/). It is a brilliant library to persist real POJOs. In short it says:

> Encapsulate all changes to your data into _Transaction_ classes and pass over me. I will write those transactions to disk and then execute on your data. When the program is restarted, I will execute those transactions in the same order on your data. Provided all such changes are deterministic, we will end up with the exact same state just before the program terminated last time.

This is simply a brilliant idea to persist POJOs. Actually, this is the exact same sequence databases store and re-execute transaction logs. 

### Postvayler

However, the thing is _Prevayler_ is a bit too verbose. You need to write _Transaction_ classes for each operation that modifies your data. And it’s also a bit old fashioned considering today’s wonderful _@Annotated_ Java world.

Here comes into scene [Postvayler](https://github.com/raftAtGit/Postvayler). It's the predecessor of _Chainvayler_, which was also a PoC project by myself for transperent POJO persistence. 

Postvayler injects bytecode into (instruments) __javac__ compiled `@Persistent` classes such that every `@Persist` method in a `@Persistent` class is modified to execute that method via `Prevayler`.

For example, the `addBook(Book)` method in the introduction sample becomes something like (omitting some details for readability):
```
void addBook(Book book) {
  if (! there is Postvayler context) {
     // no persistence, just proceed to original method
     __postvayler_addBook(book);
     return;
  }
  if (weAreInATransaction) {
     // we are already encapsulated in a transaction, just proceed to original method
     __postvayler_addBook(book);
     return;
  }
  weAreInATransaction = true;
  try {
    prevayler.execute(new aTransactionDescribingThisMethodCall());
  } finally {
    weAreInATransaction = false;
  }
}

// original addBook method is renamed to this
private void __postvayler_addBook(Book book) {
  // the contents of the original addBook method
}
```

As can been seen, if there is no _Postvayler_ context around, the object bahaves like the original POJO with an ignorable overhead.

Constructors of _@Persistent_ classes are also instrumented to keep track of of them. They are pooled weekly so GC works as expected

### Chainvayler

_Chainvayler_ takes the idea of _Postvayler_ one step forward and replicates _transactions_ among JVMs and executes them with the exact same order. So we end up with transparently replicated and persisted POJOs. 

Before a transaction is committed locally, a global transaction ID is retrieved via Hazelcast's 
[IAtomicLong](https://docs.hazelcast.org/docs/latest/javadoc/com/hazelcast/cp/IAtomicLong.html) data structure, which is basically a distributed version of Java's [AtomicLong](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicLong.html). Local JVM waits until all transactions up to retrieved transaction ID is committed and then commits its own transaction. 

Hazelcast's _IAtomicLong_ uses _Raft_ consensus algorithm behind the scenes and is [CP](https://hazelcast.com/blog/hazelcast-imdg-3-12-introduces-cp-subsystem/) (consistent and partition tolerant) in regard to [CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem) and so as _Chainvayler_. 

_Chainvayler_ uses Hazelcast's [IMap](https://docs.hazelcast.org/docs/latest/javadoc/com/hazelcast/map/IMap.html) data structure to replicate transactions among JVM's. Possibly, this can be replaced with some other mechanism, for example with [Redis pub/sub](https://redis.io/topics/pubsub), should be benchmarked to see which one performs better.

_Chainvayler_ also makes use of some ugly hacks to integrate with _Prevayler_. In particular, it uses _reflection_ to access _Prevayler_ [internals](https://github.com/raftAtGit/Chainvayler/blob/master/chainvayler/src/main/java/raft/chainvayler/impl/HazelcastPrevayler.java)
as _Prevayler_ never meant to be extened this way. Obviously this is not optimal but please just remember this is just a PoC project ;) Possibly the way to go here is, enhancing _Prevayler_ 
code base to allow this kind of extension.

__Note__, even if _persistence_ is disabled, still _Prevayler_ transactions are used behind the scenes. Only difference is, _Prevayler_ is configured not to save transactions to disk.

### Constructors

Possibly, instrumentation of constructors are the most complicated part of _Chainvayler_ and also _Postvayler_. So best to mention a bit.

First, as mentioned before, except the _root_ object of the _chained_ object graph, creating instances of _chained_ classes are done in regular ways, either with the _new_ oprerator, or via factories, builders whatever.

For example, here is a couple of code fragments from the [_Bank_](https://github.com/raftAtGit/Chainvayler/blob/master/bank-sample/src/main/java/raft/chainvayler/samples/bank/Main.java) sample to create objects:
```
Bank other = new Bank();
```
```
Customer customer = new Customer(<name>);
```
```
Customer customer = bank.createCustomer(<name>);
```
They look quite POJO way, right?

Actually many things are happening behind the scenes due to constructor instrumentation:
* First, if there is no _Chainvayler_ context around, they act like a plain POJO. They do nothing special
* Otherwise, the created object gets a unique long id, which is guaranteed to be the same among all JVMs and the object is put to the local object pool with that id
* If necessary, a [ConstructorTransaction](https://github.com/raftAtGit/Chainvayler/blob/master/chainvayler/src/main/java/raft/chainvayler/impl/ConstructorTransaction.java) is created and committed with the arguments passed to constructor.
A _ConstructorTransaction_ is necessary if:
  * We are not already in a transaction (inside a `@Modification` method call or in another _ConstructorTransaction_)
* If this is the local JVM, the JVM which created the object for the first time, it just gets back the reference of the object. All the  rest works as plain POJO world. 
* Otherwise, if this is due to a remote transaction (coming from another JVM) or a recovery transaction (JVM stopped/crashed and replaying transactions from disk)
  * Object is created due to _ConstructorTransaction_ using the exact same constructor arguments and gets the exact same id
  * Object is put to local object pool with that id
* After this point, local and remote JVMs works the same. Transactions representing `@Modification` method calls internally use _target_ object ids. So, as each _chained_ object gets the same id across all JVM sessions, `@Modification` method calls execute on the very same object. 

Constructor instrumentation also does some things which is not possible via plain Java code. In particular it:
* Injects some bytecode which is executed before calling `super class'` constructor
* Wraps `super class'` constructor call in a `try/catch/finally` clause.

These are required to effectively handle exceptions thrown from constructors (edge cases) and initiate a _ConstructorTransaction_ in the correct point.

## [Consistency](#consistency)

Each _Chainvayler_ instance is `strongly consistent` locally. That is, once the invocation of a `@Modification` method completed, all reads in the same JVM session reflect the modification. 

The overall system is `eventually consistent` for `reads`. That is, once the invocation of a `@Modification` method completed in one JVM, reads on other JVMs may not reflect the modification. 

However, the overall system is `strongly consistent` for `writes`. That is, once the invocation of a `@Modification` method completed in one JVM, writes on other JVMs reflect the modification, `@Modification` methods will wait until all other writes from other JVMs are committed locally. So they can be sure they are modifying latest version of the data.

In other words, provided all changes are deterministic, any `@Modification` method invocation on any JVM is guaranteed to be executed on the exact same data.


## [Performance](#performance)

## [Determinism](#determinism)

As mentioned, all methods which modify the data in the _chained_ classes should be [deterministic](http://en.wikipedia.org/wiki/Deterministic_system). That is, given the same inputs, they should modify the data in the exact same way.

The term `deterministic` has interesting implications in `Java`. For example, iteration order of `HashSet` and `HashMap` is not deterministic. They depend on the hash values which may not be the same in different JVM sessions. So, if iteration order is siginificant, for example finding the first object in a `HashSet` which satisfies certain conditions and operate on that, instead `LinkedHashSet` and `LinkedHashMap` should be used which provide predictable iteration order.

In contrast, random operations are deterministic as long as you use the same seed.

Another source of indeterminism is depending on the data provided by external actors, for example sensor data or stock market data. For these kind of situations, relevant `@Modification` methods should accept the data as method arguments. For example, below sample is completely safe:

```
@Chained
class SensorRecords {
  List<Record> records = new ArrayList<>();
  
  void record() {
    Record record = readRecordFromSomeSensor();
    record(record);
  }
  
  @Modification
  void record(Record record) {
    records.add(record);
  }
}
```

__Note__, _clock_ is also an external actor to the system. Since it's so commonly used, _Chainvayler_ provides a 
[Clock](https://github.com/raftAtGit/Chainvayler/blob/master/chainvayler/src/main/java/raft/chainvayler/Clock.java) 
facility which can safely be used in `@Modification` methods instead of `System.currentTimeMillis()` or `new Date()`. 
`Clock` pauses during the course of transactions and always has the same value for the same transaction regardless of which 
JVM session it's running on.

_Audits_ in the [Bank](https://github.com/raftAtGit/Chainvayler/blob/master/bank-sample/src/main/java/raft/chainvayler/samples/bank/Bank.java) sample demonstrates usage of clock facility.

## [Limitations](#limitations)

### [Garbage collection](#garbage-collection)

When _replication_ is not enabled, garbage collection works as expected. Any _chained_ object created but not accessible from the _root_ object will be garbage collected soon if there are no other references to it. This is achieved by holding references to _chained_ objects via _weak_ references. 

However, this is not possible when _replication_ is enabled. Imagine a _chained_ object is created on a JVM and it's not accessible from the _root_ object, there are only some other local references to it. Those other local references will prevent it to be garbage collected. 

When this _chained_ object is replicated to other JVMs, there won't be any local references to it, and hence nothing will stop it to be garbage collected if it's not accessible from the _root_ object. 

So, unfortunately, looks like, we need to keep a reference to all created _chained_ objects in replication mode and prevent them to be garbage collected. 

### [Clean shutdown](#clean-shutdown)

When _replication_ is enabled, clean shutdown is very important. In particular, if a writer reserves a transaction ID in the network and dies before sending the transaction to the network, the whole network will hang, they will wait indefinitely to receive that missing transaction. 

The _Bank_ sample registers a shutdown hook to the JVM and shutdowns _Chainvayler_ when JVM shutdown is initiated. This works fine for demonstration purposes unless JVM is killed with `-9 (-SIGKILL)` switch or a power outage happens.

But obviously this is not a bullet proof solution. A possible general solution is, if an awaited transaction is not received after some time, assume sending peer died and send the network a `NoOp` transaction with that ID, so the rest of the network can continue operating.

## [FAQ and more](#faq-and-more)
