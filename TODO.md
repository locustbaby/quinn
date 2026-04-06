# Quinn Fork TODO

## [TODO-1] Upstream: Fix tracing overhead in driver spawns

**Issue tracker**: #TODO-57358 (check upstream quinn-rs/quinn)
**Priority**: Medium
**Status**: Locally patched (brute-force delete), pending more elegant upstream fix

### Flamegraph 对比

- **修复前**（774a1d3，run 24031511238）：
  - Client: https://locustbaby.github.io/duotunnel/bench/flamegraphs/774a1d3-24031511238-client.svg
  - Server: https://locustbaby.github.io/duotunnel/bench/flamegraphs/774a1d3-24031511238-server.svg
  - `Instrumented<T> as Future>::poll` 占 **~37% CPU**

- **修复后**（dc4846a，run 24035021574）：
  - Server: https://locustbaby.github.io/duotunnel/bench/flamegraphs/dc4846a-24035021574-server.svg
  - `Instrumented` 帧彻底消失

### 问题描述

`quinn/src/connection.rs` 和 `quinn/src/endpoint.rs` 中，`ConnectionDriver` 和 `EndpointDriver` 被 `runtime.spawn` 时用了 `.instrument(Span::current())`：

```rust
// connection.rs (原始代码)
runtime.spawn(Box::pin(
    async { ... }.instrument(Span::current()),
));

// endpoint.rs (原始代码)
runtime.spawn(Box::pin(
    async { ... }.instrument(Span::current()),
));
```

以及 `ConnectionDriver::poll` 每次调用都无条件创建 `debug_span!`：

```rust
// connection.rs ConnectionDriver::poll (原始代码)
let span = debug_span!("drive", id = conn.handle.0);
let _guard = span.enter();
```

### 为什么有开销

1. **`Span::current()`** 每次 spawn 时做 thread-local 查询（`dispatcher::get_default`），如果当前有 active span 还会 `clone_span`，创建真实的 `Inner` 对象。`Instrumented` wrapper 在每次 `Future::poll` 时都 `enter()`/`exit()` 这个 span。

2. **`debug_span!` 无 cfg guard**：workspace `Cargo.toml` 中 tracing 依赖是：
   ```toml
   tracing = { version = "0.1.10", default-features = false, features = ["std"] }
   ```
   没有 `release_max_level_warn`，导致 `debug_span!` 在 release build 里仍是运行时求值，不会被编译期消除。

3. **实测影响**：8k QPS 压测下 `Instrumented<T> as Future>::poll` 占 **~37% CPU**，去掉后该帧彻底消失。

### 当前 patch（粗暴删除，branch: patch/no-hot-debug-span）

```rust
// connection.rs — 直接去掉 .instrument()
runtime.spawn(Box::pin(async {
    if let Err(e) = driver.await {
        tracing::error!("I/O error: {e}");
    }
}));

// endpoint.rs — 直接去掉 .instrument()
runtime.spawn(Box::pin(async {
    if let Err(e) = driver.await {
        tracing::error!("I/O error: {}", e);
    }
}));

// connection.rs ConnectionDriver::poll — 加 enabled! guard
let span = if tracing::enabled!(tracing::Level::DEBUG) {
    debug_span!("drive", id = conn.handle.0)
} else {
    tracing::Span::none()
};
let _guard = span.enter();
```

缺点：直接删除 `.instrument()` 意味着 driver task 不再继承调用者的 span 上下文，调试时日志无法关联到具体连接。

### 更优雅的修复方案

**方案 A（最小改动，推荐给上游）**：quinn 自己的 workspace `Cargo.toml` 加 `release_max_level_warn`

```toml
# quinn/Cargo.toml workspace dependencies
tracing = { version = "0.1.10", default-features = false, features = ["std", "release_max_level_warn"] }
```

效果：release build 里所有 `debug_span!`/`trace_span!` 被编译期静态消除为 `Span::none()`，`Instrumented::poll` 的 enter/exit 变成真正零成本。开发/debug build 保留完整 tracing。**不需要改任何 Rust 代码。**

> ⚠️ **重要**：`release_max_level_warn` 是 **per-crate** 编译的，只影响声明该 feature 的那个 crate 内部的 tracing 宏。在下游（如 tunnel-lib）加无效，**必须在 quinn 自己的 Cargo.toml 里加才能消除 quinn 内部的 debug_span! 开销**。这也是为什么这个修复只能作为上游 PR，下游用户无法自行解决。

**方案 B（feature gate，更灵活）**：在 quinn 的 `[features]` 里加一个 `tracing` feature

```toml
# quinn/Cargo.toml
[features]
tracing = []  # 开启后 driver 才继承 span 上下文
```

```rust
// connection.rs
let fut = async {
    if let Err(e) = driver.await {
        tracing::error!("I/O error: {e}");
    }
};
#[cfg(feature = "tracing")]
let fut = fut.instrument(Span::current());
runtime.spawn(Box::pin(fut));
```

用户可以按需开关，生产环境关闭 feature 获得零开销，开发环境开启获得完整 span 传播。

**方案 C（条件继承，最精细）**：只在 span enabled 时才 instrument

```rust
let parent = Span::current();
let fut = async {
    if let Err(e) = driver.await {
        tracing::error!("I/O error: {e}");
    }
};
let fut = if parent.is_disabled() {
    fut.instrument(Span::none())
} else {
    fut.instrument(parent)
};
runtime.spawn(Box::pin(fut));
```

运行时判断，有 active span 时才包装，无 span 时零开销。不需要改 feature，但需要类型统一（两个分支类型不同，需要 box 或宏封装）。

### 推荐给上游的方案

**方案 A**（改 workspace `Cargo.toml` 加 `release_max_level_warn`）最简单，一行改动，向后兼容，对所有用户透明。可作为 upstream PR 的核心改动，配合方案 C 的条件 instrument 作为补充。

### 上游现状分析（2026-04-06）

通过 `git log origin/main` 确认：

- `origin/main` 和 `quinn-0.11.9` **均保留** `.instrument(Span::current())`，没有任何修复
- `377af288`（rename log → tracing-log）只是 feature 改名，与 span 开销无关
- `ecae4ad1`（track callers in runtime spawn）只加了 `#[track_caller]`，无关
- workspace `Cargo.toml` 的 tracing 依赖至今仍是 `features = ["std"]`，无 `release_max_level_warn`

**结论**：这是真实存在且上游未修复的性能问题。

### 为什么下游加 `release_max_level_warn` 不行

`release_max_level_warn` 是 **per-crate 编译**的静态常量，只裁剪声明该 feature 的 crate 内部的 tracing 宏。在 tunnel-lib/client/server 的 Cargo.toml 里加，只影响这些 crate 自己的代码，**完全不影响 quinn 内部的 `debug_span!` 和 `Span::current()`**。必须在 quinn 自己的 workspace Cargo.toml 里加才有效，而这只能走上游 PR。

### 待办

- [ ] 搜索 quinn-rs/quinn issues（关键词：`Instrumented overhead`、`debug_span poll`、`tracing release_max_level`）
- [ ] 无相关 issue 则提新 issue，附上 flamegraph 链接作为证据
- [ ] PR 方向：workspace `Cargo.toml` 加 `release_max_level_warn` + 方案 C 的条件 instrument
- [ ] 上游合并后移除 tunnel 的 `[patch.crates-io]` 并升级 quinn 版本
