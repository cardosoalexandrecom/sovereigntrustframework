# 🚀 STFX Runtime Layer

## 📖 Overview

**stfx-runtime** provides a runtime-agnostic abstraction for task spawning, coordination, and timing in decentralized trust systems. This layer abstracts away the differences between async runtimes (Tokio, async-std, Embassy) and enables STF components to work across **servers, cloud deployments, and embedded devices** without modification.

As a foundational crate, `stfx-runtime` defines the **Runtime trait**, allowing pluggable backends like Tokio, async-std, Embassy, or custom runtimes. Higher layers such as TSP SDK, agents, and routers can inject custom runtimes for deployment-specific needs, such as constrained IoT environments or WASM-based applications.

---

## 🔌 Core Runtime Trait

The main crate exposes a single, focused trait for async task management:

```rust
use async_trait::async_trait;
use std::error::Error;
use std::time::Duration;
use std::future::Future;

pub trait Runtime: Send + Sync {
    type Error: Error + Send + Sync + 'static;
    type Task: Clone + Send + Sync;
    
    /// Spawn a fire-and-forget task on the runtime.
    fn spawn<F>(&self, task: F) -> Result<Self::Task, Self::Error>
    where
        F: Future<Output = ()> + Send + 'static;
    
    /// Wait for a spawned task to complete.
    async fn join(&self, task: Self::Task) -> Result<(), Self::Error>;
    
    /// Cancel a task (best-effort).
    fn cancel(&self, task: Self::Task) -> Result<(), Self::Error>;
    
    /// Sleep for a specified duration.
    async fn sleep(&self, duration: Duration) -> Result<(), Self::Error>;
    
    /// Execute a future with a timeout.
    async fn timeout<F>(&self, duration: Duration, future: F) -> Result<F::Output, Self::Error>
    where
        F: Future + Send + 'static;
}
```

This trait ensures **flexible task management** and **timing control** while remaining compatible with constrained environments like embedded systems. The `Task` associated type allows implementations to represent spawned tasks in their native form (e.g., Tokio's `JoinHandle`, Embassy's channel-based handles).

---

## 🌐 Implementation Example: `stfx-runtime-tokio`

The `stfx-runtime-tokio` crate implements `Runtime` using **Tokio**, Rust's most widely-used async runtime. It leverages Tokio's task spawning, joining, and timeout primitives directly.

### Cargo.toml dependencies:

```toml
[dependencies]
stfx-runtime = { path = "../stfx-runtime" }
tokio = { version = "1", features = ["rt", "time", "sync"] }
```

### lib.rs core implementation:

```rust
use stfx_runtime::Runtime;
use tokio::task::{JoinHandle, spawn};
use tokio::time::{sleep, timeout};
use std::error::Error;
use std::time::Duration;
use std::future::Future;

#[derive(Clone)]
pub struct TokioTask(JoinHandle<()>);

pub struct TokioRuntime;

#[async_trait::async_trait]
impl Runtime for TokioRuntime {
    type Error = std::io::Error;
    type Task = TokioTask;

    fn spawn<F>(&self, task: F) -> Result<Self::Task, Self::Error>
    where
        F: Future<Output = ()> + Send + 'static,
    {
        let handle = spawn(task);
        Ok(TokioTask(handle))
    }

    async fn join(&self, task: Self::Task) -> Result<(), Self::Error> {
        task.0.await
            .map_err(|e| std::io::Error::new(std::io::ErrorKind::Other, e))
    }

    fn cancel(&self, task: Self::Task) -> Result<(), Self::Error> {
        task.0.abort();
        Ok(())
    }

    async fn sleep(&self, duration: Duration) -> Result<(), Self::Error> {
        sleep(duration).await;
        Ok(())
    }

    async fn timeout<F>(&self, duration: Duration, future: F) -> Result<F::Output, Self::Error>
    where
        F: Future + Send + 'static,
    {
        timeout(duration, future)
            .await
            .map_err(|_| std::io::Error::new(std::io::ErrorKind::TimedOut, "timeout"))
    }
}
```

