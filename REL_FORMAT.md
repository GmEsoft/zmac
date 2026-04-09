# zmac `.REL` File Structure

> Reverse-engineered from `src/zmac.y`. This describes the **`.rel` stream that zmac writes** when `--rel` / `--rel7` is enabled; it is not intended to be a universal specification for every MACRO-80 style `.REL` producer.

## Source map

The relevant writer logic lives in these areas of `src/zmac.y`:

- `RELCMD_*` command IDs: around lines `807-821`
- low-level bit packing: `putrelbits()`, `putrel()`, `putrelname()`, `putrelsegref()`, `putrelcmd()` around lines `1446-1508`
- inline emission of bytes / relocatable words: `emit()` around lines `850-1110`
- extended link records: `can_extend_link()`, `extend_link()`, `putrelop()` around lines `9140-9211`
- file prologue / epilogue order: around lines `6948-7112`

---

## 1. High-level model

zmac writes `.REL` as a **packed bitstream**, not as a sequence of byte-aligned records.

`putrelbits()` appends fields **MSB-first** into a pending bit buffer. The writer only forces byte alignment in `flushrel()`, which zmac calls after `ENDMOD` and after `ENDPROG`.

### Primitive item layout

Each item starts with a 1-bit tag:

- `0` + `8 bits`  → a **literal byte** to place at the current location counter
- `1` + more bits → a **special item**

For special items, the next two bits determine the subtype:

- `1 ss lo hi`, where `ss != 0` → **relocatable 16-bit value** relative to segment `ss`
- `1 00 cccc ...` → **command** with 4-bit command number `cccc`

Because absolute values are usually emitted as ordinary bytes via `putrel()`, zmac effectively uses `1 00` as the command prefix.

### Endianness

Whenever zmac emits a 16-bit value, it writes:

1. low byte
2. high byte

So the payload is **little-endian inside the bitstream**, even though the bit packing itself is MSB-first.

---

## 2. Segment / scope codes

zmac uses the low two bits of `scope` as the segment code:

| Code | Symbol in `zmac.y` | Meaning |
|---|---|---|
| `0` | `SEG_ABS` | absolute / `ASEG` |
| `1` | `SEG_CODE` | code / program segment / `CSEG` |
| `2` | `SEG_DATA` | data segment / `DSEG` |
| `3` | `SEG_COMMON` | common block |

The same 2-bit encoding is used by:

- relocatable words
- `SETLOC`
- `DATASIZE`, `CODESIZE`, `COMSIZE`
- `PUBVALUE`
- `ENDMOD` entry-point reference
- external chain addresses (`putrelextaddr()`)

---

## 3. Basic encodings

### 3.1 Literal byte

Produced by `putrel(byte)`:

```text
0 bbbbbbbb
```

Meaning: store byte `bbbbbbbb` at the next location counter.

### 3.2 Relocatable 16-bit word

Produced directly inside `emit()` for segment-relative expressions that are still relocatable:

```text
1 ss llllllll hhhhhhhh
```

- `ss` = segment code
- `ll` = low byte of value
- `hh` = high byte of value

Example: `dw label` in `CSEG` becomes `1 01 lo hi`.

### 3.3 Command

Produced by `putrelcmd(relcmd)`:

```text
1 00 cccc
```

The command-specific payload follows immediately and is **not necessarily byte-aligned**.

### 3.4 Name

Produced by `putrelname(str)`:

```text
lll <char0> <char1> ...
```

- `lll` is a **3-bit length** (`0..7`)
- characters are emitted as 8-bit ASCII
- zmac uppercases names before writing them
- names are truncated to:
  - **6 chars** for `--rel`
  - **7 chars** for `--rel7`
  - **7 chars** in `--mras` mode

### 3.5 Segment/address reference

Produced by `putrelsegref(scope, addr)`:

```text
ss llllllll hhhhhhhh
```

Used as a compact payload in several commands.

---

## 4. Command set used by zmac

The command IDs are defined in `src/zmac.y` as follows:

