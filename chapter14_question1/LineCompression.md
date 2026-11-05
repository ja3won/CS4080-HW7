The original code stores a line number for every byte. Instead,store a line number only when it changes, along with the byte offset where it starts. `getLine()` uses these entries to find a byte's line number.

Update `chunk.h`. Add this structure before `Chunk`:

```c
typedef struct {
  int line;
  int offset;
} LineRun;
```

Replace `Chunk` with this version. The line entries have their own count and capacity because there may be fewer of them than bytes.

```c
typedef struct {
  int count;
  int capacity;
  uint8_t* code;

  int lineCount;
  int lineCapacity;
  LineRun* lines;

  ValueArray constants;
} Chunk;
```

Add the following declaration beside the other function declarations:

```c
int getLine(Chunk* chunk, int instruction);
```

Update `chunk.c`. Keep the includes and `addConstant()`. Replace the other functions with these:

```c
void initChunk(Chunk* chunk) {
  chunk->count = 0;
  chunk->capacity = 0;
  chunk->code = NULL;

  chunk->lineCount = 0;
  chunk->lineCapacity = 0;
  chunk->lines = NULL;

  initValueArray(&chunk->constants);
}

void freeChunk(Chunk* chunk) {
  FREE_ARRAY(uint8_t, chunk->code, chunk->capacity);
  FREE_ARRAY(LineRun, chunk->lines, chunk->lineCapacity);
  freeValueArray(&chunk->constants);
  initChunk(chunk);
}

void writeChunk(Chunk* chunk, uint8_t byte, int line) {
  if (chunk->capacity < chunk->count + 1) {
    int oldCapacity = chunk->capacity;
    chunk->capacity = GROW_CAPACITY(oldCapacity);
    chunk->code = GROW_ARRAY(uint8_t, chunk->code,
        oldCapacity, chunk->capacity);
  }

  // Record a start only when the source line changes.
  if (chunk->lineCount == 0 ||
      chunk->lines[chunk->lineCount - 1].line != line) {
    if (chunk->lineCount == chunk->lineCapacity) {
      int oldCapacity = chunk->lineCapacity;
      chunk->lineCapacity = GROW_CAPACITY(oldCapacity);
      chunk->lines = GROW_ARRAY(LineRun, chunk->lines,
          oldCapacity, chunk->lineCapacity);
    }

    chunk->lines[chunk->lineCount].line = line;
    chunk->lines[chunk->lineCount].offset = chunk->count;
    chunk->lineCount++;
  }

  chunk->code[chunk->count++] = byte;
}

int getLine(Chunk* chunk, int instruction) {
  if (instruction < 0 || instruction >= chunk->count) {
    return -1;
  }

  int left = 0;
  int right = chunk->lineCount - 1;
  int line = -1;

  while (left <= right) {
    int middle = left + (right - left) / 2;

    if (chunk->lines[middle].offset > instruction) {
      right = middle - 1;
    } else {
      // This run could contain the byte. Check for a later start.
      line = chunk->lines[middle].line;
      left = middle + 1;
    }
  }

  return line;
}
```

`writeChunk()` adds an entry when the line changes. `getLine()` uses binary search to find the entry for a byte offset, and returns `-1` for invalid offsets.

Since line numbers are no longer stored by byte offset, update their readers to call `getLine()`.

In `debug.c`, replace the line-printing block at the start of `disassembleInstruction()` with:

```c
int line = getLine(chunk, offset);
if (offset > 0 && line == getLine(chunk, offset - 1)) {
  printf("   | ");
} else {
  printf("%4d ", line);
}
```