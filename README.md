# Domain Driven Design (DDD) in Clojure
*An example implementation with explanation*

An example implementation of Domain Driven Design in Clojure with lots of information about DDD and its concepts and how it's implemented in Clojure along the way.

We're taking the same modeling example as from this blog: http://blog.sapiensworks.com/post/2016/07/14/DDD-Aggregate-Decoded-2 and re-doing it in Clojure. We also add the additional requirements from the follow up blog post: http://blog.sapiensworks.com/post/2016/08/19/DDD-Application-Services-Explained

In a nutshell, we're trying to build a banking application, and we know that it needs to support transferring money from one account to another. We've talked with the business users about how they currently manage their bank and what concepts and rules and actions and relationships they currently seem to make use of to manage their bank.

As part of trying to become a domain expert, we've started to learn about the ubiquitous language that the business users are used too. We know there are Accounts, there's this concept of Debit and Credit, there's special identifiers for referring to Accounts, they have the concept of Transfers that tracks all debits and credits between accounts. And most importantly, users can request for money to be transferred from one account to another.

This is what we thus model in this example banking application, with our only feature for now being able to transfer money between existing accounts.

The code base is structured in classic DDD patterns:

* Application Service: `clj-ddd-example` namespace, also the main namespace that can be considered the API to clients, side-effectfull.
* Domain Model: `clj-ddd-example.domain-model` namespace, where we define our domain model, pure.
* Domain Services: `clj-ddd-example.domain-services` namespace, where we define our domain services, pure.
* Repository: `clj-ddd-example.repository` namespace, where we maintain our application state, side-effectful.

Refer to each namespace's doc-string to get a full explanation of each of these concepts, and how it is implemented in Clojure.

## [Nice discussion after initial annouce of that project](https://clojureverse.org/t/domain-driven-design-ddd-in-clojure-an-example-implementation/8802/8)

**didibus**

An example implementation of Domain Driven Design in Clojure with lots
of information about DDD and its concepts and how it’s implemented in
Clojure along the way:

GitHub - didibus/clj-ddd-example: An example implementation of Domain
Driven

It implements a small banking application in the style of DDD that
lets users transfer money between accounts.

**zcaudate**

@didibus, a couple of questions:

In the example, do you have a schema or a relational modal of some sort?
what are some of the backing stores that the ddd example intends to target?

**didibus**

In the example, do you have a schema or a relational modal of some
sort?

The domain logic is modeled hierarchically, with entities, value
objects and aggregate roots all being maps.

The datastore in my example I modeled relationally, so I imagined
someone would use a SQL database. You can see it here:
clj-ddd-example/repository.clj at
e99aa7b7013529e669ce06d400765ddd10b7214f · didibus/clj-ddd-example ·
GitHub 16

I’m pretending that you’d have two tables:
1. `account-table` with columns: `account-number`,
`account-balance-value`, `account-balance-currency` and
2. `transfer-table` with columns: `transfer-id`, `transfer-number`,
`debited-account-number`, `credited-account-number`,
`transfered-amount-value`, `transfered-amount-currency`,
`creation-date`

> what are some of the backing stores that the ddd example intends to
> target?

So like I said, it would be that the database is a SQL database, say
MySQL. In DDD, you often also apply CQRS, or a simplified form,
basically your reads and writes are separated.

In a simple form, you have one SQL database with the above two tables
I mentioned. The repository gets an account from the account table,
and it commits a transfer by updating the account table and inserting
to the transfer table.

You’ll see that I recently updated my example to show a more realistic
design that handles the concurrent transfers better.

If your database is MySQL, the way I have it now in the application
code is using eventual consistency. When you get an account, it could
read a stale balance from it, because the transfers are not
synchronized. But the domain model for debiting and crediting an
account returns an event that describes the change, and not the new
state of the account.

That means that the application service when it receives that event
back from the domain model, it uses the repository to commit the
change, at that point the repository takes care of atomically applying
the change to both accounts and inserting the transfer.

This is eventually consistent, but can during the “inconsistency
window” allow some debits below 0$, resulting in possible negative
balances for some users.

We assume the domain is fine with that, by say supporting overdraft
fees for example.

