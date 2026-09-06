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

Relocate one or more properties within a single object or array.  This is an
alternative to `ASSIGN` optimized for shuffling the elements of an array when
insertions are not required, though it may also be used to rename properties
of objects.  All source and destination positions are resolved against the
state of the current container at the start of the MOVE action.
The logical algorithm (which may be implemented in any equivalent manner) is
as follows:

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
      - perform a logical splice(), replacing any queued deletion with any
        queued insertions for that index.
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
when re-assembling the string.

The `split_spec` may be a simple string used as a verbatim separator, or it may
be an object with a more elaborate specification:

    {"sep": ...,        // One or more strings to split on
     "trim": ...,       // One or more strings to trim from start/end of elements
     "discard": bool,   // Don't preserve trimmed chars when re-assembling
     "canonical": bool, // When there are multiple separator options, always
                        // reassemble with the first element of "sep" rather
                        // than the original value.
    }

This is the primary tool used to re-implement text diff/patch behavior:

    ['SPLIT',
      { sep: "\n", trim: [" ","\r"] }
      ['MATCH',
        ['ARRAY', 10,
          "Line ten",
          "Line eleven",
          "Line twelve",
          "Line thirteen"
        ],
        ["SPLICE", 11, 2
          "New Line eleven"
          // no line 12
        ]
      ],
    ]

### JGRAFT

    ['JGRAFT', min_version, metadata, action1, action2...]

This action is used for the top-level of an exported JGraft JSON file, or
optional metadata (like a comment) deeper within the structure.
It specifies the minimum version of the JGraft specification required to
interpret the contents, and then has a semi-free-form metadata object,
followed by one or more actions.  When serialized to JSON, the top-level
'JGRAFT' is not replaced by a number, and there should be no whitespace until
after the comma following min_version.  This gives JGraft JSON files a
"magic number" of

    '["JGRAFT",N,'

for whichever version `N` is being written.

The following metadata properties are pre-defined:

  - comment: a free-form text comment string
  - writer: a string identifying the tool that authored the structure

All property names matching `/[a-z][a-z0-9_]*/` are reserved for future
standards.  Implementations are encouraged to use a unique-ish property name
containing a period (such as Java-style reverse domain name) or name with
leading underscore, and place all data within an object under that.

## Context Matching <span id="matching"></span>

The matching system is invoked from either the `MATCH` action or the `MATCH`
expression function.

JGraft provides a rich collection of match specifications.  The basic match
function is `HAS` which requires certain properties to exist but does not
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
    ['BOOL', match_spec]

With no arguments, returns true if the current node is a boolean value.
(identical to `['OR', true, false]`)

With one argument, performs a type cast on the current node (only if the
current node looks boolean-ish per the table below) and then tests
`match_spec` against the cast value.  Valid booleans are defined as a
whitespace-trimmed case-insensitive match against the following:

 Input   | Derived value
---------|--------------
 bool    | pass-through
 0       | false
 ''      | false
 '0'     | false
 'f'     | false
 'false' | false
 1       | true
 '1'     | true
 't'     | true
 'true'  | true

To test whether a node can be cast to a boolean, use `['BOOL',['BOOL']]`

### NUM

    ['NUM']
    ['NUM', match_spec]

With no arguments, returns true if the current node is a number, including NaN
and Inf.  Note that JSON-compatible data structures cannot contain these
values, but they are included here for use with host language support.

With one argument, coerces the current node to a number (must be a number, or
string that can be parsed as a JSON number after trimming whitespace) and then
tests `cond` on the cast value.

To test whether a node can be cast to a number, use `['NUM',['NUM']]`

### INT

    ['INT']
    ['INT', match_spec]

Same as `NUM` above, but rejects all non-integer values.  Values formatted
with a decimal point but which evaluate to an integer are accepted for the
coercion.

### STR

    ['STR']
    ['STR', match_spec]

With no arguments, returns true if the current node is a string.

With one argument, coerces the value to a string.  Booleans are coerced to the
strings 'true' and 'false', and numbers are stringified to decimal form (no
scientific notation) or possibly to the values `'NaN'` or `'Infinity'` if the
data is able to contain those values.  (JSON is not, host extensions could)

### ARRAY

    ['ARRAY']
    ['ARRAY', length]
    ['ARRAY', offset, match_spec1, match_spec2...]

With no arguments, this merely asserts that the current node is an array.

With one argument, it asserts the current node is an array with a number of
elements (an exact number under `IS`, or at least this many under `MATCH`).

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

## Errors

Applying a JGraft to a target data structure may fail with:

  * `INVALID_GRAFT` - if the structure of the Graft itself is not well-defined
  * `INVALID_TARGET` - if the graft describes an operation that can't apply to the target
  * `NO_MATCH` - if any `MATCH` action fails to find a matching context