| Code | Name | Payload written by zmac | Meaning |
|---|---|---|---|
| `0` | `RELCMD_PUBLIC` | `name` | declares a public symbol name |
| `1` | `RELCMD_COMMON` | `name` | selects / declares a common block name |
| `2` | `RELCMD_PROGNAME` | `name` | module/program name |
| `3` | `RELCMD_LIBLOOK` | *(not emitted by zmac)* | reserved in code table only |
| `4` | `RELCMD_EXTLINK` | see [Extended linking](#5-extended-linking-records) | link-time expression fragment |
| `5` | `RELCMD_COMSIZE` | `segref(abs,size)` + `name` | final size of a common block |
| `6` | `RELCMD_EXTCHAIN` | `extaddr` + `name` | head of an external fixup chain |
| `7` | `RELCMD_PUBVALUE` | `segref(symbol_segment,value)` + `name` | final value of a public symbol |
| `8` | `RELCMD_EXTMINUS` | `segref(abs,offset)` | upcoming external reference is `symbol - offset` |
| `9` | `RELCMD_EXTPLUS` | `segref(abs,offset)` | upcoming external reference is `symbol + offset` |
| `10` | `RELCMD_DATASIZE` | `segref(abs,size)` | size of data segment |
| `11` | `RELCMD_SETLOC` | `segref(segment,addr)` | set current segment location |
| `13` | `RELCMD_CODESIZE` | `segref(code,size)` | size of code segment |
| `14` | `RELCMD_ENDMOD` | `extaddr(entry)` | end of module, plus entry address if supplied |
| `15` | `RELCMD_ENDPROG` | none | end of `.rel` stream |

Notes:

- There is **no command 12** in zmac's table.
- `CODESIZE` is deliberately emitted as **code-relative**, while `DATASIZE` and `COMSIZE` are emitted as **absolute** values; the source comment says this follows `m80` behavior.

---

## 5. Extended linking records

zmac uses `RELCMD_EXTLINK` (`4`) to serialize link-time expressions that cannot be represented as a plain relocatable word.

The payload begins with a 3-bit length and then a small opcode packet.

### 5.1 Operator packet (`'A'`)

Produced by `putrelop(op)`:

```text
EXTLINK
len=2
'A' op
```

Operator codes from `zmac.y`:

| Code | Name | Meaning |
|---|---|---|
| `1` | `RELOP_BYTE` | coerce/finalize as byte |
| `2` | `RELOP_WORD` | coerce/finalize as word |
| `3` | `RELOP_HIGH` | high byte |
| `4` | `RELOP_LOW` | low byte |
| `5` | `RELOP_NOT` | bitwise NOT |
| `6` | `RELOP_NEG` | unary minus |
| `7` | `RELOP_SUB` | subtraction |
| `8` | `RELOP_ADD` | addition |
| `9` | `RELOP_MUL` | multiplication |
| `10` | `RELOP_DIV` | division |
| `11` | `RELOP_MOD` | modulus |

### 5.2 External symbol packet (`'B'`)

Produced by `extend_link()` when the node is a simple external symbol:

```text
EXTLINK
len = 1 + symbol_length
'B' name
```

zmac truncates the embedded external name to **6 characters** here.

### 5.3 Local relocatable value packet (`'C'`)

Produced by `extend_link()` for a relocatable non-external term:

```text
EXTLINK
len=4
'C' seg lo hi
```

This represents a segment-relative literal value.

### 5.4 Expression order

`extend_link()` emits expression trees in **post-order** (left, right, operator), which is effectively a **reverse-Polish** stream for the linker.

Examples of zmac's synthesis rules:

- `LOW(x)` or `x & 255` → emitted with `RELOP_LOW`
- `HIGH(x)` → emitted with `RELOP_HIGH`
- `x << n` → rewritten as `x (2^n) MUL`
- `x >> n` → rewritten as `x (2^n) DIV`

Only the operators listed above are link-extended. Other expressions are rejected as **not relocatable**.

---

## 6. External reference chains

zmac handles unresolved external references through a classic **fixup chain** stored in each symbol's `i_chain` field.

### How it works

When zmac emits a 16-bit reference to an external symbol:

1. it writes the **previous chain head** into the output word slot (or `0` if this is the first use),
2. it updates `i_chain` to the current reference location encoded as:
   - top 2 bits = segment
   - low 16 bits = address

At end of module, zmac emits:

```text
EXTCHAIN extaddr name
```

The linker can then follow the chain through the stored word values and patch every use site.

### External `+ constant` / `- constant`

For simple forms such as:

- `extern_symbol + 5`
- `extern_symbol - 2`

zmac emits one of:

- `RELCMD_EXTPLUS`
- `RELCMD_EXTMINUS`

followed by `segref(ABS, offset)`, and then the normal chain-linked word reference.

More complicated expressions fall back to `EXTLINK` records.

---

## 7. Actual output order produced by zmac

When `outpass` begins, zmac emits records in this order:

1. `PROGNAME module_name`
2. one `PUBLIC name` per public symbol
3. `DATASIZE abs,data_size`
4. `CODESIZE code,code_size`
5. one `COMSIZE abs,size + name` per common block

During code generation it then emits, as needed:

- `SETLOC segment,address` whenever zmac changes the active relocatable location
- `COMMON name` when switching to a common block
- inline bytes (`0 + byte`)
- inline relocatable words (`1 ss lo hi`)
- extended-link command packets for expressions

At the end of assembly it emits:

1. one `EXTCHAIN` per unresolved external
2. one `PUBVALUE` per public symbol
3. `ENDMOD entry`
4. byte-align (`flushrel()`)
5. `ENDPROG`
6. byte-align again

---

## 8. Practical parser outline

A reader for zmac-generated `.REL` can use this decoding loop:

```text
read 1 bit -> tag
if tag == 0:
    read 8 bits -> literal byte
else:
    read 2 bits -> ss
    if ss != 0:
        read 16 bits (lo, hi) -> relocatable word for segment ss
    else:
        read 4 bits -> command
        decode the command-specific payload
```

Important caveats:

- command payloads may begin in the middle of a byte
- names are uppercased and length-limited
- zmac documents what **it writes**, not necessarily every record another `.REL` producer might accept

---

## 9. zmac-specific semantics to keep in mind

- `.public`, `.global`, `.entry`, and `label::` feed `PUBLIC` / `PUBVALUE`.
- `.extern`, `.extrn`, `.ext` feed `EXTCHAIN` and external fixups.
- `.aseg`, `.cseg`, `.dseg`, and `.common` determine the segment codes written into the stream.
- `end label` sets the `ENDMOD` entry-point reference.
- Expressions marked `SCOPE_NORELOC` are rejected for `.rel` output unless zmac can serialize them through `EXTLINK`.

This is enough to implement a parser, dumper, or compatibility layer for the `.REL` output that zmac currently generates.
