# hash-registers

Named symbolic access to FANUC robot registers, backed by a hash table.

## Overview

On a FANUC controller, registers are addressed by number: R[3], PR[7], DO[109]. Hardcoding those numbers throughout your Karel and TP programs makes code fragile and hard to read. This module lets you assign symbolic names to registers once, then read and write by name at runtime:

```
hashr__set_int('part_count', 0)
hashr__set_io('gripper_open', TRUE)
count = hashr__get_int('part_count')
```

It also writes the symbolic names back to the controller as native FANUC register comments via `hashr__set_comments` — so they appear on the teach pendant next to the register numbers.

The underlying storage is a `hashenv` hash table (from `lib/hash`) mapping `name → {type, id}`. The hash array can be stored in CMOS for persistence across power cycles.

---

## Files

| File | Purpose |
|------|---------|
| `lib/hashreg.kl` | Compiled Karel program — all public routines |
| `lib/hashreg.klh` | Header file — routine declarations + short aliases |
| `test/test_hashreg.kl` | Minimal test: 10 SR[n] mappings, set_comments, set_string |
| `test/tppenv.kl` | Production example: 302-register environment, CMOS storage |
| `package.json` | rossum manifest |

The hash table array itself is **not** declared in `hashreg.kl`. You declare it in your own program and point `hashreg` to it with `hashr__set_hash_table`.

---

## Register Type Constants

From `register_types.klt` (Ka-Boost) and `kliotyps.kl` (FANUC system include):

| Constant | Register |
|----------|---------|
| `DATA_REG` | Numeric register R[n] |
| `DATA_POSREG` | Position register PR[n] |
| `DATA_STRING` | String register SR[n] |
| `io_dout` | Digital output DO[n] |
| `io_din` | Digital input DI[n] |
| `io_anout` | Analog output AO[n] |
| `io_anin` | Analog input AI[n] |
| `io_flag` | Flag F[n] |
| `io_uopout` / `io_uopin` | User operator output/input |
| `INVALIDTYPE` | Sentinel / null — use for `hashr__nullenv` |

---

## API Reference

### State Management

```
hashr__set_hash_table(progName : STRING; tableName : STRING)
```
Point the module at the hash table array. `progName` is the Karel program containing the `ARRAY[N] OF hashname` variable; `tableName` is the variable name. Must be called before any other routine. Only one table is active at a time.

```
hashr__nullenv : hval_def
```
Returns a `T_ENV` with `typ = INVALIDTYPE, id = 0`. Use as the `clrData` argument to `hashenv__clear_table` when zeroing the array before population.

### Setup

```
hashr__clear_registers(typ : INTEGER; reset : BOOLEAN)
```
Clear register comments on the controller. `typ = 0` clears all types; pass `DATA_REG`, `DATA_POSREG`, `DATA_STRING`, or `io_flag` to clear only that type. `reset = TRUE` also resets the register values to zero/empty. Call this before repopulating the hash to ensure a clean state.

```
hashr__set_comments
```
Walk the hash array and write each entry's key string as the FANUC register comment for the corresponding register. Uses `SET_REG_CMT`, `SET_PREG_CMT`, `SET_SREG_CMT`, and `SET_PORT_CMT` built-ins. The comments persist on the controller and show on the teach pendant. Call this once after populating the hash.

### Setters

All setters: look up the name in the hash → get `{typ, id}` → write to the register. Silently do nothing if the name is not in the hash.

| Routine | Alias | Writes to |
|---------|-------|-----------|
| `hashr__set_int(name, val : INTEGER)` | `stint` | R[id] as integer |
| `hashr__set_real(name, val : REAL)` | `strl` | R[id] as real |
| `hashr__set_string(name, val : STRING)` | `stst` | SR[id] |
| `hashr__set_io(name, val : BOOLEAN)` | `stio` | I/O port of type `typ` at index `id` |
| `hashr__set_boolean(name, val : BOOLEAN)` | `setbol` | F[id] (flag register) |

### Getters

