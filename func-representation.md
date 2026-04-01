# FunC Representation of the Tolk Security Patterns in This Repository

This document provides a practical **FunC-first representation** of the Tolk security patterns described in this skill (`SKILL.md`, `tolk-security.md`, `tvm-async.md`).  
It is not a byte-for-byte compiler output; instead, it is a hand-written FunC model that preserves the same security intent.

---

## 1) Data layout (persistent state)

```func
;; Persistent storage schema:
;; owner          : MsgAddressInt
;; pending_owner  : MsgAddressInt? (stored as maybe-ref cell for simplicity)
;; seqno          : uint32
;; total_supply   : coins
;; stopped        : uint1
;; wallet_code    : cell

(int, cell, int, int, int, cell) load_data() impure inline {
  var ds = get_data().begin_parse();
  var owner = ds~load_msg_addr();
  var pending_owner = ds~load_maybe_ref();
  var seqno = ds~load_uint(32);
  var total_supply = ds~load_coins();
  var stopped = ds~load_uint(1);
  var wallet_code = ds~load_ref();
  ds.end_parse(); ;; strict parse, no trailing garbage
  return (owner, pending_owner, seqno, total_supply, stopped, wallet_code);
}

() save_data(int owner, cell pending_owner, int seqno, int total_supply, int stopped, cell wallet_code) impure inline {
  var b = begin_cell();
  b = b.store_slice(owner);
  b = b.store_maybe_ref(pending_owner);
  b = b.store_uint(seqno, 32);
  b = b.store_coins(total_supply);
  b = b.store_uint(stopped, 1);
  b = b.store_ref(wallet_code);
  set_data(b.end_cell());
}
```

### Why this matches the Tolk guidance
- Enforces explicit serialization/deserialization discipline.
- Uses strict `end_parse()` to avoid hidden trailing data injection.
- Keeps critical auth and replay fields (`owner`, `seqno`) in root state.

---

## 2) Opcodes and error codes

```func
const int OP_TRANSFER_OWNERSHIP = 0x1001;
const int OP_ACCEPT_OWNERSHIP   = 0x1002;
const int OP_MINT               = 0x1003;
const int OP_SET_STOPPED        = 0x1004;
const int OP_WITHDRAW_TON       = 0x1005;

const int ERR_UNAUTHORIZED      = 401;
const int ERR_BAD_SEQNO         = 402;
const int ERR_EXPIRED           = 403;
const int ERR_BAD_BODY          = 404;
const int ERR_STOPPED           = 405;
const int ERR_OVERFLOW          = 406;
```

---

## 3) Common guards (auth, replay, gas-aware structure)

```func
() require(int cond, int code) impure inline {
  if (~ cond) {
    throw(code);
  }
}

() ensure_owner(slice sender, int owner) impure inline {
  require(equal_slices(sender, owner), ERR_UNAUTHORIZED);
}

() ensure_not_stopped(int stopped) impure inline {
  require(stopped == 0, ERR_STOPPED);
}
```

---

## 4) External entrypoint (signature/seqno/expiry)

```func
() recv_external(slice in_msg_body) impure {
  ;; Typical external flow: [subwallet_id|valid_until|seqno|op|...]
  ;; (Signature check omitted here for brevity; implement with CHKSIGNU pattern.)

  var (owner, pending_owner, seqno, total_supply, stopped, wallet_code) = load_data();

  var valid_until = in_msg_body~load_uint(32);
  var msg_seqno = in_msg_body~load_uint(32);

  require(valid_until >= now(), ERR_EXPIRED);
  require(msg_seqno == seqno, ERR_BAD_SEQNO);

  ;; Increment seqno before side effects to harden replay race windows.
  seqno += 1;
  save_data(owner, pending_owner, seqno, total_supply, stopped, wallet_code);

  ;; parse op and dispatch
  var op = in_msg_body~load_uint(32);
  ;; dispatch logic can mirror recv_internal handlers or route to helper fns
}
```

---

## 5) Internal entrypoint with exhaustive opcode handling

