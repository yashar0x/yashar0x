### **Title:** Unsafe Non-Blocking Shutdown Lead To Database Corruption and Extended Validator Downtime

### **Finding Description**

The Telcoin Network’s node shutdown process uses a non-blocking mechanism (`runtime.shutdown_background()`) that allows the process to exit without ensuring that critical background tasks, particularly database writes, are complete. This flaw, located in the `launch_node` entrypoint and `epoch_manager.run()` lifecycle, risks database corruption when writes to backends are interrupted. Corruption renders the node unstartable, requiring a full re-sync. The issue is triggered by errors in critical tasks (e.g., network timeouts, consensus failures) or malicious inputs, and the lack of blocking timeouts, explicit cleanup, or custom `Drop` logic ensures that data integrity is not guaranteed.

The vulnerability originates in the node’s shutdown logic, which does not ensure that critical tasks, particularly database writes, are completed before process termination.

#### 1. Non-Blocking Shutdown Mechanism
The `launch_node` entrypoint manages the Tokio runtime and node lifecycle:

```rust
let res = runtime.block_on(async move { epoch_manager.run().await });
runtime.shutdown_background();
res
```
When `epoch_manager.run()` returns (due to an error or normal termination), the node calls `runtime.shutdown_background()`. This method is non-blocking and does not wait for background tasks, such as database writes, to complete. The process can exit immediately, interrupting ongoing operations. This is a very known method on Rust.

#### 2. Lack of Explicit Cleanup
The `EpochManager` and related components lack explicit shutdown logic to ensure that all critical tasks are finalized.

There is no mechanism to wait for database writes or other background tasks to complete before the process exits. When the main function returns, Rust deallocates memory, and the operating system terminates any remaining threads, potentially leaving operations incomplete.

#### 3. Database Write Vulnerability
The node uses a database backend to persist blockchain state, with writes performed by background worker threads.

But, if the process exits during a write operation, the database files may be left in an inconsistent state (e.g., incomplete records, corrupted locks). This renders the database unreadable or requires a full re-sync on restart.

#### 4. Insufficient Timeouts
The node may use hierarchical task managers with short timeouts (e.g., 2–5 seconds) to signal task termination.

But, these timeouts are insufficient for complex database operations, especially under high load or on slower hardware. If a write operation exceeds the timeout, the process exits, risking corruption.

#### 5. No Custom `Drop` Implementation
The `EpochManager` and database components lack a custom `Drop` trait implementation to ensure safe resource cleanup.

Without a `Drop` implementation, there is no guarantee that database handles are closed or writes are completed before the process terminates.

### **Example Failure Path**

1.  A recoverable error (e.g., network timeout, malformed peer message) occurs in a critical task, causing `epoch_manager.run()` to return.
2.  The node calls `runtime.shutdown_background()`, which initiates process termination without waiting for background tasks.
3.  A database write operation (e.g., updating the blockchain state in `libmdbx`) is interrupted, leaving `data.mdbx` in an inconsistent state.
4.  On restart, the node fails to open the corrupted database, requiring a full re-sync from the network.
5.  The validator is offline for hours or days, incurring slashing penalties and disrupting network participation.

### **Recommendation**

To prevent database corruption and ensure data integrity during shutdown, the node should:

#### 1. Replace Non-Blocking Shutdown with Blocking Timeout:
Use `runtime.shutdown_timeout(Duration::from_secs(30))` (or a similarly sufficient timeout) instead of `shutdown_background()`. This will give all background tasks, especially database writes, enough time to complete before the process exits.

```rust
// Instead of:
runtime.shutdown_background();

// Use:
runtime.shutdown_timeout(Duration::from_secs(30));
```

#### 2. Implement Explicit Cleanup Logic:
Add a shutdown sequence in the main node manager (e.g., `EpochManager`) that waits for all critical tasks, particularly database operations, to finish before allowing the process to exit.

#### 3. Custom `Drop` Implementation:
Implement a `Drop` trait for the main node manager to ensure that, when the object is dropped, it blocks until all essential resources (like database handles) are safely closed.
