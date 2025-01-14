# Mutation Support

Normally, Dagger tasks should be functional and "pure": never mutating their
inputs, always producing identical outputs for a given set of inputs, and never
producing side effects which might affect future program behavior. However, for
certain codes, this restriction ends up costing the user performance and
engineering time to work around.

Thankfully, Dagger provides the `Dagger.@mutable` macro for just this purpose.
`@mutable` allows data to be marked such that it will never be copied or
serialized by the scheduler (unless copied by the user). When used as an
argument to a task, the task will be forced to execute on the same worker that
`@mutable` was called on. For example:

```julia
x = remotecall_fetch(2) do
    Dagger.@mutable Threads.Atomic{Int}(0)
end
x::Dagger.Chunk # The result is always a `Chunk`

# x is now considered mutable, and may only be accessed on worker 2:
fetch(Dagger.@spawn Threads.atomic_add!(x, 3)) # Always executed on worker 2
fetch(Dagger.@spawn single=1 Threads.atomic_add!(x, 3)) # SchedulingException
```

`@mutable`, when called as above, gain a scope of `ProcessorScope(myid())`,
which means that any processor on that worker is allowed to execute tasks that
use the object (subject to the usual scheduling rules).

`@mutable` also has two other forms, allowing the processor and scope to be
manually supplied:

```julia
proc1 = Dagger.ThreadProc(myid(), 3)
proc2 = Dagger.ThreadProc(myid(), 4)
scope = Dagger.UnionScope(ExactScope.([proc1, proc2]))
x = @mutable OSProc() scope rand(100)
# x is now scoped to threads 3 and 4 on worker `myid()`
```

## Sharding

`@mutable` is convenient for creating a single mutable object, but often one
wants to have multiple mutable objects, with each object being scoped to their
own worker or thread in the cluster, to be used as local counters, partial
reduction containers, data caches, etc.

The `Shard` object (constructed with `Dagger.@shard`/`Dagger.shard`) is a
mechanism by which such a setup can be created with one invocation.  By
default, each worker will have their own local object which will be used when a
task that uses the shard as an argument is scheduled on that worker. Other
shard pieces that aren't scoped to the processor being executed on will not be
serialized or copied, keeping communication costs constant even with a very
large shard.

This mechanism makes it easy to construct a distributed set of mutable objects
which are treated as "mirrored shards" by the scheduler, but require no further
user input to access. For example, creating and using a local counter for each
worker is trivial:

```julia
# Create a local atomic counter on each worker that Dagger knows about:
cs = Dagger.@shard Threads.Atomic{Int}(0)

# Let's add `1` to the local counter, not caring about which worker we're on:
wait.([Dagger.@spawn Threads.atomic_add!(cs, 1) for i in 1:1000])

# And let's fetch the total sum of all counters:
@assert sum(fetch.(map(ctr->ctr[], cs))) == 1000
```

Note that `map`, when used on a shard, will execute the provided function once
per shard "piece", and each result is considered immutable. `map` is an easy
way to make a copy of each piece of the shard, to be later reduced, scanned,
etc.

Further details about what arguments can be passed to `@shard`/`shard` can be found in [Shard Functions](@ref).
