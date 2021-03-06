# Transactional Firebase

[![Circle CI](https://circleci.com/gh/vacuumlabs/firebase-transactions.svg?style=svg&circle-token=bd6366dee306c78c644fc6458085e673fe163723)](https://circleci.com/gh/vacuumlabs/firebase-transactions)

Firebase with transaction? Real transactions with almost all of the ACID goodness? Coool.

[Tutorial] (https://github.com/vacuumlabs/firebase-transactions/blob/master/wiki/tutorial.md)

[Documentation] (https://github.com/vacuumlabs/firebase-transactions/blob/master/wiki/documentation.md)

## Rationale
Firebase is doing great job syncing your data, it provides as-easy-as-it-gets read and write API and
it allows you to move blazingly fast with your project. Where's the catch? At least for us, as our
projects grow bigger, every one of them has at least some small part of it, where the transactional behavior is
highly desired. And transactional behavior is certainly not something Firebase is particularly good at.

#### The need for transactions
Ladies and gentlemen, say hello to the really old problem that
relational-database old-fashioned guys use to scare NoSQL cool-kids for almost an decade:
transferring funds.

John want to transfer 10 bucks to Emily: we have to subtract 10 bucks from his account and add 10
bucks to Emily's.

As easy as this may sound, there are several issues to think of:
- What if John does not have enough funds on his account?
- What if John has enough funds on his account but tries to double-spend them (i.e. he transfers the
  money to Alice in the very moment as he transfer them to Emily)
- What if Emily's account got closed (or, she close the account at the very same moment when the
  transfer happens)
- What if the client lose their internet connection in the middle of this whole operation?

Managing all this complexity, is far from easy; and it is only simple transfer-funds what we are
talking about! If only we have some transactional mechanism, that would allow us to focus only on
the business logic and take care of this unwanted additional complexity!

#### Validating things is HARD
Let's talk about the transfer-funds example a little more. Even if we somehow ignore
non-transactional problems, another problem exist! How do you set-up validation rules such that
John can transfer funds to Emily?

Normally, you'd expect John being able to read (and maybe under some circumstances, modify) his own
account. Definitely, he shouldn't be able neither read nor write to other users accounts. But, then
how can he actually transfer the funds?

This is not a subtle issue. You may spend months by happy hacking with Firebase and just
a few days before the initial release of your cool app, you may find out that securing it may be
really complicated, sometimes even not possible without doing some compromises such as denormalizing
the data (even more) or getting rid of using useful indexes and thus compromising the speed of the app.

## Firebase-Transactions in a nutshell

This is what happens, if client wants to do some transactional job:

- Instead of directly manipulating Firebase, client 'submits' action-descriptor. This object
  describes what action client intends to do and typically also some data the action needs
- Action-descriptor is pushed by client to the specific location within the Firebase
- The server spots new transaction-descriptor and look for a correct handler
- Once handler finishes doing transaction, the client is notified

Look up the tutorial to see, how individual steps are implemented.


## ACID, or what guarantees do you have when you write transaction-handler-code:
- Transaction is either performed fully, or none at all (Atomicity)
- While performing transaction A, transaction B can not read not-yet-committed writes of A. However,
  A can read it's own writes and got correct answers (Isolation)
- Once transaction is committed, it's definitely there (Durability)
- If transaction is aborted, none of it's writes will ever be visible
- Feel free to ensure any consistency invariants; if you find anything smelly, you can abort the
  transaction any time.

## Limitations

- Non-transactional part of the system may observe inconsistent state - i.e. if transaction makes 10
  writes, theoretically it is possible to observe (only) some of these writes already applied. Note
  however, that a) this can not happen only for non-transactional part of the system and b) even if
  transaction makes those 10 writes during 10 second interval, the inconsistent state will be
  present for significantly less time

- When writing transaction handler, you may use only some set of Firebase operations, at the moment:
  read, set, update, push. You mustn't use ad-hoc querying of your database (for example, by
  orderByX)

## Benchmarks (poor mans)

Based on randomized_complex test, I was able to process cca 50 transactions per second (each transaction
does 6 read / write operations on average). Of course, if there are lot of conflicts, the speed can
decrease significantly. The limiting factor turned out to be my CPU (not my network speed, nor the
Firebase latency), so using some proper server as a server (and not mine laptop as a server), one
should be able to get even better results.


## Some Q&A

Q: Do I really need this?

A: If you don't, good for you! However, from our experience, (at least) some transactional requirements arise
in almost any bigger-than-hello-world app. Need to implement payments properly? Transaction. Need to
conditionally update at two places at once? Transaction, probably. Your validation rules becomes too
complex? Transaction can help.

Q: Is there some best-practice how Firebase-Transactions should be used?

A: Split your data into those that need transactional updates or complex validations (part 1) and the
rest of them (part 2). Part 1 data should be editable only by transactor (i.e. not writable by any
client). Part 2 data can be handled as it is now. Being so strict about part 1 may sound little bit
too restrictive but usually it does not make sense to mix transactional and ad-hoc updates to a
field.

Q: In the beginning, you said that validations are even bigger problem, than achieving transactional
behavior. How do I do validations when using Firebase-Transactions?

A: First, 'transaction' collection (i.e. those place, where clients pushes transactions) should be
'push only': clients shouldn't be able read or modify submitted transactions. Second, 'userId'
attribute should be present in all transactions clients are submitting; you should also validate,
that presented 'userId' matched with the user, who is authenticated in Firebase (this can be
achieved easily). Finally, once you have proper userId in the transaction data, all you have to do
is to write the transaction handler properly; if user tries to do something not-allowed, just abort
the transaction.

Q: Isn't all of this against Firebase principles? Don't you lose all of the Firebase features, when
doing this?

A: Quite the opposite, such approach buys you the best from both worlds: you can get move really fast and
get all the Firebase goodness on the majority of your data, while still being able to handle the
mission-critical operations transactionally.

Q: I'd like to integrate with some external, statefull API (payment processor). Any chance I can do
something two-phase-commit-ty with this?

A: I'd love to see a pull-request for this.

## License

See [LICENSE] (https://github.com/vacuumlabs/firebase-transactions/blob/master/LICENSE)
