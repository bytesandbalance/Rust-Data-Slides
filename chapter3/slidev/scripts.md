# Slide 1

# Slide 2

Now we see how we can analyze the earthquake data using Rust's libraries.

ndarray:

This library is our tool for numerical operations. Think of it as the NumPy of Rust. It provides multi-dimensional arrays, which are essential for representing datasets and performing calculations on them.
With ndarray, we can easily manipulate our earthquake data, extract specific features (like magnitudes or locations), and prepare it for further analysis.

linfa:

This is our machine learning library. It offers a variety of algorithms and tools for tasks like classification, regression, and clustering. In our case, we'll use its k-means clustering algorithm to group earthquakes based on their characteristics.
K-means clustering helps us identify patterns and relationships within our data, potentially revealing geographical hotspots or areas with similar earthquake activity.

# Slide 3

get our earthquake data ready for clustering using the ndarray library.

First, we need to transform the EarthquakeEvent data into a format that's compatible with clustering algorithms. We do this by extracting the longitude and latitude coordinates from each event and creating a vector (a list) of these coordinate pairs.

This vector is then converted into an ndarray::Array2 object, which is a two-dimensional array specifically designed for numerical operations. Each row in this array represents a single earthquake, and the two columns hold the longitude and latitude values, respectively.

This 2D array of earthquake coordinates is the input we'll provide to the k-means clustering algorithm. By focusing only on the geographic coordinates, we're essentially clustering the earthquakes based on their locations. This will help us identify potential spatial patterns and clusters in the data.

With our data prepared in this format, we're ready to move on to the next step: applying the k-means clustering algorithm using the linfa library.

# Slide 4

The cluster_earthquake_events function performs the following steps:

Preparation:  We begin by taking the vector of earthquake events (events) and transforming them into a format suitable for clustering using the convert_to_linfa_observations function, which we saw earlier.

Model Fitting:  Next, we create and train a k-means clustering model. We specify the desired number of clusters (n_clusters) and use a random number generator for reproducibility. We fit the model to our observations data, which is the 2D array of earthquake coordinates we prepared earlier.

Cluster Assignment: Once the model is trained, we obtain cluster assignments for each earthquake event. This tells us which cluster each earthquake belongs to based on the model's analysis of the data.

Cluster Initialization: We create a vector called clusters to store the information about each cluster. Each cluster will hold the earthquake events assigned to it and its calculated centroid (average location).

Earthquake Assignment: We iterate through our original events list and the corresponding cluster_assignments.  For each earthquake, we push it into the appropriate cluster within the clusters vector.

Centroid Calculation: After all earthquakes have been assigned, we calculate the centroid (average longitude and latitude) for each cluster. This centroid represents the central point of the cluster in geographic terms.

Return Clusters: Finally, we return the clusters vector, which now contains the complete information about each cluster, including the earthquake events belonging to it and the cluster's centroid.

This function gives us a structured way to group our earthquake data into clusters based on their geographic locations. These clusters can then be used for further analysis, such as identifying earthquake hotspots or regions with similar patterns of seismic activity.

# slide 5

Here's a recap of the code we've just walked through, which performs clustering.

# Slide 6

The EarthquakeCluster struct, is designed to store information about each cluster of earthquakes after the clustering process.

This struct represents a single cluster of earthquakes. It has two main fields:

events:  This is a vector (list) of EarthquakeEvent structs. It stores all the individual earthquake events that have been grouped together into this particular cluster based on their proximity to each other.

centroid: This is a tuple of two f64 values, representing the longitude and latitude coordinates of the cluster's centroid. The centroid is the average location of all the earthquakes within the cluster, effectively serving as its central point.

# Slide 7
`MetricStatistics` is designed to hold statistical information about a single metric, such as the depth or magnitude of earthquakes. It includes three key fields:

min: The minimum value observed for the metric within the cluster.
max: The maximum value observed for the metric within the cluster.
avg: The average value of the metric within the cluster.

`ClusterStatistics` aggregates statistical information for an entire earthquake cluster. It includes the following fields:

centroid: The coordinates (latitude and longitude) of the cluster's center point.
depth_stats: An instance of MetricStatistics summarizing the depth of earthquakes in the cluster.
magnitude_stats: An instance of MetricStatistics summarizing the magnitude of earthquakes in the cluster.
duration: An optional value (Option<Duration>) representing the time elapsed since the last significant earthquake in the cluster.

# Slide 8

`calculate_cluster_statistics_async` function extracts and analyzes depth and magnitude data from each earthquake cluster.