All getters: look up name → get `{typ, id}` → read from register. Return `0` / `''` / `FALSE` on hash miss.

| Routine | Alias | Reads from | Miss return |
|---------|-------|-----------|-------------|
| `hashr__get_int(name) : INTEGER` | `gtint` | R[id] | `0` |
| `hashr__get_real(name) : REAL` | `gtrel` | R[id] | `0` |
| `hashr__get_string(name) : STRING` | `gtstr` | SR[id] | `''` |
| `hashr__get_io(name) : BOOLEAN` | `gtio` | I/O port of type `typ` | `FALSE` |
| `hashr__get_boolean(name) : BOOLEAN` | `gtbol` | F[id] | `FALSE` |

---

## Common Patterns

### 1. Basic setup: populate, comment, use

This is the core workflow. Declare the hash table in your program, populate it once, call `set_comments`, then use getters/setters throughout.

```
-- In your Karel program (e.g. 'myapp')

-- TYPE section: set up hash types
%include register_types.klt
%include hashenv.klt
hash_type_define(hashenv)
%undef hash_type_define
t_hash(hashname, hval_def, hashenv)

VAR
  tbl : ARRAY[30] OF hashname    -- size >= 1.5x your entry count
  reg : hval_def
  b   : BOOLEAN

%include hashreg.klh
%class hashenv('hash.klc', 'hashclass.klh', 'hashenv.klt')

BEGIN
  -- Step 1: configure the module
  hashr__set_hash_table('myapp', 'tbl')

  -- Step 2: clear old state
  hashr__clear_registers(0, TRUE)       -- clear all register types
  reg = hashr__nullenv
  b = hashenv__clear_table('myapp', 'tbl', reg)

  -- Step 3: populate the hash
  reg.typ = DATA_REG    ; reg.id = 1 ; b = hashenv__put('part_count',  reg, 'myapp', 'tbl')
  reg.typ = DATA_POSREG ; reg.id = 3 ; b = hashenv__put('pick_pose',   reg, 'myapp', 'tbl')
  reg.typ = io_dout     ; reg.id = 5 ; b = hashenv__put('gripper_open',reg, 'myapp', 'tbl')
  reg.typ = io_flag     ; reg.id = 1 ; b = hashenv__put('dry_run',     reg, 'myapp', 'tbl')

  -- Step 4: write register comments to controller (one-time)
  hashr__set_comments

  -- Runtime use
  hashr__set_int('part_count', 0)
  hashr__set_io('gripper_open', TRUE)
  IF NOT hashr__get_boolean('dry_run') THEN
    -- execute motion
  ENDIF
END myapp
```

### 2. Production environment file (CMOS persistence)

For large systems, store the hash in CMOS non-volatile memory. Run the setup program once after deployment; it persists across power cycles.

```
-- Program 'tppenv' — run once on the controller, then delete to free memory

%define HASH_SIZE 302        -- size your array at least 1.5x the number of entries
%define HASH_PROGRAM 'tppenv'
%define HASHTABLE 'tbl'

VAR
  tbl IN CMOS : ARRAY[HASH_SIZE] OF hashname  -- CMOS = persists without power
  tblProg : STRING[16]
  tblName : STRING[16]
  reg : hval_def
  b   : BOOLEAN

BEGIN
  IF UNINIT(tblProg) THEN tblProg = HASH_PROGRAM ; ENDIF
  IF UNINIT(tblName) THEN tblName = HASHTABLE    ; ENDIF

  hashr__set_hash_table(tblProg, tblName)

  -- Clear everything
  hashr__clear_registers(DATA_REG, TRUE)
  hashr__clear_registers(DATA_POSREG, TRUE)
  hashr__clear_registers(DATA_STRING, TRUE)
  hashr__clear_registers(io_flag, TRUE)
  reg = hashr__nullenv
  b = hashenv__clear_table(tblProg, tblName, reg)

  -- Populate all register mappings
  reg.typ = io_dout ; reg.id = 109 ; b = hashenv__put('Cellio_t1',     reg, tblProg, tblName)
  reg.typ = io_din  ; reg.id = 11  ; b = hashenv__put('Cellio_OP1_reset', reg, tblProg, tblName)
  reg.typ = DATA_POSREG ; reg.id = 4 ; b = hashenv__put('Positioner_home', reg, tblProg, tblName)
  -- ... (all entries) ...

  hashr__set_comments
END tppenv
```

