---
layout: section
background: black
theme: dracula
---

# Data Analysis with Rust (Clustering, Statistics and aggregation)

---
layout: default
---

## Libraries: `ndarray` and `linfa`

- **`ndarray`:** A powerful library for numerical operations in Rust. It provides efficient multi-dimensional arrays and supports mathematical functions on them.

- **`linfa`:**  A comprehensive machine learning library in Rust. We'll use it for k-means clustering.

---
layout: default
---

## Preparing Data (`ndarray`)

```rust
fn convert_to_linfa_observations(events: &[EarthquakeEvent]) -> Vec<[f64; 2]> {
    events
        .iter()
        .map(|event| [event.coordinates.lon, event.coordinates.lat])
        .collect()
}

let mut data_array = Array2::zeros((observations.len(), 2));
for (i, point) in observations.iter().enumerate() {
    data_array[[i, 0]] = point[0]; // longitude
    data_array[[i, 1]] = point[1]; // latitude
}
```

- Transforms the EarthquakeEvent data into a format suitable for clustering.
- Uses ndarray's Array2 to create a 2D array of earthquake coordinates (longitude and latitude). This is the input for the clustering algorithm.

---
layout: default
---

## Clustering with linfa (clustering.rs)

````md magic-move
```rust
fn cluster_earthquake_events(
    events: Vec<EarthquakeEvent>,
    n_clusters: usize,
) -> Result<Vec<EarthquakeCluster>, Error> {
    let observations: Vec<[f64; 2]> = convert_to_linfa_observations(&events);
    // (code for clustering goes here)
}
```
```rust
fn cluster_earthquake_events(
    // ... same
) -> Result<Vec<EarthquakeCluster>, Error> {
    // ... (same as before)

    let model = KMeans::params_with_rng(n_clusters, rng.clone())
        .tolerance(1e-2)
        .fit(&observations)
        .expect("KMeans fitted");

    // Get cluster assignments for each data point
    let cluster_assignments = model
        .predict(observations)
        .targets()
        .iter()
        .map(|&cluster_idx| cluster_idx as usize)
        .collect::<Vec<usize>>();

    // ... (more code to assign earthquakes to clusters)
}
```
```rust
fn cluster_earthquake_events(
    // ... same
) -> Result<Vec<EarthquakeCluster>, Error> {
    // ... (same as before)

    // Initialize EarthquakeCluster instances
    let mut clusters: Vec<EarthquakeCluster> = vec![
        EarthquakeCluster {
            events: Vec::new(),
            centroid: (0.0, 0.0),
        };
        n_clusters
    ];

    // Assign earthquake events to their respective clusters
    for (event, cluster_idx) in events.into_iter().zip(cluster_assignments) {
        clusters[cluster_idx].events.push(event);
    }

    // ... (more code to assign earthquakes to clusters)
}
```
```rust
fn cluster_earthquake_events(
    // ... same
) -> Result<Vec<EarthquakeCluster>, Error> {
    // ... (same as before)
    // Calculate centroids for each cluster
    for cluster in &mut clusters {
        let sum_lon: f64 = cluster
            .events
            .iter()
            .map(|event| event.coordinates.lon)
            .sum();
        let sum_lat: f64 = cluster
            .events
            .iter()
            .map(|event| event.coordinates.lat)
            .sum();
        let count = cluster.events.len() as f64;

        if count > 0.0 {
            cluster.centroid = (sum_lon / count, sum_lat / count);
        }
    }
    Ok(clusters)
}
```
````

---
layout: default
---

- Converts the observations from a Vec to an ``ndarray::Array2` for efficient numerical operations.
- Applies the k-means algorithm from linfa to group earthquakes based on their proximity.
- Determines the optimal cluster centers (centroids) and assigns each earthquake to its closest center.
- Initializes empty `EarthquakeCluster` instances and assigns earthquake events to their corresponding clusters.
- Calculates the centroid for each cluster by averaging the longitudes and latitudes of the earthquakes within the cluster.
- Returns a `Vec<EarthquakeCluster>`, where each cluster contains its member earthquakes and centroid.

---
layout: default
---

### EarthquakeCluster Struct (clustering.rs)

```rust
#[derive(Debug, Clone)]
pub struct EarthquakeCluster {
    pub events: Vec<EarthquakeEvent>,
    pub centroid: (f64, f64), // (lon, lat) coordinates of the cluster centroid
}
```

  - events: A vector (list) of EarthquakeEvent structs representing the individual earthquakes that belong to this cluster.
  - centroid: A tuple (f64, f64) that stores the calculated average longitude and latitude of all the earthquakes in the cluster. This serves as the central point representing the cluster's location.

---
layout: default
---

## Statistical Analysis (statistics.rs)

````md magic-move
```rust
// Define a struct to hold statistics for a single metric (depth, magnitude, or energy)
#[derive(Clone, Debug)]
pub struct MetricStatistics {
    pub min: f64,
    pub max: f64,
    pub avg: f64,
}

