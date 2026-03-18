# Stage 22: Testing & Documentation 🧪

> *Code without tests is just a hope. Code without docs is a mystery. Rust makes both easy.*

## What You'll Learn

- Writing unit tests with `#[test]`
- Test organization (modules, files)
- Integration tests in the `tests/` directory
- Documentation tests — tests embedded in your docs
- `#[should_panic]`, `#[ignore]`, test filtering
- Mocking strategies in Rust
- Benchmarking with `criterion`
- Generating beautiful documentation with `rustdoc`
- Code coverage tools

## Prerequisites

- Completed [Stage 21: Macros](../21-macros/)

## Tutorials

| # | Tutorial | Description |
|---|----------|-------------|
| 1 | [Unit Tests](./01-unit-tests.md) | #[test], assert!, assert_eq!, assert_ne!, test output |
| 2 | [Test Organization](./02-test-organization.md) | #[cfg(test)], test modules, helper functions |
| 3 | [Integration Tests](./03-integration-tests.md) | tests/ directory, testing public API, shared setup |
| 4 | [Documentation Tests](./04-doc-tests.md) | Tests in /// comments, # hiding lines, no_run |
| 5 | [Advanced Testing](./05-advanced-testing.md) | #[should_panic], #[ignore], custom test runners |
| 6 | [Mocking & Test Doubles](./06-mocking.md) | Traits for mocking, mockall crate, test strategies |
| 7 | [Benchmarking](./07-benchmarking.md) | criterion crate, statistical benchmarks, comparing impls |
| 8 | [Documentation with rustdoc](./08-rustdoc.md) | Generating docs, formatting, examples, badges |

## By the End of This Stage

You'll write comprehensive tests, generate professional documentation, and benchmark your code for performance.

---

**Previous:** [← Stage 21: Macros](../21-macros/) | **Next:** [Stage 23: Project — CLI Tool →](../23-project-cli/)