Now if using only a MySQL database, you could also simply in the
application service wrap the whole thing in a transaction and take a
FOR-UPDATE lock on the account rows when you get the accounts, and
commit the transaction in the commit transfer at the end. This would
be strongly consistent, and use a pessimistic locking strategy. The
MySQL transaction handles that using two phase commit.

If you did this, you don’t need to model domain model changes as
change events. You can just have the domain like on my prior example
return the new state of the entites and aggregates, and the repository
just overwrites the existing state with the new state. That is a
simpler design in general, and it is well suited for supporting user
frontend, because it is strongly consistent the user doesn’t see what
can appear as weird glitches as the data becomes eventually
consistent.

You could also choose a strongly consistent, but optimistic locking
approach. This would make more sense if you use a NoSQL database like
say DynamoDB, which don’t always support locks across documents.

In that scheme, you’d get accounts but the accounts would have a
version number along with it. You’d make the changes, and when you
commit the transfer in the repository, it will perform an atomic
“compare-and-set”, which basically goes, update unless version is
newer than what we read. When that happens, you’d retry the entire
application service logic, back from the get accounts calls, you’d do
this until you finally succeed.

Here too, you wouldn’t need to model domain changes as change events,
and it is sufficient to simply have domain changes return the new
state of the entity/aggregate. The difference is depending on your
concurency patterns it could avoid unnecessary locks and be faster,
but if you have a high concurency it could actually be slower as well.

There’s another option for strongly consistent, you can take a
distributed lock outside your database, which locks the application
service command itself. If your running in a single instance, you’d
just wrap the whole application service in a (locking ...), but in a
mutli-instance setup you’d need a distributed variant. You can be
smart here and lock on the account-numbers so it’s only per-account
locks.

Finally, the eventually consistent approach I have now in the example
is also a great fit for Kafka or other distributed streams like AWS
Kinesis. You can still have MySQL as your source of truth persistent
database, but you’d front it with Kafka. The change events wouldn’t
get immediately committed by the application service as it gets them
from the repository, instead the application service would publish the
events to Kafka. And a separate consumer on Kafka would handle the
messages by commiting them to your MySQL database.

You can on top of that solution implement a distributed lock like I
said for cases where you’d want strong consistency, though it kinda
defeats the purpose a bit in my opinion, in that this approach is good
for scaling out your writes, because the caller isn’t blocked waiting
for the writes, they are made async and buffered in Kafka.

There might be more, I guess the point is that really it works with
many different approaches. Since your domain model and services are
all pure, you can adapt around it to have the state stored wherever
and however you want. That doesn’t mean you can easily swap one
storage for another, because your application service is still very
coupled to the details of your repository which is coupled to which
storage solution it is designed on. And even your domain model is
slightly coupled in at least choosing if it needs to return change
events or new states and possibly for optimistic locking it might have
to have versions as fields on the entities as well, though that could
possibly be handled as metadata it in the application service itself.

One more thing, like I said, DDD seperates reads from writes. So this
domain model isn’t meant for queries and read-only use cases. It is
meant for updates or insert use cases.

For queries and read-only use cases, my example doesn’t show it, maybe
I’ll add it later to be more complete. But basically you’d have
something called a Finder, and that would just run queries on your
database however best works with your database, and it can return the
query results in whatever view structure best works for your query use
case, it wouldn’t returns things in the structure defined by the
domain model. So this Finder can be a graphQL based API, or it could
just have a bunch of functions for various queries like:
(find-transfers-over 200 :usd).

In this design, your queries can run on the same database as your
writes, but they don’t have too. You can replicate your data in other
datastores best suited for querying. Even in the strongly consistent
updates and insert cases, this can work, your queries could still be
eventually consistent as the replicas read datastore might be behind,
but when you go to make an update, you’d not use the query data to
compute the update, you’d go back to your repository and call get
again for all the entities you intend to change and change those, so
that can be strongly consistent again.

Hope that answered your question.

**zcaudate**

thanks for the write up. I think I understand the design.
- Is there a possibility of data corruption? (say if a service
  shutdown or if kafka somehow breaks).
- Have you tried benchmarking an implementation?
- I try to avoid locking when possible, especially say if multiple
accounts need to be locked for a given transaction. If one or more of
those accounts are being constantly updated there may be a deadlock
senario, especially if the system starts spamming cas operations
because of locked rows.  if transactions need to be processed in some
sort of order, then the locking/cas approach gets worse.

