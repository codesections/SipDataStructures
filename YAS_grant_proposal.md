# Persistent Data Structures for Raku 

## Synopsis

Immutable, persistent data structures give a program certain superpowers that it's very hard to have
in any other way: they allow the program to "time travel" (view previous application state); they
allow let the program share data across threads or asynchronously save it to disk without needing
locks; they enable a much more purely functional style of programming – which results in code that
many software developers find much easier to reason about.  Because of these benefits, many
languages with strong support for functional programming (Clojure, Elm, Haskell, etc) offer
persistent data structures in their standard library.  And many multiparadigm languages that attempt
to support programming in a fuctional style (C++, JavaScript, Java, Rust, Go, etc.) have
well-maintained and popular libraries offering persistent data structures.

However, despite these benefits and despite the extremely strong support Raku generally provides for
functional programming, Raku does not provide any persistent data structures, either in core or in a
library.  I am therefore seeking a grant to create a persistent data structures library for Raku.

If this grant is funded, I will build this library as an external module that can be used as soon as
I have completed it.  Additionally, I will structure the module in such a way that it can later be
upstreamed into Rakudo if we decide that persistent data types are valuable enough to include in
Roast/the core language (though accepting this grant proposal would, of course, not commit Raku to
adding these types, and the types would provide most of the same benefits as a standalone library).

### The Problem

As part of its support for functional programming, Raku provides several immutable data types – most
notably, Lists, Maps, and Sets.  These types work well, but they suffer from two well-known
problems: (1) shallow clones with interior mutability and (2) expensive copies.

The problem of interior mutability can be seen in the following code

    my Map $m1 = :key[].Map;
    my Map $m2 = $m1.clone; 
    $m2<key>.push: 2;
    say $m1; # OUTPUT: «Map.new((key => [2]))»

That is, neither immutability nor clones are deep, so changes to $m2 end up causing changes to $m1.
This behavior is not a bug; it is the intentional (and correct) consequence of the semantics of
immutable types, Scalars, and clone.  This correct behavior, however, negates much of the benefit
that _deeply_ immutable types can provide in terms of reasoning about the code and safely sharing
data between threads.

You can overcome this problem (albeit at the cost of some verbosity) by manually deep-cloning
data and/or ensuring that all the components of a type are themselves immutable.  However, solving
this problem leads you directly to the second: a deep clone of an immutable object is very
expensive, because it involves literally copying the entire object (even if the copy is only
slightly different).  This makes using large immutable objects in Raku much slower than using their
mutable equivalents.

Because copying immutable data structures is both syntactically annoying and computationally
expensive, copying data is a less appealing prospect in Raku than in languages with persistent data
structures.  This, in turn, means that Raku programs are less likely make immutable copies of data
and are more likely to share access to a mutable copy.  This is a shame in any context, because it
makes code harder to reason about and thus more likely to contain bugs.  But it's especially
problematic in concurrent programs – when programs share (rather than copy) data between threads,
they have to use locks, which is both slower and more cumbersome.  Since Raku otherwise offers
excellent support for concurrent programming, the expense of copying large data structures is a real
problem. 

### The Solution

