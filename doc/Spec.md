# JGraft

JGraft is a data structure built around basic JSON-compatible concepts and in
some cases JavaScript semantics.  The main goal is to be able to exchange it
in the form of JSON, especially between application back-ends and JavaScript
front-ends, but other languages may include data types of their own if they
don't need to serialize to JSON.
The structure primarily uses array primitives to encode directives,
Lisp-style, but switches to objects with named properties for any case where
that is more convenient.  The directives (classified below as Actions and
Match Functions) have symbolic names, but also can be represented by small
integers for better performance in production environments.  The symbolic
names are used in all examples below.

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

Assign one or more properties of the current node.  This uses the JavaScript
concept of properties, where an array has numbered properties and an object
has named properties.  Properties within sub-nodes may be specified by using
an array of property names instead of a single string / integer.
Objects along the property path will be auto-vivified if it they don't exist,
but fails with `INVALID_TARGET` if it would overwrite a scalar value with an
object or array.

To prevent inappropriate creation of sub-objects, use a `MATCH` directive that
asserts the presence or absence of the structure as desired.  Assigning to an
array more than one element beyond the end is still a fatal error, returning
`INVALID_TARGET`.

You may overwrite the current node itself by specifying a property path of `[]`.

### MOVE

  ['MOVE', src_prop, dst_prop, src_prop2, dst_prop2...]

Move the value of one property to another property, deleting the original
property.  Moves are performed in a logically simultaneous manner; every
`src_prop` in the specification refers to the initial state of the tree of
nodes and the destinations are not altered (including shifting array elements
to fill holes) until all source values have been collected.  This allows you to
swap two properties with a seuqence of `['MOVE',1,2,2,1]` or make a copy of a
property by also moving it to its current location with `['MOVE',1,2,1,1]`.
This action is also used for deleting properties, by specifying `null` for the
destination. So for example, `['MOVE',1,null,2,null]` is equivalent to
`['SPLICE',1,2]`.  Similarly, omitting the destination acts as a delete, so
in a multi-action context that could be specified as `['MOVE',2],['MOVE',1]`,
though that requires care to avoid referencing an array element that shifted.

If any source path doesn't exist, or any destination path can't be created,
the graft operation fails with `INVALID_TARGET`.

If the `MOVE` action has only one source/destination pair it executes with
less overhead.  If sequential move / delete actions are acceptable, consider
that over a single multiple-attribute action.

### SPLICE

  ['SPLICE', offset, count, replacement1, replacement2...]

Replace a span of an array, just like the splice function found in most
programming languages.  The current node must be an array or the graft fails
with `INVALID_TARGET`.  `offset` may be `null` to refer to the end of the
array, or negaitve to count backward from the end of the array, but the
resulting index must be within `[0..length]` or the graft fails with
`INVALID_TARGET`.
`count` may be `null` to replace the remainder of the array.

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
      ['SEQ', 10,
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

### SEQ

  ['SEQ', FirstIdx, value0, value1...]

This generates a specification object with numbered keys for each of the
supplied values, starting from FirstIdx:

  { FirstIdx: value0,
    [FirstIdx+1]: value1,
    ...
  }

The object is then used as normal by functions `HAS`.  (It would not be very
useful for `IS` because `IS` would fail on indices lower than FirstIdx)
This is essentially like a sparse array notation, which can't be specified
directly in JSON.

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

## Errors

Applying a JGraft to a target data structure may fail with:

  * `INVALID_GRAFT` - if the structure of the Graft itself is not well-defined
  * `INVALID_TARGET` - if the graft describes an operation that can't apply to the target
  * `NO_MATCH` - if any `MATCH` action fails to find a matching context

