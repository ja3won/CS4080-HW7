Each expression uses its own constant pool. `OP_CONSTANT` takes a pool index; the comments show the value being loaded. `OP_RETURN` ends the chunk.

For `1 * 2 + 3`

```text
OP_CONSTANT 0    // 1
OP_CONSTANT 1    // 2
OP_MULTIPLY
OP_CONSTANT 2    // 3
OP_ADD
```

For `1 + 2 * 3`

```text
OP_CONSTANT 0    // 1
OP_CONSTANT 1    // 2
OP_CONSTANT 2    // 3
OP_MULTIPLY
OP_ADD
```

For `3 - 2 - 1`

```text
OP_CONSTANT 0    // 3
OP_CONSTANT 1    // 2
OP_SUBTRACT
OP_CONSTANT 2    // 1
OP_SUBTRACT
```

Subtraction goes left to right, so this evaluates `(3 - 2) - 1`.

For `1 + 2 * 3 - 4 / -5`

```text
OP_CONSTANT 0    // 1
OP_CONSTANT 1    // 2
OP_CONSTANT 2    // 3
OP_MULTIPLY
OP_ADD
OP_CONSTANT 3    // 4
OP_CONSTANT 4    // 5
OP_NEGATE
OP_DIVIDE
OP_SUBTRACT
```

`OP_NEGATE` turns the loaded `5` into `-5` before division.