I mentioned that the problems of interior mutability and expensive copies are well known and,
fortunately, they also have a well-known solution: [persistent data
structures](https://en.wikipedia.org/wiki/Persistent_data_structure) that provide structural
sharing.  The basic idea is that when you copy an immutable object, you don't need to _literally_
copy all the data in it, you just need to point back to the copied object (which can't change
because it's immutable).  Then, the copy only needs to remember and ways it differs from the
original rather than the entire object.  (At a high level, this is similar to how some version
control systems or commands work – for example, with git-send-email, you don't send a new copy, just
a patch). 

Many early persistent data structures provided these benefits, but only at an unacceptably high
cost.  For example, a linked-list qualifies as a persistent data structure because you can add a
node to the list without mutating it and multiple modified "copies" of a linked list can share much
of their data without making literal copies (aka "tail sharing").  However, the performance profile
of a linked list (especially its lack of cache coherence, given the relative cost of a cache miss on
modern hardware) make it an extremely poor fit for most general purpose programming tasks today.

However, in the past two decades, a new bread of persistent data structures have been developed –
and these new structures combine the safely and cheap copying of previous data structures with a
much-improved performance profile.


## Prior Art

I have no interest in reinventing the wheel and persistent data structures have a long history both
in academic computer science and in practical language implementations.  Most notably, Phil Bagwell
described a practical design for a persistent data structures in a 2001 paper titled Idle Hash
Trees.  This design describes a specific data structure called a Hash Array Mapped Trie.  Here's a
simplified description of how a HAMT works:

  * create a tree where each node contains either
    - a sparse array of length 32 with references to further nodes.
    - a final value (i.e., it's a leaf node).
  * To store the values of an array into the tree created above, parse the index of each element as
    a base 32 number and then convert that number into a list of digits.  Starting at the root of
    the tree, look at the array element that matches the value of the last digit and go to (or
    create) the node referenced by that element.  When you have done so, remove the digit from your
    list and repeat.  Once you are out of digit, store the value in the current node.
  * ^^^^ created a binary Trie, hence the name.
  * Storing the values of a Hash is similar, except that the hash key must first be hashed into a 32
    bit integer and the final nodes need to be buckets to handle hash collisions.
  * This data structure is immutable, but supports creating a copy with certain values added (or
    deleted).  Doing so only requires copying the nodes in the path from the root node to the
    inserted/deleted/changed node; the rest of the tree (which is unchanged) is shared between the
    copies.
  * Note that the above omits some details, such as the use of a bitmap & dense array in place of
    the sparse array.  For a more detailed explanation of how this works, refer to either the
    original paper or to https://hypirion.com/musings/understanding-persistent-vector-pt-1

In algorithmic terms, this data structure provides access to nodes in O(log₃₂(n)) time.  A very
pedantic academic might say that this should still be considered O(log(n)) time, but in practice it is
much closer to O(n) – an HAMT with 6 levels can store over a billion items.  Memory use (compared to
full copying) is reduced by a similar factor. 
    
In 2007, Clojure was released with HAMT-based persistent data structures backing all of its
fundamental collection types.  Since then, HAMT-based data structures have been developed for many
languages, either as a core part of the language or as a third-party library.

While Raku does not have HAMT-based (or any) persistent data structures, it does have a few related
features.  First, it has strong support for laziness which can, in some cases, give Lists/Arrays
performance gains similar to those they would get from a persistent data structure (Jonathan
Worthington helpfully explained this distinction to me on StackOverflow:
https://stackoverflow.com/a/67035827/10173009).  This does not apply in all cases and does not apply
at all to Hashes/Maps/Sets/etc. Second, Rakudo currently implements the Str type in a way that has
some conceptual similarities to persistent data structures, though it does not offer the same
guarantees and that implementation is not required by the Raku specification.  Third, Jonathan
Worthington has also implemented a trio of concurrent data structures that use tail sharing (a
similar technique for creating persistent data structures).  These modules – and, in particular, the
Concurrent::Trie module – could be useful when implementing HAMT based data structures.

Given this prior art, implementing HAMT-based data structures will have a clear roadmap, even if
much implementation work remains to be done.


## Deliverables 

The primary deliverable from this grant would be a collection of persistent data types based on Hash
Array Mapped Tries (which, for type-naming purposes, I tentatively plan to refer to as Hash
Array-mapped Tries, yielding the abbreviation 'Hat' rather than 'HAMT').  I would plan to deliver
some or all of the following (time/budget permitting, as discussed in the schedule below), along
with corresponding pod6 documentation suitable for future inclusion on docs.raku.org and
corresponding tests suitable for future inclusion in Roast:

  * ListHat (persistent Array)
  * MapHat  (persistent Hash)
  * SetHat  (persistent SetHash)
  * BagHat  (persistent BagHash)
  * MixHat  (persistent MixHash)

(Note: I don't love these names, since being backed by a HAMT is an implementation detail.  But I
would like to find a name that's short and memorable enough to make the types easy to use;
"PersistentList"s seem unlikely to get much use.  I'm open to ideas here.)

This covers the Raku collection types other than Pairs; creating new Pairs is already simple enough
that creating a PairHat type would be pointless

Time permitting, I would also deliver benchmarks comparing the performance of the *Hat types with
that of the corresponding deeply copied immutable types.

## Project Details

### Proposed API for deliverable types

In general, when implementing immutable types, the copying API can be designed along two lines: it
can either require the user to use explicit copying operations, or it can offer an API that mirrors
the API of mutable types but that returns a new value rather than mutating one in place.  The
current Raku immutable types provide an API of the first type.  This API allows code like:

    my List $list = (1, 2, 3);
    my $new = |$l, |(4, 5, 6);  # correct
    my $new = $l.push: 4, 5, 6; # NO - throws an error
    
This design choice makes sense for the existing immutable types, given that they don't provide
inexpensive copying and that Raku's | operator makes combining collections easier.  It also helps
new users more quickly realize the difference between mutable and immutable types. However, even
with the | operator, creating copies of immutable types can become syntactically cumbersome,
especially if the type might be empty.

    # mutable
    class Foo {
        has @!values;
        method add($val) { @!values.push($val) }
    }
    
    # immutable 
    class Bar {
        has List $!values;
        method add($val) { $!values = (|($!values // Empty), |($val // Empty)) }
    } 
    
Again, this tradeoff makes sense for the existing types because copies are relatively expensive and
so it makes sense to have syntax that implicitly discourages their use in non-trivial cases.
However, since the *Hat types are designed for inexpensive copies, it makes sense to provide the
second type of API:

    my ListHat $l = (1, 2, 3);
    my $new = $l.push: 4, 5, 6;
   
This approach is similar to the one taken by many libraries that add persistent data structures to
languages that aren't purely functional; as one example, immutable.js.  In other languages, this
has the downside of creating a trap:

    my Array $a = 1, 2, 3;
    $a.push: 4; # works
    my ListHat $l = 1, 2, 3;
    $l.push: 4; # seems to work, but actualy does nothing
    
However, Raku offers the `is pure` trait; by implementing the relevant methods for the *Hat types
with that trait, the incorrect line would generate a warning that a pure function was used in sink
context.

I would also plan to offer an API for "batching" operations (which lets the data structure improve
performance by skipping intermediate copies).  I'm less sure of this part of the API, but think that
an API similar to the one provided by Immer could be a good fit:

   my MapHat $m = (:key1<val1> :key2<val2>);
   my $m2 = $m.next: $draft -> { $draft<key2> = 'new val' };
   # Or, more concisely
   my $m3 = $m.next: *<key2> = 'new val';
   # $m2 and $m3 are both {:key<val1> :key2('new val')}

(This API is inspired by the JavaScript Immer, not the identically-named-but-unrelated C++ library,
which provides a more traditional batch API based on a .commit method)

Of course, I am sure I will learn more in the course of the implementation, so all of the APIs shown
above could change/evolve, but I present it here as a general guideline.

### Limitations on scope

To keep the project limited in scope, there are two sets of features that I explicitly do _not_ plan
to include in the work for this grant (though either could be added later on).

First, I do not plan to implant the CHAMP optimizations to the HAMT data structure.  This
optimization, which was described in the academic literature in 2015, further reduces memory use and
increases the data's cache locality (which, given the relative significance of cache misses on
modern CPUs, provides a speed boost as well).  However, it also significantly increases the
implementation complexity.  Perhaps because of that complexity or perhaps just because it was more
recently presented, CHAMPs have not been as widely implemented in non-academic settings. Moreover,
adding the CHAMP optimizations would not change the overall performance profile, appropriate uses,
or API of the data structure, so we could add it later if we determine that the performance gains
justify the added complexity.

Similarly, I do not plan to implement the *Hat types in a way that requires them to be included in
Rakudo.  As illustrated by Jonathan Worthington's Concurrent::Stack, Concurrent::Queue, and
Concurrent::Trie data structures, it is entirely possible to implement performant Raku data
structures entirely is userland code, and I plan to follow that approach.

It's possible that integrating more tightly with the compiler/VM could further increase performance.
However I believe that it makes sense to defer that work until a working userland implementation
proves the value of persistent data types via an external library.  Creating an external library
will also allow users to provide feedback on the *Hat types earlier and, if necessary, allow us to
evolve their API without breaking Raku's stability guarantees.


## Proposed Schedule and Grant Amount

I propose to work on this grant for 10 hours/week for 14 weeks our until the completion of all
deliverables described above and to work for a reduced rate of $50/hour.  Thus, I am seeking a grant
of up to $7,000 (or less, if the implementation is completed before the end of the 14 weeks).


## Benefits to the Raku Community 

As mentioned above, adding persistent data types to Raku would have multiple benefits.  First and
most directly, these types would significantly improve the performance of Raku code that uses
immutable data.  Second, the new types would provide a more ergonomic API for copying immutable
types.

Together, these two direct benefits would encourage Raku programmers to make greater use of
immutable types and other functional programming idioms.  This, in turn, would generally increase
the likelihood that any given Raku module is robust and thread-safe (even if it was not written with
thread safety in mind).  Increased thread safety will help make the Raku ecosystem more suitable for
concurrent workloads (already a strength, given Raku's strong concurrency and parallelism primitives
and the existence of excellent concurrent modules such as Cro).

In addition to these practical benefits, adding these types (especially if they are eventually added
to core Raku) will help in Raku's marketing/adoption efforts.  Languages that have implemented
persistent data types include Clojure, Elm, Haskell, and Scala; all of these languages are (1)
functional and (2) modern – two traits that it would be both helpful and accurate for more
developers to associate with Raku.

Finally, on a more personal note, this grant would benefit a project for which I am not seeking
funding.  I am independently working on Épée, a web framework for Raku inspired by React, Svelte,
Pollen, and Hoplon (and both inspired by and designed to integrate smoothly with Cro).  Épée uses
immutable data via deep copies and thus would see performance benefits from my work on this grant.
I mention this both as a benefit of the grant and to say that, although Épée would benefit from this
grant, that benefit would not be high enough to make implementing these data structures worthwhile
if the grant is not accepted (that is, I am not seeking a grant for work that I plan to do anyway!).
I believe that my perspective as a future user of the API with a concrete use-case in mind would
prove helpful during my work on this grant.


## Biography 

I am a contributor to Rakudo, Roast, and the Raku documentation; I serve on the Raku Steering
Council and the Yet Another Society Legal Committee.  Prior to becoming a programmer, I was a
practicing attorney at Davis Polk & Wardwell, one of the top law firms in New York City – but
decided to become a software developer after helping the firm build a web application that dealt
with data breach law.  I have used JavaScript and Rust professionally, but now focus primarily on
writing free software in Raku.
