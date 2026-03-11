# hash-registers — CLAUDE.md

## Purpose

Layer 3 of Ka-Boost. Wraps the `hash` module's `hashenv` template to provide **named, symbolic access to FANUC robot registers**. Rather than using hardcoded register numbers (R[3], PR[7], DO[109]) throughout robot programs, consumers populate a hash table once with `name → {type, id}` mappings, then read/write by name at runtime. Also writes the symbolic names back to the controller as native register comments via `hashr__set_comments`, making the teach pendant display self-documenting.

The module has a single compiled Karel program (`hashreg.pc`) with module-level global state holding the active hash table's location. Only one table is active at a time per controller context.

---

## Repository Layout

```
lib/hash-registers/
├── package.json              -- rossum manifest: source=lib/hashreg.kl, depends=Hash+registers
├── readme.md                 -- original docs (TP+ env workflow only — now superseded)
├── lib/
│   ├── hashreg.kl            -- single compiled program: all public routines + hSetCmts
│   └── hashreg.klh           -- public header: 14 declarations + short aliases
└── test/
    ├── test_hashreg.kl       -- minimal test: 10 SR[n] mappings, set_comments, set_string
    └── tppenv.kl             -- production deployment: 302-register environment, CMOS storage
```

The hash table array itself is **not** declared inside `hashreg.kl`. It lives in a separate calling program (e.g. `tppenv`, `env`). `hashreg.kl` accesses it at runtime via Karel's `BYNAME` built-in using the `tblProg`/`tblName` globals set by `hashr__set_hash_table`.

---

## Key Types

### `T_ENV` (from `hashenv.klt` in `lib/hash/types/`)

```
T_ENV = STRUCTURE
  typ : SHORT    -- register type code (DATA_REG, DATA_POSREG, DATA_STRING, io_dout, etc.)
  id  : SHORT    -- register number or port ID
ENDSTRUCTURE
```

`hval_def` = `T_ENV`. `hashname` = `h_env`. `HSH_KEY_SIZE` = 24 (longer than other hash configs to accommodate names like `Cellio_OP2_buzzer`).

### Register Type Codes (from `register_types.klt` + FANUC `kliotyps.kl`)

| Constant | Value | Register type |
|----------|-------|--------------|
| `DATA_REG` | 191 | Numeric register R[n] |
| `DATA_POSREG` | 192 | Position register PR[n] |
| `DATA_STRING` | 193 | String register SR[n] |
| `INVALIDTYPE` | 17043 | Sentinel / null value |
| `io_dout` | (from kliotyps.kl) | Digital output DO[n] |
| `io_din` | (from kliotyps.kl) | Digital input DI[n] |
| `io_anout` | (from kliotyps.kl) | Analog output AO[n] |
| `io_anin` | (from kliotyps.kl) | Analog input AI[n] |
| `io_flag` | (from kliotyps.kl) | Flag F[n] |
| `io_uopout` | (from kliotyps.kl) | User operator output UO[n] |
| `io_uopin` | (from kliotyps.kl) | User operator input UI[n] |

`kliotyps.kl` is a FANUC Karel system include — not in Ka-Boost source but always available on the controller.

---

## Full API Reference

All routines are in `hashreg.kl` / declared in `hashreg.klh`. Namespace: `hashr__*`. Short aliases listed where they differ.

### State Management

```
hashr__set_hash_table(progName : STRING; tableName : STRING)   [alias: sttbl]
  -- Store progName/tableName as module globals. Must be called before any get/set/comment.
  -- progName: Karel program hosting the ARRAY[N] OF hashname variable
  -- tableName: the variable name in that program
  -- Only one table is active globally — subsequent calls overwrite the previous pair.

hashr__nullenv : hval_def                                       [alias: nlenv]
  -- Return a T_ENV with typ=INVALIDTYPE, id=0. Use as the clrData argument to
  -- hashenv__clear_table to zero-initialize the hash array.
```

### Setup / Teardown

