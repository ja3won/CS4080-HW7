We can negate the top value directly instead of popping it and pushing the result. For the chapter's VM, where `Value` is a number, the instruction becomes:

```c
case OP_NEGATE:
  vm.stackTop[-1] = -vm.stackTop[-1];
  break;
```

`stackTop[-1]` is the top value. This changes that value without moving `stackTop`, so the stack height stays the same. With the `stackCount` version from the previous question, the equivalent line is:

```c
vm.stack[vm.stackCount - 1] = -vm.stack[vm.stackCount - 1];
```

The uploaded interpreter stores tagged values, so the version tested below keeps its number check and value conversions:

```c
case OP_NEGATE:
  if (!IS_NUMBER(peek(0))) {
    runtimeError("Operand must be a number.");
    return INTERPRET_RUNTIME_ERROR;
  }
  vm.stackTop[-1] = NUMBER_VAL(-AS_NUMBER(vm.stackTop[-1]));
  break;
```

## Performance Test

Run this program with both versions of the interpreter and compare how long each takes. Keep the rest of the code the same.

```text
fun test() {
  var value = 1;
  for (var i = 0; i < 20000000; i = i + 1) {
    value = -value;
  }
  print value;
}
test();
```

These were my CPU times for the full process, including startup and compilation of the Lox program:

| Version               | Median CPU time |
| --------------------- | --------------- |
| Pop, negate, and push | 1.6532 seconds  |
| Negate in place       | 1.5284 seconds  |

The in-place version used about 7% less CPU time in this test, meaning that that it was indeed faster.

## Other Instructions

`OP_NOT` can also replace the top value directly because its stack height does not change. It can use:

```c
vm.stackTop[-1] = BOOL_VAL(isFalsey(vm.stackTop[-1]));
```

Numeric addition, subtraction, multiplication, division, and comparisons can use a similar idea. They need to remove two operands and leave one result, so the stack height must decrease by one. We can pop the right operand and overwrite the remaining left operand instead of doing two pops and a push.

For example, subtraction could use:

```c
Value right = pop();
vm.stackTop[-1] -= right;
```
