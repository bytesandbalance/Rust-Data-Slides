# Data Analysis

We explore data analysis with Rust by implementing clustering, statistical analyses, and temporal analysis of earthquake data.
We use the `ndarray` library for efficient numerical operations and the `linfa` library for k-means clustering.

We begin by preparing our data and transforming earthquake events into a format suitable for clustering. 
Then, we apply the k-means algorithm to group earthquakes based on their proximity, determining optimal cluster centers and assigning earthquakes to their respective clusters.

We conduct statistical analyses, calculating key metrics like depth and magnitude for each cluster.
This provides insights into the characteristics of earthquakes over time.

We also leverage the Polars library for temporal analysis, converting our data into a DataFrame to examine earthquake frequency patterns. 
By grouping and aggregating the data, we can extract valuable insights into seismic activity trends.