```
hashr__set_comments                                             [alias: stcmt]
  -- Walk every occupied slot in the hash array and write tbl[i].key as the
  -- native comment for the corresponding FANUC register using:
  --   SET_REG_CMT(id, key, status)   -- for DATA_REG
  --   SET_PREG_CMT(id, key, status)  -- for DATA_POSREG
  --   SET_SREG_CMT(id, key, status)  -- for DATA_STRING
  --   SET_PORT_CMT(typ, id, key, status)  -- for all I/O types
  -- The register comments persist on the controller and appear on the teach pendant.

hashr__clear_registers(typ : INTEGER; reset : BOOLEAN)          [alias: clreg]
  -- Delegate to registers__clear_comments(typ, reset) for the specified register type.
  -- typ=0 clears ALL types (DATA_REG, DATA_POSREG, DATA_STRING, io_flag).
  -- typ=DATA_REG, DATA_POSREG, DATA_STRING, or io_flag clears only that type.
  -- reset=TRUE: also resets the register values, not just comments.
  -- Typically called before repopulating the hash to ensure a clean state.
```

### Setters (write to FANUC register by name)

Each setter: looks up `name` in the hash → retrieves `{typ, id}` → calls the appropriate `registers__set_*`. Silently does nothing if name not in hash.

```
hashr__set_int(name : STRING; val : INTEGER)      [alias: stint]
  -- hashenv__get(name) → registers__set_int(reg.id, val)
  -- Writes to R[reg.id] as INTEGER.

hashr__set_real(name : STRING; val : REAL)        [alias: strl]
  -- hashenv__get(name) → registers__set_real(reg.id, val)
  -- Writes to R[reg.id] as REAL.

hashr__set_string(name : STRING; val : STRING)    [alias: stst]
  -- hashenv__get(name) → registers__set_string(reg.id, val)
  -- Writes to SR[reg.id].

hashr__set_io(name : STRING; val : BOOLEAN)       [alias: stio]
  -- hashenv__get(name) → registers__set_io(reg.typ, reg.id, val)
  -- Works for all I/O types (DO, DI, AO, AI, flag, etc.) — typ is passed through.

hashr__set_boolean(name : STRING; val : BOOLEAN)  [alias: setbol]
  -- hashenv__get(name) → registers__set_boolean(reg.id, val)
  -- Writes to F[reg.id] (flag register).
```

### Getters (read from FANUC register by name)

Each getter: looks up `name` → retrieves `{typ, id}` → reads from the register. Returns zero/empty/FALSE on hash miss.

```
hashr__get_int(name : STRING) : INTEGER           [alias: gtint]
  -- Returns registers__get_int(reg.id). Returns 0 if name not found.

hashr__get_real(name : STRING) : REAL             [alias: gtrel]
  -- Returns registers__get_real(reg.id). Returns 0 if name not found.

hashr__get_string(name : STRING) : STRING         [alias: gtstr]
  -- Returns registers__get_string(reg.id). Returns '' if name not found.

hashr__get_io(name : STRING) : BOOLEAN            [alias: gtio]
  -- Returns TRUE if registers__get_io(reg.typ, reg.id) = 1. Returns FALSE on miss or 0.

hashr__get_boolean(name : STRING) : BOOLEAN       [alias: gtbol]
  -- Returns registers__get_boolean(reg.id). Returns FALSE if name not found.
```

### Private

```
hSetCmts(tbl : ARRAY OF hashname)
  -- Internal implementation of hashr__set_comments. Iterates tbl, dispatches to
  -- SET_REG_CMT / SET_PREG_CMT / SET_SREG_CMT / SET_PORT_CMT by tbl[i].val.typ.
```

---

## Core Patterns

### Pattern 1 — Minimal setup: populate + comment + use

The standard three-phase workflow used in both test files:

