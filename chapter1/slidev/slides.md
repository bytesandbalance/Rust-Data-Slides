---
layout: section
background: black
theme: dracula
---

# Fetching Earthquake Data with Rust <span class="fragment fade-in-then-out"></span>


---
layout: default
---


Rust makes it easy to fetch and work with data from external APIs. In this tutorial, we'll build a simple tool to get earthquake data from the USGS (United States Geological Survey) API.

We'll use:

- **Traits** <span class="fragment highlight-red"></span> to define a flexible interface for fetching data from different sources.
- **Structs** <span class="fragment highlight-green"></span> to model the earthquake data and our data source.
- **The `reqwest` library** <span class="fragment slide-up"></span> to handle the network requests.

<div v-click> By the end, you'll have a solid foundation for building your own data-fetching applications in Rust. </div>


---
layout: default
---

# Understanding the `EarthquakeEvent` Struct

````md magic-move
```markdown
- Represents: A single earthquake event.
- Fields:
    - `mag`: The magnitude of the earthquake.
    - `place`: A text description of the earthquake location.
    - `time`: The time the earthquake occurred (in milliseconds since the Unix epoch).
    - ... and more fields for additional details
```
```rust{2-7}
#[derive(Debug, Serialize, Deserialize)]
pub struct EarthquakeEvent {
    pub mag: f64,
    pub place: String,
    pub time: i64,
    // ... other fields ...
}
```

```markdown
- Derives:
    - `Debug`: For easy printing/debugging.
    - `Serialize`: To convert this struct into JSON format (which the API sends us).
    - `Deserialize`: To convert JSON data from the API into this struct.
```
```rust{1}
#[derive(Debug, Serialize, Deserialize)]
pub struct EarthquakeEvent {
    pub mag: f64,
    pub place: String,
    pub time: i64,
    // ... other fields ...
}
```
````

---
layout: default
---

# The `EarthquakeDataSource` Trait

````md magic-move
```markdown
- Purpose: Defines a common way to fetch earthquake data, regardless of the specific source (e.g., USGS, another API).
- `type Error`:  Allows each data source to have its own custom error type.
```
```rust{2}
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
```markdown
- `fetch_earthquake_data`:  The core function:
    - Takes parameters to specify the data format, time range, and minimum magnitude.
    - Returns a `Result` containing either:
        - `Ok(Vec<EarthquakeEvent>)`: A vector of earthquake events if successful.
        - `Err(Self::Error)`: An error value if something goes wrong.
```
```rust{4-10}
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
````
---
layout: default
---

# The `UsgsDataSource` Struct

```rust
pub struct UsgsDataSource;
```

- **Concrete Implementation:** This struct implements the `EarthquakeDataSource` trait specifically for fetching data from the USGS API.

- **Key Tasks (Implemented in the `impl` block):**
    1. Construct the correct API URL based on the parameters.
    2. Make an HTTP request to the USGS API.
    3. Parse the JSON response into a vector of `EarthquakeEvent` structs.
    4. Handle any potential errors.

---
layout: default
---

# Implementing `fetch_earthquake_data` for `UsgsDataSource`

````md magic-move
```rust{all|8}
impl EarthquakeDataSource for UsgsDataSource {
    type Error = Errors;

    fn fetch_earthquake_data(&self, ...) -> Result<Vec<EarthquakeEvent>, Errors> {
        let url = format!(...); // Construct API URL
        println!("{:?}", url); // Print the URL for debugging

        let response = reqwest::blocking::get(&url)?; // Make the request

        match response.status() { // Handle the response
            reqwest::StatusCode::OK => {
                let earthquake_data: GeoJsonData = response.json()?;
                let earthquake_events = earthquake_data.features.into_iter()
                    .map(|feature| EarthquakeEvent { ... }).collect();
                Ok(earthquake_events)
            },
            status => Err(Errors::UnexpectedStatusCode(status.to_string())),
        }
    }
}
```
```rust
match response.status() {
    reqwest::StatusCode::OK => {
        // Parse the GeoJSON response into EarthquakeEvent objects
        let earthquake_data: GeoJsonData = response.json()?;
        let earthquake_events: Vec<EarthquakeEvent> = earthquake_data
            .features
            .into_iter()
            .map(|feature| EarthquakeEvent {
                mag: feature.properties.mag,
                place: feature.properties.place,
                time: feature.properties.time,
                updated: feature.properties.updated,
                tsunami: feature.properties.tsunami,
                coordinates: feature.geometry.coordinates,
                mag_type: feature.properties.mag_type,
                event_type: feature.properties.event_type,
            })
            .collect();
        Ok(earthquake_events)
    }
    status => Err(Errors::UnexpectedStatusCode(status.to_string())),
}
```
````

