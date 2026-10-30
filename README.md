# Document-Compressor

A lossless file compressor and decompressor written in C99, built on the
LZ78 dictionary compression algorithm. It ships as two command line
programs: `encode` compresses a file and `decode` restores it byte for byte.

All source lives in [`LZ-78Compression/`](LZ-78Compression).

## Building and usage

```sh
cd LZ-78Compression
make            # builds ./encode and ./decode (native arm64 on Apple Silicon)
make clean      # removes binaries, object files and infer output
make format     # runs clang-format on all .c/.h files
make infer      # runs the Infer static analyzer
```

Both programs take the same flags. Input defaults to stdin and output
defaults to stdout, so they can be used in pipes.

| Flag        | Meaning                                                  |
| ----------- | -------------------------------------------------------- |
| `-i <file>` | File to read (defaults to stdin)                         |
| `-o <file>` | File to write (defaults to stdout)                       |
| `-v`        | Print compressed size, uncompressed size and ratio to stderr |

```sh
./encode -v -i notes.txt -o notes.lz
./decode -v -i notes.lz  -o notes_restored.txt

cat notes.txt | ./encode | ./decode > copy.txt
```

## How LZ78 works here

LZ78 builds a dictionary of phrases while it reads the input. Every phrase
is an earlier phrase plus one new byte, so each phrase can be written out as
a **(code, symbol)** pair: "the phrase with this code, followed by this
byte". The decoder rebuilds the same dictionary from the pairs it reads, so
the dictionary itself never has to be stored in the output.

**Encoder** (`encode.c`):

1. Start at the root of the trie (the empty phrase, `EMPTY_CODE`).
2. Read a byte. If the current node has a child for that byte, step into it
   and keep reading, because the phrase is still a known one.
3. If there is no child, the phrase so far plus this byte is new. Emit
   `(current node's code, byte)`, add the byte as a new child with the next
   free code, and go back to the root.
4. When codes run out (`MAX_CODE`), reset the trie and start over.
5. At end of input, flush any partly matched phrase, then emit a
   `STOP_CODE` pair.

**Decoder** (`decode.c`):

1. Read a `(code, symbol)` pair.
2. The new word is `table[code]` with `symbol` appended. Store it at
   `table[next_code]` and write it to the output.
3. Reset the table at the same point the encoder reset its trie, and stop
   when `STOP_CODE` is read.

## Data structures and why they were chosen

The two sides of the algorithm need opposite lookups, so each side uses a
different structure:

- The **encoder** asks "given this phrase and the next byte, is there a
  longer known phrase?" That is a prefix search, so it uses a **trie**.
- The **decoder** asks "what bytes does code N stand for?" That is a lookup
  by integer key, so it uses a **flat array indexed by code**.

### Trie (`trie.h`, `trie.c`), the encoder's dictionary

```c
#define ALPHABET 256

struct TrieNode {
  TrieNode *children[ALPHABET];
  uint16_t code;
};
```

Each node is one phrase. The path from the root spells the phrase, and
`code` is the number that phrase is sent as.

Why a trie:

- **Shared prefixes match how LZ78 builds phrases.** Every new phrase is an
  existing phrase plus one byte, so it is always exactly one child below a
  node that already exists. A trie stores each prefix once, and adding a
  phrase is a single node allocation.
- **Matching is incremental.** The encoder never re-scans or hashes the
  whole phrase. It keeps a pointer to the current node and takes one step
  per input byte. Each step is O(1), so encoding is O(n) in the input size.
- **Direct-indexed children.** `children` is a fixed array of 256 pointers,
  one for every possible byte value, so `trie_step` is a single array
  index (`n->children[sym]`) with no searching, hashing or comparing. Any
  binary input works, not only text. The trade-off is memory: each node
  holds 256 pointers (about 2 KB on a 64-bit system) even if most are
  `NULL`. That was accepted because the trie is capped at `MAX_CODE`
  nodes and speed and simplicity were the priority. A hash map or sorted
  child list would use less memory but make each step slower and the code
  more complex.
- **Cheap reset.** `trie_reset` frees every subtree under the root but
  keeps the root, so the dictionary can start over without reallocating
  its entry point.

### Word and WordTable (`word.h`, `word.c`), the decoder's dictionary

```c
typedef struct Word {
  uint8_t *syms;   // the bytes of the phrase
  uint32_t len;    // number of bytes
} Word;

typedef Word *WordTable;   // array of MAX_CODE Word pointers
```

A `Word` is a phrase stored as a length-prefixed byte array. A `WordTable`
is an array of `Word *` where the index **is** the code.

Why this pairing:

- **O(1) lookup by code.** Codes are dense integers from `START_CODE` up to
  `MAX_CODE`, so an array indexed by code is the simplest and fastest
  possible map. No hashing or tree walking is needed.
- **Fixed size is known in advance.** Codes are 16 bits, so the table never
  needs more than `MAX_CODE` (65535) slots. It is allocated once with
  `calloc`, which also sets every unused slot to `NULL`.
- **Explicit length, not a C string.** `Word` stores `len` next to the
  bytes instead of relying on a `'\0'` terminator. Compressed files can
  contain any byte, including zero, so a terminator would break binary
  input.
- **Whole words ready to write.** Each entry holds the full phrase, so
  writing a decoded phrase is a single copy into the output buffer.
  The alternative, storing only `(parent code, last byte)` and walking back
  to rebuild each phrase, uses less memory but needs extra work per output
  phrase. Storing whole words keeps decoding simple and fast.
