Limitations
===========

Blur has very few built-in rule types. Should it turn into a long-term
viable product, it is likely that the set of rule types will be expanded.

Blur has trouble with file names that end with newlines. (Embedded
newlines are just fine, though.) This is because of the shell semantics
around command substitution, which silently drops any end-of-output newlines.
It is possible to fix Blur with respect to this problem, but at the time of
this writing it was not considered worth the trouble.

If a req (dependency) and target have exactly the same timestamp, Blur
assumes that the target is up-to-date. Since timestamps have one-second
(not sub-second) granularity on many systems, it is possible for a target
to be undetectably out-of-date in some circular dependency situations.
Of course, circular dependency is a bad idea.