```func
() recv_internal(int msg_value, cell in_msg_full, slice in_msg_body) impure {
  if (in_msg_body.slice_empty?()) {
    return (); ;; no-op
  }

  var cs = in_msg_full.begin_parse();
  var flags = cs~load_uint(4);
  var bounced = (flags & 1);
  var sender = cs~load_msg_addr();

  if (bounced) {
    on_bounce(in_msg_body);
    return ();
  }

  var (owner, pending_owner, seqno, total_supply, stopped, wallet_code) = load_data();

  var op = in_msg_body~load_uint(32);
  var query_id = in_msg_body~load_uint(64);

  if (op == OP_TRANSFER_OWNERSHIP) {
    ensure_owner(sender, owner);
    var new_owner = in_msg_body~load_msg_addr();
    in_msg_body.end_parse();

    pending_owner = begin_cell().store_slice(new_owner).end_cell();
    save_data(owner, pending_owner, seqno, total_supply, stopped, wallet_code);
    return ();
  }

  if (op == OP_ACCEPT_OWNERSHIP) {
    require(~ null?(pending_owner), ERR_UNAUTHORIZED);
    var po = pending_owner.begin_parse();
    var expected = po~load_msg_addr();
    po.end_parse();

    require(equal_slices(sender, expected), ERR_UNAUTHORIZED);
    owner = expected;
    pending_owner = null();
    save_data(owner, pending_owner, seqno, total_supply, stopped, wallet_code);
    return ();
  }

  if (op == OP_SET_STOPPED) {
    ensure_owner(sender, owner);
    var v = in_msg_body~load_uint(1);
    in_msg_body.end_parse();
    stopped = v;
    save_data(owner, pending_owner, seqno, total_supply, stopped, wallet_code);
    return ();
  }

  if (op == OP_MINT) {
    ensure_owner(sender, owner);
    ensure_not_stopped(stopped);

    var to = in_msg_body~load_msg_addr();
    var amount = in_msg_body~load_coins();
    var fwd_ton = in_msg_body~load_coins();
    in_msg_body.end_parse();

    ;; Overflow check before state update (representation of Tolk safe arithmetic rule)
    var new_supply = total_supply + amount;
    require(new_supply >= total_supply, ERR_OVERFLOW);

    ;; Debit/credit ordering in async systems: update state first.
    total_supply = new_supply;
    save_data(owner, pending_owner, seqno, total_supply, stopped, wallet_code);

    send_mint(to, amount, fwd_ton, query_id, wallet_code);
    return ();
  }

  if (op == OP_WITHDRAW_TON) {
    ensure_owner(sender, owner);
    var to = in_msg_body~load_msg_addr();
    var amount = in_msg_body~load_coins();
    in_msg_body.end_parse();

    send_ton(to, amount, query_id);
    return ();
  }

  ;; Exhaustive-by-policy: unknown ops fail loudly (no silent catch-all)
  throw(ERR_BAD_BODY);
}
```

---

## 6) Message sending helpers (mode hygiene)

```func
() send_ton(slice to, int amount, int query_id) impure inline {
  var body = begin_cell()
    .store_uint(0xd53276db, 32) ;; excess opcode pattern
    .store_uint(query_id, 64)
    .end_cell();

  var msg = begin_cell()
    .store_uint(0x18, 6)         ;; int msg info (nobounce here by design)
    .store_slice(to)
    .store_coins(amount)
    .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
    .store_ref(body)
    .end_cell();

  ;; Example: mode 3 = pay fees separately + ignore action errors.
  ;; Use intentionally and only where silent action error is acceptable.
  send_raw_message(msg, 3);
}

() send_mint(slice to, int amount, int fwd_ton, int query_id, cell wallet_code) impure {
  ;; Simplified mint transfer body; in production include full jetton wallet format.
  var body = begin_cell()
    .store_uint(OP_MINT, 32)
    .store_uint(query_id, 64)
    .store_coins(amount)
    .end_cell();

  var msg = begin_cell()
    .store_uint(0x18, 6)
    .store_slice(to)
    .store_coins(fwd_ton)
    .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
    .store_ref(body)
    .end_cell();

  send_raw_message(msg, 1); ;; separated fees, bounceable behavior by header flags
}
```

---

## 7) Bounce handler (state recovery)

```func
() on_bounce(slice b) impure {
  ;; Legacy bounced body starts with 0xFFFFFFFF then original op/query prefix.
  ;; Rich-bounce style handling should parse full returned body when available.

  var (owner, pending_owner, seqno, total_supply, stopped, wallet_code) = load_data();

  var bounced_prefix = b~load_uint(32);
  if (bounced_prefix != 0xffffffff) {
    return (); ;; unknown bounce format
  }

  var op = b~load_uint(32);
  var query_id = b~load_uint(64);

  if (op == OP_MINT) {
    var amount = b~load_coins();
    ;; Recover optimistic supply increment.
    require(total_supply >= amount, ERR_OVERFLOW);
    total_supply -= amount;
    save_data(owner, pending_owner, seqno, total_supply, stopped, wallet_code);
    return ();
  }
}
```

---

## 8) Mapping from Tolk patterns to FunC idioms

- **`assert(sender == owner)` in Tolk** → explicit `ensure_owner(sender, owner)` helper in FunC.
- **`assertEndAfterReading()`** → `slice.end_parse()` after each decode path.
- **Exhaustive `match` in Tolk** → strict opcode dispatch ending in `throw(ERR_BAD_BODY)`.
- **Avoid unsafe nullable unwrap `!`** → represent optional values as maybe-ref + explicit `null?` checks.
- **Carry-value + async safety** → update local state before outbound messages; restore in `on_bounce`.
- **Replay protection for external messages** → explicit `seqno` and `valid_until` checks.

---

## 9) Notes for production hardening

1. Add full external signature verification (`CHKSIGNU`) and subwallet-id checks.
2. Use precise message headers for bounce modes and fees based on your protocol.
3. Reserve balance before expensive action-phase sends where needed.
4. Keep opcode space documented and versioned.
5. Add invariant tests in sandbox for every bounce path and action-phase failure mode.

