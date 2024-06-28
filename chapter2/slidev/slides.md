---
layout: section
background: black
theme: dracula
---

# Why Go Async?

- **Responsiveness:**  Avoid blocking the entire application while waiting for tasks like network requests.
- **Efficiency:**  Handle multiple I/O-bound operations concurrently.
- **Modern:**  Async is becoming the standard for network and I/O-heavy applications.

---
layout: default
---

## The `async`/`.await` Syntax

```rust{1-3|5}
async fn fetch_earthquake_data(...) -> Result<Vec<EarthquakeEvent>, Errors> {
    // ... our async code ...
}

let earthquake_events = fetch_earthquake_data(...).await?; // .await is key
```

- **`async fn`:**  Declares a function that returns a `Future`. This future represents the eventual result of the asynchronous operation.
- **`.await`:**  Pauses execution until the future completes, yielding control to other tasks.

---
layout: default
---

## Synchronous `fetch_earthquake_data`

````md magic-move
```rust
fn fetch_earthquake_data(
    &self,
    format: &str,
    start_time: &str,
    end_time: &str,
    min_magnitude: &str,
) -> Result<Vec<EarthquakeEvent>, Errors> {
    // ... Construct URL

    // Make blocking request:
    let response = reqwest::blocking::get(&url)?;

    // ... Parse response
}
```
```rust
async fn fetch_earthquake_data(
    &self,
    format: &str,
    start_time: &str,
    end_time: &str,
    min_magnitude: &str,
) -> Result<Vec<EarthquakeEvent>, Errors> {
    // ... Construct URL
    // Now use async version:
    let response = reqwest::get(&url).await?;

    // ... Parse response (also using .await if needed)
}
```
````

- Key point: `reqwest::blocking::get` **blocks** the current thread, halting execution until the network request completes. This is what we want to avoid in asynchronous code.

---
layout: default
---

## Real-Time Fetching with `async`

````md magic-move
```rust
fn fetch_earthquake_data_real_time(
    source: &dyn EarthquakeDataSource<Error = Errors>,
    format: &str,
    polling_interval_secs: u64,
) {
    loop {
        // ... (calculate start_time and end_time)
        // Synchronous fetch
        let usgs_earthquake_data = source.fetch_earthquake_data(format, &start_time, &end_time, min_magnitude);
        // ... (handle response)
        thread::sleep(Duration::from_secs(polling_interval_secs)); // Blocking sleep
    }
}
```
```rust
async fn fetch_earthquake_data_real_time(
    source: &dyn EarthquakeDataSource<Error = Errors>,
    format: &str,
    polling_interval_secs: u64,
) {
    loop {
        // ... (calculate start_time and end_time)

        let usgs_earthquake_data = source
            .fetch_earthquake_data(format, &start_time, &end_time, min_magnitude)
            .await;  // Use .await to fetch asynchronously
        // ... (handle response)
        tokio::time::sleep(Duration::from_secs(polling_interval_secs)).await; // Non-blocking sleep
    }
}
```
````
- Key changes:
    - The function is now `async`.
    - `.await` is used after `fetch_earthquake_data` to pause and wait.
    - `tokio::time::sleep` replaces `thread::sleep` for non-blocking sleep in async context.

---
layout: default
---

## Async Traits and Structs

````md magic-move
```rust
// Define a trait for earthquake data sources
pub trait EarthquakeDataSource {
    type Error;

    fn fetch_earthquake_data(
        &self,
        format: &str,
        start_time: &str,
        end_time: &str,
        min_magnitude: &str,
    ) -> Result<Vec<EarthquakeEvent>, Self::Error>;
}

```
```rust
use async_trait::async_trait;

#[async_trait]
trait EarthquakeDataSource {
    type Error;

    async fn fetch_earthquake_data(
        &self,
        // ... (parameters)
    ) -> Result<Vec<EarthquakeEvent>, Self::Error>; // Async return
}
```
````

- **`async_trait`:** This macro from the `async_trait` crate is essential. It lets us define traits with methods that can be `async`.
- **`async fn`:** Just like with regular functions, make the trait methods `async` if they involve asynchronous operations.

---
layout: default
---

## Running Our Async Code (Tokio)

```rust
#[tokio::main] // This attribute sets up the async runtime.
async fn main() -> anyhow::Result<()> {
    let usgs_data_source = UsgsDataSource;
    let format = "geojson";
    let polling_interval_secs = 60;

    fetch_earthquake_data_real_time(&usgs_data_source, format, polling_interval_secs).await;
    Ok(())
}
```

- **`#[tokio::main]`:** This sets up the `tokio` runtime, which is responsible for managing and executing our asynchronous tasks.