---
layout: default
---

# Error Handling

````md magic-move
```rust{2}
impl EarthquakeDataSource for UsgsDataSource {
    type Error = Errors;

    fn fetch_earthquake_data(&self, ...) -> Result<Vec<EarthquakeEvent>, Errors> {
        let url = format!(...); // Construct API URL
        println!("{:?}", url); // Print the URL for debugging

        let response = reqwest::blocking::get(&url)?; // Make the request

        match response.status() { // Handle the response
            reqwest::StatusCode::OK => {
                let earthquake_data: GeoJsonData = response.json()?;
                let earthquake_events = earthquake_data.features.into_iter()
                    .map(|feature| EarthquakeEvent { ... }).collect();
                Ok(earthquake_events)
            },
            status => Err(Errors::UnexpectedStatusCode(status.to_string())),
        }
    }
}
```

```rust
#[derive(thiserror::Error, Debug)]
pub enum Errors {
    #[error("Unexpected status code: {0}")]
    UnexpectedStatusCode(String),
    #[error("Request error")]
    OtherError(#[from] reqwest::Error),
}
```
```markdown
- Custom Error Type: The `Errors` enum lets us handle different types of errors:
    - `UnexpectedStatusCode`:  When the API doesn't return a success status.
    - `OtherError`: For any other errors (wraps `reqwest::Error`).

- `Result` Type: The `fetch_earthquake_data` function returns a `Result`
    - so we can gracefully handle errors instead of the program crashing.
```
````

---
layout: default
---

# Putting it All Together: `run_fetch`

````md magic-move
```markdown
- High-Level Function:  This function uses the `UsgsDataSource` to fetch the data.
- Steps:
    1. Creates a `UsgsDataSource` instance.
```
```rust{1|8}
use common::blocking::earthquake_event::*;

pub fn run_fetch(
    start_time: &str,
    end_time: &str,
    min_magnitude: i32,
) -> Result<Vec<EarthquakeEvent>, Errors> {
    let usgs_data_source = UsgsDataSource;
    let format = "geojson";

    let usgs_earthquake_data: Result<Vec<EarthquakeEvent>, Errors> = usgs_data_source.fetch_earthquake_data(
        format,
        start_time,
        end_time,
        &min_magnitude.to_string(),
    );

    match usgs_earthquake_data {
        Ok(earthquake_events) => Ok(earthquake_events),
        Err(e) => Err(e),
    }
}

```
```markdown
- High-Level Function:  This function uses the `UsgsDataSource` to fetch the data.
- Steps:
    1. Creates a `UsgsDataSource` instance.
    2. Calls its `fetch_earthquake_data` method.
```
```rust{11-16}
use common::blocking::earthquake_event::*;

pub fn run_fetch(
    start_time: &str,
    end_time: &str,
    min_magnitude: i32,
) -> Result<Vec<EarthquakeEvent>, Errors> {
    let usgs_data_source = UsgsDataSource;
    let format = "geojson";

    let usgs_earthquake_data: Result<Vec<EarthquakeEvent>, Errors> = usgs_data_source.fetch_earthquake_data(
        format,
        start_time,
        end_time,
        &min_magnitude.to_string(),
    );

    match usgs_earthquake_data {
        Ok(earthquake_events) => Ok(earthquake_events),
        Err(e) => Err(e),
    }
}

```
```markdown
- High-Level Function:  This function uses the `UsgsDataSource` to fetch the data.
- Steps:
    1. Creates a `UsgsDataSource` instance.
    2. Calls its `fetch_earthquake_data` method.
    3. Handles potential errors by returning the `Result` from `fetch_earthquake_data`.
```
```rust{18-21}
use common::blocking::earthquake_event::*;

pub fn run_fetch(
    start_time: &str,
    end_time: &str,
    min_magnitude: i32,
) -> Result<Vec<EarthquakeEvent>, Errors> {
    let usgs_data_source = UsgsDataSource;
    let format = "geojson";

    let usgs_earthquake_data: Result<Vec<EarthquakeEvent>, Errors> = usgs_data_source.fetch_earthquake_data(
        format,
        start_time,
        end_time,
        &min_magnitude.to_string(),
    );

    match usgs_earthquake_data {
        Ok(earthquake_events) => Ok(earthquake_events),
        Err(e) => Err(e),
    }
}
```

````

---
layout: default
---

# Main Function

