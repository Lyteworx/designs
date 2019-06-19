---
state: in-progress
---

# Node Messaging API

## Summary

The current node api doesn't allow the runtime to know when a message has
been fully dealt with. This proposal looks at a new API for nodes that will
address this issue.

## Authors

 - @knolleary

## Details

The current mechanism for a node to handle messages is as follows:

 - a handler is registered for the `input` event
 - within that handler a node can call `this.send()` as many times as it wants
 - if an error is hit, it can call `this.error(err,msg)`

```
var node = this;
this.on('input', function(msg) {
    // do something with 'msg'
    if (!err) {
        node.send(msg);
    } else {
        node.error(err,msg);
    }
});
```

This simple system has a few limitations that we want to address:

 1. we cannot correlate a message being received with one being sent - particularly when there is async code involved in the handler
 2. we do not know if a node has finished processing a received message
 3. due to 1 & 2, we cannot build any timeout feature into the node
 4. we know of a use case where an embedder of node-red requires nodes to be strictly one-in, one-out, and that should be policed by the runtime. Putting aside the specifics, we cannot provide any such mode of operation with the current model

This design note explores how this mechanism could be updated to satisfy these limitations.

> This design has changed a few times. Having settled on one approach, concerns
> were raised that it was too much of a change from the existing API to be
> practical to adopt. This latest design reflects on that and tries to find
> a more modest approach that satisfies as much of the requirements as possible.

The primary use case for this is for a user to be able to create a Flow that is
notified when a message is successfully processed at a critical point of a flow.
For example, the `Email Out` node has published its message.

### `Node.done(msg,[error])`

A new function is added to the `Node` object that can be used to indicate a node
has finished processing a message.

The function takes up to two arguments:

 - `msg` - the message being marked as complete.
 - `error` - an _optional_ error message

Calling this function with only the `msg` argument will trigger any in-scope `Complete`
nodes. This is a new node type that will be added under this proposal.

If the `error` argument is also provided, then any in-scope `Catch` nodes will
be triggered instead of the `Complete` node.

```
var node = this;
this.on('input', function(msg) {
    // do something with 'msg'
    if (!err) {
        node.send(msg);
    } else {
        node.error(err,msg);
    }
    node.done(msg);
});
```

```
var node = this;
this.on('input', function(msg) {
    // do something with 'msg'
    if (!err) {
        node.send(msg);
        node.done(msg);
    } else {
        node.done(msg, err);
    }
});
```


This will also be made available in the `Function` node.

#### `Complete` node

This design has changed quite a few times. It has bounced between adding a new
node to handle the 'done' events, and reusing the `Status` node.

Having modelled the `Status` node approach, a number of issues were identified that
made it less ideal.

 - Existing flows using Status nodes will suddenly start receiving the 'done' status
   events. If the flows are not expecting them, that could have bad side-effects.
 - A workaround to that would be to add an option to the Status node to opt into
   receiving 'done' events. But that gets messy.
 - Overloading the Status event with the Done event also gets messy when there is
   also an error to report.

Having a new node type to handle the 'done' events is the cleanest way for a flow
author to create a flow that can react to the different types of event - status,
error and done.

This node will be very similar in design to the Catch/Status node. However, it will
*not* offer the default 'Handle all' type of behaviour the others do. The user
*must* target it at specific nodes in the flow. They could chose to select all,
but it would not be a top-level menu option as it is with the Catch/Status nodes.

#### Timeout handling

This is moving to a separate Design note and not part of the delivery of this
design.