```
```rust
pub struct ClusterStatistics {
    pub centroid: (f64, f64),
    pub depth_stats: MetricStatistics,
    pub magnitude_stats: MetricStatistics,
    pub duration: Option<Duration>, // Time since last significant earthquake
}
```
````

---
layout: default
---


#### Extracting Depth and Magnitude

```rust{1-3|5-10|11-17|18-22|24|all}
async fn calculate_cluster_statistics_async(
    cluster: &EarthquakeCluster,
) -> (MetricStatistics, MetricStatistics) {
    // Extract depth and magnitude data from the cluster's EarthquakeEvent instances
    let depth_data: Vec<f64> = cluster
        .events
        .iter()
        .map(|event| event.coordinates.depth)
        .collect::<Vec<f64>>()
        .into();

    let magnitude_data: Vec<f64> = cluster
        .events
        .iter()
        .map(|event| event.mag)
        .collect::<Vec<f64>>()
        .into();

    let (depth_stats, magnitude_stats) = tokio::join!(
        calculate_metric_statistics_async(&depth_data),
        calculate_metric_statistics_async(&magnitude_data),
    );

    (depth_stats, magnitude_stats)
}
```

---
layout: default
---

### Calculating Statistics for All Clusters

````md magic-move
```rust{1-3|4-6|8-14|15-21}
async fn calculate_all_cluster_statistics_async(
    clusters: Vec<EarthquakeCluster>,
) -> Result<Vec<ClusterStatistics>, Box<dyn Error>> {
    let tasks = clusters.into_iter().map(|cluster| {
        let centroid = cluster.centroid.clone(); // Clone centroid
        let cluster_clone = cluster.clone(); // Clone cluster for async block

        async move {
            let duration_task = calculate_time_since_last_significant_earthquake(&cluster_clone);

            let (depth_stats, magnitude_stats) =
                calculate_cluster_statistics_async(&cluster_clone).await;

            let duration = duration_task.await;

            Ok(ClusterStatistics {
                centroid,
                depth_stats,
                magnitude_stats,
                duration,
            })
        }
    });
// comes later
}

```
```rust{all|3}
async fn calculate_all_cluster_statistics_async(
    clusters: Vec<EarthquakeCluster>,
) -> Result<Vec<ClusterStatistics>, Box<dyn Error>> {
    // as before
    let results: Result<Vec<_>, Box<dyn Error>> = futures::future::try_join_all(tasks).await;

    // Handle errors or extract successful results
    match results {
        Ok(statistics) => Ok(statistics),
        Err(err) => Err(err),
    }
}
```
````


## Temporal Analysis in Rust with Polars

````md magic-move
```rust
use polars::prelude::*;
use common::earthquake_event::EarthquakeEvent;

pub fn events_to_dataframe(events: Vec<EarthquakeEvent>) -> Result<DataFrame, PolarsError> {
    let timestamps: Vec<i64> = events.iter().map(|e| e.time).collect();
    let magnitudes: Vec<f64> = events.iter().map(|e| e.mag).collect();
    let latitudes: Vec<f64> = events.iter().map(|e| e.coordinates.lat).collect();
    let longitudes: Vec<f64> = events.iter().map(|e| e.coordinates.lon).collect();

    let df = DataFrame::new(vec![
        Series::new("timestamp", timestamps),
        Series::new("magnitude", magnitudes),
        Series::new("latitude", latitudes),
        Series::new("longitude", longitudes),
    ])?;

    Ok(df)
}
```
```rust {2-3|1|all}
pub async fn temporal_analysis(df: &DataFrame) -> Result<DataFrame, PolarsError> {
    let df = df.clone()
        .lazy()
        .with_columns([
            // Convert the "timestamp" column from milliseconds to datetime
            col("timestamp")
                .cast(DataType::Datetime(TimeUnit::Milliseconds, None))
                .alias("datetime"),
        ])
        .with_columns([
            // Extract the month from the "datetime" column
            col("datetime")
                .dt()
                .month()
                .alias("month"),
            // Extract the year from the "datetime" column
            col("datetime")
                .dt()
                .year()
                .alias("year"),
        ])
        .group_by([col("year"), col("month")])
        .agg([
            // Count the number of "magnitude" entries per group
            col("magnitude").count().alias("count"),
        ])
        .collect()?;

    Ok(df)
}
```
````