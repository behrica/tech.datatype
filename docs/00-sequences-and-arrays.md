# Sequences And Arrays


## Turing Machines and LISP


We start our discussion talking about the basis of what is computable and why.  In the
beginning, there was the [Turing Machine](https://en.wikipedia.org/wiki/Turing_machine)
and it was good.  No one built an exact turing machine of course but the axioms developed
for doing computions using a turing machine have proved to have a universal truth about
them.


The first turing machine design had tape that you could scroll forward or backward and a
finite state machine that controlled reading, scrolling, and moving this tape.  This was
sufficient to describe all of the computations that we know about in
an abstract sense.  This first turing machine relates closely to the type of sequence
based programming we find in normal course of functional programming.  Whether your
favorite language is typed or not the base mechanics of how you do your computation are
the same.  Generally speaking, you have sequences, you have map, filter, and reduce.
Adding in some minimal control logic and variable declaration and many computational
problems seem to dissolve into fairly simple blocks of the above operations.  We can add
distinct (if we have some form of scalar memory along with sets) and group-by to this
system but we cannot in fact efficiently add sort.


John McCarthy came along and in 1958 and invented LISP which is I think a beautiful
language because it is based on simple principles build a complete turing machine.  He
had the insight to build the compiler in LISP and to make the boundaries between the
compiler and the program blurry so that you could flow between programming the actual
machine and extending the language itself via programming the compiler.  LISP is
`homoiconic` which means the language is completely described using datastructures and
elements that exist within the language thus making programming the compiler via reading
source code and transforming it is the same as reading in data to the program and
transforming it via normal programming routines.


## Math And Notation


Around the same time that McCarthy was working in LISP, Ken Iverson was getting a PhD
from Harvard in mathematics.  Ken was working under a legendary visionary and educator
named Howard Aiken.  Within months of finishing his PhD, Ken was asked by Aiken to
prepare a course on doing business data processing.


During the development of the course, Ken realized that the mathematical notation that
he was using was lacking in expressiveness so he designed new notation.  Later, while
working at IBM his team published two books: "A Programming Language" (APL) and
"Automatic Data Processing."  He and a team completed an initial working batch-mode
implementation of the language in 1965 and a timeshare version in 1967.  He received the
Turing Award in 1979 and continued on to make the opensource J programming language.


APL is based on array programming; so you are dealing with contiguous groups of items.
This has important implications and benefits.  In fact, there are versions
of Turing machines that have random access memory and you can show the two types of
machines equivalent but in practice the two types of programming, sequence vs. array,
tend to have different shapes.


The foundation of APL (A Programming Language) appears to be nearly completely different
than LISP but I intend to show a natural progression from the familiar sequence based
constructs of LISP that we all love to the N-dimensional constructs of APL-type
languages.


One thing to keep in mind throughout all of this is the fact that regardless of how
advanced the hardware underlying your computation system is, the thing it would like to
be doing the most is the same thing, over and over again.  APL had amazing performance
working in very constrained environments precisely because the language is structured
such that you do operations on hyper-rectangles of data instead of on one datum.


## Toward Unification


### Sequences of LISP


Throughout this discussion you will not see any language based security directives
(public, private) as they are entirely irrelevant to this discussion.

We start by defining a what a sequence is.  A sequence is an abstract interface that has
two members, current and next.    Current points to the current object
in the sequence and next points to another sequence interface or nothing.

```java
interface SequenceElement
{
  Object current();
  SequenceElement next();
}

interface Sequence
{
  SequenceElement first();
}
```

Should we get into a situation where we need to avoid boxing things, we have no choice
but move to a mutable iterator.  It should be noted that a mutable iterator is the
clumsy approximation to a sequence that is forced by the necessity to deal with concrete
types directly for efficiency concerns:

```java
interface ByteIterator
{
  byte current();
  bool hasNext();
  byte nextByte();
}

interface ByteIterable
{
  ByteIterator iterator();
}
```

Note that we attempt the most minimal transformation from sequences to iterators.  Note
also that the iterator has a `current` member.  This allows a simplification of
algorithms as compared to using only `nextByte` as the algorithm itself does not need to
store both the iterator and the current item.  It also matches our sequence definition
more precisely leading to a natural progression from sequences to iterators where
efficiency concerns are beginning to dominate.  For primitive types, the iterator
interface producs 2 fewer objects per element so you have a significantly less amount of
garbage generated via iterating through primitive types than you would via sequencing
through them.


Given these two definitions we can expect a few things:

* map, filter, etc. have lazy processing semantics
* count realises the entire sequence
* sort is not possible without another piece of engineering
* reverse is O(N)


### Arrays of APL


We now abstract the concept of an index-addressable storage into 2 interfaces, readers
and writers.

```java
interface Countable
{
  int size();
}

interface ByteReader extends Countable
{
  byte read(int idx);
}

interface ByteWriter extends Countable
{
  void writer(int idx, byte value);
}
```

In addition to these strongly typed interfaces, there are weakly typed bulk
interfaces:

```clojure
(defprotocol PSetConstant
  (set-constant! [item offset value elem-count]))

(defprotocol PWriteIndexes
  (write-indexes! [item indexes values options]))

(defprotocol PReadIndexes
  (read-indexes! [item indexes values options]))
```

Both share some level of interface and we have some expectations about them.  For a lot
of reasons, we use strongly typed interfaces where we have an efficiency concern and
weakly typed interfaces elsewhere.  Chatty, strongly typed interfaces are less ideal
than bulk interfaces but we cannot always avoid using them.  The peak performance of a
chatty interface given the most sophisticated optimizations we know of is still far
below the peak performance of a bulk interface regardless of typing if for nothing else
than we have to pay for the cost of crossing the interface a large number of times.


We don't explicitly state anything about where read or write are getting/putting their
data allowing us to map those interfaces to anything we like.

The reader concept is a natural extension of the sequence concept that allows
us to implement algorithms that imply random access generically.

* size is constant time
* reverse is constant time
* We now have binary-search, which gives us a rough version of sort
* Analog of map is reader-map, which does the operation on read of each element
* Reductions still only require iterators, creating an iterator from a reader is a
constant time operation
* Parallelizing operations across a range of integers is far easier (and more efficient)
than implementing any form of parallel map
* Given data and index readers, we can return a new reader that arbitrarily changes the
order of the values in the original data reader
* Concatenation produces a new reader in constant time


Moving to more sophisticated operations, we build `arg` versions of our
usual paradigms.  Arg means the operation returns results in index space:

* `binary-search` - return tuple of `[found? index]` where if found is true, index
points to where the element is and if found is false, index points to where one would
call `insert` to produce a sorted sequence
* `argsort` - return sorted list of indexes
* `argfilter` - return a potentially shorter list of indexes based on a boolean
  operation
* `argmax, argmin, argcompare` - Return index of the the min, max, item that compared
the highest as compared to the rest of the items


#### Mutables - Arrays With Insert/Remove

An object that knows how to insert elements and is convertible in constant time to a
reader that reads elements by index and a writer that writes to that indexed data store.
This is neither a list nor a vector but Java called it a list and vector is heavily
overloaded with other semantic definitions.  `insert` can be called with the return
of `size` and a new value and the result is the addition of the value to the end of
the mutable and an increment to the return value of `size`.


```java
interface MutableRemove
{
  void remove(int idx);
}

interface ByteMutable extends MutableRemove
{
  void insert(int index, byte value);
}
```

In addition to the strongly typed interface, we have bulk weakly typed interfaces:

```clojure
(defprotocol PRemoveRange
  (remove-range! [item idx n-elems]))

(defprotocol PInsertBlock
  (insert-block! [item idx values options]))
```

### Containers


We still need to define a storage mechanism.  But with the above definitions, we can
define a few purely in terms of the above elements but with different important
properties.  These properties include:
*  Backed by native `malloc` data
*  Implements `mutable` interfaces
*  Sparse expectation of data compression and algorithm optimization.  Efficient
conversion to CSR and CSC formats.
*  constant convertible in constant time to a tuple of `{:java-array :offset :length}`
*  constant convertible to a length-mutable datastructure.
*  constant convertible to a integer long value that can be used by native functions.


#### Container Rules


1.  We can formalize the operation of making a reader out of one container and a writer
    out of another and copying each element into a `copy!` operation.  In this way we
    can efficiently copy data from one buffer into another in such a way as to sometimes
    offer `System/arraycopy`, `memcpy`, or `dma-transfer` as an optimization.
2.  We also formalize set-constant! operation such that it sometimes can be implemented
    with `memset` or `dma-set`.
3.  We add the ability to create sub buffers; a buffer that may be offset and
    have a shorter length (or both) than the parent buffer but still share the same
    backing store.  This is an important property to both simplify algorithms (they do
    not have to have these aspects passed in as options) and to create 'pools' where we
    have a general large memory allocation and then reuse sub-portions of it for
    different potentially unrelated purposes.


These definitions allow some important optimizations especially if we expect this
language to transfer to gpgpu or multi-gpgpu programming where all three of these rules
apply albeit without the ability to directly make a reader/writer.  We expect in those
environments to have access to a dynamic compilation system similar to
[tvm](https://github.com/dmlc/tvm) and as such all variations can be implemented
dynamically and efficiently with existing compiler technology.


## Summary Of Part 1


We begin with sequences and an efficient (although clumsy) formulation of the sequence
concept where we may be dealing with large numbers of primitive types.  We then show a
typed formulation of index-addressable memory in the form of `read`, `write`, and
constant time `size` operations.  We define the concept of a container which includes at
least both read/write concepts and may also, in the case of a resizeable container
include the concept of insert and removal of elements.


We [next](01-operations.md) dive into performing actual arithmetic given the above
definitions and combining them with a concept we call 'chaining'.