- **Slot `EMPTY_CODE` holds the empty word**, so `word_append_sym` can
  build single-byte phrases with the same code path as longer ones.

### BitVector (`bv.h`, `bv.c`), bit-level packing

```c
typedef struct BitVector {
  uint8_t *vector;   // packed bits, 8 per byte
  uint32_t length;   // number of bits
} BitVector;
```

A `BitVector` is an array of bytes addressed one bit at a time. Bit `i` is
in byte `i / 8` at position `7 - (i % 8)`, and `bv_set_bit`, `bv_clr_bit`
and `bv_get_bit` read or change it with shifts and masks.

Why it is needed: the codes written to the file are **variable width**.
The encoder writes each code using only as many bits as the largest code
handed out so far needs (`blength(next_code)`, roughly
`floor(log2(next_code)) + 1`). Early in a file a code may need only 2 or 3
bits, and it never needs more than 16. Those widths do not line up with
byte boundaries, so pairs have to be packed bit by bit. Rounding every code
up to 16 bits would waste a lot of space, especially at the start of a
file.

A `BitVector` gives exact control over each bit with very little overhead:
a single byte array plus a counter, with no per-bit allocation.

### I/O buffers (`io.c`)

All file access goes through fixed `BLOCK`-sized (4096 byte) buffers
instead of one `read`/`write` system call per byte or per pair:

| Buffer    | Type                     | Used by | Purpose |
| --------- | ------------------------ | ------- | ------- |
| `read_sym` buffer | `uint8_t[4096]`  | encode  | Reads input a block at a time and hands out one byte per call |
| `buffer`  | `BitVector` (4096 × 8 bits) | encode | Collects packed `(code, symbol)` bits and writes a block when full |
| `rbuffer` | `BitVector` (4096 × 8 bits) | decode | Reads packed bits a block at a time for `read_pair` to unpack |
| `wbuff`   | `uint8_t[4096]`          | decode  | Collects decoded word bytes and writes a block when full |

Why: system calls are expensive compared to copying into memory, and the
algorithm naturally works one byte or one pair at a time. Buffering turns
millions of tiny calls into a few large ones. 4096 bytes matches a typical
filesystem block and memory page size. The buffers are `static` inside
`io.c`, so callers just call `read_sym`, `buffer_pair`, `read_pair` and
`buffer_word` and never deal with buffer state. `flush_pairs` and
`flush_words` write whatever partial block is left at the end.

### FileHeader (`io.h`)

```c
#define MAGIC 0x8badbeef

typedef struct FileHeader {
  uint32_t magic;        // identifies a file made by encode
  uint16_t protection;   // original file's permission bits
} FileHeader;
```

The header is the first thing written to every compressed file.

- `magic` lets `decode` reject input that was not produced by `encode`
  ("Incorrect magic number.") instead of producing garbage.
- `protection` records the original file's `st_mode` so the permissions
  can be restored.

The header is always stored **little endian**. On big endian machines
`read_header` and `write_header` swap the bytes with the helpers in
`endian.h`, so a file compressed on one machine decodes correctly on
another.

### Endianness helpers (`endian.h`)

`is_big` and `is_little` detect the machine's byte order at runtime using a
`union` of a `uint16_t` and a two byte array: write `0xFF00` as the
integer, then check which byte comes first. `swap16`, `swap32` and `swap64`
reverse byte order with masks and shifts. They are `static inline` in the
header because they are tiny and called often, so there is no function call
overhead and no extra `.c` file.

### Reserved codes (`code.h`)

| Constant     | Value        | Meaning |
| ------------ | ------------ | ------- |
| `STOP_CODE`  | 0            | Marks the end of the encoded data |
| `EMPTY_CODE` | 1            | The empty phrase (trie root / table slot 1) |
| `START_CODE` | 2            | First code given to a real phrase |
| `MAX_CODE`   | `UINT16_MAX` | Upper limit; dictionary resets when reached |

Codes are `uint16_t`, which caps the dictionary at 65535 entries. This
bounds memory for both the trie and the word table, keeps the largest code
at 16 bits, and the reset at `MAX_CODE` lets very long inputs keep
compressing with a fresh dictionary instead of growing without limit.

## Compressed file format

```
+-----------------------+---------------------------------------------+
| FileHeader (6 bytes)  | packed pairs ...                  | STOP    |
| magic | protection    | [code: k bits][symbol: 8 bits] ...| pair    |
+-----------------------+---------------------------------------------+
```

- `k` is `blength(next_code)` and grows as the dictionary grows. The
  encoder and decoder compute it the same way, so the width never has to be
  stored.
- Bits within each code and symbol are written least significant bit
  first.
- The last byte is zero padded up to a byte boundary.

## Source map

| File          | Contents |
| ------------- | -------- |
| `encode.c`    | Compressor entry point and LZ78 encoding loop |
| `decode.c`    | Decompressor entry point and LZ78 decoding loop |
| `trie.c/.h`   | Trie used by the encoder |
| `word.c/.h`   | `Word` and `WordTable` used by the decoder |
| `bv.c/.h`     | Bit vector used for bit-level packing |
| `io.c/.h`     | Buffered I/O, file header, pair and word read/write |
| `endian.h`    | Byte order detection and swapping |
| `code.h`      | Reserved code constants |
| `DESIGN.pdf`  | Original design document |
