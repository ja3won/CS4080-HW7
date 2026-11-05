The expression `4 - 3 * -2` is evaluated as `4 - (3 * (-2))`. Each sequence below uses its own constant pool. The numbers after `OP_CONSTANT` are pool indexes, and the comments show the values being loaded. `OP_RETURN` ends the chunk.

## Without OP_NEGATE

We can get `-2` by subtracting `2` from `0`. The expression becomes `4 - (3 * (0 - 2))`.

```text
OP_CONSTANT 0    // 4
OP_CONSTANT 1    // 3
OP_CONSTANT 2    // 0
OP_CONSTANT 3    // 2
OP_SUBTRACT
OP_MULTIPLY
OP_SUBTRACT
OP_RETURN
```

## Without OP_SUBTRACT

We can replace subtraction with addition of a negated value. The expression becomes `4 + (-(3 * (-2)))`.

```text
OP_CONSTANT 0    // 4
OP_CONSTANT 1    // 3
OP_CONSTANT 2    // 2
OP_NEGATE
OP_MULTIPLY
OP_NEGATE
OP_ADD
OP_RETURN
```

Both sequences produce `10`.

I think keeping both instructions makes sense because subtraction and negation are common operations. Separate instructions keep the bytecode shorter and reduce the number of instructions the interpreter has to execute. The extra opcode only adds a small amount of complexity. Other useful redundant instructions could load common constants like `0` and `1`. I would also consider `OP_INCREMENT` and `OP_DECREMENT`, since they could replace loading `1` and then adding or subtracting.
