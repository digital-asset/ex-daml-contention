# Sample code

**This repo contains sample code to help you get started with DAML. Please bear in mind that it is provided for illustrative purposes only, and as such may not be production quality and/or may not fit your use-cases. You may use the contents of this repo in parts or in whole according to the BSD0 license:**

> Copyright © 2020 Digital Asset (Switzerland) GmbH and/or its affiliates
>
> Permission to use, copy, modify, and/or distribute this software for any purpose with or without fee is hereby granted.
>
> THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.

# Handling Contention in DAML

This project demonstrates issues around contention in DAML and
possible solutions for handling them.

## Setup

First build the project and start sandbox via
`daml start --json-api-port=none --start-navigator false --sandbox-option --log-level=info`.
This will create contracts with KeyIds 0 up to 999.

Wait a couple of seconds until you see `[DA.Internal.Prelude:540]:
"Initialization finished"` in the terminal. At this point all
contracts have been created.

If you want to reset the ledger simply kill the `daml start` process
and run it again.

## The Issue

With a fresh ledger you can run the following command

```
daml script --dar .daml/dist/daml-contention-1.0.0.dar --ledger-host localhost --ledger-port 6865 --script-name OneShot:oneShot --input-file oneshot.json
```

This will call `Inc` on all contracts with key ids 0 up to 999 in a
single transaction using the `Batch` template. This should run through
successfully since no other ledger clients are active.

Now to simulate contention, start a DAML trigger using

```
daml trigger --dar .daml/dist/daml-contention-1.0.0.dar --ledger-host localhost --ledger-port 6865 --trigger-name ContentionTrigger:contentionTrigger --ledger-party p
```

This will start a trigger that will repeatedly call `Noop` on all
contracts with ids between 100 and 200 and therefore create
contention.


Next run the script again using

```
daml script --dar .daml/dist/daml-contention-1.0.0.dar --ledger-host localhost --ledger-port 6865 --script-name Scripts:noLock --input-file party.json
```

You might get lucky and it works the first time but if you run it
again it will almost certainly fail since there is too much
contention. The exact error message can vary but one potential error is the following:

```
Error: User abort: Submit failed with code 10: Could not find a suitable ledger time after 0 retries
```

## Solving Contention with a Lock

To solve this issue we introduce locks for every hundred contracts, so
there is one lock for contracts 0-99, one for 100-199, ….

We wrap the `T` template in a `TLock` template that provides the same
choices but first checks the lock. If the lock is active, we simply do
nothing and return an error that the caller can use to retry later.
If the lock is not active, we proceed as before.

Our modified trigger in `LockTrigger` then uses `TLock` instead of `T`.

Finally, we modify our script to first acquire the lock and send
separate batches for each set of 100 messages. Once we have processed
a batch we free the lock again.


To run this, first kill the `daml start` process and then start it
again.

Now start the new trigger using

```
daml trigger --dar .daml/dist/daml-contention-1.0.0.dar --ledger-host localhost --ledger-port 6865 --trigger-name LockTrigger:lockTrigger --ledger-party p
```


If you run the old DAML Script, i.e., the one that does not acquire a lock you will still get the error. Try it out using

```
 daml script --dar .daml/dist/daml-contention-1.0.0.dar --ledger-host localhost --ledger-port 6865 --script-name Scripts:noLock --input-file party.json
```

However, if you use the new DAML Script via

```
daml script --dar .daml/dist/daml-contention-1.0.0.dar --ledger-host localhost --ledger-port 6865 --script-name Scripts:withLock --input-file party.json
```

it should go through. While the batch is running our trigger will see
that the lock is active and simply do nothing.

## Automatic batching using a trigger

In the previous example, we split things up in our DAML Script and
handled locking and unlocking there. However, that can get a bit
tedious.  To simplify this, we can handle all the logic for splitting
up a large batch into smaller batches and for managing locks in a
trigger. The trigger will listen for a specific template that lists
the KeyIds of the whole batch and the desired size of the smaller
batches. It then processes the smaller batches one by one, locking and
unlocking before and after as required.

To try it out first restart the `daml start` process.

Next, start the auto batching trigger using

```
daml trigger --dar .daml/dist/daml-contention-1.0.0.dar --ledger-host localhost --ledger-port 6865 --trigger-name AutoBatch:autoBatch --ledger-party p
```

Now create a request for the trigger by running the following DAML Script:

```
daml script --dar .daml/dist/daml-contention-1.0.0.dar --ledger-host localhost --ledger-port 6865 --script-name Scripts:autoBatch --input-file party.json
```

You should see the following output printed from the trigger:

```
[DA.Internal.Prelude:540]: "Initializing state"
[DA.Internal.Prelude:540]: "Acquiring lock"
[DA.Internal.Prelude:540]: "Executing batch number: 0"
[DA.Internal.Prelude:540]: "Acquiring lock"
[DA.Internal.Prelude:540]: "Executing batch number: 1"
[DA.Internal.Prelude:540]: "Acquiring lock"
[DA.Internal.Prelude:540]: "Executing batch number: 2"
[DA.Internal.Prelude:540]: "Acquiring lock"
[DA.Internal.Prelude:540]: "Executing batch number: 3"
[DA.Internal.Prelude:540]: "Acquiring lock"
[DA.Internal.Prelude:540]: "Executing batch number: 4"
[DA.Internal.Prelude:540]: "Acquiring lock"
[DA.Internal.Prelude:540]: "Executing batch number: 5"
[DA.Internal.Prelude:540]: "Acquiring lock"
[DA.Internal.Prelude:540]: "Executing batch number: 6"
[DA.Internal.Prelude:540]: "Acquiring lock"
[DA.Internal.Prelude:540]: "Executing batch number: 7"
[DA.Internal.Prelude:540]: "Acquiring lock"
[DA.Internal.Prelude:540]: "Executing batch number: 8"
[DA.Internal.Prelude:540]: "Acquiring lock"
[DA.Internal.Prelude:540]: "Executing batch number: 9"
```

Our request consists of a 1000 requests and we specified a batch size
of 100 so it takes 10 smaller batches to process everything.