After running: delete `tppenv.pc` from the controller. The CMOS array (`tbl`) survives. Runtime programs point to it with `hashr__set_hash_table('tppenv', 'tbl')`.

### 3. Runtime program referencing a CMOS hash

Once the setup program has run, any runtime program can use the same table by name — no re-population needed.

```
-- In program 'robot_cycle' (runs every cycle)
%include hashreg.klh   -- just the header; hashreg.pc must be loaded on controller

BEGIN
  hashr__set_hash_table('tppenv', 'tbl')   -- point to the CMOS hash from tppenv

  IF hashr__get_io('e_stop') THEN ABORT ; ENDIF

  speed = hashr__get_real('move_speed')
  hashr__set_io('laser_enable', TRUE)
  hashr__set_int('cycle_count', hashr__get_int('cycle_count') + 1)
END robot_cycle
```

### 4. Clearing a specific register type only

Use `hashr__clear_registers` with a specific type code to reset only that category:

```
-- Reset only string registers (useful after job change)
hashr__clear_registers(DATA_STRING, TRUE)

-- Reset only flags
hashr__clear_registers(io_flag, TRUE)

-- Reset all (typ=0)
hashr__clear_registers(0, TRUE)
```

---

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Not calling `hashr__set_hash_table` first | `tblProg`/`tblName` are uninitialized; all getters return 0/empty/FALSE, all setters are silent no-ops | Always call `hashr__set_hash_table(prog, tbl)` before any other routine |
| Skipping `hashenv__clear_table` before population | Stale UNINIT entries cause incorrect hash probing; lookups may hit wrong entries | Always call `hashenv__clear_table(prog, tbl, hashr__nullenv)` before populating |
| Wrong program/variable names in `set_hash_table` | BYNAME fails silently; all operations work on wrong memory or do nothing | Verify the VAR name and program name match exactly and the program is loaded |
| Using `hashr__set_io` for DATA_REG | `registers__set_io` receives `typ=DATA_REG` — wrong dispatch, likely no-op or error | Use `hashr__set_int` or `hashr__set_real` for R[n] |
| Using `hashr__set_boolean` for DO[n] / DI[n] | `registers__set_boolean` targets F[n] (flag), not I/O ports | Use `hashr__set_io` for any I/O port (DO, DI, AO, AI, flag all work) |
| ARRAY too small for entry count | `hashenv__put` returns FALSE silently when the static array is full; entries are dropped without error | Set `HASH_SIZE` to at least 1.5× your total entries |
| Missing `%undef hash_type_define` after custom type | If `hashenv.klt` is re-included (e.g. from a second `%class`), the TYPE is emitted twice → compile error | Add `%undef hash_type_define` immediately after calling `hash_type_define(hashenv)` |

---

## Build Flow

This module compiles to a single `.pc` file (`hashreg.pc`). Add `"hash-registers"` to your module's `depends` in `package.json`.

```sh
cd lib/hash-registers
rossum .. -w -o -t    # generate build.ninja with tests
ninja                 # compile hashreg.kl → hashreg.pc, plus test programs
kpush                 # deploy to controller
```

Test programs (`test_hashreg.kl`, `tppenv.kl`) are standalone Karel programs — run them from the teach pendant, not via KUnit HTTP. They produce visible side effects (register comments and values) that you verify on the controller.

The hash implementation (`hash.klc` / `hashenv`) is compiled inline into each program that uses `%class hashenv(...)` — there is no separate `hashenv.pc`.

See the top-level [Ka-Boost readme](../../readme.md) for full build setup instructions.