```
-- Phase 1: Declare and configure the hash table (in the application program)
%include register_types.klt
%include hashenv.klt
hash_type_define(hashenv)
%undef hash_type_define
t_hash(hashname, hval_def, hashenv)

VAR
  tbl    : ARRAY[20] OF hashname   -- table lives here, in 'myapp'
  reg    : hval_def
  b      : BOOLEAN

%include hashreg.klh
%class hashenv('hash.klc', 'hashclass.klh', 'hashenv.klt')

BEGIN
  -- Phase 1: configure state
  hashr__set_hash_table('myapp', 'tbl')

  -- Phase 2: clear + populate hash
  reg = hashr__nullenv
  b = hashenv__clear_table('myapp', 'tbl', reg)

  reg.typ = DATA_REG ; reg.id = 1
  b = hashenv__put('part_count', reg, 'myapp', 'tbl')

  reg.typ = DATA_POSREG ; reg.id = 3
  b = hashenv__put('pick_pose', reg, 'myapp', 'tbl')

  reg.typ = io_dout ; reg.id = 5
  b = hashenv__put('gripper_open', reg, 'myapp', 'tbl')

  -- Phase 3: write names as register comments (one-time setup)
  hashr__set_comments

  -- Runtime use
  hashr__set_int('part_count', 0)
  hashr__set_io('gripper_open', TRUE)
  count = hashr__get_int('part_count')
END myapp
```

### Pattern 2 — Production environment file (CMOS persistence)

`tppenv.kl` pattern: large hash in CMOS, runs once as a one-shot setup program:

```
%define HASH_SIZE 302
%define HASH_PROGRAM 'tppenv'
%define HASHTABLE 'tbl'

VAR
  tbl IN CMOS : ARRAY[HASH_SIZE] OF hashname   -- persists across power cycles
  tblProg : STRING[16]
  tblName : STRING[16]

BEGIN
  IF UNINIT(tblProg) THEN tblProg = HASH_PROGRAM ; ENDIF
  IF UNINIT(tblName) THEN tblName = HASHTABLE ; ENDIF

  hashr__set_hash_table(tblProg, tblName)

  -- Clear all register types
  hashr__clear_registers(DATA_REG, TRUE)
  hashr__clear_registers(DATA_POSREG, TRUE)
  hashr__clear_registers(DATA_STRING, TRUE)
  hashr__clear_registers(io_flag, TRUE)

  -- Clear hash
  reg = hashr__nullenv
  b = hashenv__clear_table(tblProg, tblName, reg)

  -- Populate (302 entries)
  reg.typ = io_dout ; reg.id = 109
  b = hashenv__put('Cellio_t1', reg, tblProg, tblName)
  -- ... 301 more ...

  -- Label FANUC registers
  hashr__set_comments
END tppenv
```

This program runs once after deploying to the controller, writes all register comments, then can be deleted to reclaim memory. The hash itself persists in CMOS.

### Pattern 3 — Calling from a runtime program (separate from setup)

After the setup program has run and populated the CMOS hash, runtime programs reference the same table by name without re-populating:

```
-- In runtime program 'robot_main' — after 'tppenv' has been run:
%include hashreg.klh   -- just the header, no %class needed

BEGIN
  hashr__set_hash_table('tppenv', 'tbl')   -- point to the CMOS hash

  -- Now use by name
  speed = hashr__get_real('move_speed')
  hashr__set_io('gripper_open', TRUE)
  IF hashr__get_boolean('e_stop') THEN
    ABORT
  ENDIF
END robot_main
```

Note: `hashreg.pc` must be loaded on the controller, and `%include hashreg.klh` + `%include hashreg.kl` (or `FROM hashreg` declarations) must be present.

### Pattern 4 — Mixed register types in one hash

The `typ` field enables a single hash to cover all register categories:

