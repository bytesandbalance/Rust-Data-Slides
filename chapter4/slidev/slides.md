---
layout: section
background: black
theme: dracula
---
# Adding Observability with Telemetry

---
layout: default
---


- **Logs & Traces:** Go beyond basic printing to gain deeper insights into our async code's behavior.
- **OpenTelemetry:**  The industry-standard way to collect and analyze telemetry data.
- **Jaeger:** A powerful visualization tool to explore the traces of our application.

---
layout: default
---

## Refactoring `main.rs`

Before we add OpenTelemetry, let's refactor `main.rs` to improve code organization. We'll extract the core logic into a separate `process` function. This sets the stage for easier instrumentation later.

````md magic-move
```rust
// imports
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    // ... Define start_date and end_date, Create date iterator and Fetch data asynchronously
        // ... Log the start and end times being fetched
        println!("Fetching data for Start: {} End: {}", start_time_clone, end_time_clone);
        // ...
    }
    // ... Combine results, Cluster events and Calculate statistics
    let clusters_statistics = calculate_all_cluster_statistics_async(clusters).await?;
    // ... Print statistics
    for cluster_statistic in clusters_statistics {
        println!("{}", cluster_statistic);
    }
    Ok(())
}
```
```rust
// ... imports

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    // ... (The rest will be OpenTelemetry setup)
    process(chrono::Utc::now(), chrono::Duration::days(5 /* * 365 */)).await?;
    // ... (The rest will be OpenTelemetry setup)
    Ok(())
}
```
```rust
async fn process(
    end_date: chrono::DateTime<chrono::Utc>,
    back_until: chrono::Duration,
) -> Result<(), Box<dyn Error>> {
    // ... (extracted logic from main) ...
    // ... Calculate statistics
    let clusters_statistics = calculate_all_cluster_statistics_async(clusters).await?;
    // ... Print statistics
    for cluster_statistic in clusters_statistics {
        println!("{}", cluster_statistic);
    }
    Ok(())
}
```
````

---
layout: default
---

## Tracing Data Flow in Our Rust Application

<div class="grid grid-cols-2 gap-15">
    <div class="left-side">
        <div class="tracing_subscriber" v-click="1">
            <p class="text-center text-2xl" v-click="2">tracing</p>
            <div class="data-point" v-click="9"></div>
            <div class="data-point" v-click="10"></div>
            <div class="data-point" v-click="11"></div>
            <div class="log-point" v-click="12"></div>
            <div class="log-point" v-click="13"></div>
        </div>
    </div>
    <div class="subscriber" v-click="3">
        <p class="text-center text-2xl" v-click="4">tracing_subscriber</p>
        <div class="console-layer Subscriber layer" v-click="5">
            fmt
            <div class="otel-spans">
                <div class="log-point" v-click="12"></div>
                <div class="log-point" v-click="13"></div>
            </div>
        </div>
        <div class="otel-layer layer" v-click="6">
            tracing_opentelemetry
            <div class="otel-spans">
                <div class="span" v-click="9"></div>
                <div class="span" v-click="10"></div>
                <div class="span" v-click="11"></div>
            </div>
        </div>
    </div>
    <div class="exporters-container" v-click="7">
        <div class="otel-exporter exporter-1">
            OTLP Exporter 1
            <div class="otel-spans">
                <div class="span" v-click="9"></div>
                <div class="span" v-click="10"></div>
                <div class="span" v-click="11"></div>
            </div>
        </div>
        <div class="otel-exporter exporter-2">
            OTLP Exporter N
        </div>
    </div>
    <div class="jaeger bottom-right" v-click="8">
        Jaeger
        <div class="jaeger-spans">
            <div class="jaeger-point" v-click="9"></div>
            <div class="jaeger-point" v-click="10"></div>
            <div class="jaeger-point" v-click="11"></div>
        </div>
    </div>
     <v-drag-arrow pos="461,150,59,0" op70 v-click="4" />
    <v-drag-arrow pos="550,257,-93,80" op70 v-click="6" />
    <p v-after class="absolute bottom-57 left-115 transform -rotate-42 text-orange-500" v-click.after="6">EnvFilter</p>
    <v-drag-arrow pos="461,405,57,1" op70 v-click="8" />
</div>

<style>

.tracing_subscriber, .otel-exporter, .jaeger {
    border: 2px solid;
    padding: 1rem;
    height: 150px; /* Adjust height as needed */
}

.subscriber {
    border: 2px solid;
    padding: 1rem;
    height: 200px; /* Adjust height as needed */
}

.layer {
    border-top: 2px solid; /* Only add top border for layers */
    padding: 0.5rem;
    height: 60px;
}

.otel-spans { /* Add styling for the spans container */
    display: flex; /* Arrange spans horizontally */
}

.span {
    width: 10px;
    height: 10px;
    background-color: orange; /* Use a different color for spans */
    border-radius: 50%;
    margin: 5px;
}

.data-point {
    width: 10px;
    height: 10px;
    background-color: blue;
    border-radius: 50%;
    margin: 5px;
}

.log-point {
    width: 10px;
    height: 10px;
    background-color: red;
    border-radius: 50%;
    margin: 5px;
}

.jaeger-point {
    width: 10px;
    height: 10px;
    background-color: green;
    border-radius: 50%;
    margin: 5px;
}