How would the eventually consistent model handle something a bit more
complex? ie, a transaction involving multiple steps:
- Check Account A has enough Balance to Transfer
- Check Account B has a valid Account to accept the Asset, if not,
  create the account.
- Transfer if valid, otherwise deduct the transfer fee from Account A.

**didibus**

> Is there a possibility of data corruption? (say if a service
> shutdown or if kafka somehow breaks).

It depends how you’re publishing the change events, if you look at my
example right now, I publish an event of events, so technically
there’s no possibility that half the events for the transaction are
published and the app crashes failing to publish say the event about
updating one of the accounts.

Some people do like to have one Kafka stream per entity/aggregate
though, I guess there’d be a small risk then that the app crashes
prior to having published all the events.

Kafka is designed itself to recover from crashes, I believe it writes
its stream to disk so it doesn’t lose data in case of a node crash,
and you can set a replication factor that duplicates all messages to
multiple nodes, so if one has a disk explode the other node would have
a copy of the data.

> Have you tried benchmarking an implementation?

Have not no. While I mention Kafka, I actually don’t use it myself
:stuck_out_tongue:

At my work, I have familiarity with AWS, and personally I’d lean on
AWS SQS if that is an option to you over Kafka. The simplicity of
managing an SQS queue is just bliss. If you use a FIFO queue, you get
exactly once delivery and guaranteed ordering, and all the normal
replication and fault tolerance you’d expect. But I don’t know the
cost relative to running your own Kafka cluster or using AWS MKS or
even Kinesis though. SQS FIFO queues have a RPS of 300 r/s which you
can increase to 3000 r/s if you batch messages in batches of 10, and
there’ a high throughput mode that goes up to 3000 r/s no batch, and
30000 r/s with batch. It has under 100ms latencies per send/receive.

> How would the eventually consistent model handle something a bit
> more complex? ie, a transaction involving multiple steps:
> - Check Account A has enough Balance to Transfer

You need to be creative :stuck_out_tongue: , you are pretty much
trading correctness for scale/availability, and you might need to push
back on the exact requirements.

In my example I do “Check Account A has enough Balance to Transfer”,
but it is best effort, because it could be checking an old version of
the Account for which there is a queued-up change not reflected
yet. The solution is that the “bank” uses overdraft fees to cover this
edge case, so if someone transfers more money than they have, the bank
will basically cover the transfer out-of-pocket, and then charge the
customer money + interest for the negative balance. If the bank sees
you went under 0, it would charge you overdraft fees, if you managed
to refill it before it found out, maybe you just get away with it, but
the balance is positive again and all is good.

> Check Account B has a valid Account to accept the Asset, if not,
> create the account.

In this case, I would push back on the requirement, I’d say, look, if
Account B at the time we check it doesn’t have a balance in the
correct currency (assuming that’s the “Asset” we are checking for),
we’ll just error and refuse the transfer. The user either has to go
add one first, or they just retry and this time (if one was being
created) it will succeed.

> Transfer if valid, otherwise deduct the transfer fee from Account A.

Find another way to monetize :stuck_out_tongue:

With eventual consistency, you can’t guarantee that your reads are
getting the most up to date view, all the workarounds are simply ways
to make things strongly consistent temporarily when you need too and
go back to being eventually consistent otherwise.

Generally, I think people simply accept this, and deal with the
limitations, because they’d rather scale instead. Also, pre-computers,
business and users and humans constantly delt with eventual
consistency, information slowly trickles from one bank teller to the
next, one bank to the other, etc. And similarly, business/human
processes simply figured out processes that worked within that
reality. Eventually consistent software systems need a similar thing.

That’s why it’s not for every use-case, but it works for some.

**zcaudate**

sweet. thanks for that.

**mdiin**

Thank you @didibus for building this example of DDD in Clojure. It
highlights the core concepts and how they help structure the
functional core and imperative shell of an application. Having spent
some time over the last year working with implementing DDD, I wish I
had had your example back at the beginning. I love the clarity of your
doc strings, and how they explain the concepts along with the code.

I have been meaning to write this since you first published your
example, but other things always got in the way.

# About this fork

I hope to explore different repository implementation and getting more
hands of experience in Clojure and

_Have fun!_
