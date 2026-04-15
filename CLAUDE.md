# bytepack

## Ownership

**Owns:**

- Binary serialization/deserialization library (header-only, vendored third-party)
- `binary_stream<BufferEndian>` template + `buffer_view` wrapper
- All concepts/type traits (SerializableBuffer, NetworkSerializable*, IntegralType, etc.)
- Endianness control (default: `std::endian::big` for network byte order)
- Catch2-based test suite

**Does NOT Own:**

- codex project build/integration decisions (how bytepack is consumed)
- upstream bytepack maintenance (this is vendored; any patches are local-only)
- LSP framing strategy (linter_lsp_server decides how to use bytepack)
- MCP transport logic (codexx_mcp_server decides integration)

## Invariants (NEVER violate)

1. `binary_stream` is non-copyable, non-movable (enforced via deleted copy/move constructors in original source)
2. `buffer_view` is always non-owning; lifetime of underlying data is caller's responsibility
3. `write()`/`read()` return `bool` (not exception-based); false = buffer overflow or size violation
4. Endianness conversion uses `std::ranges::reverse` byte-by-byte (not `hton`/`ntoh`); see TODO comment in code for performance note
5. Vector/dynamic container writes ALWAYS prefix with size field (even for empty containers) to allow deserializer to detect emptiness
6. All public APIs are `noexcept`; no exceptions thrown
7. No preprocessor macros in public API
8. C++20 standard (not C++23); codex's C++23 features do NOT apply inside bytepack

## Architectural Constraints

**Dependency Rules:**

- bytepack has ZERO external dependencies (not even abseil; vendored standalone)
- codex includes bytepack as INTERFACE library; no transitive dependencies injected
- Tests use vendored Catch2 (extern/Catch2/), NOT GTest

**Design Patterns:**

- Non-owning buffer pattern (buffer_view delegates lifetime to caller)
- Concept-based polymorphism (avoids virtual dispatch)
- Template-driven endianness selection (compile-time, zero runtime cost)
- Variadic `write()`/`read()` for bulk operations

## Modification Rules

**Safe to Change:**

- Test cases (CMake option `-DBYTEPACK_BUILD_TESTS=ON` controls build)
- Documentation in `doc/` directory
- Examples in `examples/` (not used by codex)
- Local patches to bytepack.hpp (preserve upstream intent; document deviation)

**Requires Coordination:**

- Changes to `binary_stream<BufferEndian>` API signature (consumed by LSP/MCP servers)
- Changes to endianness default or reversal strategy (network interoperability risk)
- Changes to `buffer_view` memory model

**Almost Never Change:**

- Core serialization logic in `binary_stream::write()`/`read()` overloads (load-bearing)
- Size-prefixing strategy for vectors (protocol-breaking)
- `noexcept` guarantees (used by protocol handlers that assume no exceptions)

## Common Pitfalls

- **Buffer overflow silently returns false.** Client code MUST check return values; do NOT assume write succeeded.
- **Endianness swap cost.** Byte-by-byte reverse has performance penalty (see TODO comment at line ~219). Use little-endian template param only when sender/receiver both native-endian.
- **Size prefix assumes `std::uint32_t`.** Default vector write uses 4-byte size field. If communicating with systems expecting 2-byte or 8-byte sizes, use explicit SizeType template param: `stream.write<std::uint16_t>(vec)`.
- **Empty vector still writes 4 bytes.** Size field is mandatory even for empty containers; deserializer cannot omit reading it.
- **No validation of deserialized data.** Client code receiving untrusted network data must validate (e.g., string lengths) after `read()` succeeds.
- **`buffer_view` does NOT extend buffer lifetime.** Passing temporary buffer to `binary_stream(buffer_view)` constructor creates dangling reference.

## How Claude Should Help

**Expected Tasks:**

- Integration points: help consuming code (LSP server, MCP server) use `binary_stream` correctly
- Test additions: new Catch2 test cases for specific serialization scenarios
- Documentation: clarify API usage, endianness semantics, size-prefix behavior
- Local patches: if upstream bytepack behavior conflicts with codex protocol needs, document deviation + rationale

**Conservative Approach Required:**

- ANY change to `write()`/`read()` method signatures (breaking change for protocol consumers)
- Endianness default modifications (network interoperability breakage)
- Removal of `noexcept` guarantees (affects error handling contracts downstream)
- Bulk code generation via Gemini (bytepack is small, hand-review all changes)

## Notes

- License: MIT (Faruk Eryilmaz, 2023-2024)
- Upstream: https://github.com/farukeryilmaz/bytepack
- This package is intentionally isolated; changes should NOT affect other codex packages unless explicitly coordinating with LSP/MCP server owners
