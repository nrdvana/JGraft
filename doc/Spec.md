# JGraft

## Introduction

JGraft is a data structure that describes how to edit a tree of data.  It can
describe edits where portions of the source and destination are fully defined
and thus reversible, edits applicable to any compatible data structure, or
conditional edits where the change depends on the state of the target data.
It is a data structure rather than a parsed language to make it easier to
programmatically generate, combine fragments, extract fragments, and inspect.

The data that JGraft operates on is defined using basic JSON-compatible types:
`null`, booleans, numbers, JSON-compatible strings, arrays, and objects.
Implementations may include other native host-language data types if the
application doesn't need to interoperate with different host languages.
(implementations allowing other host language types are responsible for
defining equality, cloning, and serialization behavior for them, which is
outside the scope of portable JGraft)

The JGraft data structure contains "actions" and "expressions" in addition
to literal data.  Actions have the potential to mutate the tree or exhibit
other side effects, but they are restricted to ensure the input tree remains
a tree.  Expressions include Match-Patterns and Functions, and have no side
effects.  JGraft is designed so that untrusted data and grafts can be
processed safely when implementations enforce appropriate resource limits,
such as memory usage limits, node creation limits, and stack depth limits.
To this end, it has no flow control or free-form looping construct.

## Representation

The JGraft tree contains actions and expressions, but JSON has no mechanism to
tag data with a type.  So, when serialized to JSON they take the form of
arrays in the style of the Lisp programming language, where the first element
indicates the operation and remaining elements become an argument list for it.
The operation (first element) may be specified using either a name or numeric
ID; the ID is useful for compact encoding in production, and the name is
useful for development and debugging.
In contexts allowing a mixture of data and expressions, array literal values
get "escaped" by wrapping them with another array.  In other words, in the
JSON representation of JGraft in a context that allows data and expressions,
an array containing exactly one element which is also an array means a literal
array having the elements of the inner array.  The elements may then
recursively contain expressions or array literals by this convention.

Implementations may choose to use host-language objects of some sort to
represent actions and expressions, then perform the array encoding during
serialization and reconstruct the objects during deserialization.
Implementations could also choose to leave the JGraft data
structure in its JSON-compatible form and interpret it on the fly.
They should make it clear to the developer whether the data structure the
application code interacts with is using objects and literal arrays, or the
direct "arrays-are-special" representation of JGraft.
The remainder of this specification uses the JSON form of actions and
expressions, since the object model of a host language is left unspecified.

## Action Overview

Actions describe the overall algorithm of the JGraft.  Each action dictates
the meaning of its arguments; they may include other sub-actions, match
expressions, objects of configuration properties, and so on.  If you want to
validate the JGraft or build host language objects for the actions, it needs
to be handled recursively per-action.  Actions occupy a namespace apart from
expressions, and are not interchangeable.

The root of a JGraft is the `JGRAFT` action.  The `JGRAFT` action may contain
metadata and scope-based configuration.  It establishes a scope for constants
and stash variables, available to all sub-actions.  The constants help reduce
redundant data within that tree, and the stash facilitates transfer of values
between actions.

The primary intended workflow for JGraft is to use a `MATCH` or `IF` action to
verify that the input data tree matches expectations (or find a matching
location in the tree with some fuzzy matching) and then within that use
`ASSIGN`, `MOVE` or other actions to alter the tree.

Here is a summary:

