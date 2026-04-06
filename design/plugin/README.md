# Plugin System

SDream uses a plugin architecture for all extensions: filters, device
providers, auto-converters, and transfer bridges. The plugin boundary is a
stable **C ABI** for maximum portability and FFI compatibility. A C++
header-only SDK wraps the C ABI with RAII, templates, and strong types for
ergonomic C++ plugin development.

A C++ header-only SDK wraps the C ABI with RAII, templates, and strong types
for ergonomic C++ plugin development.

## Sub-Pages

| Document | Contents |
|----------|----------|
| [ABI](abi.md) | C ABI surface, filter factory struct, parameter descriptors, versioning, discovery |
| [Cross-Language Support](cross-language.md) | State-machine step protocol, language scaffolding for C/Rust/Python/C# |