This provides a direct mapping from the `Runtime` trait to Tokio's native primitives, with zero overhead.

---

## 🌐 Implementation Example: `stfx-runtime-embassy`

The `stfx-runtime-embassy` crate implements `Runtime` for **Embassy**, a lightweight async executor for embedded systems. Since Embassy lacks native task handles and cancellation, this implementation uses **channels** to simulate task tracking.

### Cargo.toml dependencies:

```toml
[dependencies]
stfx-runtime = { path = "../stfx-runtime" }
embassy-executor = "0.4"
embassy-time = "0.2"
embassy-sync = "0.4"
```

### lib.rs core implementation:

```rust
use stfx_runtime::Runtime;
use embassy_executor::Spawner;
use embassy_sync::channel::{Channel, Sender, Receiver};
use embassy_time::{Timer, Duration as EmbassyDuration, with_timeout};
use std::future::Future;
use std::time::Duration;

#[derive(Clone)]
pub struct EmbassyTask {
    done_signal: Sender<'static, ()>,
}

pub struct EmbassyRuntime {
    spawner: Spawner,
}

impl EmbassyRuntime {
    pub fn new(spawner: Spawner) -> Self {
        Self { spawner }
    }
}

#[async_trait::async_trait]
impl Runtime for EmbassyRuntime {
    type Error = ();
    type Task = EmbassyTask;

    fn spawn<F>(&self, task: F) -> Result<Self::Task, Self::Error>
    where
        F: Future<Output = ()> + Send + 'static,
    {
        // Create a channel for task completion notification
        let (tx, rx) = embassy_sync::channel::channel::<()>(1);

        // Wrap task to signal completion
        let wrapped = async move {
            task.await;
            let _ = tx.send(()).await;
        };

        self.spawner.spawn(wrapped).map_err(|_| ())?;
        Ok(EmbassyTask { done_signal: tx })
    }

    async fn join(&self, task: Self::Task) -> Result<(), Self::Error> {
        // Wait for completion signal
        let _ = task.done_signal.recv().await;
        Ok(())
    }

    fn cancel(&self, _task: Self::Task) -> Result<(), Self::Error> {
        // Embassy doesn't support cancellation natively.
        // This is best-effort: task continues but signal is dropped.
        Ok(())
    }

    async fn sleep(&self, duration: Duration) -> Result<(), Self::Error> {
        Timer::after(EmbassyDuration::from_millis(duration.as_millis() as u64)).await;
        Ok(())
    }

    async fn timeout<F>(&self, duration: Duration, future: F) -> Result<F::Output, Self::Error>
    where
        F: Future + Send + 'static,
    {
        let timeout_dur = EmbassyDuration::from_millis(duration.as_millis() as u64);
        with_timeout(timeout_dur, future)
            .await
            .map_err(|_| ())
    }
}
```

This demonstrates how constrained runtimes like Embassy can adapt the `Runtime` trait using channels for task tracking, enabling embedded STF deployments.

---

## 🔗 Integration with TSP SDK

The **TSP SDK** can be refactored to accept a generic `Runtime` parameter, eliminating its hard coupling to Tokio. This enables deployment across diverse environments.

### In tsp_sdk Cargo.toml:

```toml
[dependencies]
stfx-runtime = "0.1"
stfx-runtime-tokio = "0.1"  # default for testing
stfx-transport = "0.1"
```

### In tsp_sdk/src/lib.rs:

