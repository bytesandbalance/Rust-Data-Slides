---
layout: section
background: black
theme: dracula
---

# Data Analysis with Rust (Clustering & Statistics)

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

```rust
pub struct ClusterStatistics {
    pub centroid: (f64, f64),
    pub depth_stats: MetricStatistics,
    pub magnitude_stats: MetricStatistics,
    pub duration: Option<Duration>, // Time since last significant earthquake
}
```


---
layout: default
---

- `ClusterStatistics`: This struct encapsulates the statistics calculated for each cluster:
  - `centroid`: The geographic center of the cluster (longitude, latitude).
  - `depth_stats`: Statistics about the earthquake depths within the cluster (min, max, average).
  - `magnitude_stats`: Statistics about the earthquake magnitudes (min, max, average).
  - `duration`: The time since the last significant earthquake in the cluster (magnitude > 5).

---
layout: default
---

### Calculating Statistics (statistics.rs)
```rust
async fn calculate_cluster_statistics_async(
    cluster: &EarthquakeCluster,
) -> (MetricStatistics, MetricStatistics)
```
- This async function calculates depth and magnitude statistics for a single cluster. It does this by extracting the relevant data from the cluster and then calculating the statistics in parallel using tokio::join!.
- The return value is a tuple containing a struct representing depth statistics and a struct representing magnitude statistics.

---
layout: default
---

#### Extracting Depth Data
````md magic-move
```rust
async fn calculate_cluster_statistics_async(
    cluster: &EarthquakeCluster,
) -> (MetricStatistics, MetricStatistics) {
    // ...
}
```

```rust
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
}
```
````
It iterates through the cluster's events, maps each event to its depth, and collects the depths into a vector.


---
layout: default
---

#### Extracting Magnitude Data
````md magic-move
```rust
async fn calculate_cluster_statistics_async(
    cluster: &EarthquakeCluster,
) -> (MetricStatistics, MetricStatistics) {
   // Extract depth and magnitude data from the cluster's EarthquakeEvent instances
    let depth_data: Vec<f64> = cluster
        .events
        .iter()
        .map(|event| event.mag)
        .collect::<Vec<f64>>()
        .into();
}
```
```rust
async fn calculate_cluster_statistics_async(
    cluster: &EarthquakeCluster,
) -> (MetricStatistics, MetricStatistics) {
    // ...(same as before)
    let magnitude_data: Vec<f64> = cluster
        .events
        .iter()
        .map(|event| event.mag)
        .collect::<Vec<f64>>()
        .into();
}
```
````
Similar to depth data extraction, but it retrieves the magnitudes of the earthquakes in the cluster

---
layout: default
---

### Concurrent Calculation of Statistics

````md magic-move
```rust
async fn calculate_cluster_statistics_async(
    cluster: &EarthquakeCluster,
) -> (MetricStatistics, MetricStatistics) {
    // ...(same as before)
    let magnitude_data: Vec<f64> = cluster
        .events
        .iter()
        .map(|event| event.mag)
        .collect::<Vec<f64>>()
        .into();
}
```
```rust
async fn calculate_cluster_statistics_async(
    cluster: &EarthquakeCluster,
) -> (MetricStatistics, MetricStatistics) {
    // ...(same as before)

    let (depth_stats, magnitude_stats) = tokio::join!(
        calculate_metric_statistics_async(&depth_data),
        calculate_metric_statistics_async(&magnitude_data),
    );

    (depth_stats, magnitude_stats)
}
```
````
Calculates the `depth_stats` and `magnitude_stats` concurrently using `tokio::join!`, a macro that allows running multiple futures simultaneously. This improves performance by calculating statistics for both depth and magnitude simultaneously.


---
layout: default
---

### Calculating Statistics for All Clusters

````md magic-move
```rust
pub async fn calculate_all_cluster_statistics_async(
    clusters: Vec<EarthquakeCluster>,
) -> Result<Vec<ClusterStatistics>, Box<dyn Error>> {
    let tasks = clusters.into_iter().map(|cluster| {
        let centroid = cluster.centroid.clone(); // Clone centroid
        let cluster_clone = cluster.clone(); // Clone cluster for async block

        // Comes later
    });

    // Comes later
}
```
```rust
pub async fn calculate_all_cluster_statistics_async(
    clusters: Vec<EarthquakeCluster>,
) -> Result<Vec<ClusterStatistics>, Box<dyn Error>> {
    // Same as before

        async move {
            let duration_task = calculate_time_since_last_significant_earthquake(&cluster_clone);

            let (depth_stats, magnitude_stats) = calculate_cluster_statistics_async(&cluster_clone).await;
            let duration = duration_task.await;

            Ok(ClusterStatistics {
                centroid,
                depth_stats,
                magnitude_stats,
                duration,
            })
        }
    };// Comes later
}

```
```rust
pub async fn calculate_all_cluster_statistics_async(
    clusters: Vec<EarthquakeCluster>,
) -> Result<Vec<ClusterStatistics>, Box<dyn Error>> {
    // Same as before

    // Collect and await all the async tasks
    let results: Result<Vec<_>, Box<dyn Error>> = futures::future::try_join_all(tasks).await;

    // Handle errors or extract successful results
    match results {
        Ok(statistics) => Ok(statistics),
        Err(err) => Err(err),
    }
}
```

````
---
layout: default
---

- This function iterates through a vector of `EarthquakeClusters` and creates a future for each cluster.
- The futures concurrently calculate ClusterStatistics for each cluster using `calculate_cluster_statistics_async`.
- It then collects all the results using `futures::future::try_join_all`.
- If all futures succeed, it returns `Ok(Vec<ClusterStatistics>)`; otherwise, it returns the first encountered error.

---
layout: default
---

## Conclusion

  - Rust empowers your earthquake data analysis:

    - Efficient Clustering: Leveraging `ndarray` for data representation and linfa for powerful machine learning algorithms like k-means clustering, you can effectively identify geographical patterns in earthquake occurrences.

    - Meaningful Statistics: By computing insightful statistics like depth and magnitude distributions, as well as time since last significant events, you gain deeper insights into the characteristics of each earthquake cluster.

    - Asynchronous Performance:  With tokio, you can fetch and process earthquake data asynchronously, enhancing the responsiveness and efficiency of your analysis pipeline.
