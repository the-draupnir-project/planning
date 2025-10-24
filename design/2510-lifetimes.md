# Lifetimes

## Background

Draupnir protections have ad-hock lifetime management through the use of
`unregisterListener` methods that are called by owners.

This appears to work well, except they do allow classic races in the owner, for
example the `ProtectionsManager`.

Consider the `startProtection` and `changeProtectionSettings` methods. These
methods create a new protection after awaiting for some asynchronous operation
to complete.

Unfortunately, this allows for the `unregisterListener` hook to be called during
the operation. And then for a new protection to be instantiated which will not
be cleaned up. Since the cleanup has already happened.

This is a classic violation of RAII, since the connection of the protection to
the protection manager is done after the protection is constructed, rather than
as a part of construction.

## Proposal

We introduce a new _Lifetime_ abstraction that tracks a set of callbacks, and
whether `Symbol.asyncDispose` has been called.

_Lifetimes_ are hierarchical. A child lifetime can be created with a _Lifetime_
that is linked to the parent. This is important because protections usually
register their own resources, particularly with the timeout scheduler.

Every dispose handle can be called more than once, even if cleanup has already
occurred. But the cleanup operation only happens once. This lets the dispose
hook be used in multiple places.

New resources registered with a lifetime while cleanup is taking place, or after
cleanup has taken place, will immediately be disposed. This keeps cleanup
idempotent.

When a child lifetime cleans up, it calls the parent to deregister itself. This
prevents memory leaks in situations where child lifetimes have some autonomy in
cleaning themselves up or can crash.

## The relationship to `AbortSignal`

`AbortSignal` is meant for cancelling operations immediately, such as background
tasks or ongoing IO operations. Rather than cleaning up resources, like
disabling a protection. This is also why dispose is async on `Lifetime`, so that
we can track when disposal has finished before we restart a protection or a
draupnir instance.

`AbortSignal` is still important, and the _Lifetime_ does need an
`toAbortSignal` method so that background tasks can be cancelled immediately as
the protection begins to be cleaned up.

## The relationship to the persistent replica runtime

`AbortSignal`'s can be used to fascilistate pre-emption on lock acquisition, and
IO.