```rust
use stfx_runtime::Runtime;
use stfx_transport::Transport;
use std::time::Duration;

pub struct SecureStore<T: Transport, R: Runtime> {
    transport: T,
    runtime: R,
}

impl<T: Transport, R: Runtime> SecureStore<T, R> {
    pub fn new(transport: T, runtime: R) -> Self {
        Self { transport, runtime }
    }
    
    pub async fn send_message(&mut self, msg: &[u8]) -> Result<usize, T::Error> {
        self.transport.send(msg).await
    }
    
    pub async fn receive_with_timeout(
        &mut self,
        buf: &mut [u8],
        timeout_dur: Duration,
    ) -> Result<usize, R::Error> {
        let receive_future = self.transport.receive(buf);
        self.runtime.timeout(timeout_dur, receive_future).await
    }
    
    pub async fn spawn_background_task<F>(&self, task: F) -> Result<R::Task, R::Error>
    where
        F: std::future::Future<Output = ()> + Send + 'static,
    {
        self.runtime.spawn(task)
    }
    
    pub async fn shutdown_gracefully(&self, tasks: Vec<R::Task>) -> Result<(), R::Error> {
        for task in tasks {
            self.runtime.join(task).await?;
        }
        Ok(())
    }
}
```

This decouples TSP from specific runtimes, enabling HTTP relays for public internet use AND embedded IoT deployments without code changes.

### Usage in different environments:

**Cloud/Server (Tokio):**
```rust
use stfx_runtime_tokio::TokioRuntime;
use stfx_transport_http::HttpTransport;
use tsp_sdk::SecureStore;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let transport = HttpTransport::new();
    let runtime = TokioRuntime;
    
    let mut store = SecureStore::new(transport, runtime);
    store.send_message(b"Hello TSP over stfx-runtime").await?;
    Ok(())
}
```

**Embedded/IoT (Embassy):**
```rust
use stfx_runtime_embassy::EmbassyRuntime;
use stfx_transport_ble::BleTransport;
use tsp_sdk::SecureStore;

#[embassy_executor::main]
async fn main(spawner: Spawner) {
    let transport = BleTransport::new();
    let runtime = EmbassyRuntime::new(spawner);
    
    let mut store = SecureStore::new(transport, runtime);
    let _ = store.send_message(b"TSP over BLE").await;
}
```

---

## 📡 Additional Runtimes

### async-std Runtime (`stfx-runtime-async-std`)

An alternative to Tokio with similar capabilities. Useful for projects already invested in async-std or requiring specific scheduling characteristics.

```rust
use stfx_runtime_async_std::AsyncStdRuntime;

let runtime = AsyncStdRuntime;
let mut store = SecureStore::new(transport, runtime);
```

---

## 🌍 Real-World Use Cases

- **TSP Message Routing:** Spawn concurrent message handlers with timeout enforcement across all deployment targets (cloud, edge, IoT).

- **Agent Lifecycle Management:** Start agents, wait for graceful shutdown, and cancel long-running operations (handshakes, credential negotiation).

- **Embedded Credential Exchange:** BLE-based TSP VID registration on resource-constrained IoT devices using Embassy, with timeout protection.

- **DIDComm Integration:** Higher-level protocols can use `stfx-runtime` for task coordination without reimplementing async primitives per environment.

- **Hybrid Deployments:** Single TSP codebase runs on servers (Tokio), edge devices (async-std), and microcontrollers (Embassy).

---

## 🧭 Alignment with SOLID & STF Principles

### 🔄 Dependency Inversion (DIP)

**Higher-level components** (TSP SDK, agents, routers) depend on the abstract `Runtime` trait, not concrete implementations. Implementations (Tokio, Embassy, async-std) depend on the trait—never the reverse.

This eliminates coupling between TSP and specific runtimes:

```rust
// ✅ Correct: TSP depends on abstraction
fn handle_message<R: Runtime>(runtime: &R, msg: &[u8]) -> Result<(), R::Error> {
    let task = runtime.spawn(async { /* process */ })?;
    runtime.join(task).await
}

// ❌ Wrong: TSP would depend on concrete Tokio
fn handle_message_tokio(msg: &[u8]) { tokio::spawn(...); }
```

### 📌 Single Responsibility (SRP)

`stfx-runtime` has **one job**: provide a stable abstraction for async task management. Each implementation crate focuses on one runtime:

- `stfx-runtime-tokio` → Tokio task management
- `stfx-runtime-embassy` → Embassy execution
- `stfx-runtime-async-std` → async-std runtime

### 📖 Open/Closed Principle (OCP)

The `Runtime` trait is **stable and closed for modification**. New runtimes (glommio, smol, custom) extend functionality **without changing** the core trait:

