# JGraft

JGraft is a data structure that describes edits to a tree of data.
The data structure is built around basic JSON-compatible concepts, and in
some cases JavaScript semantics.  The main goal is to be able to exchange it
in the form of JSON, especially between application back-ends and JavaScript
front-ends, but other languages may include data types of their own if they
don't need to serialize to JSON.

The structure primarily uses array primitives to encode directives in a manner
similar to the Lisp programming language, though it also uses JSON objects
with named properties for any case where that is more convenient.
The directives (classified below as Actions or Match Functions) have
symbolic names, but also can be represented by small integers for better
performance in production environments.
The symbolic names are used in all examples below.

## Paths

Several actions and match functions use the concept of a 'path' (of JSON
property names).  A path is specified as a string, number or array:

    'a'                node.a
    0                  node[0]
    []                 node
    ['a']              node.a
    [0]                node[0]
    ['a',1,'b',2]      node.a[1].b[2]
    [-1]               node[node.length-1]
    [false]            node[node.length]
    [null, ...]        special per-action
    [true, ...]        special per-action

As shown above, negative numbers count backward from the end of an array,
false refers to the nonexistent one-beyond-the-end element of an array which
may sometimes be assigned to, and paths beginning with `null` or `true` are
used as an escape sequence for special purposes depending on the current
action.

## Actions

A JGraft data structure is a tree of actions, Lisp-style, where each action is
an array that begins with the action name (or numeric opcode) and contains
parameters for that action, which will often include sub-actions.

### AT

    ['AT', path, subaction1, subaction2...]

Navigate to a sub-node of the current node, then execute all sub-actions with
that node as the current node.  The path must exist or the graft operation
fails with `INVALID_TARGET`.

### MATCH

    ['MATCH', match_spec, subaction1, subaction2...]

