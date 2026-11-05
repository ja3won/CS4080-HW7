`OP_CONSTANT` can only store indexes up to 255. For bigger indexes, `OP_CONSTANT_LONG` stores the index in three bytes, starting with the low byte.

Update `chunk.h` by adding the new opcode at the end of the existing `OpCode` enum:

```c
  OP_METHOD,
  OP_CONSTANT_LONG
```

Also declare the new function:

```c
void writeConstant(Chunk* chunk, Value value, int line);
```

Add `writeConstant()` in `chunk.c`, `#include <assert.h>` with the includes, then add this function:

```c
void writeConstant(Chunk* chunk, Value value, int line) {
  assert(chunk->constants.count <= 0xffffff);
  int constant = addConstant(chunk, value);

  if (constant <= 255) {
    writeChunk(chunk, OP_CONSTANT, line);
    writeChunk(chunk, (uint8_t)constant, line);
    return;
  }

  // Write the bytes from low to high.
  writeChunk(chunk, OP_CONSTANT_LONG, line);
  writeChunk(chunk, (uint8_t)constant, line);
  writeChunk(chunk, (uint8_t)(constant >> 8), line);
  writeChunk(chunk, (uint8_t)(constant >> 16), line);
}
```

`addConstant()` saves the value and returns its index. Small indexes use `OP_CONSTANT`; bigger ones use `OP_CONSTANT_LONG`. The casts split the index into bytes.

For example, index 65,536 is written as `00 00 01`. The bytes all use the same source line.

Update the disassembler in `debug.c` by adding this helper near `constantInstruction()`:

```c
static int longConstantInstruction(const char* name, Chunk* chunk,
                                   int offset) {
  uint32_t index = chunk->code[offset + 1];
  index |= (uint32_t)chunk->code[offset + 2] << 8;
  index |= (uint32_t)chunk->code[offset + 3] << 16;
  printf("%-16s %4u '", name, (unsigned)index);
  printValue(chunk->constants.values[index]);
  printf("'\n");
  return offset + 4;
}
```

This combines the three bytes into one index. It returns `offset + 4` to skip the opcode and its three bytes.

Add this case to `disassembleInstruction()`:

```c
case OP_CONSTANT_LONG:
  return longConstantInstruction("OP_CONSTANT_LONG", chunk, offset);
```

The compiler and VM also need to handle the new instruction.

In `compiler.c`, replace `emitConstant()` with:

```c
static void emitConstant(Value value) {
  if (currentChunk()->constants.count > 0xffffff) {
    error("Too many constants in one chunk.");
    return;
  }

  writeConstant(currentChunk(), value, parser.previous.line);
}
```

The limit check makes sure the index fits in three bytes. The assertion checks the same limit inside `writeConstant()`.

In the execution switch in `vm.c`, add:

```c
case OP_CONSTANT_LONG: {
  uint32_t index = READ_BYTE();
  index |= (uint32_t)READ_BYTE() << 8;
  index |= (uint32_t)READ_BYTE() << 16;
  push(frame->closure->function->chunk.constants.values[index]);
  break;
}
```

The VM reads the bytes in the same order they were written. Each `READ_BYTE()` is separate so they are read in order.

Other instructions, like global-variable instructions, still use one-byte indexes.

## Sacrifices?

The downside is that the interpreter has another instruction to handle, and one opcode is used. Long loads also take more bytes than short loads.

Short loads take two bytes, and long loads take four. Long loads can use indexes up to 16,777,215.