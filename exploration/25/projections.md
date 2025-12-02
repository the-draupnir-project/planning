# Projections Exploration Report

## Context

In order to implement previews for
[policy list subscription](https://github.com/the-draupnir-project/planning/issues/2),
we would need a generic _revision_ interface that is accessible on the
_protection_ interface. Having a generic _revision_ type would also make it
easier to derive new sets of revisions for the purpose of observing the effects.
For example, viewing the effects of subscribing to a _policy room_ on the
_member ban synchronisation protection_ would require _revising_ the _policy
list revision_ for Draupnir's _watched policy rooms_, the _membership policies
revision_, and a new _intent_ _revision_ that could explain what the protection
was going to do.

## Unknowns

## Experiment

We generalised the _revision_ and _revision issuer_ abstraction into a generic
_projection node_ and _projection_ type. This work was inspired by the _replica_
concept from the Persistent Replica Runtime proposal. We then implemented a
_projection_ for the _member ban synchronisation protection_ and another for the
_server ban synchronisation protection_.

Policy previews were then derived by destructuring the active protections from
the protections manager. Then hand coding a function to create deltas from each
input starting from the _policy list revision_ for Draupnir's _watched policy
rooms_.

### Exploration points

- Determine whether if it is reasonable to rewrite all the revisions in the data
  flow from the _policy list revision issuer_ all the way to the _protection
  projection intents_.
- Determine if cloning a snapshot of the pipeline and reducing with arbitrary
  data in isolation of the original is possible.
- Determine how two inputs to a _projection_ interact when the data flow of two
  inputs are in a fork because they share a transient dependency. And thus the
  _projection_ is convergence of the fork.
- Determine how to handle hooks for plugins without making the plugin handle the
  allocation of the listener. And while keeping the "hook registry" generic.

## Outcome

- A generic `Projection` and `ProjectionNode` type were successfully created and
  used.

- We were unable to arbitrarily clone a pipeline of _projections_ because only
  the projection intents of protections were moved to the new abstraction.
  However, we were able to reduce the deltas from each source revision to
  provide a preview. And this was surprisingly simple. So it should be straight
  forward to make this process generic later. Remember, the trick is to focus on
  the deltas, not the projection nodes.

- We were unable to experiment with how two inputs to a _projection_ interact
  around a fork and merge. The suspicion I have is that if there needs to be
  synchronisation or progress updates then the revision is badly designed in
  some way. But it does need investigating.

- When experimenting with _projection nodes_ it became clear that in some
  situations the reducer for producing the delta had similar code as the reducer
  for producing the next _projection node_ when reducing that delta. We thought
  about what it would look like to merge the two steps into one join. But it
  became apparent that doing so would mean maintaining an independent code path
  that is rarely used for reducing the delta into a projection node. Ie if you
  wanted to restore a projection node from the deltas it produces then you would
  have to use a rarely used code path. Which is very unideal. The only way you
  could do this is by only reproducing projection nodes from input deltas, not
  the output deltas. It needs thought about whether this is something we want to
  do for PRR.

- Additionally we spared some thought for how to persist projection nodes and
  run reducers without loading the entire projection. One idea was to shard the
  projection node data and input delta and only run part of the reducer at a
  time. But it's not clear to me how feasible it is to find common keys in input
  to partition on especially when there are multiple input projections. It would
  be highly desirable to do this over adding IO into reducer code which is meant
  to be deterministic.

- While we didn't have time to explore implementation of a protection hook
  registry. We think that it is possible to create a protection hook registry
  where each handle on a protection after construction is connected up with the
  associated input. And disconnected when the protection is disposed. We think
  this can be done by maintaining a metaobject with a description of each hook
  that takes the protected rooms set, context, and protection itself as input.