```
-- I/O
reg.typ = io_dout ; reg.id = 109 ; b = hashenv__put('valve_1', reg, prog, tbl)
reg.typ = io_din  ; reg.id = 11  ; b = hashenv__put('sensor_a', reg, prog, tbl)
reg.typ = io_anout; reg.id = 1   ; b = hashenv__put('laser_pwr', reg, prog, tbl)
reg.typ = io_flag ; reg.id = 5   ; b = hashenv__put('dry_run', reg, prog, tbl)

-- Registers
reg.typ = DATA_REG    ; reg.id = 1  ; b = hashenv__put('cycle_count', reg, prog, tbl)
reg.typ = DATA_POSREG ; reg.id = 4  ; b = hashenv__put('home_pos', reg, prog, tbl)
reg.typ = DATA_STRING ; reg.id = 1  ; b = hashenv__put('job_id', reg, prog, tbl)

-- Getters and setters dispatch correctly by stored typ:
hashr__set_io('valve_1', TRUE)       -- DO[109] = ON
hashr__set_io('dry_run', FALSE)      -- F[5] = OFF (uses io_flag, dispatches correctly)
hashr__set_int('cycle_count', 0)     -- R[1] = 0
job = hashr__get_string('job_id')    -- SR[1]
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Calling any getter/setter before `hashr__set_hash_table` | `tblProg`/`tblName` are uninitialized; `hashenv__get` resolves via BYNAME to empty strings → `b = FALSE` → all setters silent no-op, all getters return 0/empty/FALSE | Always call `hashr__set_hash_table(progName, tableName)` first |
| Not calling `hashenv__clear_table` before populating | Stale UNINIT entries from previous runs cause incorrect probing; entries appear to be found when they are not | Call `hashenv__clear_table(prog, tbl, hashr__nullenv)` before every population run |
| Passing wrong `progName`/`tableName` to `set_hash_table` | BYNAME resolves to a different program or fails silently; all operations are no-ops or work on wrong data | Verify the program containing the `ARRAY[N] OF hashname` VAR is loaded and names match exactly |
| Using `hashr__set_io` for DATA_REG or DATA_POSREG | `registers__set_io(typ=DATA_REG, id, val)` likely does nothing or errors; io setters pass `reg.typ` through which must be an I/O type | Use `hashr__set_int` / `hashr__set_real` for R[n], `hashr__set_boolean` for F[n] |
| Using `hashr__set_boolean` for I/O | Calls `registers__set_boolean(reg.id, val)` which targets flag F[n], not DO/DI | Use `hashr__set_io` for all I/O port types |
| ARRAY[N] too small for number of entries | hasharray's static backend returns FALSE on `put` when full; entries are silently dropped; later `get` misses | Set `HASH_SIZE` to at least 1.5× the expected entry count to keep load factor below hasharray's implicit limit |
| Calling `hashr__set_comments` when table has never been populated | No-op (all keys are '' so nothing is written), but wastes a full array scan | Ensure `hashenv__clear_table` and population are done first |
| Missing `hash_type_define(hashenv)` + `%undef hash_type_define` in consumer | Karel compiler error if hashenv.klt is re-included elsewhere, or T_ENV is not defined in scope | Match the exact pattern from tppenv.kl: include, define, %class, %undef |

---

## Dependencies

**hash-registers depends on:**
- `ktransw-macros` — `namespace.m`, `define_type.m`, `t_hash`, `declare_function`
- `Hash` — `hashenv.klt` (T_ENV type + HSH_KEY_SIZE=24), `hash.klc` (static array backend), `hashclass.klh`
- `registers` — `registers__set_int/real/string/io/boolean`, `registers__get_int/real/string/io/boolean`, `registers__clear_comments`

**Modules that depend on hash-registers:**
- None in current Ka-Boost `lib/` — this is a leaf-layer module. Consumers are application programs deployed to the controller (e.g. `tppenv.kl`).

---

## Build / Integration Notes

- `package.json` has `"source": ["lib/hashreg.kl"]` — rossum compiles exactly one Karel program to `hashreg.pc`.
- Tests: `test/tppenv.kl` and `test/test_hashreg.kl`. Both are standalone programs, not KUnit tests — they run on the controller and produce observable side effects (register comments, register values). They cannot be run through `kunit?filenames=`.
- The `tppenv.kl` pattern is a **one-shot setup tool**, not a recurring test. After running it, delete `tppenv.pc` from the controller to free memory. The hash data persists in CMOS.
- In production deployment: the application program that hosts `tbl IN CMOS` is typically separate from both the setup program and runtime programs. `hashr__set_hash_table` links them at runtime by name.
- `%define HASH_SIZE` must be set before the `VAR tbl : ARRAY[HASH_SIZE] OF hashname` declaration. Karel evaluates this at compile time.
- The `hash.klc`/`hashclass.klh`/`hashenv.klt` triple is always inlined via `%class` — hashenv has no separate `.pc` file; its code is baked into whatever program calls `%class hashenv(...)`.
