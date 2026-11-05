In `vm.h`, replace the fixed-size stack and `stackTop` with these fields:

```c
typedef struct {
  Chunk* chunk;
  uint8_t* ip;
  Value* stack;
  int stackCount;
  int stackCapacity;
} VM;
```

In `vm.c`, add `#include "memory.h"` with the existing includes so the array macros are available.

Resetting the stack only needs to set its count to zero. Initialize the pointer and capacity when creating the VM, and free the array when shutting it down.

```c
static void resetStack() {
  vm.stackCount = 0;
}

void initVM() {
  vm.stack = NULL;
  vm.stackCapacity = 0;
  resetStack();
}

void freeVM() {
  FREE_ARRAY(Value, vm.stack, vm.stackCapacity);
  vm.stack = NULL;
  vm.stackCapacity = 0;
  resetStack();
}
```

In `push()`, grow the array if it is full, then store the new value. `GROW_CAPACITY()` starts with a small capacity and doubles it for later growth.

```c
void push(Value value) {
  if (vm.stackCount == vm.stackCapacity) {
    int oldCapacity = vm.stackCapacity;
    vm.stackCapacity = GROW_CAPACITY(oldCapacity);
    vm.stack = GROW_ARRAY(Value, vm.stack,
        oldCapacity, vm.stackCapacity);
  }

  vm.stack[vm.stackCount++] = value;
}

Value pop() {
  return vm.stack[--vm.stackCount];
}
```

Popping reduces the count before reading the top value. Resetting and popping keep the allocated memory available for future pushes.

Update the stack-debugging loop to use the count:

```c
for (int i = 0; i < vm.stackCount; i++) {
  printf("[ ");
  printValue(vm.stack[i]);
  printf(" ]");
}
```

The main benefit is that the stack can grow instead of writing past a fixed-size array. Small programs also avoid reserving a large stack that they may never use. The downside is checking the capacity every time a value is pushed. Since pushes happen often, this adds runtime overhead. Growing the array may also require copying its values, and doubling the capacity can leave some unused space. Allocation can still fail if memory runs out. However, doubling makes growth less frequent, so pushes take constant time on average.
