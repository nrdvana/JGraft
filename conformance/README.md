# JGraft Conformance Tests

This directory contains JSON files describing many instances of

    { name:         // short phrase describing test
      desc:         // optional longer comment explaining the test
      input:        // any JSON value
      jgraft:       // any valid or invalid JGraft
      output:       // the result of applying the graft
      needProto:    // bool; the test requires "__proto__" support
      errors: {     // possible ways the graft can fail
        NO_MATCH: bool,        // can fail with NO_MATCH
        INVALID_TARGET: bool,  // can fail with INVALID_TARGET
        INVALID_OPERAND: bool, // can fail with INVALID_OPERAND
        INVALID_GRAFT: bool,   // can fail with INVALID_GRAFT
        RESOURCE_LIMIT: bool,  // should fail with RESOURCE_LIMIT
      },
      limits: {     // the limits that should be applied
        mem:          // maximum number of (estimated) bytes allocated
        ops:          // maximum number of operations performed
        new_obj:      // maximum number of new objects/arrays created
        prop_per_obj: // maximum number of properties added to any one object
        depth:        // maximum recursion depth of any expression
      }
    }

Since the actual error produced may depend on the degree of static anaysis
performed by an implementation, the structure lists which errors *may* occur
rather than mandating a specific outcome.  However, the tests that are designed
to hit a RESOURCE_LIMIT should always hit that limit, and the rest should never
hit a limit.

An implementation which does *no* static analysis and interprets the JGraft
structure on the fly should always expect to produce the `output` or if one is
specified for the test, or NO_MATCH if that is a possibility.  Otherwise it
should fail with one of the permitted errors.  An implementation which does
extensive static analysis should aim to detect the first available of
INVALID_GRAFT, INVALID_OPERAND, INVALID_TARGET, or NO_MATCH, and only match
`output` when no errors are permitted.  Test suites for advanced implementations
should not fail overall if a new test for a semantic error is introduced which
the engine detects as a lesser error; this merely indicates that more static
analysis *could* be done, not that it must be done.

The `needProto` flag indicates when a test requires use of the `__proto__`
property.  Some implementations (e.g. JavaScript environments with old JSON
modules) reject this property and may need to skip over these tests.  In these
files, it is either true or omitted entirely, since the most common value is
false.

The top level of each file is structured as:

    { version:      // version of JGraft required
      name:         // short phrase describing collection
      desc:         // describes what the collection have in common
      tests: [      // one or more test structures
        ...
      ]
    }