[https://github.com/node-red/designs/blob/master/designs/timeout-api.md]()

---

## Alternative Designs

The following designs were considered for this feature and are kept here for
future reference.

### Alternative Option 0: Pass in scoped `send` and `done` functions

This alternative would provide full correlation between calls to `send` and `done`
with the original message. However it has two significant drawbacks that make it
impractical:

 - it is a significant api change that cannot be made backwards compatible. We
   would be forcing nodes to drop support for Node-RED 0.x and will lead to
   a bad user experience.
 - there is overhead in creating the necessary closure for every single message -
   both in terms of memory and through-put.


<details>

If the event handler is registered with three arguments, the runtime will pass in functions that should be used to send or mark the msg as handled.

```
this.on('input', function(msg, send, done) {
    // do something with 'msg'
    if (!err) {
        // send can be called as many time as needed (including not at all)
        send(msg);
        send(msg);
        send(msg);
        // Once complete, done is called
        done();
    } else {
        // If an error occurs, call done providing the error.
        done(err);
    }
});
```

The `done` function takes two arguments: `done(error, message)`.


Usage            | Meaning
-----------------|------------
`done()`         | success. Any `success` node targeting this node will be triggered using the original msg
`done(null,msg)` | success. Any `success` node targeting this node will be triggered using the provided msg
`done(err)`      | failure. Any `catch` node targeting this node will be triggered using the original msg
`done(err,msg)`  | failure. Any `catch` node targeting this node will be triggered using the provided msg



 - a new node will be added to compliment the `Catch` node that can be used to trigger a flow when a node finished processing a message. It's current name is the `Success` node - but it needs to change.
 - this feels the most 'node.js-like'. The presence of a `done` callback is familiar to many apis.
 - the functions can be scoped to the received message so the node does not need to provide the message back
 - the runtime can tell if the handler expects these extra arguments or not, so can adapt its behaviour to match
 - `node.send` should not be used in this case as its use will stop the runtime from being able to correlate message received with message sent. We _probably_ won't enforce this - tbd.
 - `node.error` can still be used as a handler may need to log multiple errors before completing.

> (HN): According to the discussion held on May 17th,
>       if `send` and `done` is omitted from the handler arguments (i.e. original form of handler is used), `done` is implicitly called after callback execution.

**What if `done` is never called?** - If a handler is registered that takes the `send` and `done` arguments, the runtime requires it to eventually call `done` for each message received. Not calling `done` should be considered a bug with the node implementation. The question is what happens if it doesn't get called.

The easy option is to do nothing. But that will allow buggy implementations to exist, so we should avoid this option.

The right approach will be to timeout the function. A timeout would be considered an error and logged as such. The runtime will set a default timeout of **30 seconds (TBD)**. A node will be able to set its own timeout value by setting a property on itself (`this.TIMEOUT = 60000` (TBD)). This also allows a future extension where a user can set custom timeout values per node in the editor (but this proposal does not extend that far today).

> (HN): According to the discussion held on May 17th,
>       fixed timeout value may lead to unpredictable behavior of flows because execution time and order may vary for each execution.  So, we make default timeout behavior of nodes off and add allow specifying timeout of each node independently using new API, say `node.setTimeout(<value>)`.  Global setting of timeout, e.g. `RED.setNodeTimeout(<value>)`, is also useful.

**What if a node that has been timed out then wakes up and calls `done` or `send`?** - should the runtime then block a timed out node from calling `send` or `done` (at least... prevent any messages it then sends from being passed on? I can see use cases for both allowing the message to pass on and for stopping it. Does this need to be a per-node policy? Or a choice made in the editor? Hmmm.

> (HN): According to the discussion held on May 17th,
>       correct handling of early timeout of node is different for each flow.  So, we assume timeout of a node throws exception and can be caught by `catch` node.

> (HN): Note: Because processing of `send` and `done` is on critical path of node processing, we must take care of reducing their execution overhead on implementation.

</details>

### Alternative Option 1: Add a `node.complete` function - v1

<details>

The first proposal is to add a new function to the Node object that can be called when a node has finished handling a message.

```
this.on('input', function(msg) {
    // do something with 'msg'
    if (!err) {
        node.send(msg);
    } else {
        node.error(err,msg);
    }
    node.done(null,msg);
});
```

 - this relies on the user passing msg through - something that could be a source of programming error.
 - it would need clear semantics over when it was called and how it relates to `node.error`.

</details>

### Alternative Option 2: Add a `node.complete` function - v2

<details>

The second proposal is similar to the first, but the `complete` function can also be used to indicate a failure:

```
this.on('input', function(msg) {
    // do something with 'msg'
    if (!err) {
        node.send(msg);
        node.complete(msg, null, msg);
    } else {
        // Log the error, but don't provide the msg obj here
        node.error(err);
        // Provide the err - which will trigger any Catch nodes
        node.complete(msg,err);
    }
});
```

 - this relies on the user passing msg through - something that could be a source of programming error.
 - it would need clear semantics over when it was called and how it relates to `node.error`.

</details>


### Proposal of `function` node extension for Node Messaging API (HN)

In order to add support for the new style message handler, We propose following extension for `function` node.

- add `send` and `done` variable that correspond to newly introduced arguments of new style message handler,
- add `node.setTimeout` function for specifying node timeout in milliseconds.

For compatibility with the old style message handler, we expect old style handler is used for `function` node in default (no need for calling `done` callback function).
The new style handler can be activated by calling `node.useNewStyleHandler()` or selecting checkbox of setting panel of `function` node.

## History

  - 2019-06-19 - another iteration of the design
  - 2019-03-26 - rewritten to cover new design proposal
  - 2019-02-27 - migrated from Design note wiki