/* For cascaded exporters */
.exporters-container {
    grid-column: 1;  /* Places in the second column of the grid */
    grid-row: 2;     /* Places in the second row of the grid */
}

.otel-exporter {
    border: 2px solid;
    padding: 1rem;
    height: 120px; /* Adjust height as needed */
    position: relative; /* Allows relative positioning for child elements */
}

.exporter-2 {
    top: -60px;      /* Adjust for desired overlap */
    left: 20px;       /* Adjust for desired horizontal offset */
}

/* Add more styles for other components */
</style>

---
layout: default
---

## OpenTelemetry Setup: Imports

```rust
use opentelemetry::{global::shutdown_tracer_provider, KeyValue}; // Core OpenTelemetry components and utilities
use opentelemetry_sdk::{trace, Resource};                       // SDK for building trace providers
use tracing::level_filters::LevelFilter;                       // Filtering log and trace events by level (e.g., INFO, ERROR)
use tracing_subscriber::{layer::SubscriberExt, EnvFilter, Layer, Registry}; // Building blocks for constructing trace subscribers
```

- **`opentelemetry`:** The heart of OpenTelemetry, providing fundamental types and functions.
- **`opentelemetry_sdk`:** The SDK for building and configuring how OpenTelemetry collects and processes data.
- **`tracing`:** A framework for instrumenting Rust code with structured logs and spans.
- **`tracing_subscriber`:** Tools for composing subscribers to collect the trace data.

---
layout: default
---
## OpenTelemetry Setup: Logging

```rust {all|5,6,7,8|9,10}
// ... imports
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
   // 1. Logging setup
   let console_env_filter = EnvFilter::builder()   // Filter for log levels
        .with_env_var("RUST_LOG")                  // Read filter settings from the RUST_LOG env variable
        .with_default_directive(LevelFilter::INFO.into()) // Default to INFO level if not specified
        .from_env_lossy();                        // Create the filter
    let console_logger = tracing_subscriber::fmt::layer()    // Formatted console log output
        .with_filter(console_env_filter);                // Apply the filter

   // (Rest of the code...)
}
```

- **`console_env_filter`:**  A filter that determines which log messages to display on the console. It reads the configuration from the `RUST_LOG` environment variable, which allows we to control the logging verbosity at runtime. The default log level is set to `INFO`, meaning that only log messages with a severity of `INFO` or higher will be displayed.
- **`console_logger`:** A layer that formats log messages and outputs them to the console. The filter ensures that only log messages that meet the criteria defined in the `console_env_filter` are printed.


---
layout: default
---

## OpenTelemetry Setup: Exporter

```rust
// ... imports

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    // ... (logging setup from the previous slide)

   // 2. OpenTelemetry exporter
   let otlp_exporter = opentelemetry_otlp::new_exporter()    // Create a new OTLP exporter
       .tonic();                                      // Use the Tonic gRPC implementation for communication

    // (Rest of the code...)
}
```

- **`otlp_exporter`:** This creates an OpenTelemetry Protocol (OTLP) exporter. This is how our app will send telemetry data out over the network. It uses gRPC (a high-performance RPC framework) as the communication protocol.  The exporter will send data to `http://localhost:4317` by default, but this is configurable.

---
layout: default
---

## OpenTelemetry Setup: Tracer Pipeline

```rust{all|8|9|10-12|13}
// ... imports
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    // ... (logging and exporter setup from previous slides)
   // 3. OpenTelemetry tracer pipeline
   let otlp_tracer = opentelemetry_otlp::new_pipeline()
       .tracing()                                  // Install tracing layer to connect with the tracing library
       .with_exporter(otlp_exporter)                // Add the OTLP exporter to the pipeline
       .with_trace_config(trace::config()         // Configure how traces are sampled and processed
           .with_resource(Resource::new(vec![
               KeyValue::new("service.name", "earthquake_tracing_app"),
           ])))                                     // Add a service name resource attribute
       .install_batch(opentelemetry_sdk::runtime::Tokio)?; // Install the tracer with a batch processor
    // (Rest of the code...)
}
```

---
layout: default
---

## OpenTelemetry Setup: Tracing

```rust {6|7,8,9}
// ... imports
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    // ... (logging, exporter, and tracer setup from previous slides)
   // 4. Tracing setup
   let tracing_env_filter = EnvFilter::builder().with_default_directive(LevelFilter::TRACE.into()).from_env_lossy();
   let telemetry = tracing_opentelemetry::layer()
       .with_tracer(otlp_tracer)
       .with_filter(tracing_env_filter);
   // (Rest of the code...)
}
```

---
layout: default
---

## OpenTelemetry Setup: Final Steps

```rust {al|6|7|8|10-11|12}
// ... imports
#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    // ... (all the setups from previous slides)

    let subscriber = Registry::default().with(telemetry).with(console_logger);
    tracing::subscriber::set_global_default(subscriber)?; // 👈 Make it global
    process(chrono::Utc::now(), chrono::Duration::days(5 /* * 365 */)).await?;

    tracing::info!("sleeping before shutting down trace providers");
    tokio::time::sleep(tokio::time::Duration::from_secs(10)).await;
    shutdown_tracer_provider();
    Ok(())
}

```

---
layout: default
---

## Jaeger
Run this:

```bash
docker run -d --name jaeger   -p 16686:16686   -p 6831:6831/udp   jaegertracing/all-in-one:latest
cd process_async_opentelemetry
cargo run
```

Now go to http://localhost:16686/
