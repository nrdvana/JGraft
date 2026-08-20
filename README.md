# JGraft

JGraft is a specification for a data structure which describes how to modify a
tree of JSON-like data.  While it is naturally applicable to JSON, it can also
be used for actual JavaScript data contianing arbitrary JavaScript objects.

It is designed to solve the same sort of problems that you would use the
popular unified-diff format for, but applied to trees of data.  In particular,
it can diff text elements within a tree, which RFC 6902 "json patch" format
cannot, and is itself a recursive data structure so that patches can be
combined more easily.  It also emphasizes an algorithm of looking for matching
elements before applying edits, so patches can be applied at adjusted array
offsets much like how text patches can be applied at a different line number
within a modified text file.  The format is entirely specified as structured
data rather than inventing any syntax, so no parsing is required, aside from
a JSON parser if you want to serialize the structure.

## Examples

A very basic data update looks like:

```
[ 2,     // opcode 2 is "assign property/value pairs"
  "x",6  // cur_node.x=6
]
```

The property name can be a path:

```
[ 2,
  ["foo","bar","baz"],42  // cur_node.foo.bar.baz=42
]
```

This JGraft is not reversible because the old value is not known.  If you
request context matching for the update, like:

```
[ 0,         // opcode 0 is "match context before applying actions"
  {"x":5},   // context spec
  [2,"x",6]  // action
]
```

the JGraft becomes reversible because the context spec clause contains the old
value for what is being overwritten in the following actions.

This can expand to multiple attributes on the same object:

```
[ 0,
  {"x":5,"y":0,"z":0,"patched":false},
  [2,"x",6,"y",-2,"z",1,"patched",true]
}
```

This says "On an object where x is 5 and y is 0 and z is 0 and patched is
false, set x to 6 and set y to -2 and set z to 1 and set patched to true".
The context spec does not need to constrain all the properties being updated,
but if it does, the JGraft will be reversible.

One of the key features of JGraft is the ability to patch text just like the
traditional Unix diff format:

```
[ 7, "\n",   // opcode 7 is "split(separator, actions)"
  [ 0,
    [ "seq", 12,   // describe array element sequence starting from [12]
      "line of text",
      "thing we want to change",
      "other thing we want to change",
      "line of text"
    ],
    [ 6, 13, 2,    // opcode 6 is "splice(ofs, n_del, values...)"
      "new first line",
      "new second line"
    ]
  ]
}
```

Using the "split" opcode converts the current node from a string of text to an
array, runs a series of actions on it, then re-joins it back to a string.
This is equivalent to

```
@@ -13,4 +13,4 @@
 line of text
-thing we want to change
-other thing we want to change
+new first line
+new second line
 line of text
```

(note that JGraft always uses 0-based array indices rather than 1-based line
numbers of diff/patch)

Importantly, if the lines of text described by the context spec are found at a
different array offset than 12, any actions referring to an array index of
that node get updated to match, just like Unix `patch` locating the context
lines at a different line number.

## JGraft Actions

JGraft is fundamentally just a tree of actions to perform on the tree of data.
The structure is optimized for size and processing efficiency, so the basic
action is an array where element 0 is a numeric opcode, and the meaning of the
remaining elements are defined by that opcode.

#### Opcodes

<table>
<tr><th>Code<th>Op(Parameters)<th>Description
<tr><td> 0  <td>Context(context_spec, actions...)
    <td>Assert that the current node matches the `context_spec` (described
        below) and then execute one or more actions.
        In an array, if the `context_spec` doesn't match at the expected
        offset, it may be found at another offset and all offsets of the
        current node referenced by the `actions` will be shifted to match.
<tr><td> 1  <td>Subpath(path, actions...)
    <td>Change the current node to a sub-path of the current node,
        then execute the actions.
<tr><td> 2  <td>Assign(prop, value, prop, value...)
    <td>Assign one or more properties of the current node, provided as a
        key/value list.  The current node must be an object or array.
        The `prop` values may be arrays to specify a deeper path within the
        current node, or an empty array to specify the current node itself.
        All elements of a path must exist except the final property name.
        Values are literal.  If the value is omitted, the property is deleted.
        (which means this op can delete only one property per action)
<tr><td> 3  <td>Default(prop, value, prop, value...)
    <td>Apply default values to a property.  This is just like the assignment
        op but only applies when a property doesn't already exist.  Existing
        values are unchanged.  This is useful for vivifying objects along a
        path before trying to assign properties of the path.
<tr><td> 4  <td>Move(from, to, from, to...)
    <td>Move (rename) a property.  For objects, `from` must exist and `to`
        must not exist.  For arrays, `from` and `to` must be numeric, but may
        be negative to count backward from the end of the array, where -1 is
        the final element of the array.  Also, the elements inbetween will be
        shifted rather than overwriting the `to` element.
        If multiple from/to pairs are given, they will be performed
        simultaneously, giving you the ability to swap elements.
<tr><td> 5  <td>Copy(from, to)
    <td>Just like Move, but clone the source element instead of removing it.
<tr><td> 6  <td>Splice(offset, del_count, value...)
    <td>Perform an array splice at the given offset, deleting `del_count`
        elements, and then inserting zero or more values.  `offset` may be
        'N' to reference the end of the list, or negative to count backward
        from the end of the list.  `del_count` elements must exist at `offset`,
        but `del_count` may be the special value '*' to delete all elements
        until the end of the array.
<tr><td> 7  <td>Split(separator, actions...)
    <td>Split the current node (which must be a string) on a separator token,
        then execute the `actions` with the current node set to the resulting
        array, then reassemble the string with a 'join' using the separator.
        The separator may also be an object with additional options; see below.
</table>