The code performs the following actions:

Data Extraction:

It extracts depth and magnitude information from the EarthquakeEvent instances stored within the cluster.
The map function is used to transform each EarthquakeEvent into its respective depth or magnitude value, creating two separate vectors, depth_data and magnitude_data.

Asynchronous Calculation:

The tokio::join! macro is utilized to execute the calculation of statistics for both depth and magnitude concurrently. This takes advantage of asynchronous programming to potentially improve performance.
Statistical Calculation:

The calculate_metric_statistics_async function is called for both depth_data and magnitude_data to compute their respective statistics (minimum, maximum, and average).

Return Results:

The function returns a tuple containing the calculated depth and magnitude statistics, both in the form of MetricStatistics structs.

# Slide 9
`calculate_all_cluster_statistics_async` takes a vector of `EarthquakeCluster` objects as input and returns a vector of `ClusterStatistics` objects containing the calculated statistics for each cluster.  Here's how it works:

1. Create Tasks:
    - It iterates through the `clusters` vector and creates asynchronous tasks for each cluster.
    - The `clone` method is used to create copies of the `centroid` and `cluster` data for each task. This is necessary because Rust's ownership rules wouldn't allow us to move the original data into multiple asynchronous tasks.
    - `async move`:
        - The move keyword transfers ownership of the variables captured by the block (in this case, cluster_clone and centroid) from the surrounding scope into the block.
        - This means the async block becomes the sole owner of these variables, ensuring they live as long as the task needs them, even if the original cluster is dropped or modified elsewhere.
    - Inside each task, we calculate the `duration_task` and the `depth_stats` and `magnitude_stats` using the `calculate_cluster_statistics_async` function, which we saw in a previous slide.  The `.await` operator is used to wait for the results of these asynchronous calculations.
    - Each task produces a `ClusterStatistics` struct containing the calculated statistics for that cluster.

2. Join All Tasks:
   - The `futures::future::try_join_all` function is used to wait for all the asynchronous tasks to complete. This function collects the results of all the tasks into a single `Result` value.

3. Handle Results:
    - If all tasks complete successfully, the `Ok` variant of the result contains a vector of `ClusterStatistics` objects, representing the calculated statistics for all clusters.
    - If any task fails, the `Err` variant will contain the first error encountered.

Important Note:

* `dyn Error`: This indicates that the error type can be any type that implements the `Error` trait. This is known as a "trait object" and allows for flexibility in error handling. We're using `Box<dyn Error>` to store the error on the heap, ensuring that the function's return type has a fixed size regardless of the specific error type. This makes it easier to work with errors that might originate from different parts of our code. The choice between Box<dyn Error> and a concrete error type comes down to flexibility vs. specificity. If you need the flexibility to handle multiple error types, or if you want to work with errors generically, Box<dyn Error> is your friend. If you know exactly what kind of error you'll encounter and performance is a top priority, using a specific error type is the way to go.

Trait Object Ergonomics:

Rust's trait objects (like dyn Error) are designed to be used with Box, Rc (reference-counted pointer), or Arc (atomic reference-counted pointer). This is because trait objects themselves don't have a known size at compile time.
By using Box, we allocate the trait object on the heap, and the function only needs to return a pointer to that memory, which has a fixed size.


## Temporal Analysis in Rust with Polars

Code Explanation:

- Mapping and Collecting: In `events_to_dataframe`, we map over the list of `EarthquakeEvent` to extract relevant fields (timestamps, magnitudes, etc.) into separate vectors. The collect step gathers these mapped values into vectors, which are then used to create series for the `DataFrame`. This structured approach ensures accurate data representation.

- Cloning and Lazy Evaluation: In `temporal_analysis`, we clone the original `DataFrame` to avoid directly modifying the original data. We use lazy evaluation for efficient, deferred execution, allowing us to chain operations like datetime conversion, feature extraction, and grouping without immediate computation. This is especially beneficial for performance when handling large datasets.

- Handling Parsing Errors: Both functions return a `Result<DataFrame, PolarsError>`. This error handling mechanism ensures that if something goes wrong—like an issue with the data conversion or grouping—a `PolarsError` is returned. This allows the calling code to handle the error gracefully rather than causing a runtime panic.

- Returning the Aggregated Data: The `temporal_analysis` function returns a `DataFrame` that groups earthquake data by year and month. For each group, it counts the number of earthquake events. This resulting `DataFrame` is essential for understanding trends in earthquake occurrences over time.
