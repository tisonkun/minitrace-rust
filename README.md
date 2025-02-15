> [!WARNING]
> `minitrace` has been renamed to [`fastrace`](https://github.com/fastracelabs/fastrace). Please follow the migration guide to update your code. **This repository is no longer maintained.**

Edit your `Cargo.toml` and find and replace `minitrace` with `fastrace` in the source code:

```diff
# Cargo.toml

[dependencies]
- minitrace = "0.6"
+ fastrace = "0.6"
```

<hr>

<div align="center">

  ![minitrace: Extremely fast tracing library for Rust](https://raw.githubusercontent.com/tikv/minitrace-rust/master/etc/img/head-img-640.svg)

  [![Crates.io](https://img.shields.io/crates/v/minitrace.svg?style=flat-square&logo=rust)](https://crates.io/crates/minitrace)
  [![Documentation](https://img.shields.io/docsrs/minitrace?style=flat-square&logo=rust)](https://docs.rs/minitrace/)
  [![CI Status](https://img.shields.io/github/actions/workflow/status/tikv/minitrace-rust/ci.yml?style=flat-square&logo=github)](https://github.com/tikv/minitrace-rust/actions)
  [![Coverage](https://img.shields.io/coveralls/github/tikv/minitrace-rust?style=flat-square)](https://coveralls.io/github/tikv/minitrace-rust?branch=master)
  [![License](https://img.shields.io/crates/l/minitrace?style=flat-square)](https://github.com/tikv/minitrace-rust/blob/master/LICENSE)

</div>
<br>

minitrace is a tracing library [10~100x faster](#benchmarks) than others:

![benchmark](https://raw.githubusercontent.com/tikv/minitrace-rust/master/etc/img/head-benchmark.svg)

Features:

- Easy to use
- [Extremely fast](#benchmarks)
- [Library-level tracing](#what-is-library-level-tracing)
- [Compatible with the log crate and its macros](minitrace/examples/log.rs)
- Compatible with [Jaeger], [Datadog], and [OpenTelemetry]

## Resources

- [Docs]
- [Examples]
- [FAQ](#faq)

## Getting Started

## In Libraries

Libraries should include `minitrace` as a dependency without enabling any extra features.

```toml
[dependencies]
minitrace = "0.6"
```

Add a `trace` attribute to the function you want to trace. In this example, a `SpanRecord` will be collected every time the function is called, if a tracing context is set up by the caller.

```rust
#[minitrace::trace]
pub fn send_request(req: HttpRequest) -> Result<(), Error> {
    // ...
}
```

Libraries are able to set up an individual tracing context, regardless of whether the caller has set up a tracing context or not. This can be achieved by using `Span::root()` to start a new trace and `Span::set_local_parent()` to set up a local context for the current thread.

The `full_name!()` macro can detect the function's full name, which is used as the name of the root span.

```rust
use minitrace::prelude::*;

pub fn send_request(req: HttpRequest) -> Result<(), Error> {
    let root = Span::root(full_name!(), SpanContext::random());
    let _guard = root.set_local_parent();

    // ...
}
```

## In Applications

Applications should include `minitrace` as a dependency with the `enable` feature set. To disable `minitrace` statically, simply remove the `enable` feature.

```toml
[dependencies]
minitrace = { version = "0.6", features = ["enable"] }
```

Applications should initialize a `Reporter` implementation early in the program's runtime. Span records generated before the reporter is initialized will be ignored. Before terminating, `flush()` should be called to ensure all collected span records are reported.

When the root span is dropped, all of its children spans and itself will be reported at once. Since that, it's recommended to create root spans for short tasks, such as handling a request, just like the example below. Otherwise, an endingless trace will never be reported.

```rust
use minitrace::collector::Config;
use minitrace::collector::ConsoleReporter;
use minitrace::prelude::*;

fn main() {
    minitrace::set_reporter(ConsoleReporter, Config::default());

    loop {
        let root = Span::root("worker-loop", SpanContext::random());
        let _guard = root.set_local_parent();

        handle_request();
    }

    minitrace::flush();
}
```

## Benchmarks

**By different architectures:**

![Benchmark result by architecture](etc/img/benchmark-arch.svg)

|                      | x86-64 (Intel Broadwell) | x86-64 (Intel Skylake) | x86-64 (AMD Zen) | ARM (AWS Graviton2) |
|----------------------|--------------------------|------------------------|------------------|---------------------|
| tokio-tracing        | 124x slower              | 33x slower             | 36x slower       | 29x slower          |
| rustracing           | 45x slower               | 10x slower             | 11x slower       | 9x slower           |
| minitrace (baseline) | 1x (3.4us)               | 1x (3.2us)             | 1x (3.8us)       | 1x (4.2us)          |

**By creating different number of spans:**

![Benchmark result by number of spans](etc/img/benchmark-spans.svg)

|                      | 1 span      | 10 spans   | 100 spans   | 1000 spans  |
|----------------------|-------------|------------|-------------|-------------|
| tokio-tracing        | 19x slower  | 61x slower | 124x slower | 151x slower |
| rustracing           | 13x slower  | 26x slower | 45x slower  | 55x slower  |
| minitrace (baseline) | 1x (0.4us)  | 1x (0.8us) | 1x (3.4us)  | 1x (27.8us) |

Detailed results are available in [etc/benchmark-result](etc/benchmark-result).

## Projects using minitrace

Feel free to open a PR and add your projects here:

- [TiKV](https://github.com/tikv/tikv): A distributed transactional key-value database
- [Conductor](https://github.com/the-guild-org/conductor): Open-source GraphQL Gateway
- [Apache OpenDAL](https://github.com/apache/opendal): A data access layer for various storage
- [Databend](https://github.com/datafuselabs/databend): Cost-Effective alternative to Snowflake
- [foyer](https://github.com/mrcroxx/foyer): Hybrid in-memory and disk cache in Rust

## FAQ

### Why is minitrace so fast?

There are some articles posted by the maintainer of minitrace:

- [The Design of A High-performance Tracing Library in Rust (Chinese)](https://www.youtube.com/watch?v=8xTaxC1RcXE)
- [How We Trace a KV Database with Less than 5% Performance Impact](https://en.pingcap.com/blog/how-we-trace-a-kv-database-with-less-than-5-percent-performance-impact/)

### What is library-level tracing?

Library-level tracing refers to the capability of incorporating tracing capabilities directly within libraries, as opposed to restricting them to application-level or system-level tracing.

Tracing can introduce overhead to a program's execution. While this is generally acceptable at the application level, where the added overhead is often insignificant compared to the overall execution time, it can be more problematic at the library level. Here, functions may be invoked frequently or performance may be critical, and the overhead from tracing can become substantial. As a result, tracing libraries not designed with speed and efficiency in mind may not be suitable for library-level tracing.

In the realm of the minitrace library, library-level tracing is engineered to be fast and lightweight, resulting in zero overhead when it's not activated. This makes minitrace an excellent choice for use in performance-sensitive applications, and it can be seamlessly integrated into libraries in a similar fashion to the log crate, something other tracing libraries may not offer.

### How does minitrace differ from other tracing libraries?

While many tracing libraries aim for extensive features, minitrace prioritizes performance and simplicity.

For example, minitrace doesn't introduce new logging macros, e.g. `info!()` or `error!()`, but seamlessly integrates with the [`log`](https://crates.io/crates/log) crate. This allows you to use existing logging macros and dependencies, with logs automatically attached to the current tracing span.

### Will minitrace incorporate 'level' for spans?

The concept of 'level' may not be an optimal feature for tracing systems. While `tokio-tracing` incorporates this feature, the underlying motivation for having levels in a span primarily revolves around performance. More specifically, it relates to the performance implications of tracing elements that are not of interest. However, tracing differs from logging in two key aspects: 

1. Disregarding a low-level span might inadvertently discard a high-level child span. 
2. The process of filtering, or 'level' as it's often called, in a tracing system should be applied to a trace as a whole rather than individual spans within a trace. 

In this context, minitrace offers a more efficient solution by filtering out entire traces that are not of interest through its unique tail-sampling design. Therefore, the concept of 'level', borrowed directly from logging systems, may not be suitable for minitrace.

### Will minitrace support OpenTelemetry feature 'X'?

minitrace is focused on high performance tracing only. You can open an issue for the missing tracing features you want to have.

Note that we always prioritize performance over features, so that not all tracing feature requests may be accepted. 

### What's the status of this library?

**API Unstable**: The API is not stabilized yet, may be changed in the future. 

**Code base Tested**: minitrace has been tested with high coverage. However, applications utilizing minitrace have not been widely deployed, so that minitrace is currently **NOT** regarded as battle-tested. 

[Docs]: https://docs.rs/minitrace/
[Examples]: minitrace/examples
[OpenTelemetry]: https://opentelemetry.io/
[Jaeger]: https://crates.io/crates/minitrace-jaeger
[Datadog]: https://crates.io/crates/minitrace-datadog
