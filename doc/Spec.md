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
The directives (classified below as Actions and Match Functions) have symbolic
names, but also can be represented by small integers for better performance in
production environments.  The symbolic names are used in all examples below.

## Actions

A JGraft data structure is a tree of actions, Lisp-style, where each action is
an array that begins with the action name (or numeric opcode) and contains
parameters for that action, which will often include sub-actions.

### AT

    ['AT', path, subaction1, subaction2...]

Navigate to a sub-node of the current node, then execute all sub-actions with
that node as the current node.  The path must exist or the graft operation
fails with `INVALID_TAGET`.

### MATCH

    ['MATCH', context_struct, subaction1, subaction2...]

Assert that the current node matches a [Context specification](#context-match),
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

    ['IF', context, action, else_action]
    ['IF', context1, action1, context2, action2... else_action]

Like the `MATCH` action, this compares the current node to a Context
specification, optionally performing a fuzzy match.  Unlike `MATCH`, it is not
fatal if it does not match, and will execute an optional `else_action` on
failure.
The action may have additional conditions; if two or more array elements
remain after the last action element, they are considered to be an additional
match_struct and action.  If one element remains, it is an `else_action`.
The `else_action` is also optional.

### ASSIGN

    ['ASSIGN', prop1, val1, prop2, val2, ...]

Assign one or more properties (or properties of sub-objects) at the current
node.  This uses the JavaScript concept of properties, where an array has
numbered properties and an object has named properties.  If the `propN` is an
array, it specifies a path to a sub-object.  Negative numbers reference
array elements counting backward from the end, and `false` references a logical
element beyond the end of the array.  The empty path refers to the node itself.

Objects along the property path will be auto-vivified if they don't exist, but
the action fails with `INVALID_TARGET` if it would need to overwrite a scalar
value with an object or array.
To prevent inappropriate creation of sub-objects, use a containing `MATCH`
directive that asserts the presence or absence of the structure as desired.
Assigning to an array more than one element beyond the end is still a fatal
error, returning `INVALID_TARGET`.

    ['ASSIGN', 0, {x:1,y:1}]            // node[0]= { x: 1, y : 1 };
    ['ASSIGN', [0,'x'], 5]              // node[0].x= 5;
    ['ASSIGN', [-1,'x'], 6]             // node[node.length-1].x= 6;
    ['ASSIGN', [false], {x:6}]          // node[node.length]= { x: 6 };

If the `valueN` is an array, it means to get or calculate the value via an
expression. (described below in [Expression Functions](#expression-functions).
Any data referenced using the `GET` expression will be assigned by value, so
effectively deep-cloned, though implementations may optimize this with a
Copy-On-Write strategy.

    ['ASSIGN', ['b',0], ['GET','a',4]]  // node.b[0]= deep_clone(node.a[4]);
    ['ASSIGN', 'a', ['DEL']]            // delete node['a'];
    ['ASSIGN', 'a', [[1,2,3]]]          // node.a= [1,2,3];

Assignments are performed in order, so later value expressions see the
modified state of the node.  If you need to hold onto current
values temporarily without overwriting them, you can store them at a path
starting with `null`.  This temporary variable exists only for the duration of
the `ASSIGN` action.

    ['ASSIGN', [null], ['GET']]         // temp= deep_clone(node);
    ['ASSIGN', 'b', ['GET',null,'a']]   // node.b= temp.a;
    ['ASSIGN', 'a', ['GET',null,'b']]   // node.a= temp.b;

Deletions to array elements cause the following elements to be shifted to fill
the gap.  There is no way to perform a logical insert operation using the
`ASSIGN` action, other than explicitly re-assigning every element; see `SPLICE`.

### MOVE

    ['MOVE', src_prop1, dst_prop1, src_prop2, dst_prop2...]

Rename one or more properties within a single object or array.  This is an
optimized version of 'ASSIGN' primarily intended for shuffling the elements of
an array, though it may also be used as a shorthand for renaming properties of
objects.  The logical algorithm (which may be implemented in any equivalent
manner) is as follows:

  - For arrays:
    - For each pair of `src_prop`, `dst_prop`,
      - Index `src_prop` must exist in the array, and must be distinct from
        any other `src_prop` in this action.
      - Index `dst_prop` must be in the range [0..length] (optionally using
        negative number notation to count backward from the end of the array,
        or the special value 'false' to refer to the length of the array) or
        the value `null` which means to delete it.
      - Queue the value at `src_prop` for insertion at index `dst_prop` if
        `dst_prop` was not null.
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
      - Queue the assignment of the value from `src_prop` to `dst_prop`,
        unless `dst_prop` was `null`.
      - Queue the deletion of `src_prop`.
    - Perform all queued deletions
    - Perform all queued assignments

    [`MOVE',0,-1,-1,0]                  // ins[node.length-1]= node[0];
                                        // ins[0]=             node[node.length-1];
                                        // node.splice(node.length-1, 1, ins[node.length-1]);
                                        // node.splice(0, 1, ins[0]);
    
    ['MOVE',1,false,2,false]            // ins[node.length]= [ node[1], node[2] ];
                                        // node.splice(node.length, 0, ...ins[node.length]);
                                        // node.splice(1, 2);

### SPLICE

    ['SPLICE', offset, count, replacement1, replacement2...]

Replace a span of an array, just like the splice function found in most
programming languages.  The current node must be an array or the graft fails
with `INVALID_TARGET`.  `offset` may be `false` to refer to the end of the
array, or negaitve to count backward from the end of the array, but the
resulting index must be within `[0..length]` or the graft fails with
`INVALID_TARGET`.
`count` may be `null` to replace the remainder of the array.
The replacement values follow the same expression notation used by `ASSIGN`.
All expressions in the replacement are evaluated *before* performing the
splice, unlike the `ASSIGN` action.

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

## Context Matching Functions <span id="context-match"></span>

JGraft provides a rich collection of match specifications.  The basic match
function is `HAS`, described below.  This is used any time a scalar value or
plain object is encountered.  Other functions are specified as lisp-style
arrays where the first element is a function name, and the remaining elements
are passed as arguments to that function.  Literal arrays can be specified
by wrapping them in an additional array, such that the literal array appears
where the function name would normally appear.

### HAS

    null         // cur_node === null
    0            // cur_node === 0
    'str'        // cur_node === 'str'
    { a:1, b:2 } // cur_node.a === 1 && cur_node.b === 2
    [[1,2]]      // cur_node[0] === 1 && cur_node[1] === 2

For scalars, this function is equivalent to JavaScript's `===` operator.
For objects (including array values being compared to specification objects)
all of the properties seen in the specification object must match the value
object, recursively.  Extra properties in the value object are ignored.

### IS

    ['IS', exactly_this, or_exactly_this...]

This is a variant of 'HAS' that forbids extra properties in a value object
that were not in the specification object.  Additionally, specification
objects cannot match array values.  Nested objects within the specification
are also handled as 'IS' tests, until the next function.  Objects within
a nested function revert to using 'HAS' unless that function is 'IS' or 'LIKE'.

The function can take additional arguments to perform an implied 'OR'.

### EXISTS

    ['EXISTS']

The property exists on the object (or array)

### DEFINED

    ['DEFINED']

The property exists and value is not `null` (and not `undefined` if the host
language has that distinction).

### BOOL

    ['BOOL']
    ['BOOL', cond]

With no arguments, returns true if the current node is a boolean value.
(identical to `['OR', true, false]`)

With one argument, performs a type cast on the current node (only if the
current node looks boolean-ish per the table below) and then tests `cond` on
the cast value.  Valid booleans are defined as a whitespace-trimmed
case-insensitive match against the following:

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
    ['NUM', cond]

With no arguments, returns true if the current node is a number, including NaN
and Inf.  Note that JSON-compatible data structures cannot contain these
values, but they are included here for use with host language support.

With one argument, coerces the current node to a number (must be a number. or
string that can be parsed as a JSON number after trimming whitespace) and then
tests `cond` on the cast value.

To test whether a node can be cast to a number, use `['NUM',['NUM']]`

### REAL

    ['REAL']
    ['REAL', cond]

Same as `NUM` above, but rejects `NaN` and `Infinity` values.

### INT

    ['INT']
    ['INT', cond]

Same as `NUM` above, but rejects all non-integer values.  Values formatted
with a decimal point but which evaluate to an integer are accepted for the
coercion.

### STR

    ['STR']
    ['STR', cond]

With no arguments, returns true if the current node is a string.

With one argument, coerces the value to a string.  Booleans are coerced to the
strings 'true' and 'false', and numbers are stringified to decimal form (no
scientific notation) or possibly to the values `'NaN'` or `'Infinity'` if the
data is able to contain those values.

### ARRAY

    ['ARRAY']
    ['ARRAY', offset, cond[i], cond[i+1], cond[i+2]...]

With no arguments, this merely asserts that the current node is an array.

With arguments, this asserts that a range of the array matches the supplied
conditions, according to `HAS`.  The `offset` may be negative to count
backward from the end of the array.

    // Assert that the array ends with 'x', 'y', 'z'
    ['ARRAY', -3, 'x', 'y', 'z']

### AND

    ['AND', cond, and_cond...]

All following expressions must be true

### OR

    ['OR', cond, or_cond...]

Any one of the following expressions must be true

### NOT

    ['NOT', cond, or_cond...]

True when none of the arguments is true.
(this would perhaps be more accurately labeled 'NOR')

### SPLIT

    ['SPLIT', split_spec, cond1, cond2...]

This is the same as the `SPLIT` action, but operates in matching context as a
boolean that returns true if all the conditions return true.  Using this in
a match is slower than using the `SPLIT` action and matching within the
resulting array, but this gives you the chance to search for substrings in
content during a fuzzy search when you don't know which specific node path
needs to be split.

With a simple enough split and match condition, implementations may be able to
compile this into a regular expression.

### LT

    ['LT', 65536]
    ['LT', 'ASCII-STRING']

Less-than test.  Returns true when the current node is the same type as the
specified value and sorts less than the value according to its type.
This is only portably defined for numbers and pure-ASCII strings, currently,
since unicode strings have locale baggage.

An unsupported type in the specification generates an `INVALID_GRAFT` error.
If the current node's type does not match the specification value's type, this
generates an `INVALID_TARGET` error.
See `NUM` and `STR` if you wish to coerce the current node first.

### LE

Less-or-equal test.  Same design as `LT`.

### GT

Greater-than.  Same design as `LT`.

### GE

Greater-or-equal test.  Same design as `LT`.

## Errors

Applying a JGraft to a target data structure may fail with:

  * `INVALID_GRAFT` - if the structure of the Graft itself is not well-defined
  * `INVALID_TARGET` - if the graft describes an operation that can't apply to the target
  * `NO_MATCH` - if any `MATCH` action fails to find a matching context