```rust
// Add a new runtime without touching stfx-runtime
pub struct MyCustomRuntime { ... }

impl Runtime for MyCustomRuntime {
    // Implement the five methods
}
```

### 🧩 Interface Segregation (ISP)

The `Runtime` trait exposes **only essential operations**:
- `spawn(task)` — spawn a task
- `join(task)` — wait for completion
- `cancel(task)` — stop a task (best-effort)
- `sleep(duration)` — delay execution
- `timeout(duration, future)` — enforce timing bounds

**No bloat, no forced dependencies.** Clients use exactly what they need. This is a **textbook example of ISP** — the interface is minimal, client-driven, and composable. Additional concerns (intervals, blocking, introspection) layer on top rather than bloating the core trait.

### ♻️ Liskov Substitution (LSP)

Any `Runtime` implementation is **substitutable for another** without breaking higher-level code:

```rust
// Works with any Runtime implementation
let runtime: Box<dyn Runtime> = Box::new(TokioRuntime);
// or
let runtime: Box<dyn Runtime> = Box::new(EmbassyRuntime::new(spawner));
// Code remains unchanged
```

### 🏗️ Modularity & Reusability (STF Core Principles)

The trait is **reusable independently** across any project. Implementations are **loosely coupled** and **pluggable**:

- Can mix runtimes at runtime or compile-time
- Third-party developers extend without forking STFX
- Each crate evolves independently with stable interfaces
- TSP SDK, agents, and routers all benefit from the same abstraction

---

## 🎯 Summary: SOLID-Compliant Runtime Abstraction

**stfx-runtime** demonstrates all five SOLID principles in practice, making it a model implementation of the STF architectural philosophy. It is:

- ✅ **Reusable** — independent, pluggable, cross-project
- ✅ **Modular** — loosely coupled implementations behind a stable abstraction
- ✅ **Extensible** — new runtimes without modifying existing code
- ✅ **Testable** — mock runtimes for deterministic unit tests
- ✅ **Deployable** — same TSP code runs on servers, cloud, and embedded devices

---

## 📚 Future Considerations

### Graceful Shutdown Pattern

Applications can use `join()` to implement graceful shutdown:

```rust
pub async fn shutdown_gracefully<R: Runtime>(
    runtime: &R,
    tasks: Vec<R::Task>,
) -> Result<(), R::Error> {
    for task in tasks {
        runtime.join(task).await?;
    }
    Ok(())
}
```

### Testing with Mock Runtimes

Create a mock `Runtime` for deterministic testing:

```rust
pub struct MockRuntime {
    tasks: Arc<Mutex<Vec<bool>>>,  // Track task completion
}

impl Runtime for MockRuntime {
    fn spawn<F>(&self, task: F) -> Result<Self::Task, Self::Error> {
        // Immediate synchronous execution for testing
        // or store task for later inspection
    }
    // ...
}
```

### Future Extensions (Out of Scope)

The following are explicitly **not** part of `stfx-runtime` to keep the abstraction focused:
- ❌ `block_on()` — sync/async bridge (incompatible with embedded)
- ❌ Intervals/periodic tasks — compose with `spawn` + `sleep` loop
- ❌ Task introspection — not needed for STF workflows
- ❌ Current time access — external concern, bring your own clock
- ❌ Channel abstractions — use runtime-specific `tokio::sync`, `embassy_sync`, etc.

If future STF components need these, they should be separate crates with their own abstractions.

---

## 📖 Further Reading

For a comprehensive analysis of design decisions behind `stfx-runtime`, including alternatives considered and rationale for each choice, see:

**[STFX Runtime — Architecture Decision Analysis](./stfx-runtime_decision_analysis.md)**

This document covers:
- Trade-offs between abstraction and simplicity
- Why this scope (5 methods) vs. narrow or vast alternatives
- How Embassy's limitations were addressed with channel-based simulation
- Naming decisions and alignment with SOLID principles
- Feature boundaries and intentional scope exclusions
- Future evolution path and extensibility patterns