Assert that the current node matches a [match specification](#matching),
then execute all sub-actions in sequence.  If it doesn't match, the graft
operation will fail with `NO_MATCH`, unless the user has enabled "fuzzy"
matching in which case the engine looks at nearby array indices to find a
match.

The exact algorithm for permitting "fuzzy" matches is dependent on the engine
and parameters supplied by the user.
If a fuzzy match is successful, the array offsets discovered during that
process are applied to any property path in the sub-actions which reference
those arrays.  This is analogous to how unix `patch` applies "hunk at offset"
by first searching for the matching lines then continuing as if they'd been
specified for that location.

### IF

    ['IF', match_spec1, action, else_action]
    ['IF', match_spec1, action1, match_spec2, action2... else_action]

For each pair of (`match_spec`,`action`), test whether the current node
matches and if so, apply the action, else continue to the next pair.
If the list ends with a single element, it is treated as an "else" action.
If the action aborts with an error, the `IF` action likewise fails.

### ASSIGN

    ['ASSIGN', prop1, val1, prop2, val2, ...]

Examples:

    ['ASSIGN', 0, {x:1,y:1}]            // node[0]= { x: 1, y : 1 };
    ['ASSIGN', [0,'x'], 5]              // node[0].x= 5;
    ['ASSIGN', [-1,'x'], 6]             // node[node.length-1].x= 6;
    ['ASSIGN', [false], {x:6}]          // node[node.length]= { x: 6 };
    ['ASSIGN', ['b',0], ['a',4]]        // node.b[0]= deep_clone(node.a[4]);
    ['ASSIGN', 'a', null]               // node['a']= null;
    ['ASSIGN', 'a', []]                 // node['a']= deep_clone(node);
    ['ASSIGN', 'a', [[1,2,3]]]          // node.a= [1,2,3];

Assign one or more properties at the current node (or at sub-paths of it).
The basic usage of this action is to assign literal values to named
properties, but each property *and* value argument may instead be a
[path](#paths).  This means that `[]` in the value position refers to the node
itself (empty path) rather than a literal empty array to become the new value.
As usual, an array literal value can be specified by wrapping
it with another array, to disambiguate it from a path.  All assignments are
considered to be "by value", deep-cloning objects rather than building a graph
of object references.  Assignments are performed in order, so later value
expressions see the modified state of the node.

Objects along the property path being assigned will be auto-vivified if they
don't exist, but the action fails with `INVALID_TARGET` if it would need to
overwrite a scalar value with an object or array.
To prevent inappropriate creation of sub-objects, use a containing `MATCH`
directive that asserts the presence or absence of the structure as desired.
Assigning to an array more than one element beyond the end is still a fatal
error, returning `INVALID_TARGET`.

Additionally, using a path that begins with `null` reads and writes to a
temporary variable instead of the current node, giving you a place to store
temporaries.  The temporary variable space exists only for the duration of
this `ASSIGN` action.

Examples:

    ['ASSIGN', [null], []]                // temp= deep_clone(node);
    ['ASSIGN', 'b', [null,'a']]           // node.b= temp.a;
    ['ASSIGN', 'a', [null,'b']]           // node.a= temp.b;

### MOVE

    ['MOVE', src_prop1, dst_prop1, src_prop2, dst_prop2...]

Relocate one or more values of properties within a single object or array.
This is an alternative to `ASSIGN` optimized for shuffling the elements of an
array when insertions are not required, though it may also be used to rename
properties of objects.
All source and destination positions are resolved against the state of the
current container at the start of the MOVE action.  The logical algorithm
(which may be implemented in any equivalent manner) is as follows:

  - For arrays:
    - For each pair of `src_prop`, `dst_prop`,
      - Index `src_prop` must exist in the array, and must be distinct from
        any other `src_prop` in this action.
      - Index `dst_prop` must be in the range [0..length] (optionally using
        negative number notation to count backward from the end of the array,
        or the special value `false` to refer to the length of the array) or
        the value `null`.
      - Queue the value at `src_prop` for insertion at index `dst_prop` if
        `dst_prop` was not `null`.
      - Queue `src_prop` for deletion.
    - Iterating backward over each index where a change was queued,
      - perform a logical splice(), performing any deletions and insertions
        queued at that index.
  - For objects:
    - For each pair of `src_prop`, `dst_prop`,
      - `src_prop` must exist in the object, and must be distinct from any
        other `src_prop` in this action.
      - `dst_prop` must be distinct from any other `dst_prop`, or the special
        value `null`.  `dst_prop` is not required to exist in the object.
      - Queue the assignment of the current value of `src_prop` to `dst_prop`,
        unless `dst_prop` was `null`.
      - Queue the deletion of `src_prop`.
    - Perform all queued deletions
    - Perform all queued assignments

Examples:

    [`MOVE',0,-1,-1,0]                  // ins[node.length-1]= node[0];
                                        // ins[0]=             node[node.length-1];
                                        // node.splice(node.length-1, 1, ins[node.length-1]);
                                        // node.splice(0, 1, ins[0]);
    
    ['MOVE',1,false,2,false]            // ins[node.length]= [ node[1], node[2] ];
                                        // node.splice(node.length, 0, ...ins[node.length]);
                                        // node.splice(1, 2);
    
    ['MOVE','a','b','c','a']            // tmp1= node.a;
                                        // tmp2= node.c;
                                        // delete node.a;
                                        // delete node.c;
                                        // node.b= tmp1;
                                        // node.a= tmp2;

### SPLICE

    ['SPLICE', offset, count, replacement1, replacement2...]

Replace a span of an array, just like the splice function found in most
programming languages.  The current node must be an array or the graft fails
with `INVALID_TARGET`.  `offset` may be `false` to refer to the end of the
array, or negative to count backward from the end of the array, but the
resulting index must be within `[0..length]` or the graft fails with
`INVALID_TARGET`.
`count` may be `null` to replace the remainder of the array.

The replacement values may be literal values or [paths](#paths), relative to
the same current node as the `SPLICE` action.  All replacement values are
resolved before starting the splice operation.

### SPLIT

    ['SPLIT', split_spec, subaction1, subaction2...]

This can only be applied at a string node, or it fails with `INVALID_TARGET`.
It splits the string according to `split_spec` to create an array, then runs
each sub-action on that array, then re-assembles the string from the elements
of the array.  The separator and any trimmed characters are preserved to use
when re-assembling the string; each generated array element carries them as
hidden metadata.

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
is considered to match at every position except at the character immediately
following the end of the previous match.  This means the separator of '' can
be used to split the string into its individual characters.  It also means
that a regular expression which can match the empty string will generate a
single-character element any time none of the alternatives matched.  For
example, the pattern `/a*/` applied to the string "baabca" results in:

    ['', 'b', 'b',  'c', '']

(Note that implementations may place limits on the resources consumed by a
JGraft, and splitting a string into an array of individual characters is a
quick way to reach those limits.)

Once a separator is found, it uses the value of `keep` to determine what to do
with it.  The default is to hide the separator from the resulting array but
store it (attached invisibly to the preceeding element) until it is time to
reconstruct the string.
The value "<" means to keep the separator as part of the current element in
the generated array, and likewise ">" means to keep it as part of the
following element.  The value '@' means to include the separator as its own
element of the resulting array, in which case these elements get reassembled
with an empty string as the join character.

`ltrim` and `rtrim` are used to hide unimportant characters from the resulting
array element strings, such as leading/trailing whitespace (like `diff -b`)
and you can (exclusive to the others) specify `trim` to set them both to the
same value.  `ltrim` is implicitly anchored to the start of the string, and
`rtrim` is implicitly anchored to the end, allowing a common value for `trim`.
If you want to permanently remove the trimmed characters, use `discard: true`.

If the separators are not added to the array using `keep` they are stored in
a parallel array of metadata associated with the array.  Likewise, trimmed
characters are stored there.  As actions like `ASSIGN`, `MOVE`, and `SPLICE`
process the array, assignments from one element of the array to another carry
the separator and trimmings to the new element.  Assignments of values from
any other source (constants, a different array, etc) do not carry this
metadata, and eliminate trimmings of the assigned line.  Deletions on the
array perform a parallel deletion on the metadata array, and every insert
operation clones the separator of the previous or following element such that
every element continues to have a defined separator.  Trimmings do not get
cloned.
If the array becomes empty at any point, the separator reverts to `default`.
`default` must be specified when `sep` is a regular expresion and `keep` is
not used; if `sep` is a string then it also implies `default` and if `keep`
is used then separators are not relevant to the join operation.

Once all actions have been applied to the array, it gets reassembled into a
string.  Each element must be a string, or it fails with `INVALID_GRAFT`.
Each element gets any hidden trimmed characters re-added, and any hidden
separator is appended only if there is a following element.  In order to get a
"newline at the end of the file" the final element of the array needs to be an
empty string.  It is the responsibility of the nested actions to preserve or
add this element as desired.

This is the primary tool used to re-implement text diff/patch behavior:

    ['SPLIT',
      { sep: ["|", "\n", "\r\n"], default: "\n", trim: ["{",0,null," "] }
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

### JGRAFT

    ['JGRAFT', metadata, action1, action2...]

This action is used for the top-level of an exported JGraft JSON file, or
optional metadata deeper within the structure.
When serialized to JSON, the top-level 'JGRAFT' action name is not replaced by
a number, and there should be no whitespace until after the following comma.
This gives JGraft JSON files a "magic number" of

    '["JGRAFT",'

The `metadata` is an object with semi-free-form properties, where top-level
names matching `/^[a-z][A-Za-z0-9]*$/` are reserved for the protocol, leading
underscores are forbidden, and any others are user-defined.  The following
properties are currently defined:

  - v: integer; the minimum JGraft specification version required to
       correctly interpret the contents of this structure
  - writer: a string identifying the tool that authored the structure
  - comment: a free-form text comment string
  - regex: an object holding default configuration for regular expressions
    - common: an array of [common subexpressions](#common-subexpressions)

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

## <span id="matching">Context Matching</span>

JGraft provides a rich collection of match specifications.  The basic match
function is `HAS`, which requires certain properties to exist but does not
specify them in full.  This is used any time a scalar value or plain object
is encountered.  The matching system has its own collection of match functions
specified as lisp-style arrays where the first element is a function name and
the remaining elements are passed as arguments to that function.
Literal arrays can be specified by wrapping them in an additional array, such
that the literal array appears where the function name would normally appear.

The following functions all implicitly operate on the current node and return
a boolean of whether the current node passes the test:

Name      | Description
----------|----------------------------------------------------------
HAS       | Switch to partial matching
IS        | Switch to exact matching
AND       | All conditions match at current node
OR        | At least one condition matches at current node
NOT       | None of the conditions match at current node
EXISTS    | Current node exists (including `null` values)
BOOL      | Match any boolean, or cast to boolean for more matching
NUM       | Match any number, or cast to numeric for more matching
INT       | Match any integer, or cast to integer for more matching
STR       | Match any string, or cast to string for more matching
ARRAY     | Match any array, or match sub-range of an array
SPLIT     | Split a string to match against the resulting array

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

This is the default function used when a non-array is seen in a match
specification.  For scalars, this function is equivalent to JavaScript's `===`
operator.  For objects, the current node must be an object, and each specified
property must match the same on the current node according to `HAS`.
Arrays are used to specify alternate match functions, and will receive a
current node of whichever sub-property is being matched.  Literal arrays are
specified as an array within an array, and mean that the current node must be
an array where each specified element matches according to `HAS`.

Extra properties or array elements in the current node (beyond what was
specified) are ignored.

### IS

    ['IS', match_spec, match_spec...]

This is a variant of 'HAS' that forbids extra properties in a value object
that were not in the specification object.  Nested objects within the
specification are also handled as 'IS' tests, until the next function boundary.
Objects within a nested function revert to 'HAS' semantics (unless of course
that function is 'IS').

The function can take additional arguments to perform an implied 'OR'.

### AND

    ['AND', match_spec, match_spec...]

All following patterns must match at the current node.

### OR

    ['OR', match_spec, match_spec...]

Any one of the following patterns must match at the current node.

### NOT

    ['NOT', match_spec, or_match_spec...]

Matches when none of the arguments match at the current node.

### EXISTS

    ['EXISTS']

The property exists on the object (or array).  It may be `null`.

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

With one or two arguments, also verify that the number is within the range of
`[min,max]` (inclusive).  `min` or `max` may be `null` to omit the respective
test.  In contexts where infinity is available, positive and negative infinity
may also be used for the `min` and `max` values.  In contexts where NaN is
available, NaN may not specify a bound of the range, and declaring any range
causes NaN values in the data to fail to match.

### INT

    ['INT']
    ['INT', min, max]

Same as `NUM` above, but rejects all non-integer values.

### STR

    ['STR']
    ['STR', regex_spec]

With no arguments, returns true if the current node is a string.

You may specify a [regular expression](#regex) to additionally constrain which
strings match.

### ARRAY

    ['ARRAY']
    ['ARRAY', length]
    ['ARRAY', offset, match_spec1, match_spec2...]

With no arguments, this merely asserts that the current node is an array.

With one argument, it asserts the current node is an array with a number of
elements (an exact number under `IS`, or at least this many under `HAS`).

With arguments, this asserts that a range of the array matches the supplied
conditions, according to `HAS`.  The `offset` may be negative to count
backward from the end of the array.

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
in any parameter that allows regular expressions with the full power and
capabilities of your host language.

If you need to portably serialize the JGraft, the regular expressions may be
specified as (you guessed it) lisp-style structure describing the pattern.
Think of it as a pre-parsed regular expression.  The notation is not intended
to be written by hand, but aims to be readable enough to be debugged visually.  

Implementations can try to flatten this structure into the syntax relevant for
the host language, or just directly implement the character-matching engine.
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

Character matching is defined in terms of the unicode character set, which
JSON already provides.

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

The initial value of the common expression namespace can be supplied by
enclosing ['JGRAFT' actions](#jgraft) via the metadata property
`regex.common`.  Temporary overrides can also be specified via Sequence
functions that begin with a `{}` configuration block.  In both cases, the list
is specified as an array of key/value properties to apply to the namespace in
order.

Action Example:

    ['JGRAFT',{
      regex: {
        common: [
          "word", ["[", '', 48, 58, 65, 91, 95, 96, 97...],
          ...

Function Example:

    [{"common":["linebreak", ["|", "\n", "\r\n"]]}, ...]

### Functions

The functions are named according to the common regex notation they implement:

Function                | Description
------------------------|-----------------------------------------------------
`['', ...]`             | Sequence - match each element in order
`[{}, ...]`             | Sequence with configuration
`["^"]`                 | StartAnchor - Start of string
`["$"]`                 | EndAnchor - End of string
`["|", ... ]`           | Alternation - Match any one alternative
`["[", ... ]`           | Charclass - A set of distinct characters
`["{", min, max, ... ]` | Repetition - Match zero, one or more times
`["&", ...]`            | Ref - reference a common subexpresion

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

Match any one alternative.  Each parameter may be a single string which must
match exactly, or an array indicating a function.  The order of alternatives
should not matter in theory, because the output is a simple boolean and does
not track positional matches of the pattern components.  Implementations which
convert this specification to the host language's regex engine are free to
attempt to re-order the alternatives in the order that yields the highest
performance.

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
resolve to a charset definition.

Examples:

    // .
    ["[", 0,null]
    ["[", false]    // because it starts from the full set of chars
                    // and removes nothing
    
    // [0-9]
    ["[", "0123456789"]
    ["[", "0", "9"]
    ["[", 48, 58]
    
    // [0-9a-zA-Z_.-]
    ["[", "0","9", "A","Z", "a","z", "_-."]
    ["[", 0x30, 0x39, 0x41, 0x5A, 0x61, 0x7A, "_-."]
    
    // [^\n]
    ["[", false, "\n"]

#### Repetition

    ["{", min, max, pattern...]

The first two parameters specifiy the minimum and maximum repeat count for
which all remaining parameters (the pattern components) must be found.
`min` must be an integer greater or equal to zero, and `max` must be greater
or equal to `min`, or `null` to enable unlimited matching.

#### Ref

Inject the value of a common subexpression

## Errors

Applying a JGraft to a target data structure may fail with:

  * `INVALID_GRAFT` - the structure of the Graft itself does not match the
                      spec, or is semantically invalid in some way.
  * `INVALID_TARGET` - the graft describes an operation that can't apply due
                       to the type or structure of the target
  * `NO_MATCH` - a `MATCH` action failed to find a matching context

