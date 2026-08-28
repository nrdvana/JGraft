## Actions

A JGraft data structure is a tree of actions, Lisp-style, where each action is
an array that begins with the action name (or numeric opcode) and contains
parameters for that action, which will often include sub-actions.

The symbolic names are described below, and used in all examples, but in
production environments where size and speed matter, each symbolic name gets
replaced by a small integer.

### AT

  ['AT', path, subaction1, subaction2...]

Navigate to a sub-node of the current node, then execute all sub-actions at
that node.  The path must exist.

### MATCH

  ['MATCH', context_struct, subaction1, subaction2...]

Assert that the current node matches a [Context description](#context-match),
then execute all sub-actions in sequence.  If it doesn't match, the patch
operation will fail, unless the user has enabled "fuzzy" matching in which case
the engine looks at nearby array indices to find a match.  The exact algorithm
for permitting "fuzzy" matches is dependent on the engine and parameters
supplied by the user.
If a fuzzy match is successful, the array offsets discovered during that
process are applied to any property path in the sub-actions which reference
those arrays.

### IF

  ['IF', context_struct, action, else_action]
  ['IF', context1_struct, action1, match2_struct, action2... else_action]

Like the 'AT' action, this compares the current node to a Context description,
optionally performing a fuzzy match.  Unlike 'AT', it is not fatal if it does
not match, and will execute an optional `else_action` on failure.  The action
may have additional conditions; if two or more array elements remain after the
last action element, they are considered to be an additional match_struct and
action.  If one element remains, it is an `else_action`.  The `else_action` is
also optional.

### ASSIGN

  ['ASSIGN', prop1, val1, prop2, val2, ...]

Assign one or more properties of the current node.  Sub-nodes may be specified
by using an array of property names instead of a single property name string.
Objects along the property path will be auto-vivified if it they don't exist,
but auto-vivification may not overwrite scalar values with objects.
To prevent inappropriate creation of sub-objects, use a MATCH directive that
asserts the presence or absence of the structure as desired.  Assigning to an
array more than one element beyond the end is an error.

You may overwrite the current node itself by specifying a property path of `[]`.
If the final value is omitted it is treated as `null`.

### DELETE

  ['DELETE', prop1, prop2, ...]

Delete one or more properties.  If the property is a path, only the final
property of the path is deleted.  Objects along the path are not auto-vivified,
nor do they need to exist.  If the property being deleted is part of an array,
elements beyond it are shifted.  When multiple properties are deleted from the
same array, the property names (or paths) should refer to the original element
index, not the index after being shifted.  So for example, `['DELETE',1,2,3]`
deletes elements 1, 2 and 3 from the array then shifts the remainder down; it
doesn't delete element 1, (shift), 2, (shift), 3 which would be original
elements 1,3,5.  However, subsequent actions will refer to the modified array.

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
`['SPLICE',1,2]`.

### SPLICE

  ['SPLICE', offset, count, replacement1, replacement2...]

Replace a span of an array, just like the splice function found in most
programming languages.  `offset` may be `null` to refer to the end of the array,
or negaitve to count backward from the end of the array, but the resulting
index must be within `[0..length]` or the action fails.  `count` may be `null`
to replace the remainder of the array.

### SPLIT

  ['SPLIT', split_spec, subaction1, subaction2...]

This can only be applied at a string node.  It splits the string on the
`split_spec` to create an array, then runs each sub-action on that array, then
re-assembles the string from the elements of the array.  The separator and any
trimmed characters are preserved to use when re-assembling the string.

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

## Context Matching <span id="context-match"></span>

JGraft provides a rich collection of match specifications.  The basic match
function is 'HAS', described below.  This is used any time a scalar value or
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

### LIKE

  ['LIKE', this, or_this...]

This is a variant of 'HAS' that uses JavaScript's '==' operator for relaxed
comparisons.  However, comparing scalars to objects still returns false.
Nested objects within the specification are also handled as 'LIKE' tests,
until the next function.  Objects within a nested function revert to using
'HAS' unless that function is 'IS' or 'LIKE'.

The function can take additional arguments to perform an implied 'OR'.

### EXISTS

  ['EXISTS']

The property exists on the object (or array)

### DEFINED

  ['DEFINED']

The value is not `null` or `undefined`.

### AND

  ['AND', this, and_this...]

All following expressions must be true

### OR

  ['OR', this, or_this...]

Any one of the following expressions must be true

### NAND

  ['NAND', this, and_this...]

True as long as at least one option is false.
(Both `NOR` and `NAND` function as `NOT` when given a single argument)

### NOR

  ['NOR', this, or_this...]

True when none of the arguments is true.
(Both `NOR` and `NAND` function as `NOT` when given a single argument)

### SEQ

  ['SEQ', FirstIdx, value0, value1...]

This generates a specification object with numbered keys for each of the
supplied values, starting from FirstIdx:

  { FirstIdx: value0,
    [FirstIdx+1]: value1,
    ...
  }

The object is then used as normal by functions `HAS` or `LIKE`.  (It would not
be very useful for `IS` because `IS` would fail on indices lower than FirstIdx)
This is essentially like a sparse array notation, which can't be specified
directly in JSON.

### SPLIT

  ['SPLIT', split_spec, cond1, cond2...]

This is the same as the `SPLIT` action, but operates in matching context as a
boolean that returns true if all the conditions return true.  Using this in
a match is slower than using the SPLIT action and matching within the resulting
array, but this gives you the chance to search for substrings in content during
a fuzzy search when you don't know which specific node path needs to be split.

With a simple enough split and match condition, implementations may be able to
compile this into a regular expression.
