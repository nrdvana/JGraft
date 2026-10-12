# JGraft

## Introduction

JGraft is a data structure that describes how to edit a tree of data.  It can
describe edits where portions of the source and destination are fully defined
and thus reversible, edits applicable to any compatible data structure, or
conditional edits where the change depends on the state of the target data.
It is a data structure rather than a parsed language to make it easier to
programmatically generate, combine fragments, extract fragments, and inspect.

JGraft is constructed of basic JSON-compatible data types:
`null`, booleans, numbers, JSON-compatible strings, arrays, and objects.
Implementations may include other native host-language data types if the
application doesn't need to interoperate with different host languages.
(implementations allowing other host language types are responsible for
defining equality, ordering, cloning, and serialization behavior for them,
which is outside the scope of portable JGraft)
While performing edits, JGraft additionally defines types for
[dates and date-times](#dates), though these export back to JSON-compatible or
application-configured data types before committing the edit.

The JGraft data structure describes "actions" and "expressions" in addition
to literal data.  Actions have the potential to mutate the tree or exhibit
other side effects, but they are restricted to ensure the input tree remains
a tree.  Expressions include match patterns and functions, and have no side
effects.  JGraft is designed so that untrusted data and grafts can be
processed safely when implementations enforce appropriate resource limits,
such as memory usage limits, node creation limits, and stack depth limits.

## Representation

The JGraft tree contains actions and expressions, but JSON has no mechanism to
tag data with a type.  So, when serialized to JSON they take the form of
arrays in the style of the Lisp programming language, where the first element
indicates the operation and remaining elements become an argument list for it.
The operation (first element) may be specified using either a name or numeric
ID; the ID is useful for compact encoding in production, and the name is
useful for development and debugging.
In [contexts](#contexts) allowing a mixture of data and expressions, array
literal values get "escaped" by wrapping them with another array.
In other words, in the JSON representation of JGraft in a context that allows
data and expressions, an array containing exactly one element which is also an
array means a literal array having the elements of the inner array.
The elements may then recursively contain expressions or array literals by this
convention.

Implementations may choose to use host-language objects of some sort to
represent actions and expressions at runtime, then perform the array encoding
during serialization and reconstruct the objects during deserialization.
Implementations could also choose to leave the JGraft data
structure in its JSON-compatible form and interpret it on the fly.
They should make it clear to the developer whether the data structure the
application code interacts with is using objects and literal arrays, or the
direct "arrays-are-special" representation of JGraft.
The remainder of this specification uses the JSON form of actions and
expressions, since the object model of a host language is left unspecified.

### Contexts

Every action and expression declares the context of each of its arguments.
Contexts help reduce the verbosity of the encoding by giving more complex
meaning to simpler JSON structures.

#### Value Context

This constructs a new value or tree of values from a specification which is
a mixture of structure and expressions.

  * Non-object/non-arrays are literal values to appear in the constructed
    result.
  * Objects in the specification construct an object of the same properties
    where each property value is evaluated recursively.
  * Escaped arrays (an array within an array of one element) likewise
    construct an array where each element of the specification is recursively
    evaluated to generate the value for that element of the result.
  * Other arrays represent expressions, which will be evaluated to generate the
    corresponding value for the result.

Reference expressions used in value context may only refer to paths which
exist.  Missing path properties generate an `INVALID_TARGET` error.

The constructed value is logically a new tree, however implementations may
re-use existing tree fragments so long as they can ensure that the result is
not used in a way that could degrade the target data tree to a graph.

#### Match Context

This determines whether the current node has a shape compatible with the
specification structure.

  * Non-object/non-arrays describe literal values to be compared for equality
    with the target.
  * Objects and escaped arrays are considered structure to be matched.
  * Other arrays represent expressions, which are handled specially depending
    on their type.  Match expressions become part of the matching algorithm,
    while all other expression types evaluate to a plain value and then use
    the value as structure to match against.

The section [Matching](#matching) describes the process in detail.

#### Regex Context

Regular expressions are implemented using mostly the same specification
structure as general matching, but operating on a virtual array of characters.

  * One-character strings describe characters
  * Strings with more than one character implicitly become [`SEQ`](#ex-seq)
    expressions, matching a sequence of characters
  * Non-negative integers describe unicode codepoints
  * Other numbers, objects, escaped-arrays, and dates in the specification
    generate `INVALID_GRAFT`, or if they are the result of an expression they
    generate `INVALID_OPERAND`.
  * Referenced subexpressions (via [`EXPR`](#ex-expr) ) get re-interpreted
    according to these rules.

The section [Regex Matching](#regex-matching) describes the process in
detail.

#### Path Context

[Paths](#paths) are used to refer to data within one of the
[namespaces](#namespaces), and may be used for reading or writing.

  * Non-object/non-arrays are interpreted as a single argument to a
    [`NODE` expression](#ex-node).
  * Arrays represent expressions, but they are restricted to path expressions,
    and may be further restricted to specific namespaces.
  * Objects and escaped arrays are illegal in this context.

#### Other Contexts

Other contexts may be defined by individual actions or expressions, usually
in terms of providing arguments to an implied expression.

### Dates

Dates are a problematic domain to make a specification for.  JGraft tries to
keep the implementation burden low, like not requiring timezone tables, but
applications almost certainly want some integration with the date objects of
the host language.  Meanwhile, JSON doesn't even define a notation for dates.

JGraft takes an approach where dates and date-time values are represented by
temporary objects that logically exist only within the JGraft implementation.
They are created by the [`CASTDATE` function](#ex-castdate), and any operation
that would write them into the target tree of data automatically exports them
to a format of the application's choosing.  In the simplest case, the
application provides host-language date objects in the target tree of data,
the JGraft implementation uses those same objects, and no export steps are
required when writing them back to the data tree.  In other cases, the
application might be parsing ISO 8601 date strings from the tree of data and
returning them to that format as they re-enter the target data, losing any
metadata that would differentiate them from plain strings.  JGraft dates and
date-times may be saved into the const, stash, and temporary namespaces
without losing their metadata.  When exported as an "undo" JGraft, dates are
represented by a `CASTDATE` expression.

JGraft defines three distinct types: date, datetime, and timestamp.  They are
considered immutable values, but expose virtual properties which may be
referenced in [paths](#paths).  `date` lacks a time component, and has virtual
properties of "year", "month" and "day", according to the rules of the
Gregorian calendar and represented in their natural 1-based format.
`datetime` additionaly has "hour", "minute" and "second" virtual properties,
where `second` may be fractional to any numeric precision allowed by JSON and
may also be a leap-second value (0 <= second < 61).  Implementations don't
need to check whether a leap second was historically valid; just preserve the
field and compare it last during less-than tests.  Implementations may need to
hold the `second` value in an extra data field if the host date objects
don't support leap seconds or arbitrary precision.  JGraft instances should
generally avoid *creating* datetimes with leap seconds, as they can't be
exported in some host language encironments.
`timestamp` additionally has a `tz_offset` virtual property, which is a number
of minutes offset from UTC.  `timestamp` references a specific point in time,
and can be compared with timestamps from differing timezone offsets.

Since each of these is a different type, the usual rules apply when testing
them: dissimilar types cannot "match" in a match expression, and the `LESS`,
`LESSEQ`, `MIN`, and `MAX` expressions will fail with an `INVALID_OPERAND`
error.  You can use the `CASTDATE` function to cast values to the same
precision for comparison.

## Namespaces

Actions and expressions can make references to data in six namespaces:

  * Current Node       - a tree of data supplied as input, implicitly
                         referenced by many expressions and directly by the
                         [`NODE` expression](#ex-node).
  * Constants          - a read-only tree of values declared in the
                         [`JGRAFT` action](#act-jgraft) and referenced by the
                         [`CONST` expression](#ex-const).
  * Common Expressions - a read-only collection of parsed/validated expresson
                         trees declared in the [`JGRAFT` action](#act-jgraft)
                         and referenced by the [`EXPR` expression](#ex-expr).
  * Stash              - a read/write tree of temporaries shared between
                         actions in the scope of the same `JGRAFT` action,
                         referenced by the [`STASH` expression](#ex-stash).
  * Match Captures     - a read-only tree of data captured by the named
                         sequences in an enclosing `MATCH` or `IF` action,
                         referenced by the [`CAP` expression](#ex-cap).
  * Temporaries        - a read/write tree of temporaries that only exists for
                         the duration of one action, referenced by the
                         [`TEMP` expression](#ex-temp).

## Paths

Several actions and expressions use the concept of a "path".  A path is a
list of object properties, array indices, or aliases that descend into the tree
of a namespace.

In general, strings may only refer to object properties and integers may only
refer to array elements.  This is particularly important when auto-vivifying a
path; integers will create arrays and strings will create objects.
However, the `CONST`, `EXPR`, `STASH` and `TEMP` namespaces permit a leading
integer as a property name (coerced to a string) even though the namespaces
aren't arrays, for the improved serialization size.

Negative numbers are available as aliases for array elements.  Specifically,
a value of `-n` (where `n > 0`) means `[SUB, [LENGTH [...path_to_array]], n]`,
or in other words, the array element at `[array.length - n]`.  The value
`false` is an alias for `[LENGTH [...path_to_array]]` of the current array.
This position immediately after the final element may not be read, because no
element exists there, but it may be written to append to the array.
(`-0` would be the ideal alias, but too many languages drop that distinction
when decoding JSON)  The values `null` and `true` may not appear in paths.
Certain actions may extend path syntax for particular arguments; MOVE permits
a half-integer as the final destination selector.  A unspecified or illegal
type of literal value appearing in a path raises an `INVALID_GRAFT` error.

Paths always navigate deeper into the tree and have no concept of "up a level"
like a filesystem's `..` convention.  A path element *may* be another
expression which evaluates to become the name or number or alias of a
property; the computed value must conform to the same rules as literal path
elements or it results in `INVALID_OPERAND`.

## Action Overview

Actions describe the overall algorithm of the JGraft.  Each action dictates
the meaning of its arguments; they may include other sub-actions, match
expressions, objects of configuration properties, and so on.  Consequently,
validation or inflation of host language objects needs to be handled
recursively per-action.  Actions occupy a namespace apart from expressions,
and are not interchangeable.

The root of a JGraft is the `JGRAFT` action.  The `JGRAFT` action may contain
metadata and scope-based configuration.  It establishes a scope for common data
and expressions and stash variables available to all sub-actions.  The common
data and expressions help reduce redundancy within that tree, and the stash
facilitates transfer of values between actions.

The primary intended workflow for JGraft is to use a `MATCH` or `IF` action to
verify that the input data tree matches expectations (or find a matching
location in the tree with some [fuzzy matching](#fuzzy-matching) ) and then
within that use `COPY`, `MOVE`, or other actions to alter the tree.

The list of actions is fairly concise:

Name                  | ID | Description
----------------------|----|----------------------------------------------
[JGRAFT](#act-jgraft) | -1 | Declare metadata; top-level element
[AT](#act-at)         |  0 | Navigate to sub-node and execute sub-actions
[MATCH](#act-match)   |  1 | Assert current node matches pattern
[IF](#act-if)         |  2 | Choose action based on which pattern matches
[COPY](#act-copy)     |  3 | Assign-by-value to properties
[MOVE](#act-move)     |  4 | Move existing values between properties
[SPLICE](#act-splice) |  5 | Perform standard splice() on an array
[SPLIT](#act-split)   |  6 | Split string into array, apply actions, re-join
[LOG](#act-log)       | -2 | Emit diagnostic message and data

The `SPLICE` action is an optimization vs. `COPY` or `MOVE` for making
changes to a range of an array.

The `SPLIT` action is a tool for parsing strings in order to make changes
within them, then reassembling the modified string.  It can also be used to
access specific substrings and copy them into stash variables for use in other
actions.

The `LOG` action can be used to provide improved diagnostics to a user
applying a JGraft.

## Expression Overview

Expressions are essentially "functions without visible side effects", though
there are two special categories:

  * "Reference Expressions" reference a [path](#paths) within the input data
    or one of the special namespaces JGraft offers, and in some cases may be
    written as well as read.
  * "Match Expressions" compare the current node to a structural pattern and
    return a boolean of whether it matched.  Match expressions have special
    behavior when calling other match expressions.

Everything else is just a "Function".  Functions don't need to be pure in the
mathematical sense, but the "no visible side effects" rule is important
because they may be evaluated any number of times during fuzzy-matching, and
because JGraft does not define an order for enumerating object properties
during the matching process.  They *may* have a side effect of incrementing
a resource counter that aborts the graft operation due to resource usage, but
that is not visible *to the expressions*.

### Matching

The matching system compares the current node with a specification structure
composed of literal data and/or subexpressions.  It operates on several pieces
of scoped state:

  * `exact` - whether unspecified properties/elements cause a match to fail.
  * `regex` - whether the match is applied to a virtual array of characters.
  * `sequence` - how expressions may match variable numbers of array elements.

The `exact` flag is set for an expression subtree using the `IS` or `HAS`
expressions.  The `sequence` mode is enabled with the [`SEQ`](#ex-seq),
[`SEQMIN`](#ex-seqmin) or [`SEQMAX`](#ex-seqmax) expressions but is not
inherited by subexpressions which match deeper into the data tree beyond the
current array.  The `regex` flag is set by any expression declared in
[regex context](#regex-context) such as within the [`STR` expression](#ex-str).

For all contiguous portions of the match specification composed of plain
structures or match expressions, the matching algorithm continues building on
the same match state.  If the match algorithm encounters an `EXPR` path
expression, the referenced structure (which may contain expressions) becomes
part of the current match specification and likewise continues building on the
same match state.
If the matching algorithm encounters any other type of expression, it evaluates
it as a function and uses the resulting value (which may not contain
expressions) as a literal structure to match against.
If the function expression includes an inner match expression, the inner match
expression begins a new match state independent of any ancestor match state.
It retains the current node, but begins with default HAS semantics and no
sequence context, and only gets regex context if matching against a virtual
array of characters.  This is covered in more detail under
[expression interaction](#expression-interaction).

The baseline for a match is that nothing can compare equal unless it is the
same type.  Plain objects can only match plain objects, arrays can only match
arrays, and each type of scalar can only match that same type of scalar.
For non-object non-array elements of the specification, matching checks for
equality as follows:

  * `null`, `true`, and `false` are equal only to themselves.
  * Numbers compare using the implementation's native representation of JSON
    numbers after decoding.  JGraft does not specify behavior related to
    floating point roundoff; see the [`NUM` expression](#ex-num) for a quick
    way of checking whether a value is within a range.
  * Strings compare codepoint-to-codepoint, though implementations may be
    configurable in determining how an array of codepoints is generated.
  * `date` can only match `date`, and all fields must be identical.
  * `datetime` can only match `datetime`, and all fields must be identical.
  * `timestamp` can only match `timestamp`, and both must refer to the same
    instant after adjusting for time zone offset.

For an object `spec_obj` in the specification being compared to current node
`data_obj`, the matching algorithm iterates the properties of `spec_obj` and
temporarily sets the current node to the corresponding property of `data_obj`,
even if no such property exists.  It then recursively tests the specification
for that property.  This may succeed even on a nonexistent current node due to
the [`NONE` expression](#ex-none) which returns true when a pattern *doesn't*
match.

For arrays in the specification, the matching algorithm takes steps identical
to object-matching, iterating the array elements instead of object properties,
and likewise fails in `exact` context if array elements exist which were not
included in the specification.
When sequence context is not involved, every element of the specification
matches against one element of the data (or a nonexistent element at the end).
However, if sequence context is involved with any element of the array, the
array match is handled according to the rules described in
[sequence matching](#sequence-matching).

#### Sequence Matching

Sequence matching is based on a regular language automata theme.  Expressions
evaluated in sequence context (initiated implicitly by
[regexes](#regex-matching) or by applying a [`SEQ`](#ex-seq) /
[`SEQMIN`](#ex-seqmin) / [`SEQMAX` expression](#ex-seqmax) to array elements)
have behavior defined in terms of "initial state" and "generated successor
states", with the initial state being a position within [0,length] where a
match needs to occur, and successor states being all the ways that the match
can resolve successfully from that starting position.
Implementations are likely to need to iterate the successor states as they look
for an overall match for the array specification, but no specific order is
required for this iteration and implementations are free to apply optimizations
that eliminate impossible states or collapse redundant successor states.
This also gives implementations the option to optimize things with a DFA
compilation when the specification is composed purely of regular language
constructs.

Sequence context can be configured to resolve in one of three ways:

  * boolean - Nothing needs to know which elements matched, and the
              algorithm may terminate as soon as it finds any match.
  * longest - The algorithm finds the greatest number of elements which can be
              matched by the specification.
  * shortest - The algorithm finds the fewest number of elements which can be
              matched by the specification.

Sequence context is only resolved in terms of the whole array, not as decisions
within individual expressions or named sequences.  Various regex contexts will
set either `boolean` or `longest` resolution.  When performing structural
matches, the default is `boolean` unless any named sequences are found among
the array element specifications.  Named sequences specify a `longest` or
`shortest` preference for the subexpression itself.  Of all the final states of
all sequence context expressions that can match the complete specification for
the array, the algorithm chooses one according to these priorities:

  * Longest overall or shortest overall, if initial mode was not `boolean`.
  * For each named sequence N in order specified:
    * Longest or shortest match according to its preference.
  * For each named sequence N in order specified:
    * Leftmost start of a match.

Sequence context affects the behavior of several other match expressions:

  * [`NONE`](#ex-none) becomes a zero-width assertion.
  * [`ANY`](#ex-any) matches however many elements its subexpression does.
  * [`ALL`](#ex-all) matches at least as many elements as required for all of
    its subexpressions to match.  (see the expression description for details)
  * [`REP`](#ex-rep) expressions are permitted.

A `SEQMIN` / `SEQMAX` expression may only be used at the root of a match
specification being applied to an array's elements, and a `SEQ` expresison may
only initiate sequence context there (though `SEQ` may be used in any existing
sequence context as a general way to specify a sequential matches).
The restrictions on entering sequence context ensure that `NONE`, `ANY`, and
`ALL` don't gain surprising semantics without directly requesting them, and the
specific restriction on `SEQMIN` and `SEQMAX` ensures that named sequences are
always referring to unambiguous ranges of array elements without needing to
define an iteration order for testing alternatives or greedy semantics for
every `REP` expression.  These restrictions are applied *after* replacing any
common subexpression references.

After a successful sequence-context match of an array, named sequences get
recorded in an internal data structure of the match state.
It is an error for two named sequences within the same match specification to
share the same name.  Since all sequences will participate in the match, all
named sequences will be defined by the end of the match.
If the match was occurring as the test of a `MATCH` or `IF` action, a
successful overall match makes each name available within the
[capture namespace](#ex-cap) locally within its contained actions.
If the match was occurring within a `CAPTURE` function, the named captures
become the properties of the returned object.

#### Fuzzy Matching

Implementations may offer a feature for looking for matches at a slightly
different path than the one specified, such as by trying nearby array elements
if the path named a specific array element.

The exact algorithm for performing the search for a "fuzzy" match is dependent
on the engine and parameters supplied by the user.  This specification leaves
fuzzy matching open to experimentation by implementations.  However, all
implementations should follow these guidelines:

  * If a path referring to the source tree was altered to allow a match to
    succeed, all access to the node(s) where the alteration(s) were made must
    receive the same alteration, but only for actions (and their expressions)
    under the scope of that match.
  * The alteration to a path must be linked to the node (object or array)
    such that if the node gets moved to a different path, the alteration
    metadata now affects only the new path.
    This includes moves to/from other namespaces like the stash.
  * Alterations to a path must only apply *before* resolving aliases.
    If a match expression using a negative array index is altered, it will
    not have any effect on the same array element when specified as a positive
    integer.
  * Fuzzy matching may not be used in regular expression context.
  * Implementations are strongly discouraged from attempting fuzzy matching on
    an array where the specification could match a variable number of elements.
    If the first pass had to test multiple possible match patterns on an array,
    falling back to fuzzy matching is probably unwanted.
  * Implementations are encouraged to restrict fuzzy matching to arrays/objects
    where every property affected by an action was previously tested by its
    containing `MATCH` action.  i.e. record every path that was tested during a
    successful fuzzy match and then if the actions try to access any path
    outside of those paths, abort the attempt and return `NO_MATCH` as if the
    fuzzy match wasn't attempted.
  * Implementations should not enable fuzzy matching by default.

Future versions of this specification may specify fuzzy matching in more
detail as common useful strategies and antipatterns emerge.

### Regex Matching

Regex matching can only begin when the current node is a string.
The subject string is broken into a virtual array of characters, and then the
match expressions are applied to that virtual array.
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

When matching against a virtual character array,

  * Numbers match against the numeric value of codepoints.
  * Strings of one character match against the character itself.
  * Strings of multiple characters are interpreted as `SEQ` expressions.
  * No other types can match anything within the virtual array of characters.
  * If the current node is changed to something other than a virtual array
    of characters (via 2-argument `HAS` or 2-argument `IS`), the algorithm
    switches back to Non-Regex Matching for the duration of that scope.

In some cases, only a boolean result is required, and the match ends as soon
as the expression is satisfied.  Other cases require finding the longest
match, and run through every possibility of the match expression to see which
is the longest, though may end early if the whole string is consumed.  If an
expression is constructed from only "textbook regular" match expressions, an
implementation can compile the expression into a DFA, or possibly a
host-language regular expression for improved performance.
Elaborate non-regular match expressions may be used in regex context, but can
cause a great deal of iteration and quickly run into resource limits.

### Expression Summary

The following expressions are available:

Name                     | Type  | ID | Description
-------------------------|-------|----|----------------------------------------
[EXPR](#ex-expr)         | Ref   | -5 | An element of the common subexpressions
[CAP](#ex-cap)           | Ref   | -4 | A path in the capture namespace
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
[NUM](#ex-num)           | Match |  9 | Match numbers, optional ranges
[BOOL](#ex-bool)         | Match | 10 | Match booleans
[DATE](#ex-date)         | Match | 11 | Match date-like things
[WITH](#ex-with)         | Match | 12 | Scoped overrides to TEMP namespace
[SEQ](#ex-seq)           | Match | 13 | Match a sequence of adjacent elements
[SEQMAX](#ex-seqmax)     | Match | 14 | Named sequence, prefer longest match
[SEQMIN](#ex-seqmin)     | Match | 15 | Named sequence, prefer shortest match
[REP](#ex-rep)           | Match | 16 | Match repeatedly on adjacent elements
[START](#ex-start)       | Match | 17 | Match at first element of array
[END](#ex-end)           | Match | 18 | Match at length of array
[CHARS](#ex-chars)       | Match | 19 | Match to a set of characters
[CASE](#ex-case)         | Fn    | 50 | If/else that returns a value
[NOT](#ex-not)           | Fn    | 51 | Logical NOT of value exprs
[AND](#ex-and)           | Fn    | 52 | Logical AND of value exprs
[OR](#ex-or)             | Fn    | 53 | Logical OR of value exprs
[CASTBOOL](#ex-castbool) | Fn    | 54 | Cast value to boolean
[CASTNUM](#ex-castnum)   | Fn    | 55 | Cast value to number
[CASTSTR](#ex-caststr)   | Fn    | 56 | Cast value to string
[CASTDATE](#ex-castdate) | Fn    | 57 | Cast value to host date
[CAPTURE](#ex-capture)   | Fn    | 58 | Match, then return captured sequences
[SPLIT](#ex-split)       | Fn    | 59 | Split a string into an array
[JOIN](#ex-join)         | Fn    | 60 | Join array into a string
[LESS](#ex-less)         | Fn    | 61 | Less-than test
[LESSEQ](#ex-lesseq)     | Fn    | 62 | Less-or-equal test
[MIN](#ex-min)           | Fn    | 63 | Minimum of list
[MAX](#ex-max)           | Fn    | 64 | Maximum of list
[COALESCE](#ex-coalesce) | Fn    | 65 | First non-null value of list
[ADD](#ex-add)           | Fn    | 66 | Add numbers
[SUB](#ex-sub)           | Fn    | 67 | Subtract numbers
[MUL](#ex-mul)           | Fn    | 68 | Multiply numbers
[DIV](#ex-div)           | Fn    | 69 | Divide numbers
[MOD](#ex-mod)           | Fn    | 70 | Calculate Modulus

### Expression Interaction

To quickly understand how the types interact, consider these examples:

    [IS, { "a": [ANY, false, true, { c: 2 }] }]
    // implies [IS, { "a": [ANY, false, true, [IS, { c: 2 }]] }]
    // node.a === false || node.a === true || node.a.c === 2

In this case, the outer match expression passes `node.a` as the current node,
and a `IS` context.  The `ANY` expression tests whether `node.a` matches
against any of of the values.  When it tests the third value, it performs an
`IS` operation rather than the default of `HAS`.  The boolean result of `ANY`
is consumed as the result for whether property 'a' matched.

    [IS, { "a": [NONE, { "b": 1 }] }]
    // implies [IS, { "a": [NONE, [IS, {"b": 1}]] }]
    // !deep_equals(node.a, {b:1})

In this case, the outer `IS` match expression passes `node.a` as the current
node to the `NONE` expresison with `IS` context.  The `NONE` expression tests
whether `node.a` is not exactly identical to `{b:1}`.

    [IS, [NODE,"a"], [NODE,"b"] }]
    // deep_equals(node.a, node.b)

The `IS` and `HAS` expressions have a 2-argument form to specify both the
pattern and the target to match.  In this case, `node.a` becomes the current
node, and `node.b` becomes the fully-evaluated structure to match against.
Return values from functions cannot contain expressions; arrays are considered
literally.

    [IS, { "a": [OR, false, { b: 1 }] }]
    // node.a === true
    // probably not what was intended

In this example where someone probably meant to use `ANY` instead of `OR`,
a match expression calls a function (non-match expression) and uses the return
value as *a value to match against*.  Objects are considered true, so the
result of `OR` is true and `node.a` is compared to true.

    // This doesn't work:
    [IS, { "a": [LESS, [CASTNUM], 8] }]
    // This does:
    [IS, { "a": [IS, [LESS, [CASTNUM], 8], true] }]
    // (parseNumber(node.a) < 8) === true

If you want the return value of a function to be used as the match *result*,
simply use the two-argument form of `IS` or `HAS` to compare it to `true`.
In this case, `LESS` is not a match expression so it can't be used directly
within the structure of the outer `IS` in the intended manner.
The inner `IS` compares two values instead of comparing one value to the
current node, and the return value gets consumed as the match result because
`IS` is a match expression.

## Errors

Applying a JGraft to a target data structure may fail with:

  * `INVALID_GRAFT`   - The structure of the JGraft itself violates the spec.
  * `INVALID_TARGET`  - The graft describes an operation that can't apply due
                        to the type or structure of the target.
  * `INVALID_OPERAND` - The runtime value given to an action or expression
                        is invalid.  This may be the fault of either the
                        target data or the described graft.
  * `NO_MATCH`        - A `MATCH` action failed to find a matching context.
  * `RESOURCE_LIMIT`  - Applying the graft would exceed configured limits,
                        such as too much memory, too much recursion, too many
                        generated properties, or too deep of a tree.

Note that `INVALID_TARGET` and `INVALID_OPERAND` errors which occur inside of
a match expression get trapped and simply cause it to fail to match, which may
in turn generate a `NO_MATCH` outcome for the overall graft operation.
The `INVALID_GRAFT` and `RESOURCE_LIMIT` errors always immediately terminate
the graft operation regardless of match context.  Likewise, unspecified errors
from application callbacks or other implementation aspects should terminate
the graft operation.

When applying a graft fails, the recommended behavior for an implementation is
to emit the error along with some diagnostic properties describing the current
node and action/expression which failed, and make the partially-edited data
structure available for inspection.  Implementations may also provide a
callback allowing the application to intercept a failure and resolve it in a
way that allows the remainder of the JGraft to continue.

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

### <span id="act-jgraft">JGRAFT</span>

    ['JGRAFT', metadata, action1, action2...]

This action is used for the top-level of an exported JGraft, or optional
metadata deeper within the structure.
When serialized to JSON, the top-level 'JGRAFT' action name is not replaced by
a number, and there should be no whitespace until after the following comma.
This gives JGraft JSON files a "magic number" of

    ["JGRAFT",

The `metadata` is an object with semi-free-form properties, where top-level
names matching `/^[a-z][A-Za-z0-9]*$/` are reserved for the protocol, and any
others are user-defined.
The following properties are currently defined:

  - v: integer; the minimum JGraft specification version required to
       correctly interpret the contents of this structure
  - writer: a string identifying the tool that authored the structure
  - comment: a free-form text comment string
  - const: an array declaring constants
  - expr: an array declaring value context macros

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

The `JGRAFT` action acts like a scope for declaring constants and common
expressions, and also provides a fresh empty `stash` of shared temporary
variables for the actions within it.
The algorithm when beginning a `JGRAFT` action is roughly:

  - Verify the version `v` is supported.
  - If the metadata contains a `const` property,
    - Set the `const` namespace to a shallow clone of any inherited `const`
      namespace, or empy object at the root.
    - For each (`name`,`val_expr`) pair of elements in the `const` array,
      - `name` must be a single property string, or integer (which gets
        coerced to a string)
      - `val_expr` may be a [value or expression](#value-context).
      - Set property `const[name]` to the value, after evaluating any
        expressions within.  `CONST` expressions evaluate according to the
        `const` namespace in its current state before this assignment. `STASH`
        expressions still refer to the parent scope's stash, if one exists.
  - If the metadata contains an `expr` property,
    - Set the `expr` namespace to a shallow clone of the parent `expr`
      namespace, or empty object at the root.
    - For each (`name`,`val_expr`) pair of elements in the `expr` array,
      - `name` must be a single property string, or integer (which gets
        coerced to a string)
      - `val_expr` may be a [value or expression](#value-context).
      - Set property `expr[name]` to the value-construction structure *without*
        evaluating any of it.  In other words, it captures no context and paths
        like `NODE` or `TEMP` will be interpreted in the context of the `EXPR`
        expresison that invokes it.
  - Set the `stash` namespace to a new empty object.

### <span id="act-at">AT</span>

    ['AT', path, subaction1, subaction2...]

Change the current node to a sub-node.

Navigate to a [path](#paths) within the current node, then execute all
sub-actions with that node as the current node.  The path must exist or the
graft operation fails with `INVALID_TARGET`.  The current node for an action
may never reference data outside the input tree, so `path` is limited to `NODE`
expressions.  The change to the current node is only visible within the scope
of this action.

### <span id="act-match">MATCH</span>

    ['MATCH', match_spec, subaction1, subaction2...]

Execute actions on a node having the intended shape.

Assert that the current node matches a [match specification](#match-context),
then execute all sub-actions in sequence.  If it doesn't match (or throws an
`INVALID_OPERAND` or `INVALID_TARGET`) the graft operation will fail with
`NO_MATCH`, unless the user has enabled [fuzzy matching](#fuzzy-matching) in
which case the engine looks for nearby potential matches.
If a fuzzy match is successful, the alterations to the paths described in
`match_spec` are likewise applied during all sub-actions.
This is analogous to how unix `patch` applies "hunk at offset" by first
searching for the matching lines then continuing as if they'd been specified
for that location.

### <span id="act-if">IF</span>

    ['IF', match_spec1, action, else_action]
    ['IF', match_spec1, action1, match_spec2, action2... else_action]

Conditionally apply actions.

For each pair of (`match_spec`,`action`) processed in sequence, test whether
the current node matches `match_spec`.  If so, apply the action and stop,
else continue to the next pair.
If the list ends with a single element, it is treated as an "else" action.
Lacking an "else" action, the `IF` action completes successfully even if
no sub-action was performed.  If the sub-action fails with an error, the
`IF` action likewise fails.

`IF` uses the same [fuzzy matching mechanism](#fuzzy-matching) as `MATCH`.
However, implementations should always attempt every condition of the `IF`
statement *without* additional fuzzy match alterations before trying them all
again with fuzzy searching enabled.  Alterations inherited from a fuzzy match
at a parent action will still be applied to both attempts.

### <span id="act-copy">COPY</span>

    ['COPY', val_expr, path, val_expr1, path2, ...]

Construct/clone values and assign them to paths.

For each pair of (`val_expr`, `path`), resolve any expressions in `val_expr`
(specified in [value context](#value-context)) then assign the resulting scalar
or tree to the `path` (specified in [path context](#path-context)).
Each pair sees the effects of the previous assignment.

Reference expressions in the `val_expr` must exist, or the graft fails with
`INVALID_TARGET`.  However, objects along the property path being assigned will
be auto-vivified if they don't exist, but the action fails with `INVALID_TARGET`
if it would need to convert a scalar, object, or array to a different type.
Recall that arrays are implied when an object path contains a number, and
objects are implied when the path contains a string.
To prevent inappropriate creation of sub-objects, use a containing `MATCH`
directive that asserts the presence or absence of the structure as desired.
Unlike in JavaScript, assigning to (or a path through) an array element greater
than `length` does not auto-vivify and fails with `INVALID_TARGET`.

Examples:

    ['COPY', {x:1,y:1}, 0]               // node[0]= { x: 1, y : 1 };
    ['COPY', 5, [NODE, 0,'x']]           // node[0].x= 5;
    ['COPY', 6, [NODE, -1,'x']]          // node[node.length-1].x= 6;
    ['COPY', {x:6}, false]               // node[node.length]= { x: 6 };
    ['COPY', [NODE,'a',4], [NODE,'b',0]] // node.b[0]= clone(node.a[4]);
    ['COPY', null, 'a']                  // node['a']= null;
    ['COPY', [NODE], 'a']                // node['a']= clone(node);
    ['COPY', [[1,2,3]], 'a']             // node.a= [1,2,3];
    ['COPY', [NODE], [TEMP,0]]           // temp['0']= clone(node);
    ['COPY', [TEMP,0,'a'], 'b']          // node.b= clone(temp['0'].a);
    ['COPY', [TEMP,0,'b'], 'a']          // node.a= clone(temp['0'].b);
    ['COPY', [NODE,'a'], [TEMP,0]]       // temp['0']= clone(node.a);
    ['COPY', [NODE,'b'], 'a']            // node.a= clone(node.b);
    ['COPY', [TEMP,0], 'b']              // node.b= clone(temp['0']);
    ['COPY', [NODE,'b'], [STASH,0]]      // jgraft.stash['0']= clone(node.b);
    ['COPY', [CONST,'big_data'], 'b']    // node.b= clone(jgraft.const.big_data);

### <span id="act-move">MOVE</span>

    ['MOVE', src_path1, dst_path1]
    ['MOVE', src_path1, dst_path1, src_path2, dst_path2...]

Relocate subtrees from a source property to a destination property without
copying them.

For each pair of (`src_path`,`dst_path`) processed in sequence, find the value
of the source property, delete the leaf source property from its container,
and then set the destination to refer to that value.
The `src_path` must refer to a property that exists, and the `dst_path` must
be validated to be conflict-free (with respect to auto-vivification) before
any alteration to the target data is performed.  It is permitted to delete a
source property and then immediately auto-vivify through that same property
to create the destination.  If `src_path` and `dst_path` are the same, the
pair are essentially a no-op.
`src_path` and `dst_path` are both [path context](#path-context) with the
exception that a literal `null` may be used as `src_path` to write a literal
`null` value to `dst_path` (which lets you preserve the existence of a
property that would otherwise get deleted without needing a followup `COPY`
action to fix it) or as `dst_path` as a way of requesting deletion of the
property without preserving the value anywhere.

When the source property is an element of an array, the deletion of the
element implies that following elements will be shifted up to fill the gap.
However, this shifting does not occur until after all pairs have been processed.
Until then, the array element is made inaccessible to source paths, though a
new value may be written into it which cancels the deletion for that element.
The queued deletions (along with the metadata of being inaccessible to source
paths) bind to the instance of the array, following it even if it moves to a
different namespace.  Implementations need to also consider that the array
can get implicitly deleted from the tree if an ancestor node is overwritten or
deleted.

A second special case for arrays is that the final element of a path ending
within an array may refer to a "half index" of the array, like 1.5.
Values written to half-index elements get queued for insertion, to be applied
during the same process that performs deletions at the end of the `MOVE`
action.
Elements written to a half-index are inaccessible from any namespace; you
cannot subsequently use them as source paths, nor write through them to deeper
sub-paths, nor auto-vivify through them.  If you need to modify a value that
will be written into a half-index, first move it to the temporary namespace,
apply subsequent `MOVE` pairs to it, then move from the temporary to the
half-index path.

Negative half-index values are permitted in mostly the same manner that
negative array indices alias positive indices, however -0.5 refers to the
insertion point before element 0 rather than after the final element of the
array.

Enumerated for an array currently of length 2, destination array indices are
handled like this:

Index  | Behavior
-------|--------------------
-0.5   | Queue insertion before element 0
 0     | Overwrite element 0
 0.5   | Queue insertion between 0 and 1
 1     | Overwrite element 1
 1.5   | Queue insertion after element 1 and before a potential element 2
 2     | Append immediately as element 2
 2.5   | Fail with `INVALID_TARGET`
 false | Append immediately as element 2
 -1    | Overwrite element 1
 -1.5  | Queue insertion at `0.5`
 -2    | Overwrite element 0
 -2.5  | Queue insertion at `-0.5`
 -3    | Fail with `INVALID_TARGET`
 -3.5  | Fail with `INVALID_TARGET`

At the end of the `MOVE` action, every array instance where an insertion or
deletion was queued gets conceptually rebuilt from the list of remaining
elements (including half-index elements) in the numeric order of their index.
If multiple values were queued to the same half-index they get enumerated in
the same order the assignments happened.

The practical consequences of these restrictions and special array handling
are:

  - No expensive deep-copying of objects is required, nor processing to
    determine whether deep-copying would be required.
  - It is not possible to degrade the data from a tree to a graph, because
    for every reference added, the old reference is immediately destroyed
    (or made inaccessible).
  - Arrays do not get resized or shifted on every single move pair
  - If an editor has re-ordered the elements of an array, and knows the source
    and destination index of each, it can easily export that edit as a `MOVE`
    argument list from those pairs by adding ".5" to each destination.  Every
    deletion will be paired with an insertion and the `MOVE` action can run in
    `O(N)` time without actual splice() calls.

Examples:

    ['MOVE', 0,[TEMP,0],           // tmp['0']= node[0];
                                   // restrict(node, 0);
            -1,0,                  // node[0]= node[node.length-1];
                                   // unrestrict(node, 0);
                                   // restrict(node, node.length-1);
            [TEMP,0],-1]           // node[node.length-1]= tmp['0'];
                                   // unrestrict(node, node.length-1);
                                   // delete tmp['0'];
    
    ['MOVE',1,-0.5,2,false]        // pos0_queue= [ node[1] ];
                                   // restrict(node, 1);
                                   // node.push(node[2]);
                                   // restrict(node, 2);
                                   // node.splice(1,2);
                                   // node.splice(0, 0, ...pos0_queue);
    
    ['MOVE','a','b','c','a']       // node.b= node.a;
                                   // delete node.a;
                                   // node.a= node.c;
                                   // delete node.c;
    
    ['MOVE',[NODE,'x','x'],[TEMP,0], // tmp['0']= node.x.x;
                                     // delete node.x.x;
            'x',[TEMP,0,'x'],        // tmp['0'].x= node.x;
                                     // delete node.x;
            [TEMP,0],'x']]           // node.x= tmp['0'];
                                     // delete tmp['0'];

### <span id="act-splice">SPLICE</span>

    ['SPLICE', array_elem_path, count, replacement1, replacement2...]

Replace a span of an array, just like the splice function found in most
programming languages.  The `array_elem_path` must resolve to an element of
an array (or to one-beyond-the-end insertion point of an array) or the graft
fails with `INVALID_TARGET`.  `count` is the number of elements to repalce.
Negative numbers are aliases for the count that would end just before element
`length-N`, and `null` is an alias for `length-offset`.  The resulting count
must be within `[0..length-offset]`.

The replacements are evaluated in [value context](#value-context).  If they
contain reference expressions the nodes get deep-cloned as per the `COPY`
action, but implementations may optimize this if they can determine that the
reference will be the only remaining reference to a subtree of one of the
array elements being deleted.  All replacement values are resolved before
making any alteration to the target array.

### <span id="act-split">SPLIT</span>

    ['SPLIT', split_spec, subaction1, subaction2...]

Execute actions within a temporary array of substrings.

This can only be applied at a string node, or it fails with `INVALID_TARGET`.
It splits the string according to `split_spec` to create an array, then runs
each sub-action on that array, then re-assembles the string from the elements
of the array.  The separators and any trimmed characters are preserved to use
when re-assembling the string; the generated array carries them as hidden
metadata.  Actions that modify the array also modify the metadata in parallel
as described below.

The `split_spec` may be a simple string used as a verbatim separator, a match
expression to be evaluated in [regex longest-match context](#regex-context),
or an object with a more elaborate specification:

    {"sep": ...,        // A string or regex to split on
     "element": ...,    // A regex that must match prior to testing for 'sep'
     "default": ...,    // The default separator used for joining new elements
     "keep": ...,       // Preserve the separator within the generated array
     "ltrim": ...,      // A string or regex to trim from start of elements
     "rtrim": ...,      // A string or regex to trim from end of elements
     "trim": ...,       // sets 'ltrim' and 'rtrim' to the same value
     "discard": bool,   // Don't preserve trimmed chars when re-assembling
    }

The split operation iterates through the characters of the string looking for
[the longest match](#regex-context) of the separator pattern.  If you specify
an `element` regex, a test for separator will only match when the element
characters (from the end of the previous separator until the current match
position) also match the `element` pattern.   The `element` pattern is
implicitly anchored to the start and end of the element characters.
This gives you a way to pass over escape sequences without accidentally
detecting a separator, and without requiring the complexity of "look-behind"
or "look-ahead" or capture groups in the separator pattern.

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

(Note that splitting a string into an array of individual characters is a
quick way to reach a `RESOURCE_LIMIT` error)

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
characters are stored there.  As actions like `COPY`, `MOVE`, and `SPLICE`
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
string.  Each element must be a string, or it fails with `INVALID_OPERAND`.
Each element gets any hidden trimmed characters re-added, and any hidden
separator is appended only if there is a following element.  In order to get a
"newline at the end of the file" the final element of the array needs to be an
empty string.  It is the responsibility of the nested actions to preserve or
add this element as desired.

This is the primary tool used to re-implement text diff/patch behavior:

    [SPLIT,
      { sep: [ANY, "\n", "\r\n"], default: "\n", trim: [CHARS," "] },
      [MATCH,
        [ARRAY, 10,
          "Line ten",
          "Line eleven",
          "Line twelve",
          "Line thirteen"
        ],
        [SPLICE, 11, 2,
          "New Line eleven"
          // no line 12
        ]
      ],
    ]

### <span id="act-log">LOG</span>

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

`message` is evaluated in [value context](#value-context).  It must resolve
to a string and should be generally printable, and may not contain unicode
Surrogates, Noncharacters, or control characters (codepoints 0x00–0x1F and
0x80–0x9F).

`data` is also evaluated in [value context](#value-context) but may resolve
to any type of value.  It provides data to the user relevant to the message.
This may be shown to the user in some form such as JSON, or inspected
programmatically by code using a JGraft library.
If the data is not being consumed immediately, implementations should ensure
that it remains stable, such as deep-cloning it or setting up copy-on-write.
This is already handled if the standard operation of the engine is to
shallow-clone the ancestors of each change.

## Expressions

### Reference Expressions

#### <span id="ex-expr">EXPR</span>

    [EXPR, name]

Reference a pre-defined expression.

The common expression namespace is populated by the [`JGRAFT` action](#act-jgraft).
It is a flat namespace, so referencing one of the expressions uses only a
single name rather than a path.  The other expression will be used as if it
had been specified in place of this `EXPR` expression.  It is possible to
set up infinite recursion using EXPR references, but likely to quickly run
into `RESOURCE_LIMIT` errors.  Implementations are not expected to optimize
tail-calls like a functional language would (but are permitted to).

#### <span id="ex-temp">TEMP</span>

    [TEMP, path...]

Reference data from the temporary namespace.

The temporary namespace is local to the current action and may be further
localized to the enclosing [`WITH` expression](#ex-with).
If the first element of `path` is an integer, it will be coerced to a string.

#### <span id="ex-stash">STASH</span>

    [STASH, path...]

Reference data from the stash namespace.

The stash namespace is reset by every [`JGRAFT` action](#act-jgraft), and can
receive data from `MOVE` and `COPY` actions which is shared with other actions
until the end of the enclosing `JGRAFT` scope.
If the first element of `path` is an integer, it will be coerced to a string.

#### <span id="ex-const">CONST</span>

    [CONST, path...]

Reference data from the const namespace.

The const namespace can only be initialized during a [`JGRAFT` action](#act-jgraft).
If the first element of `path` is an integer, it will be coerced to a string.

#### <span id="ex-node">NODE</span>

    [NODE]           // the current node itself
    [NODE, path...]  // data within the current node

Reference data relative to the current node.

The path may be empty to reference the current node itself.

### Match Expressions

#### <span id="ex-has">HAS</span>

    [HAS, match_spec]
    [HAS, value_expr, match_spec]

Match when the target has at least the properties described in `match_spec`.

The `match_spec` argument is declared using [match context](#match-context)
and the `value_expr` is declared using [value context](#value-context).
In the 1-argument form, the value being compared is the current node.

The `HAS` expresison is the default for match context; a `[HAS,...]`
expression at the root of a match structure is redundant unless used in
the 2-argument form to establish an alternate current node.

Examples:

    null                                node === null
    0                                   node === 0
    'str'                               node === 'str'
    [HAS, null]                         node === null
    [HAS, 42]                           node === 42
    { a: 1 }                            node.a === 1
    { a: [OR, [NOT,[EXISTS]], 1 }       !('a' in node) || node.a === 1
    { a: [[]] }                         isArray(node.a)
    { a: [ARRAY, 3, 6, 7] }             isArray(node.a) && node.a[3] === 6
                                          && node.a[4] === 7
    [HAS, [TEMP,0], { a: 1 }]           temp['0'].a === 1

#### <span id="ex-is">IS</span>

    [IS, match_spec]
    [IS, value_expr, match_spec]

Match according to `HAS` but fail if the object/array contains unspecified
properties.

More precisely,

  * Every property named by the specification must successfully match,
    including against a nonexistent current node.
  * Every property actually present in the target must have a corresponding
    property in the specification.

The `match_spec` argument is [match context](#match-context) and the
`value_expr` is [value context](#value-context).  In the 1-argument
form, the value being compared is the current node.

As described in the [matching section](#matching), the entire tree of match
structure and match expressions within the `IS` expression inherit `IS`
semantics unless overridden with a `HAS` expression, but a match expression
used as a direct child of a function expression resets to `HAS` semantics.

Examples:

    [IS, null]                          node === null
    [IS, 8]                             node === 8
    [IS, 'str']                         node === 'str'
    [IS, {a:1}]                         is_deeply(node, {a:1})
    [IS, [ANY, {a:1}, {b:1}]]           is_deeply(node, {a:1})
                                         || is_deeply(node, {b:1})
    [IS, [TEMP,0], {a:1}]               is_deeply(temp['0'], {a:1})

#### <span id="ex-none">NONE</span>

    [NONE, match_expr...]

Match when the target does not match any of the [match-context](#match-context)
expressions.

This is like the function [`NOT`](#ex-not), but with
[match expression](#expression-overview) semantics rather than function
expression semantics.

#### <span id="ex-any">ANY</span>

    [ANY, match_expr...]

Match when the target matches any of the [match-context](#match-context)
expressions.

This is like the function [`OR`](#ex-or), but with
[match expression](#expression-overview) semantics rather than function
expression semantics.  When evaluated at an array cursor position, it can
produce multiple successor cursor positions.

#### <span id="ex-all">ALL</span>

    [ALL, match_expr...]

Match when the target matches all of the [match-context](#match-context)
expressions.

This is like the function [`AND`](#ex-and), but with
[match expression](#expression-overview) semantics rather than function
expression semantics.

In sequence context, each argument is treated as a constraint that must be
satisfiable from the starting position; of the successor positions generated
by each subexpressions, any that is less than the maximum of each
subexpression's minimum position gets discarded, and all the rest are returned
as potential successor positions.  In other words, the combined match extends
at least as far as needed for the minimum requirement of each subexpression,
up to the maximum matchable by any subexpression.

#### <span id="ex-exists">EXISTS</span>

    [EXISTS]

Match when the current node exists, including the value `null`.

This is also particularly useful when negated as `[NONE,[EXISTS]]`.  As this
implies, the matching system must consider property expressions even when the
property doesn't exist on the target.

#### <span id="ex-array">ARRAY</span>

    [ARRAY]
    [ARRAY, length]
    [ARRAY, offset, match_spec1...]

With no arguments, this matches when the current node is an array.

With one argument, it asserts the current node is an array with at least this
number of elements in the context of `HAS`, or exactly this number of elements
in the context of an `IS` expression.

With more than one argument, this asserts a sequence of matches (as per the
[`SEQ` expression](#ex-seq)) starting at `offset`.  It is equivalent to the
expressions:

    // In 'HAS' match context:
    [[ [REP,offset,offset,[EXISTS]], [SEQ, match_spec1...] ]]
    // In 'IS' match context:
    [[ [REP,offset,offset,[EXISTS]], [SEQ, match_spec1...], [END] ]]

The `offset` may be negative to count backward from the end of the array.

    // Assert that the array ends with 'x', 'y', 'z'
    [ARRAY, -3, 'x', 'y', 'z']

#### <span id="ex-str">STR</span>

    [STR]
    [STR, regex_expr]

With no arguments, this matches when the current node is a string.

With one argument it asserts that the node is a string and matches a regular
expression, specified in [regex context](#regex-context).  The regex is not
implicitly anchored.  This expression is the primary way of invoking regular
expression matching when the specific bounds of the match do not need to be
known. It does not invoke "longest" semantics since it only need a boolean
result.

#### <span id="ex-num">NUM</span>

    [NUM]
    [NUM, min]
    [NUM, min, max]

With no arguments, this matches when the current node is any JSON number,
or additionally NaN and Inf if those extensions are relevant.

With one or two arguments, also verify that `min <= node <= max`.
`min` or `max` may be `null` to omit the respective test.
In implementations where infinity or NaN are available, neither may be
used as the range boundary.

#### <span id="ex-bool">BOOL</span>

    [BOOL]

Match when the current node is a boolean.

#### <span id="ex-date">DATE</span>

    [DATE]

Match when the current node is a valid input for the
[`CASTDATE` expression](#ex-castdate).  See the details of that expression
and also the [dates section](#dates) for details.

#### <span id="ex-with">WITH</span>

    [WITH, temp_prop, value_expr, match_expr]
    [WITH, temp_prop0, val0, temp_prop1, val1, ..., match_expr]

Set up a lexical overlay on the TEMP namespace.

This sets one or more properties at the root of the TEMP namespace to new
values that are only visible for the duration of the `WITH` expression.
`temp_prop` may be any string or integer, but not an expression.
Each `value_expr` is declared in [value context](#value-context) and evaluated
before assignment.
The `match_expr` is declared in [match context](#match-context) and the result
is passed through as the result of the `WITH` expression.

The purpose of this expression is to capture values within the tree of the
current node before descending into the tree with match expressions where they
would no longer be accessible.  Since expressions have no side effects and the
changes to TEMP are local to this expression, implementations are free to store
references to other trees rather than deep-clonding the data.

#### <span id="ex-seq">SEQ</span>

    [SEQ]
    [SEQ, match_expr...]

Match a series of array elements against a series of match expressions.

This can only match when the current node is derived from a match cursor of an
array.  It produces all the successor cursor positions that can satisfy the
sequence of `match_expr`, returning false if no position can satisfy them.
It is permitted to have zero arguments, in which case it returns true and
produces the successor cursor position of it's initial cursor position.

#### <span id="ex-rep">REP</span>

    [REP, min, max, match_expr]

Match repetitions of an expression against a series of array elements.

`min` and `max` are [value context](#value-context) expressions which must
evaluate to integers.  The value of `min` must be non-negative.  The value of
`max` may be -1 as an alias for infinity, or 0 as an alias for "same as min".
Otherwise, `max` must have a value greater than `min`.
`match_expr` is a [match context](#match-context) expression.

This can only match when the current node is derived from a match cursor of an
array.  It produces all the successor cursor positions that can satisfy a count
of matches between `min` and `max`.

Regex Notation | REP notation
---------------|--------------------------
 `(...)*`      | [REP,0,-1,...]
 `(...)?`      | [REP,0,1,...]
 `(...)+`      | [REP,1,-1,...]
 `(...){5}`    | [REP,5,0,...]
 `(...){3,5}`  | [REP,3,5,...]

#### <span id="ex-start">START</span>

    [START]

Match when the current node is derived from a match cursor of an array and the
cursor is at its minimum value.  This is a zero-width assertion. (returns the
input cursor position as the successor position)

#### <span id="ex-end">END</span>

    [END]

Match when the current node is derived from a match cursor of an array and the
cursor is at its maximum value. (i.e one-beyond-the-end of the array)
This is a zero-width assertion.

#### <span id="ex-chars">CHARS</span>

    [CHARS, ...
      true,                // include following elements
      false,               // exclude following elements
      "abcdef",            // 6 literal characters
      "a","z",             // a range of characters to be added
      "a",null,            // open-ended range
      48,57,               // unicode codepoint range
      [EXPR,name]          // reference a common CHARS expression

Match when the current node is a character in the defined set.

Boolean values switch between inclusion and exclusion.  The default is
inclusion to an empty set.  If the first element is false, it instead becomes
exclusion from the complete set of Unicode.  Further boolean values affect
the elements that follow but do not themselves modify the set.

A string of more than one character is considered a list of characters to
include.  A single character begins an inclusive range of characters if the
following element is also one character.  `null` may be used as an alias for
the maximum Unicode character.  An integer is treated as a unicode codepoint
value (and must fall within the valid unicode range).  Since integers and
`null` represent single characters, they also begin or end ranges when
possible.

Finally, you may use an [`EXPR expression`](#ex-expr) to reference a `CHARS`
expression from the common expression namespace.  It must resolve to exactly
a `CHARS` expression; anything else results in `INVALID_GRAFT`.

Examples:

    // Regex /./
    [CHARS, false, "\n"]

    // Regex /./s
    [CHARS, 0,null]
    [CHARS, false]
    
    // Regex /[0-9]/
    [CHARS, "0123456789"]
    [CHARS, "0","9"]
    [CHARS, 48,57]
    [CHARS, "0",57]
    
    // Regex /[0-9a-zA-Z_.-]/
    [CHARS, "0","9", "A","Z", "a","z", "_-."]
    [CHARS, 0x30,0x39, 0x41,0x5A, 0x61,0x7A, "_-."]
    
    // "All printable but no punctuation"
    // Regex /[^\P{Print}\p{Punct}]/
    // (assuming you define these classes in the JGRAFT action)
    [CHARS, [EXPR,"regex_Print"], false, [EXPR,"regex_Punct"]]

### Function Expressions

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