````md magic-move
```markdown
- Entry Point: This is where the program starts.
```
```rust{1-2}
pub mod fetch;
use fetch::run_fetch; // Import the function

fn main() -> anyhow::Result<()> {
    let start_time = "2014-01-01";
    let end_time = "2014-02-01";
    let min_magnitude = 3;

    let earthquake_events = run_fetch(start_time, end_time, min_magnitude)?;
    for event in earthquake_events {
        println!("{:?}", event);
    }

    Ok(())
}
```
```markdown
1. Gets parameters (time range, magnitude).
```
```rust{5-7}
pub mod fetch;
use fetch::run_fetch; // Import the function

fn main() -> anyhow::Result<()> {
    let start_time = "2014-01-01";
    let end_time = "2014-02-01";
    let min_magnitude = 3;

    let earthquake_events = run_fetch(start_time, end_time, min_magnitude)?;
    for event in earthquake_events {
        println!("{:?}", event);
    }

    Ok(())
}
```
```markdown
2. Calls `run_fetch` to get the earthquake data.
```
```rust{9}
pub mod fetch;
use fetch::run_fetch; // Import the function

fn main() -> anyhow::Result<()> {
    let start_time = "2014-01-01";
    let end_time = "2014-02-01";
    let min_magnitude = 3;

    let earthquake_events = run_fetch(start_time, end_time, min_magnitude)?;
    for event in earthquake_events {
        println!("{:?}", event);
    }

    Ok(())
}
```
```markdown
4. Returns `Ok(())` to indicate success.
```
```rust{14}
pub mod fetch;
use fetch::run_fetch; // Import the function

fn main() -> anyhow::Result<()> {
    let start_time = "2014-01-01";
    let end_time = "2014-02-01";
    let min_magnitude = 3;

    let earthquake_events = run_fetch(start_time, end_time, min_magnitude)?;
    for event in earthquake_events {
        println!("{:?}", event);
    }

    Ok(())
}
```
````

---
layout: default
---

# Result Types


````md magic-move
```rust{all|1}
fn main() -> anyhow::Result<()> {
    let start_time = "2014-01-01";
    let end_time = "2014-02-01";
    let min_magnitude = 3;

    let earthquake_events = run_fetch(start_time, end_time, min_magnitude)?;
    for event in earthquake_events {
        println!("{:?}", event);
    }

    Ok(())
}
```
```markdown
- `anyhow::Result<()>`:
  - `anyhow::Result` is a type provided by the `anyhow` crate. It's designed for flexible error handling,
  letting you work with a wide range of error types without needing to explicitly handle each one.
  - The `()` (empty tuple) within the angle brackets signifies that the successful outcome of the `main`
  function doesn't produce a value. The function is primarily focused on side effects.
    * If an error occurs within `main`, it's wrapped in the `anyhow::Result` and propagated upward.
```

```rust{all|5}
pub fn run_fetch(
    start_time: &str,
    end_time: &str,
    min_magnitude: i32,
) -> Result<Vec<EarthquakeEvent>, Errors> {
    let usgs_data_source = UsgsDataSource;
    let format = "geojson";

    let usgs_earthquake_data: Result<Vec<EarthquakeEvent>, Errors> = usgs_data_source.fetch_earthquake_data(
        format,
        start_time,
        end_time,
        &min_magnitude.to_string(),
    );

    match usgs_earthquake_data {
        Ok(earthquake_events) => Ok(earthquake_events),
        Err(e) => Err(e),
    }
}
```
```markdown
- `Result<Vec<EarthquakeEvent>, Errors>`:
  - This is Rust's standard `Result` type, which is central to error handling in the language.
  It represents the potential outcome of an operation that might either succeed or fail.
  - In this context, the `Vec<EarthquakeEvent>` indicates that a successful result will be
  a vector (a collection) of `EarthquakeEvent` objects.
  - `Errors` is a custom enum defined in our project to represent different error scenarios
  that could arise during the fetching process.
```
````

---
layout: default
---

# The `?` Operator

The question mark `?` is a powerful tool for handling `Result` values concisely. Here's what it does:

1. **Unwrapping Success:** If the `Result` value is `Ok(value)`, the `?` extracts the underlying `value` and allows the code to continue.

2. **Early Return on Error:** If the `Result` value is `Err(error)`, the `?` operator performs the following:
    * It tries to convert the `error` into the error type expected by the function's return type (in this case, `anyhow::Result<()>` for `main` or `Result<Vec<EarthquakeEvent>, Errors>` for `run_fetch`).
    * It immediately returns that converted error from the function.

---
layout: default
---

# Conclusion and Next Steps

- **Key Takeaways:**
    - Traits provide a flexible way to define interfaces for data sources.
    - Structs model the data we're working with.
    - Error handling is crucial for robust applications.

- **Ideas for Extending:**
    - Fetch data from other earthquake sources.
    - Visualize the earthquake data on a map.
    - Create a command-line interface for the tool.


---
layout: center
---

<video controls width="600">
  <source src="/home/pegah/output.mp4" type="video/mp4">
</video>