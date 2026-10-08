# Rust core assessment

Assessment target: commit `a271a20f8434616c2737fd87d9210ce5fed0daf1`.

## Executive assessment

The parser is technically ambitious and contains strong algorithmic and
performance engineering. Its public Rust wrapper also uses several good Rust
patterns, including RAII ownership, tree-bound node lifetimes, mutable access
for parser and cursor operations, and streaming iterators for query results.

The implementation is not yet a sound modern-Rust foundation, however. The
safe API currently exposes multiple paths to undefined behavior. These are not
objections to the presence of `unsafe` itself: a parser with a C ABI, generated
grammar tables, compact arena storage, and pointer-based trees will require
unsafe code. The problem is that important arena, callback, and allocator
invariants are not encoded in types and are violated in several reachable
paths.

The highest-priority work is targeted safety hardening. The parsing algorithm
and overall component structure do not need to be rewritten.

## Critical safety issues

### 1. `Parser: Send` permits borrowed and non-`Send` loggers

`Logger<'a>` accepts `Box<dyn FnMut(LogType, &str) + 'a>` without a `Send`
bound (`lib/binding_rust/lib.rs:302`). `Parser::set_logger` stores the closure
behind a raw pointer (`lib/binding_rust/lib.rs:743-781`), while `Parser` is
unconditionally declared `Send` and `Sync` (`lib/binding_rust/lib.rs:3847-3848`).

Safe callers can therefore:

- store a closure borrowing a local value and retain it after that value dies;
- capture `Rc`, `RefCell`, or another thread-affine value;
- move the parser to another thread; and
- invoke or drop that value on the destination thread.

Both a borrowed-lifetime example and an `Rc` cross-thread example compile
through the public safe API. The former permits a use-after-free; the latter
violates the standard library's `Send` boundary and can cause a data race or
memory corruption.

Recommended repair: require a stored logger to be
`Box<dyn FnMut(LogType, &str) + Send + 'static>` if `Parser: Send` is required.
If borrowed or non-`Send` loggers must remain supported, parameterize `Parser`
by the callback lifetime and make thread transfer conditional. Reassess `Sync`
separately and document the proof beside every manual `Send`/`Sync`
implementation.

### 2. Scratch-node construction can use a stale arena after relocation

`subtree_new_scratch_node` receives a cached arena pointer, calls
`subtree_reuse_children`, and then constructs a handle using the original
arena (`lib/src_rust/subtree.rs:556-565`). Reserving the child/header storage
can grow the arena with `realloc` (`lib/src_rust/subtree/storage.rs:75-94,
790-803`). Its production caller snapshots the arena before this operation
(`lib/src_rust/parser/actions.rs:135-143`).

If `realloc` moves the arena, the newly allocated header is resolved against
the freed old base. Pointer subtraction then combines unrelated allocations,
and subsequent traversal can dereference a malformed handle. This path is
reachable through ordinary safe parsing.

Recommended repair: do not pass an arena snapshot into scratch-node
construction. Re-read the current arena after every operation capable of
allocation or relocation, and add a forced-relocation regression case at this
boundary.

### 3. Tree editing retains raw child pointers across moving allocations

Edit work items store `NonNull<Subtree>` pointers into arena-resident child
arrays (`lib/src_rust/subtree.rs:180-183` and
`lib/src_rust/subtree/edit.rs:119-170`). The edit loop can clone or allocate
while these pointers and an arena snapshot remain live
(`lib/src_rust/subtree/edit.rs:64-114,174-212`). Arena growth uses `realloc`.

After a relocation, queued child pointers and the destination pointer for the
current write can refer to freed storage. Safe `Tree::edit` can consequently
reach a dangling write, use-after-free, or memory corruption.

Recommended repair: represent edit work as stable arena-relative offsets or
as parent-handle/child-index paths. Resolve each item against the current arena
immediately before use and refresh the arena after every allocating call.

### 4. S-expression measurement performs out-of-bounds pointer arithmetic

