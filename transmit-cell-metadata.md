---
title: Transmitting Cell Metadata in Jupyter Execute Requests
authors: John Lam (jflam@microsoft.com), Matthew Seal (zzz@zzz), Carol Willing (zzz@zzz)
issue-number: <pre-proposal-issue-number>
pr-number: <proposal-pull-request-number>
date-started: 2021-02-10
---

# Summary

This proposal discusses transmitting cell metadata as part of the Jupyter
Messaging Protocol execute message requests. It is up to the kernel to interpret
this metadata as it sees fit. 

# Motivation

By transmitting cell metadata inline with the execute message request,
implementations will have a reliable channel to transmit additional metadata to
the kernel. Extensions will have a place to store additional information that
was often transmitted using magic commands. Here are some use cases for this
proposal:

-   Automatically route requests to an appropriate kernel via libraries like
    [allthekernels](https://github.com/minrk/allthekernels) without need for
    additional metadata within the cell itself
-   Create or find a conda environment without needing to use magics, like
    [pick](https://github.com/nteract/pick)
-   Support polyglot (more than one language/kernel within a single notebook)
    scenarios, like [sos](https://vatlab.github.io/sos-docs/)
-   Provide hints to the kernel for localization purposes, like how the
    ACCEPT_LANGUAGE HTTP header works
-   Provide hints to the kernel about client capabilities, like how web browser
    client capabilities hints works

# Guide-level explanation

Transmitting cell metadata enables many scenarios. Some of the scenarios are
briefly described in the Motivation section. In this section, we'll consider a
scenario where you'd like to run the cell code using a specific kernel. Today
you'd typically have the user include a magic command in the cell to identify
the conda environment. This interferes with other extensions that may want to
use the contents of the cell, e.g., autocomplete providers would now need to be
aware of and ignore the syntax of magics.

## Simple example

For example, in the [allthekernels](https://github.com/minrk/allthekernels)
project, users select the kernel using a `><language>` command:

```R
>python3
1+1
```

But in our example, let's imagine that we use cell metadata to specify the
kernel instead. Now, let's consider a minimal JSON fragment for the above cell: 

```json
{
  "cell_type" : "code",
  "execution_count": 1, 
  "metadata" : {
    "kernel": "python3",
  },
  "source" : "1+1",
}
```

You can see that the cell metadata dict contains an entry that specifices that
the `kernel` is `python3`. But where did the `"kernel": "python3"` metadata come
from? What wrote it into the cell metadata in the first place? So elaborating a
bit more on the user experience here, you could imagine a client extension
providing some additional UI affordances such as a cell drop-down that lets the
user pick from a list of installed kernels on the user's machine. The user picks
one, and the kernelspec is written to that cell's metadata.

In this example, there is also a corresponding `allthekernels` kernel that is
installed on the user's machine that knows how to multiplex between different
kernel processes that are running on the user's machine. When the user runs the
cell, the Jupyter implementation will send an
[execute](https://jupyter-client.readthedocs.io/en/stable/messaging.html#execute)
message to the kernel. 

Here's a minimal representation of the execute message for the above cell:

```js
{
  "header" : {
      "msg_id": "...",
      "msg_type": "...",
      //...
  },
  "parent_header": {},
  "content": {
    "code": "1+1",
    "metadata": {
      "kernel": "python3", 
    },
  },
  "content": {},
  "buffers": [],
}
```
 
In this case the `allthekernels` kernel sees the `"kernel": "python3"` entry
in the message, and locates and activates a child kernel to handle the request,
and passes the message onto the child kernel for processing.

There could be other cell metadata that was transmitted from the client as well.
Some of that metadata could have been put there by client extensions, like in
the case of `allthekernels`. Other metadata could be put there by the Jupyter
implementation itself, e.g., language or client capabilities like screen size.

## Metadata Key Conflicts

There is the possibility for conflicts across extensions that want to add their
own cell metadata to notebook file. We recommend that extensions namespace their
metadata keys to minimize the possibility of conflicts between extensions. For
example, in the `allthekernels` case it could look like:

```json
{
  "cell_type" : "code",
  "execution_count": 1, 
  "metadata" : {
    "allthekernels:kernel": "python3",
  },
  "source" : "1+1",
}
```

## Kernels declaring the need for Cell Metadata

Kernels should have a way to declare that they require metadata to be sent. For a 
kernel like `allthekernels` it *needs* to have 

TODO: in detail section.


# Reference-level explanation

This is the technical portion of the JEP. Explain the design in
sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

[Execute](https://jupyter-client.readthedocs.io/en/stable/messaging.html#execute)
message to the kernel. The general form of a message is:

```js
{
  "header" : {
    "msg_id": "...",
    "msg_type": "...",
    //...
  },
  "parent_header": {},
  "metadata": {},
  "content": {},
  "buffers": [],
}
```

Different message types have different schemas for the content dict. The schema
of the content dict of an Execute message follows:

```js
content = {
  // Source code to be executed by the kernel, one or more lines.
  "code" : str,

  // A boolean flag which, if True, signals the kernel to execute
  // this code as quietly as possible.
  // silent=True forces store_history to be False,
  // and will *not*
  //   - broadcast output on the IOPUB channel
  //   - have an execute_result
  // The default is False.
  "silent" : bool,

  // A boolean flag which, if True, signals the kernel to populate history
  // The default is True if silent is False.  If silent is True, store_history
  // is forced to be False.
  "store_history" : bool,

  // A dict mapping names to expressions to be evaluated in the
  // user's dict. The rich display-data representation of each will be evaluated after execution.
  // See the display_data content for the structure of the representation data.
  "user_expressions" : dict,

  // Some frontends do not support stdin requests.
  // If this is true, code running in the kernel can prompt the user for input
  // with an input_request message (see below). If it is false, the kernel
  // should not send these messages.
  "allow_stdin" : True,

  // A boolean flag, which, if True, aborts the execution queue if an exception is encountered.
  // If False, queued execute_requests will execute even if this request generates an exception.
  "stop_on_error" : True,
}
```

We propose the addition of a new `metadata` dict to the `content` dict schema in
nbformat. Cell metadata would be transmitted to the kernel in this dict.
Consider the following cell in a notebook:

```js
{
  "cell_type" : "code",
  "execution_count": 1, // integer or null
  "metadata" : {
    "pick_conda_env" : "python38", // identify the conda env to run this cell in
    "collapsed" : True, 
    "scrolled": False, 
  },
  "source" : "1+1",
  "outputs": [{
    // list of output dicts (described below)
    "output_type": "stream",
    ...
  }],
}
```

The contents of the cell `metadata` dict in the notebook would be transmitted as
part of the EXECUTE message in the proposed `metadata` dict within the `content`
dict:

```js
{
  "header" : {
      "msg_id": "...",
      "msg_type": "...",
      //...
  },
  "parent_header": {},
  "content": {
    "code": "1+1",
    "metadata": {
      "pick_conda_env": "python38", 
      "collapsed": True, 
      "scrolled": False, 
    },
  },
  "content": {},
  "buffers": [],
}
```
# Rationale and alternatives

- Why is this choice the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does this feature exist in other tools or ecosystems, and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your JEP with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.


# Unresolved questions

- What parts of the design do you expect to resolve through the JEP process before this gets merged?
- What related issues do you consider out of scope for this JEP that could be addressed in the future independently of the solution that comes out of this JEP?

# Future possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the Jupyter community at-large. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how the this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
JEP you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future JEP; such notes should be
in the section on motivation or rationale in this or subsequent JEPs.
The section merely provides additional information.
