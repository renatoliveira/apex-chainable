# Apex Chainable [![Codacy Badge](https://app.codacy.com/project/badge/Grade/7024ec2e01c24c03a323e565e029a5a6)](https://www.codacy.com/gh/rsoesemann/apex-chainable/dashboard?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=rsoesemann/apex-chainable&amp;utm_campaign=Badge_Grade)

<a href="https://githubsfdeploy.herokuapp.com?owner=rsoesemann&repo=apex-chainable-batch">
  <img alt="Deploy to Salesforce"
       src="https://raw.githubusercontent.com/afawcett/githubsfdeploy/master/src/main/webapp/resources/img/deploy.png">
</a>

Apex Batches can be chained by calling the successor batch from the `finish()` method of the previous batch.
But such hardcoding makes this model inflexible. It's hard to build the chain from outside, neighter from a central class
nor on runtime dependant on business logic.

The same applies when the `execute()` method of `Schedulable` or `Queueable` classes call other classes.

## With `Chainable`

The `Chainable` wrapper class of this repository overcomes those drawbacks.

 - No need to hardcode successor batch in `finish()` method
 - Created batch chains of arbitrary length without changing existing Batch classes
 - Support `Batchable`, `Queueable` and `Schedulable` classes as chain members
 - Allows sharing and passing of variables between chain members

```java
      new FirstBatch().setShared('result', new Money(0))
            .then(AnotherBatch())
            .then(QueueableJob())
            .then(ScheduledJob())
            ...
            .execute();
```

## Without `Chainable`

```java
class FirstBatch implements Batchable<SObject> {
    Iterator<SObject> start(BatchableContext ctx) { ... }

    void execute(BatchableContext ctx, List<Account> scope) { ... }

    void finish(BatchableContext ctx) {
        Database.enqueueBatch(new SecondBatch());
    }
}
```

```java
class AnotherBatch implements Batchable<SObject> {
    Iterator<SObject> start(BatchableContext ctx) { ... }

    void execute(BatchableContext ctx, List<Account> scope) { ... }

    void finish(BatchableContext ctx) {
        System.schedule('name', cron, new ScheduledJob());
    }
}
```

## Implementation & example

To use the Chainable interface the batchables, queueables or schedulables should extend their corresponding classes from the library: `ChainableBatch`, `ChainableQueueable` and `ChainableSchedulable`.

For instance, if the processing is to be done on accounts and then contacts on the Salesforce org, the implementations might look like this:

```java
public without sharing class AccountProcessing extends ChainableBatch {
    // implement the counterpart of the `start` method from the original
    // Database.Batchable
    public override System.Iterable<Object> start(Chainable.Context ctx) {
        return [SELECT Id, Name FROM Account];
    }

    // implement the counterpart of the `execute` method from the original
    // Database.Batchable
    public override void execute(
        Chainable.Context ctx,
        System.Iterable<Object> iterableInstance
    ) {
        for (Account a : (List<Account>) iterableInstance) {
            System.debug('Account Name: ' + a.Name);
        }
    }

    // implement the counterpart of the `finish` method from the original
    // Database.Batchable
    public override void finish(Chainable.Context ctx) {
    }

    // you can define the the number of records to be processed on each
    // iteration
    protected override Integer batchSize() {
        return 1;
    }
}
```

Assuming that a "ContactProcessing" class also exists, they can be chained as:

```java
new AccountProcessing().then(new ContactProcessing()).execute();
```

## Deferring

When all the "links" in the chain cannot be completely identified beforehand in order to assemble them in a single chain and trigger its execution, the chain execution can be *deferred* until the end of the transaction. All chainable processes that have been *deferred* will be automatically chained together in a **single** chain and executed sequentually.

```java

// automation 1
new FirstBatch()
        .then(AnotherBatch())
        .setShared('result', new Money(0)) // shared variables will be available across other following deferred chainables
        .executeDeferred();

// automation 2
new QueueableJob()
        .then(ScheduledJob())
        ...
        .executeDeferred();

// the framework would internally build the chain in a separate transaction with a definition like this
new FirstBatch()
        .then(AnotherBatch())
        .setShared('result', new Money(0))
        .then(QueueableJob())
        .then(ScheduledJob())
        .execute();

```

### Considerations

In order to leverage the deferring of the Chainable instances there are some nuances compared to its direct execution.

* Override the `getDeferArgs` and `setDeferredArgs` if the class receives external input before its execution via constructor parameters or setters to serialize and deserialize them (See the `SampleDeferArgQueueable`).
* Must have a no-arg constructor (if no explicit constructor exists, the default implicit one will be used)
* Must not be an inner class (due to difficulty on dynamic name inferring)