`subtree_string` measures output using a one-byte stack buffer
(`lib/src_rust/subtree/debug.rs:229-248`). In measurement mode,
`subtree_write_to_string` repeatedly advances a pointer by each `snprintf`
return value (`lib/src_rust/subtree/debug.rs:52-226`). For any nontrivial
S-expression, this forms pointers beyond the allocation. Rust's pointer
arithmetic rules make that undefined behavior even when the pointers are not
dereferenced.

This is exposed through safe `Node::to_sexp` and `Display`
(`lib/binding_rust/lib.rs:1953-1962,2028-2031`).

Recommended repair: use an integer count during the measurement pass. Advance
an output pointer only during the real write pass, with an explicit remaining
capacity.

### 5. The active deallocator has two independently mutable owners

The runtime owns `ts_current_free` (`lib/src_rust/alloc.rs:51-100`), while the
Rust binding independently stores `FREE_FN`
(`lib/binding_rust/lib.rs:3803-3822`). Runtime-owned buffers are later freed
through the binding copy in `Tree::included_ranges`, `Node::to_sexp`, and
`CBufferIter::drop`.

Calling the public C `ts_set_allocator` symbol directly, including from a
mixed Rust/C process, updates the runtime hook without updating `FREE_FN`. A
buffer allocated by a custom allocator can then be released with libc, causing
allocator mismatch, corruption, or a crash.

Recommended repair: make the runtime allocator module the only owner of all
allocation hooks. Binding code should release runtime-owned buffers through
that owner instead of caching another free-function pointer.

## Modern Rust and maintainability issues

### Unsafe operations are not locally reviewable

The workspace uses Rust edition 2021 and does not enable
`unsafe_op_in_unsafe_fn`. Much of the core executes inside broad `unsafe fn`
bodies, so pointer dereferences, unchecked indexing, FFI calls, and fabricated
lifetimes do not require small explicit unsafe blocks. Only a small portion of
the unsafe API has complete `# Safety` contracts.

Recommended repair:

1. Enable `unsafe_op_in_unsafe_fn` as a warning, then as a denial.
2. Add complete pointer, lifetime, aliasing, ownership, and index preconditions
   to each exposed unsafe function.
3. Move individual unsafe operations into small blocks with local `SAFETY`
   explanations.
4. Introduce an arena context that couples a handle with its current storage
   domain and prevents raw arena snapshots from surviving allocation.

### `Array<T>` is a C/POD buffer presented as an unconstrained generic

`Array<T>` exposes its raw pointer, size, and capacity fields
(`lib/src_rust/utils.rs:44-48`). Safe slice methods trust those fields.
`grow_by` assumes the all-zero bit pattern is valid for `T`, allocation uses
unchecked byte-size arithmetic, and clear/delete/splice operations do not have
normal Rust destructor semantics.

Current uses appear confined to integers, pointers, FFI-style records, compact
handles, and manually managed nested arrays. No current misuse involving a
`String`, `Vec`, `Box`, `Rc`, or `Arc` element was found. The type still permits
future invalid-zero, over-aligned, zero-sized, or destructor-bearing uses.

Recommended repair: make the fields private, separate copyable/POD storage
from manually owned nested arrays, use checked capacity arithmetic, and
restrict zero-initializing growth through an explicit unsafe marker trait or a
specialized operation.

### Callback panic behavior is unsuitable for a safe Rust API

Logger, input, and progress callbacks are safe `FnMut` values invoked through
`extern "C"` trampolines. A panic cannot unwind normally through that boundary
and can abort the process. The surrounding code also mixes `C` and `C-unwind`
declarations, which obscures the intended policy.

Recommended repair: catch panics in the trampolines, retain the panic payload,
cancel and clean up the core operation, then resume the panic on the Rust side.
Alternatively, explicitly document an abort-on-panic policy, although that is
a poor fit for the current safe callback API.

### Recursive diagnostic traversal can exhaust the native stack

S-expression and DOT renderers recurse once per syntax-tree level
(`lib/src_rust/subtree/debug.rs`). Deep attacker-controlled syntax can overflow
the native stack when safe formatting or diagnostics are requested.

