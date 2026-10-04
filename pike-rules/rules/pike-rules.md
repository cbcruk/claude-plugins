# Pike's Rules

Rules for choosing algorithms, data structures, and when to optimize. Apply them
to every change that adds or rewrites logic. Adapted from
[Rob Pike's 5 Rules of Programming](https://www.cs.unc.edu/~stotts/COMP590-059-f24/robsrules.html).

## Don't guess where the time goes

Bottlenecks occur in surprising places. Do not add caching, memoization,
batching, pooling, or any other speed hack on the strength of reading the code.
Add one only after a measurement shows that this spot is the bottleneck.

## Measure before tuning

Before changing code for speed, measure it — a profiler, a benchmark, or timing
around the suspect path — and report the numbers. Tune only when one part
clearly dominates the rest; a change that shaves a few percent off a path that
is not the hot one is churn.

When asked to "make it faster" with no measurement in hand, measure first and
show where the time goes before editing.

## n is usually small

Fancy algorithms have big constants and lose when n is small, and n is usually
small. Default to a linear scan, a plain array, or a nested loop. Reach for a
more elaborate algorithm only when you know n is frequently large — from the
data's real size, not from what it could become.

## Simple beats fancy

Fancy algorithms are buggier and harder to implement. Use simple algorithms and
simple data structures — arrays, maps, sets, plain records. When in doubt, use
brute force.

## Data dominates

Design the data before the code. When the logic is getting tangled, change the
shape of the data first — a lookup table instead of a branch chain, a map keyed
by what is being searched for, a record that carries the field the code keeps
recomputing. With the right data structures, the algorithm becomes
self-evident.