Name                     | ID | Description
-------------------------|----|----------------------------------------------
[JGRAFT](#action-jgraft) | -1 | Declare metadata; top-level element
[AT](#action-at)         |  0 | Navigate to sub-node and execute sub-actions
[MATCH](#action-match)   |  1 | Assert current node matches pattern
[IF](#action-if)         |  2 | Choose action based on which pattern matches
[ASSIGN](#action-assign) |  3 | Assign-by-value to properties
[MOVE](#action-move)     |  4 | Move existing values between properties
[SPLICE](#action-splice) |  5 | Perform standard splice() on an array
[SPLIT](#action-split)   |  6 | Split string into array, apply actions, re-join
[LOG](#action-log)       | -2 | Emit diagnostic message and data

The `SPLICE` action is an optimization vs. `ASSIGN` or `MOVE` for making
changes to a range of an array.

The `SPLIT` action is a tool for parsing strings in order to make changes
within them, resulting in a string.  It can also be used to access specific
substrings and copy them into stash variables for use in other actions.

The `LOG` action can be used to provide improved diagnostics to a user
applying a JGraft.

## Expression Overview

Expressions are essentially "functions without visible side effects", though
there are two special categories:

  * "Reference Expressions" reference a [path](#paths) within the input data
    or one of the special namespaces JGraft offers, and can be both read and
    written by actions.
  * "Match Expressions" compare the current node to a structural pattern and
    return a boolean of whether it matched, though that boolean is implicitly
    consumed in the context of another match expression.

Everything else is just a "Function".  Functions don't need to be pure in the
mathematical sense, but the "no visible side effects" rule is important
because they may be evaluated any number of times during fuzzy-matching, and
because JGraft does not define an order for enumerating object properties
during the matching process.  They *may* have a side effect of incrementing
a resource counter that aborts the graft operation due to resource usage, but
that is not visible *to the expressions*.

Here is a summary:

Name                     | Type  | ID | Description
-------------------------|-------|----|---------------------------------
[TEMP](#ex-temp)         | Ref   | -3 | A path in the temp namespace
[STASH](#ex-stash)       | Ref   | -2 | A path in the stash namespace
[CONST](#ex-const)       | Ref   | -1 | A path in the const namespace
[NODE](#ex-node)         | Ref   |  0 | A path in the current node
[HAS](#ex-has)           | Match |  1 | Partial structured match
[IS](#ex-is)             | Match |  2 | Exact structured match
[NONE](#ex-none)         | Match |  3 | Logical NOT/NOR of match exprs
[ANY](#ex-any)           | Match |  4 | Logical OR of match exprs
[ALL](#ex-all)           | Match |  5 | Logical AND of match exprs
[EXISTS](#ex-exists)     | Match |  6 | Current node exists
[ARRAY](#ex-array)       | Match |  7 | Match arrays, specific elements
[STR](#ex-str)           | Match |  8 | Match strings, optional regex
[NUM](#ex-num)           | Match |  9 | Match numbers, optional range
[BOOL](#ex-bool)         | Match | 10 | Match booleans
[DATE](#ex-date)         | Match | 11 | Match date-like things
[CASE](#ex-case)         | Fn    | 31 | If/else that returns a value
[NOT](#ex-not)           | Fn    | 32 | Logical NOT of value exprs
[AND](#ex-and)           | Fn    | 33 | Logical AND of value exprs
[OR](#ex-or)             | Fn    | 34 | Logical OR of value exprs
[CASTBOOL](#ex-castbool) | Fn    | 35 | Cast argument to boolean
[CASTNUM](#ex-castnum)   | Fn    | 36 | Cast argument to number
[CASTSTR](#ex-caststr)   | Fn    | 37 | Cast argument to string
[SPLIT](#ex-split)       | Fn    | 38 | Split a string into an array
[JOIN](#ex-join)         | Fn    | 39 | Join array into a string

To quickly understand how these types interact, consider these examples:

    [IS, { "a": [ANY, false, true] }]
    // node.a === false || node.a == true

When a match expression encounters another match expression, the outer match
expression calls the inner one with a modified current node and context of
`IS` vs. `HAS` matching (discussed next) and consumes the boolean return value
as the status of whether it matched.  In this case, the outer passes `node.a`
as the current node, and the inner one tests two values against that, and
returns a boolean of whether `node.a` equaled any of them.  The outer `IS`
action consumes the boolean as the result for matching property 'a'.

    [IS, { "a": [NONE, { "b": 1 }, { "c": 1 }] }]
    // !deep_equals(node.a, {b:1}) && !deep_equals(node.a, {c:1})

The `HAS` expression tests that specified properties of the current node
match, and the `IS` expression additionally tests that no extra properties
existed on the object.  Every match expression (other than `IS` and `HAS`)
receive the `IS` vs. `HAS` semantics from the caller if the caller was also
a match expression.  They are *not* preserved through a regular function
call, and revert to the default `HAS` semantics if a match expression is
used inside a function.  In this case, the `NONE` expression inherits `IS`
semantics.

    [IS, { "a": [OR, false, true] }]
    // node.a === ( false || true )

When a match expression encounters a function expression, it calls the
function *without* updating the current node, and uses the return value as a
literal value to match against.  The return value of this `OR` expression is
`true`, so then the `IS` expression compares `node.a === true` to determine
whether that property is valid.

    [IS, { "a": [NODE, 'b'] }]
    // deep_equals(node.a, node.b)

Likewise, a ref expression within a match expression resolves to a value using
the same current node as the outer match action.  Then the value is used in the
match expression as a literal to match against.  This `NODE` expression
evaluates to `node.b`, and then the `IS` expression checks whether `node.a` is
an identical tree to that value.

    [OR, [ANY, 2, 3], [IS, [NODE,'b'], 2]]]
    // node === 2 || node === 3 || node.b === 2

When a function contains a match expression, the match expression tests
against the current node and returns a boolean.  If you want to test against
a different value (which need not even be in the tree) the `HAS` and `IS`
functions have a 2-argument form that overrides the current node with a value
of your choice.  In this case, a reference expression sets the current node to
`node.b` then tests against the literal `2`, and the boolean result gets
combined with the boolean result of testing whether `node` is the value `1`
or `2`.

    [IS, { "a": [IS, [LT, [CASTNUM], 8], true], "b": 5 }]
    // parseNumber(node.a) < 8 && node.b === 5

If you want the return value of a function to be consumed as the match result
for matching a property, simply use the `IS` or `HAS` expression to compare it
to `true`.  In this case, `LT` is not a match expression so it can't be used
directly within the structure of the outer `IS` in the intended manner.  Use a
second `IS` with the 2-argument form to compare the output of the `LT`
function to `true`.  That gets used as the match result for `node.a` because
`IS` is a match expression.

## Paths

Several actions and expressions use the concept of a 'path'.  This is a
description of how to navigate from some reference tree node to a deeper node
by following object property names or array index numbers.
When an action or expression declares that it takes a path in a positional
argument, it means a [NODE expression](#ex-path),
[CONST expression](#ex-const), [STASH expression](#ex-stash), or
[TEMP expression](#ex-temp).  To aid compact encoding, a plain number or
string (or `false`) is interpreted as a single argument to a `NODE`
expression.  Some actions also restrict which path types may be used, like how
`MOVE` can't make use of a `CONST` path.
Paths always descend the tree and have no concept of "up a level" like a
filesystem's `..` convention.  A path element *may* be another expression
(limited to functions) which evaluates to become the name or number or alias
of a property.

In general, strings may only refer to object properties and integers may only
refer to array elements.  This is particularly important when auto-vivifying a
path; integers will create arrays and strings will create objects.
However, the `STASH`, `CONST`, and `TEMP` namespaces permit a leading integer
as a property name even though the namespaces aren't arrays, for the improved
serialization size.  Negative numbers are available as aliases for the array
element at `[array.length-X]`.  The value `false` is available as an alias
for `length` of the current array, which is an element that doesn't exist but
may be written, growing the array.
(`-0` would be the ideal alias, but too many languages drop that distinction
when decoding JSON)
The values `null` and `true` may not appear in paths.

## Dates

Dates are a problematic domain to make a specification for.  JGraft tries to
keep the implementation burden low, like not requiring calendar math or
Unicode tables, but applications almost certainly want some integration with
the date objects of the host language.  Meanwhile, JSON doesn't even define
a notation for dates.

JGraft takes an approach of using expressions to identify when dates are
intended, and then comparing them directly field-to-field for year, month,
day, hour, minute, and second.  The `CASTDATE` function allows
`YYYY-MM-DD HH:MM:SS` notation and ISO 8601 variations like the "T"
separator, "Z" suffix, and sub-seconds specified with a decimal point. It
parses them to an array of 6 numbers, or 3 numbers for `YYYY-MM-DD` strings.
The numbers are parsed as-is, and not adjusted to be zero-based.
Portable JGraft should exclusively use UTC, and does not support time zone
notations, because correctly handling time zones would require date math.

If an implementation wants to make use of host language date objects, the
`CASTDATE` function can return them.  However, the implementation should
ensure that date object comparisons behave equivalently to comparing arrays of
`[y,m,d]` or `[y,m,d,h,n,s]` numbers.

## Errors

Applying a JGraft to a target data structure may fail with:

  * `INVALID_GRAFT` - The structure of the JGraft itself violates the spec.
  * `INVALID_TARGET` - The graft describes an operation that can't apply due
                       to the type or structure of the target.
  * `INVALID_OPERAND` - The runtime value given to an action or expression
                        is invalid.  This may be the fault of either the
                        target data or the described graft.
  * `NO_MATCH` - A `MATCH` action failed to find a matching context.
  * `RESOURCE_LIMIT` - Applying the graft would exceed configured constraints,
                       such as too much memory, too many generated properties,
                       or too deep of a tree.

Note that `INVALID_TARGET` and `INVALID_OPERAND` errors which occur inside of
a match expression get trapped and simply cause it to fail to match, which may
in turn generate a `NO_MATCH` error.

The recommended behavior for failures in a JGraft implementation are to emit
the error along with some diagnostic properties describing the current node
and action/expression which failed, and make the partially-edited data
structure available for inspection.  See also the [LOG](#log) action.
Implementations may also provide a callback allowing the application to
intercept a failure and resolve it in a way that allows the remainder of the
JGraft to continue.

Ideally, an implementation should be shallow-cloning each ancestor node of an
altered node back to the root as edits occur, so that the input data structure
remains unchanged while the output gets to share un-altered sub-trees.  This
isolates the partially-edited error state from the input data.  However, if
shared structures are difficult for the host language or it is understood that
the input data has already been deep-cloned for the library's exclusive use,
the implementation may just mangle the input tree and provide that to the user
as-is during errors.

Implementations are also encouraged to provide an option of building an
"undo" JGraft describing how to revert the changes it made.
If this option is enabled, during an error it should deliver one to the caller
that describes how to restore the original from the current mangled state.

## Actions

### JGRAFT

    ['JGRAFT', metadata, action1, action2...]

This action is used for the top-level of an exported JGraft JSON file, or
optional metadata deeper within the structure.
When serialized to JSON, the top-level 'JGRAFT' action name is not replaced by
a number, and there should be no whitespace until after the following comma.
This gives JGraft JSON files a "magic number" of

    '["JGRAFT",'

The `metadata` is an object with semi-free-form properties, where top-level
names matching `/^[a-z][A-Za-z0-9]*$/` are reserved for the protocol, and any
others are user-defined.
The following properties are currently defined:

  - v: integer; the minimum JGraft specification version required to
       correctly interpret the contents of this structure
  - writer: a string identifying the tool that authored the structure
  - comment: a free-form text comment string
  - const: an object of constants for use in [`ASSIGN`](#assign) actions
  - regexCommon: an array declaring [common subexpressions](#common-subexpressions)

When exported to JSON, outer metadata should contain `v` (minimum version
to correctly process the structure), and it should be at least as high as
any minimum version seen within.  It is permitted to declare `v` on
sub-structures, for composability between different sources.
Future versions of JGraft are expected to be backward compatible, but if
incompatibilities arise, implementations should aim to support older versions
dynamically so that they can process sub-structures from prior versions.

For user-defined properties, a good pattern is to use a unique-ish property
name containing a period (such as Java-style reverse domain name) or other
special character like '#ProjectName' and place all custom data within an
object under that.

The `JGRAFT` action acts like a scope for declaring constants, and also
provides a fresh empty `stash` of shared temporary variables for the actions
within it.  It also declares scoped regex common subexpressions.
The algorithm when beginning a `JGRAFT` action is roughly:

  - Set the `stash` namespace to a new empty object.
  - Set the `const` namespace to the specified object, but inherit the
    properties of the parent `const` namespace (or possibly values implicit
    to the declared version of the JGraft spec).
  - Create a local regex common-subexpression namespace which inherits the
    properties of the parent (or possibly values implicit to the declared
    version of the JGraft spec).
  - For each name/value pair specified in `regexCommon`, resolve references
    in the expression and then assign the result to the named property of
    the common-subexpression namespace.

### AT

    ['AT', path, subaction1, subaction2...]

Navigate to a sub-node of the current node, then execute all sub-actions with
that node as the current node.  The path must exist or the graft operation
fails with `INVALID_TARGET`.  The path cannot reference namespaces outside
the target tree of data, such as with `[null,...]`.

### MATCH

    ['MATCH', match_spec, subaction1, subaction2...]

Assert that the current node matches a [match specification](#matching),
then execute all sub-actions in sequence.  If it doesn't match, the graft
operation will fail with `NO_MATCH`, unless the user has enabled "fuzzy"
matching in which case the engine looks at nearby array indices to find a
match.  If a fuzzy match is successful, the array offsets discovered during
that process are applied to all property paths of the sub-actions which
reference those arrays.  This is analogous to how unix `patch` applies
"hunk at offset" by first searching for the matching lines then continuing
as if they'd been specified for that location.

The exact algorithm for performing the search for a "fuzzy" match is
dependent on the engine and parameters supplied by the user.  While this
specification leaves fuzzy matching open to experimentation, implementations
should probably *not* apply offsets when array indices specified as negative
numbers happen to alias indices where an offset is in effect.
This specification makes no attempt to define behavior of depth-wise fuzzy
matching through paths of objects, though a future revision might if common
useful strategies emerge.

### IF

    ['IF', match_spec1, action, else_action]
    ['IF', match_spec1, action1, match_spec2, action2... else_action]

For each pair of (`match_spec`,`action`), test whether the current node
matches and if so, apply the action, else continue to the next pair.
If the list ends with a single element, it is treated as an "else" action.
Lacking an "else" action, the `IF` action completes successfully even if
no sub-action was performed.  If the sub-action aborts with an error, the
`IF` action likewise fails.

`IF` uses the same fuzzy matching mechanism as `MATCH`, if enabled, however
implementations should always attempt every condition of the `IF` statement
*without* additional fuzzy matching before trying again with fuzzy matching.
Offsets derived from a fuzzy-match in a parent action will still be applied
to both attempts.

### ASSIGN

    ['ASSIGN', prop1, val1, prop2, val2, ...]

Assign a new value to one or more properties of the current node (or
sub-paths of it).  The values may be literal, or may come from paths relative
to the current node (but see below for exceptions).  All assignments are
performed "by value", deep-cloning objects or arrays, so the resulting state
of the data will not reference objects of the JGraft nor can this be used to
create cyclic or even acyclic graphs of object references.  The list of
assignments is performed sequentially, so later property and value expressions
see the modified state of the node's tree.  For each property/value pair, the
value is completely resolved and deep-cloned before modifying the destination
path.  If the value is a path, it must exist or the action fails with
`INVALID_TARGET`.

The basic usage of this action is to assign literal values to named
properties, but each property may be a deeper [path](#paths) specified as an
array, and each value may be an array to designate it as a path rather than a
literal value.  This means that `[]` in the value position refers to an empty
path (i.e. the node itself) rather than a literal empty array to become the
new value.  As usual, an array literal value can be specified by wrapping
it with another array, to disambiguate it from a path.

Objects along the property path being assigned will be auto-vivified if they
don't exist, but the action fails with `INVALID_TARGET` if it would need to
convert a scalar, object, or array to an opposing type.  Recall that arrays
are implied when an object path contains a number, and objects are implied
when the path contains a string.
To prevent inappropriate creation of sub-objects, use a containing `MATCH`
directive that asserts the presence or absence of the structure as desired.
Assigning to an array element greater than `length+1` is a fatal error,
returning `INVALID_TARGET`.  (unlike JavaScript which would permit this)
Likewise, a path through an array element at `length+1` or greater does
*not* auto-vivify and fails with `INVALID_TARGET`.

Examples:

    ['ASSIGN', 0, {x:1,y:1}]            // node[0]= { x: 1, y : 1 };
    ['ASSIGN', [0,'x'], 5]              // node[0].x= 5;
    ['ASSIGN', [-1,'x'], 6]             // node[node.length-1].x= 6;
    ['ASSIGN', [false], {x:6}]          // node[node.length]= { x: 6 };
    ['ASSIGN', ['b',0], ['a',4]]        // node.b[0]= clone(node.a[4]);
    ['ASSIGN', 'a', null]               // node['a']= null;
    ['ASSIGN', 'a', []]                 // node['a']= clone(node);
    ['ASSIGN', 'a', [[1,2,3]]]          // node.a= [1,2,3];

Additionally, paths that begin with `null` access a special namespace.  The
following path element must be an integer, or one of the following special
names (or its ID):

Name      | ID | Description
----------|----|-------------------------------------------------------------
`"const"` | -1 | constants defined in the enclosing `JGRAFT` action
`"stash"` | -2 | variables shared among actions of enclosing `JGRAFT` action

All non-negative integers may be used for temporaries local to this `ASSIGN`
action.  The top-level node of `const` and `stash` may not be assigned to,
nor *any* path within `const`.  This namespace is not actually exposed as an
array, so you cannot write to the `false` element to append to it, nor does it
have a `length` property.  Elements of this namespace must exist prior to
being read, but can be auto-vivified in the usual manner.  As a special case,
if the first property path element within `const` or `stash` is an integer,
it gets coerced to a string.  This allows auto-generated numeric elements
(which encode more compactly in JSON) for those without implying that they are
arrays.

Examples:

    ['ASSIGN', [null,0], []]                // temp0=  clone(node);
    ['ASSIGN', 'b', [null,0,'a']]           // node.b= clone(temp0.a);
    ['ASSIGN', 'a', [null,0,'b']]           // node.a= clone(temp0.b);
    
    ['ASSIGN', [null,0], ['a']]             // temp0=  clone(node.a);
    ['ASSIGN', 'a', ['b']]                  // node.a= clone(node.b);
    ['ASSIGN', 'b', [null,0]]               // node.b= clone(temp0);
    
    ['ASSIGN', [null,-2,0], ['b']]          // jgraft.stash['0']= clone(node.b);
    ['ASSIGN', 'b', [null,-1,'big_data']]   // node.b= clone(jgraft.const.big_data);

### MOVE

    ['MOVE', src_prop1, dst_prop1]
    ['MOVE', src_prop1, dst_prop1, src_prop2, dst_prop2...]

Relocate one or more values of properties within the current node's tree.
For each pair of (source,destination), processed in sequence, set the
destination to refer to the value of the source, by reference, and then
delete the source reference to that sub-tree.  The source property must
exist, and it is an error for the destination path to pass through or
terminate at the node the source path references.

This is an alternative to `ASSIGN` for when the source property is being
deleted afterward and copy-by-value is unnecessary.  It has most of the same
semantics as `ASSIGN`, but a few key differences:

  * The pairs are specified as (source,destination) instead of
    (destination,source).
  * Literal values cannot be specified (other than `null`)
    so string/number arguments are interpreted as one-element paths.
  * The `const` namespace is not usable because `MOVE` alters source data.
  * You can use half-index numbers (`.5`) to insert between existing
    elements of an array.
  * Array insertions and deletions are postponed until the end of the `MOVE`
    action, then resolved in tandem per array.

The value `null` may be used as a source to write a literal `null` to a path.
This is provided as a convenience to preserve the existence of a property that
would otherwise get deleted, without needing a followup `ASSIGN` action to fix
it.  The value `null` may be used as a destination as a way of requesting
deletion of the value without re-adding it anywhere.

The final path element of an array destination may be a half-index number
(an index plus or minus 0.5), to indicate an insertion point between
elements.  Positive numbers count from the start of the array, and negative
numbers count backward, but as a special case `-0.5` refers to the insertion
point before element `0` rather than the point right after the final element
`-1`.  As with `ASSIGN`, you may define an element at the current numeric
length of the array (optionally using the alias `false`) which will be visible
to subsequent source/dest pairs.  Since this is addressable, there is no need
to create an insertion queue there.  An insertion queue is associated with the
array object itself; relocating that array elsewhere in the tree carries the
queued insertions with it.
Implementations must either queue the insertion in relation to an array
reference, or keep track of subsequent path mutations affecting those arrays.

Enumerated for an array currently of length 2, it looks like this:

Index  | Meaning
-------|--------------------
-0.5   | Before element 0
 0     | Overwrite element 0
 0.5   | Between elements 0 and 1
 1     | Overwrite element 1
 1.5   | Boundary after element 1 and before a potential element 2
 2.5   | Illegal, until after defining element 2
 2     | Append immediately as element 2
 false | Append immediately as element 2
 -1    | Overwrite element 1
 -1.5  | Between elements 0 and 1
 -2    | Overwrite element 0
 -2.5  | Before element 0
 -3    | Illegal, until after defining element 2
 -3.5  | Illegal, until after defining element 2

As usual, negative numbers used in paths must alias to a position relative to
existing elements of the array, so `-length - 0.5` is the lowest allowed
negative number.

A deletion naturally implies shifting the following elements to fill the gap;
to prevent confusion in the indices of subsequent source and destination
paths, the shifting does not occur until the end of the `MOVE` action after
all source/dest pairs have been processed.
Until then, reading from the the moved array index becomes an `INVALID_TARGET`
error unless a new value is written there first.
If you write a new value to an element where a deletion was scheduled, the
deletion is cancelled, including if you manually assign the value `null` to it
or cause it to auto-vivify (as per `ASSIGN`) by specifying a destination
path through it.  When processed, each span of deletions is replaced by any
insertions from that span.  The sequential ordering of remaining elements,
queued deletions and queued insertions is preserved as they are resolved.

Since insertions and deletions do not occur until all movement pairs have been
processed, all index (and half-index) numbers counting forward from the start
of the array refer to the same coordinates throughout the `MOVE` action.
Index (and half-index) numbers which count backward from the end of the array
are resolved based on the current length of the array (not including pending
insertions) at that point in processing movement pairs.

Half-index values may *only* be the final element of the destination path of
the pair.  You cannot subsequently use them as source paths, nor write
through them to deeper sub-paths, nor auto-vivify through them.  The value
written essentially disappears from the namespace until the end of the `MOVE`
action when all the insertions and deletions get resolved.
To use multiple `MOVE` pairs to construct the tree that needs inserted,
perform the changes in the temporary namespace before a final move to a
half-index destination.

The practical consequences of array handling are:

  - No expensive deep-copying of objects is required, nor processing to
    determine whether deep-copying would be required.
  - It is not possible to degrade the data from a tree to a graph, because
    for every reference added, the old reference is immediately destroyed
    (or made inaccessible) and the no-prefix rule prevents cycles.
  - If an editor has re-ordered the elements of an array, and knows the source
    and destination index of each, it can easily export that edit as a `MOVE`
    argument list from those pairs by adding ".5" to each destination.  Every
    deletion will be paired with an insertion and the `MOVE` action can run in
    `O(N)` time without actual splice() calls.

Examples:

    ['MOVE', 0,[null,0],           // tmp['0']= node[0];
                                   // mark_vacant(node, 0);
            -1,0,                  // node[0]= node[node.length-1];
                                   // mark_populated(node, 0);
                                   // mark_vacant(node, node.length-1);
            [null,0],-1]           // node[node.length-1]= tmp['0'];
                                   // mark_populated(node, node.length-1);
                                   // mark_vacant(tmp, '0');
    
    ['MOVE',1,-0.5,2,false]        // pos0_queue= [ node[1] ];
                                   // mark_vacant(node, 1);
                                   // node.push(node[2]);
                                   // mark_vacant(node, 2);
                                   // node.splice(1,2);
                                   // node.splice(0, 0, ...pos0_queue);
    
    ['MOVE','a','b','c','a']       // node.b= node.a;
                                   // delete node.a;
                                   // node.a= node.c;
                                   // delete node.c;
    
    ['MOVE',['x','x'],[null,0],    // tmp['0']= node.x.x;
                                   // delete node.x.x;
            'x',[null,0,'x'],      // tmp['0'].x= node.x;
                                   // delete node.x;
            [null,0],'x']]         // node.x= tmp['0'];
                                   // delete tmp['0'];

### SPLICE

    ['SPLICE', offset, count, replacement1, replacement2...]

Replace a span of an array, just like the splice function found in most
programming languages.  The current node must be an array or the graft fails
with `INVALID_TARGET`.  `offset` may be `false` as an alias for the length of
the array, or negative to count backward from the end of the array where -1
is the final element, but the resulting index must be within `[0..length]` or
the graft fails with `INVALID_TARGET`.
`count` is the number of elements to repalce.  Negative numbers are aliases
for the count that would end just before element `length-N`, and `null` is an
alias for `length-offset`.  The resulting count must be within
`[0..length-offset]`.

The replacement values (which may be paths) use the same specification as the
values of the [`ASSIGN` action](#assign), and the copy-by-value semantics of
the `ASSIGN` action with the exception that implementations may re-use trees
from the deleted portion of the array if only one reference is preserved.
All replacement values are resolved before starting the splice operation.

### SPLIT

    ['SPLIT', split_spec, subaction1, subaction2...]

This can only be applied at a string node, or it fails with `INVALID_TARGET`.
It splits the string according to `split_spec` to create an array, then runs
each sub-action on that array, then re-assembles the string from the elements
of the array.  The separator and any trimmed characters are preserved to use
when re-assembling the string; the generated array carries them as hidden
metadata.  Actions that modify the array also modify the metadata in parallel
as described below.

The `split_spec` may be a simple string used as a verbatim separator, or it may
be an object with a more elaborate specification:

    {"sep": ...,        // A string or regex to split on
     "element": ...,    // A regex that must match prior to checking 'sep'
     "default": ...,    // The default separator used for joining new elements
     "keep": ...,       // Preserve the separator within the generated array
     "ltrim": ...,      // A string or regex to trim from start of elements
     "rtrim": ...,      // A string or regex to trim from end of elements
     "trim": ...,       // sets 'ltrim' and 'rtrim' to the same value
     "discard": bool,   // Don't preserve trimmed chars when re-assembling
    }

Regular expressions are specified using either a host Regex object, or in the
[portable JSON array notation](#regex) specified below.  The split operation
iterates through the characters of the string looking for a match, and notes
how many characters matched.  (When using the portable regular expressions,
this will always be the longest match, but host language regex objects may
behave differently and vary between versions of the host language)
If you specify an `element` regex, a test for separator will only match when
the element characters (from the end of the previous separator until the
current match position) also match the `element` regex.   The `element` regex
is implicitly anchored to the start and end of the element characters.
This gives you a way to pass over escape sequences without accidentally
detecting a separator, and without adding the complexity of "look-behind" or
"look-ahead" or capture groups to the portable regexes.

A zero-length separator (including a successful zero-length regex match)
is considered to match at every boundary between characters except the one
between the end of the previous separator and the following character
(i.e. the current position) which prevents an infinite loop of empty
elements.  This means the separator of `''` can be used to split the string
into its individual characters.  It also means that a regular expression which
can match the empty string will generate a single-character element any time
none of the alternatives matched.

For example, the pattern `/a*/` applied to the string "baabca" results in:

    ['b', 'b',  'c', '']

(Note that implementations may place limits on the resources consumed by a
JGraft, and splitting a string into an array of individual characters is a
quick way to reach those limits.)

Once a separator is found, it uses the value of `keep` to determine what to do
with it.  The default is to hide the separator from the resulting array but
store it (attached invisibly to the preceding element) until it is time to
reconstruct the string.
The value "<" means to keep the separator as part of the current element in
the generated array, and likewise ">" means to keep it as part of the
following element.  The value '@' means to include the separator as its own
element of the resulting array.  In all three cases, separator metadata is not
used, and elements get reassembled with an empty string as the separator since
the actual separator is part of the data.

`ltrim` and `rtrim` are used to hide unimportant characters from the resulting
array element strings, such as leading/trailing whitespace (like `diff -b`)
and you can (exclusive to the others) specify `trim` to set them both to the
same value.  `ltrim` is implicitly anchored to the start of the element string,
and `rtrim` is implicitly anchored to the end, allowing a common value for
`trim`.  The longest match of `ltrim` gets removed first, followed by removing
the longest match of `trtim` from the remainder, to resolve cases where the
trimming could overlap.
If you want to permanently remove the trimmed characters, use `discard: true`.

If the separators are not added to the array using `keep` they are stored in
a parallel array of metadata associated with the array.  Likewise, trimmed
characters are stored there.  As actions like `ASSIGN`, `MOVE`, and `SPLICE`
process the array, assignments from one element of the array to another carry
the separator and trimmings to the new element.  Assignments of values from
any other source (constants, a different array, etc) do not carry this
metadata, and eliminate trimmings at the assigned element, though the
separator of that element remains.  Deletions on the array perform a parallel
deletion on the metadata array, and every insert operation clones the
separator of the previous element (and if none, the following element) such
that every element continues to have a defined separator.  Trimmings do not
get cloned.
If the array becomes empty at any point, the separator reverts to `default`.
`default` must be specified when `sep` is a regular expression and `keep` is
not used; if `sep` is a string then it also implies `default` and if `keep`
is used then separators are not relevant to the join operation.

Once all actions have been applied to the array, it gets reassembled into a
string.  Each element must be a string, or it fails with `INVALID_TARGET`.
(a particularly generous implementation might detect whether the non-string
came unconditionally from the graft itself and report `INVALID_GRAFT` instead)
Each element gets any hidden trimmed characters re-added, and any hidden
separator is appended only if there is a following element.  In order to get a
"newline at the end of the file" the final element of the array needs to be an
empty string.  It is the responsibility of the nested actions to preserve or
add this element as desired.

This is the primary tool used to re-implement text diff/patch behavior:

    ['SPLIT',
      { sep: ["|", "\n", "\r\n"], default: "\n", trim: ["{",0,null," "] },
      ['MATCH',
        ['ARRAY', 10,
          "Line ten",
          "Line eleven",
          "Line twelve",
          "Line thirteen"
        ],
        ["SPLICE", 11, 2,
          "New Line eleven"
          // no line 12
        ]
      ],
    ]

### LOG

    ['LOG', level, message]
    ['LOG', level, message, data]

As JGraft applies a patch it may emit diagnostic information, such as when
a match succeeded at an offset from the declared array.  You can emit your own
custom diagnostics as well, using this action.  The motivation is that while
a tool like `patch` can make a fairly straightforward diagnostic about why
applying a text diff failed, a JGraft mismatch can be much harder to explain.
An author of a JGraft might have more domain-specific information about why
something wouldn't match, and can encode that with some `IF` actions in a way
that the graft operation fails with a useful error message.

The `level` argument is one of the following strings, or the corresponding ID:

String | ID | Meaning
-------|----|----------------------------------------------------------------
"info" |  0 | shown if the user asks for details
"warn" |  1 | flagged for the user even if they didn't ask for details
"fail" |  2 | a fatal error that ends the graft attempt with `NO_MATCH`

`message` may be a literal string, or an array containing a path relative to
the `const` namespace which resolves to a string.  The string should be
generally printable, and may not contain unicode Surrogates, Noncharacters,
or control characters (codepoints 0x00–0x1F and 0x80–0x9F).

`data` is an optional literal value or path with the same rules as described
for values in the `ASSIGN` action.  It provides data to the user relevant to
the message.  This may be shown to the user in some form such as JSON, or
inspected programmatically by code using a JGraft library.  If you want to
provide multiple pieces of data you may first assemble an object structured
as you like using an `ASSIGN` action to write to the `stash` namespace, and
then reference that object in this action.  If the data is not being consumed
immediately, implementations ensure it remains stable, such as deep-cloning
it or setting up copy-on-write.  This is already handled if the standard
operation of the engine is to shallow-clone the ancestors of each change.

## <span id="matching">Context Matching</span>

JGraft provides a rich collection of match specifications.  The matching
system has its own collection of match functions specified as lisp-style
arrays where the first element is a function name and the remaining elements
are passed as arguments to that function, but scalars, objects, and array
literals (specified using an array of one array) are used as shorthand for
the `HAS` and `IS` functions.  `HAS` requires only specified properties to
exist in the current node; `IS` requires that all properties match between
the specification and current node.  `HAS` and `IS` establish a scope for
how child nodes are interpreted.

The following functions all implicitly operate on the current node and return
a boolean of whether the current node passes the test:

 Name      | ID | Description
-----------|----|----------------------------------------------------------
 HAS       | 3  | Switch to partial matching
 IS        | 4  | Switch to exact matching
 NOT       | 0  | None of the conditions match at current node
 OR        | 1  | At least one condition matches at current node
 AND       | 2  | All conditions match at current node
 EXISTS    | 5  | Current node exists (including `null` values)
 BOOL      | 10 | Match any boolean
 NUM       | 11 | Match any number, or restricted range of numbers
 INT       | 8  | Match any integer, or restricted range of integers
 STR       | 7  | Match any string, or strings restricted by a regex
 ARRAY     | 9  | Match any array, or array with specified sub-range
 SPLIT     | 6  | Split a string to match against the resulting array

### HAS

    null                                node === null
    0                                   node === 0
    'str'                               node === 'str'
    { a: 1 }                            node.a === 1
    [OR, 1, 2]                          node === 1 || node === 2
    { a: [EXISTS] }                     'a' in node
    { a: [AND,[EXISTS],[NOT, null]] }   'a' in node && node.a !== null
    { a: [OR, [NOT,[EXISTS]], 1 }       !('a' in node) || node.a === 1
    { a: [[]] }                         isArray(node.a)
    { a: [IS, [[]]] }                   isArray(node.a) && node.a.length == 0
    { a: [IS, [[1,2]]] }                isArray(node.a) && node.a.length == 2
                                        && node.a[0] === 1 && node.a[1] === 2
    { a: [ARRAY, 3, 6, 7] }             isArray(node.a) && node.a[3] === 6
                                        && node.a[4] === 7

This is the default function implied at the top of a match specification.
It establishes a scope where the following rules apply:

For scalars, this function is equivalent to JavaScript's `===` operator.

For objects, the current node must be an object, and each specified property
which is not a function must match the same value on the current node
according to `HAS`.  If a specified property is missing from the current node,
a comparison is still attempted during which the current node is effectively
an undefined value which doesn't compare equal to anything.  The comparison
could still return true if it makes use of `NOT`.

Arrays are used to specify alternate match functions, and will receive a
current node of whichever sub-property is being matched.  Literal arrays are
specified as an array of one element holding the literal array.  They requiure
the current node to be an array where each specified element matches according
to `HAS`.  Like with objects, comparisons will be attempted for specification
of elements beyond the end of the current node.

Extra properties / array elements in the current node (beyond what was
specified) are ignored.

### IS

    ['IS', match_spec...]

This is a variant of 'HAS' that forbids extra properties in the current node
which were not in the specification object.  This sets up a scope where all
contained non-functions are also interpreted as `IS` tests.

The function can take additional arguments to perform an implied 'OR'.

### AND

    ['AND', match_spec, match_spec...]

All following patterns must match at the current node.  There must be at least
two arguments.  Though side-effects should not be visible from matching,
implementations should still adhere to short-circuiting behavior, testing the
arguments in order and stoping at the first failure.

### OR

    ['OR', match_spec, match_spec...]

Any one of the following patterns must match at the current node.  There must
be at least two arguments.  Implementations should adhere to short-circuiting
behavior in case side effects become visible, testing the arguments in order
and stoping at the first success.

### NOT

    ['NOT', match_spec]
    ['NOT', match_spec, or_match_spec...]

Matches when none of the arguments match at the current node.  When more than
one argument is used, it behaves identical to `[NOT,[OR,...]]`.

### EXISTS

    ['EXISTS']

True when the current node is defined, including when the value is `null`.

### BOOL

    ['BOOL']

Return true if the current node is a boolean.  This is essentially just
shorthand for `[OR, true, false]`, but slightly more performant.

### NUM

    ['NUM']
    ['NUM', min, max]

With no arguments, returns true if the current node is a number, including NaN
and Inf.  Note that JSON-compatible data structures cannot contain these
values, but they are included here for use with host language support.

With two arguments, also verify that the number is within the range of
`[min,max]` (inclusive).  `min` or `max` may be `null` to omit the respective
test.  In contexts where infinity is available, positive and negative infinity
may also be used for the `min` and `max` values.  In contexts where NaN is
available, NaN may not specify a bound of the range, and declaring any range
causes NaN values in the data to fail to match.

### INT

    ['INT']
    ['INT', min, max]

Same as `NUM` above, but rejects all non-integer values.  (the container used
by the host language may be a float, so long as it has an integer value)

### STR

    ['STR']
    ['STR', regex_spec]

With no arguments, returns true if the current node is a string.

You may specify a [regular expression](#regex) to additionally constrain which
strings match.  The regex is not implicitly anchored.

### ARRAY

    ['ARRAY']
    ['ARRAY', length]
    ['ARRAY', offset, match_spec1, match_spec2...]

With no arguments, this merely asserts that the current node is an array.

With one argument, it asserts the current node is an array with at least this
number of elements in the scope of `HAS`, or exactly this number of elements
in the scope of an `IS` function.

With more than one argument, this asserts that a range of the array matches
the specification of the respective element, inheriting the `HAS` or `IS`
scope.  The `offset` may be negative to count backward from the end of the
array.

    // Assert that the array ends with 'x', 'y', 'z'
    ['ARRAY', -3, 'x', 'y', 'z']

### SPLIT

    ['SPLIT', split_spec, match_spec]

Matches when the current node is a string that, when split into an array of
strings, matches the `match_spec` argument.
This uses the same `split_spec` as the `SPLIT` action.
The match against the array is also subject to fuzzy matching.

## <span id="regex">Regular Expressions</span>

If the host language has direct support for objects encapsulating a regular
expression, and JSON serialization is not required, you may use those objects
in any parameter that allows regular expressions, to take advantage of the
full power and capabilities of your host language.

If you need to portably serialize the JGraft, the regular expressions may be
specified as (you guessed it) lisp-style structure describing the pattern.
Think of it as a pre-parsed regular expression.  The notation is not intended
to be written by hand, but aims to be readable enough for debugging.

Implementations can try to flatten this structure into the syntax relevant for
the host language's regex engine, or just directly implement the character-
matching engine, which may actually be easier.
The capabilities of this regular expression specification are intentionally
limited to improve the odds that each host language can compile it into the
host's native regexes, and textbook "regular" so that they give a clean
definition for "longest match".
In particular, it does not deal with case insensitivity, unicode databases,
back-references, or sub-pattern captures, and does not need to worry about
concepts like "greedy" matching.
This does require that implementations be able to perform finite automata
semantics for the longest match, which is an easy fit for the standard C
library, but a bit harder for engines like JavaScript, Python, or Perl
which match according to a predictable backtracking order.

Character matching is defined over a sequence of Unicode code point values.
It is left to implementations or host languages to determine (or provide
configuration for) how input strings are converted to this sequence,
including the treatment of malformed encodings and unpaired surrogates.
For example, an implementation might reject malformed UTF-8 or strings
containing unpaired surrogates, replace malformed input, or expose surrogate
code points individually. These choices are outside the JGraft specification.

JGraft character classes may include surrogate code point values
(U+D800–U+DFFF). These can only match when the host's interpretation of the
input string exposes corresponding values in the character sequence.

### Common Subexpressions

There is a "common subexpression" namespace available to the regular
expressions.  This helps make up for the lack of built-in unicode tables and
case insensitivity and general verbosity of the JGraft regex structure.
The common subexpression namespace is a simple key/value store where the key
is a string of printable ASCII or non-ASCII unicode and the value is any
structure that can be interpreted as a valid JGraft regular expression.

When references are made to common subexpressions, they are resolved as if
the referenced structure had been specified in-line.  Subexpressions cannot
contain cyclic references.  In particular, if a common subexpression is
specified using a reference, the reference must already exist and gets
resolved immediately.

The initial value of the common expression namespace can be supplied by an
enclosing ['JGRAFT' actions](#jgraft) via the metadata property
`regexCommon`.  Temporary overrides can also be specified via regex Sequence
functions that begin with a `{}` configuration block.  In both cases, the list
is specified as an array of property/value pairs to apply to the namespace, in
order.  Integers are permitted as the property name (implicitly coerced to
strings) since numeric iteration is the simplest thing to generate and
integers encode more compactly in JSON than strings.

Action Example:

    ['JGRAFT',{
      regexCommon: [
        "word", ["[", 48, 57, 65, 90, 95, 95, 97...],
        ...

Function Example:

    [{"common":["linebreak", ["|", "\n", "\r\n"]]}, ...]

### Functions

The functions are named according to their typical regex syntax:

Function                | Description
------------------------|-----------------------------------------------------
`['', ...]`             | Sequence - match each element in order
`[{}, ...]`             | Sequence with configuration
`["^"]`                 | StartAnchor - Start of string
`["$"]`                 | EndAnchor - End of string
`["|", ... ]`           | Alternation - Match any one alternative
`["[", ... ]`           | Charclass - A set of distinct characters
`["{", min, max, ... ]` | Repetition - Match zero, one or more times
`["&", name]`           | Ref - reference a common subexpression

#### Sequence

    ['', ...]
    [config, ...]

This is provided as a container for a list of other elements.  It may
optionally begin with an object specifying configuration details:

  - common: an array specifying [common subexpressions](#common-subexpressions)

#### StartAnchor

Matches only at the start of the string.

#### EndAnchor

Matches only at the end of the string.  (not right before a newline like regex
engines often do)

#### Alternation

    ['|', this, or_that...]

Match any one alternative.  Requires two or more parameters.  Each parameter
may be a single string which must match exactly, or an array indicating a
function.  The order of alternatives should not matter in theory, because the
output is a simple boolean and does not track positional matches of the
pattern components.
Implementations which convert this specification to the host language's regex
engine are free to attempt to re-order the alternatives in the order that
yields the highest performance.

#### Charset

    ["[", ...
    true,                // include following elements
    false,               // exclude following elements
    "abcdef",            // 6 literal characters
    "a","z",             // a range of characters to be added
    "a",null,            // open-ended range
    48,58,               // unicode codepoint range
    ['&',name]           // reference a named charset for inclusion/exclusion

Define a set of characters.  Boolean values switch between inclusion and
exclusion.  The default is inclusion to an empty set.  If the first element
is false, it instead becomes exclusion from the complete set of Unicode.
Further boolean values are allowed in order to exclude individual items from
ranges, etc.
When true or false values are encountered, it switches between inclusion
and exclusion respectively.  A string of more than one character is considered
a list of characters to include.  A string of one character begins a range if
the following element is also one character (or number or `null`, for an
open-ended range), otherwise it is interpreted as a single character to
include or exclude.  A number is treated as a unicode codepoint value, and
likewise begins or ends a range.  Finally, you may use `['&',name]` notation
to reference a [common subexpression](#common-subexpressions), which must
resolve to a charset definition.  (a name resolving to anything else results
in `INVALID_GRAFT`)

Examples:

    // .
    ["[", 0,null]
    ["[", false]    // because it starts from the full set of chars
                    // and removes nothing
    
    // [0-9]
    ["[", "0123456789"]
    ["[", "0", "9"]
    ["[", 48, 57]
    
    // [0-9a-zA-Z_.-]
    ["[", "0","9", "A","Z", "a","z", "_-."]
    ["[", 0x30, 0x39, 0x41, 0x5A, 0x61, 0x7A, "_-."]
    
    // [^\n]
    ["[", false, "\n"]

#### Repetition

    ["{", min, max, pattern...]

The first two parameters specify the minimum and maximum repeat count for
which all remaining parameters (the pattern components) must be found.
`min` must be an integer greater or equal to zero, and `max` must be greater
or equal to `min`, or `null` to enable unlimited matching.

Regex Notation | Function notation
---------------|--------------------------
 `*`           | ['{',0,null,...]
 `?`           | ['{',0,1,...]
 `+`           | ['{',1,null,...]
 `{5}`         | ['{',5,5,...]
 `{3,5}`       | ['{',3,5,...]

#### Ref

    ["&", name]

Inject the value of a common subexpression.  Name is a property name or path
within the [common subexpression](#common-subexpressions) namespace.