Recommended repair: use an explicit frame stack, preserving the current
public API without imposing a native-recursion limit.

### Representation knowledge is duplicated around unsafe code

Examples include:

- the low-bit subtree tag repeated in construction, masking, and immutable and
  mutable classification;
- four-bit inline geometry limits repeated in eligibility, packing, decoding,
  and mutation;
- the internal/leaf discriminator described incorrectly in safety comments;
- stack metrics repeated across `StackNode`, `WindowEntry`, and `StackHead`;
- compressed parse-table decoding implemented separately for terminal and
  nonterminal rows; and
- parser and query callback/option FFI bridges copied across entry points.

These copies can diverge while continuing to compile. Several directly govern
unsafe pointer interpretation.

Recommended repair: create named representation owners and shared typed
conversion/decoding helpers before further storage optimization.

### Query implementation remains a broad C-shaped port

`lib/src_rust/query.rs` is a close translation of the former C implementation.
Several functions span hundreds of lines and combine allocation, pointer
arithmetic, alias-sensitive array mutation, query construction, and cursor
state transitions inside broad unsafe regions. No separate current
mis-dereference was established in these functions, but the proof surface is
too large for reliable local review.

The module documentation is also stale: it says the Rust query port is
inactive and `query.c` is live, although the Rust functions now export the ABI
and `query.c` has been deleted.

Recommended repair: first correct the documentation and establish explicit
unsafe boundaries. Then split the largest routines around ownership-preserving
algorithmic phases. Avoid a wholesale algorithm rewrite.

## Relationship to upstream Tree-sitter 0.27

Commit `98de2bc1` ("feat: start working on v0.27") updates workspace package
versions and dependencies from 0.26.3 to 0.27.0. It does not import upstream
0.27's C runtime sources.

This repository had already removed the canonical C implementations:

- commit `8e2ca006` deleted `parser.c`, `subtree.c`, and `tree.c` after their
  Rust replacements were introduced;
- commit `ff58e76d` activated `query.rs` and deleted `query.c`.

The default crate build includes `lib/src_rust/mod.rs` as `core_impl` and
compiles `lib/src/lib.c`. That C file contains only
`lexer_log_shim.c`, retained because stable Rust cannot define the required C
variadic logging function. The build script has an optional
`TREE_SITTER_CORE_IMPL=c` mode, but it requires
`TREE_SITTER_C_CORE_SRC_DIR` to point at a separate pre-rewrite C source tree;
the canonical C runtime is not vendored in this checkout.

The version number therefore means that the fork's API, grammar tooling, and
crate set have been advanced to the 0.27 release line. It does not mean the
runtime is built from the upstream 0.27 C implementation.

## Prioritized remediation

1. Fix the logger lifetime/`Send` defect.
2. Fix both stale-arena relocation paths.
3. Remove undefined pointer arithmetic from S-expression formatting.
4. Establish one allocator/deallocator owner.
5. Enable and document explicit unsafe boundaries.
6. Harden `Array<T>` and couple subtree handles to their arena domain.
7. Define callback panic behavior and replace recursive diagnostics.
8. Consolidate duplicated representation and stack-state knowledge.
9. Correct stale query and migration documentation.

After the first five items, re-audit all changed ownership paths. Arena
relocation and callback ownership are cross-cutting invariants, so fixing one
call site without tracing every consumer could leave equivalent defects.

## Overall judgment

| Area | Assessment |
| --- | --- |
| Parser algorithm and performance engineering | Strong |
| Overall module architecture | Good |
| Public wrapper API design | Mostly good |
| Idiomatic modern Rust | Weak to moderate |
| Unsafe encapsulation | Weak |
| Current safe-API soundness | Unacceptable until the critical issues are fixed |
| Need for an algorithmic rewrite | No |
| Need for significant targeted cleanup | Yes |

The project can become a strong Rust-native Tree-sitter runtime without
discarding its algorithm or compact storage strategy. Its next development
phase should treat memory-safety proofs and type-enforced ownership as primary
requirements rather than follow-up cleanup.
