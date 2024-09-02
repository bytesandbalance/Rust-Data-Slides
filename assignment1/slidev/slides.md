---
layout: section
background: black
theme: dracula
---

# Implementing and Using Macros for Plotting in Rust <span class="fragment fade-in-then-out"></span>

---
layout: default
---

## What are Macros?

- **Macros** in Rust allow you to write code that generates other code.
- **Benefits**:
  - Eliminate repetitive patterns.
  - Provide flexibility and consistency.

---
layout: default
---

## Objective of the Assignment

- **Objective**: Implement the `plot_data!` macro with support for different chart types
- The macro will take several parameters like:
  - DataFrame reference
  - Columns for x and y axes
  - Type of chart (line, bar, scatter)
  - File name, title, and axis labels


- **Hints**:
  - Use `plotters` crate documentation for reference.
  - Focus on correctly handling the different chart types.
  - Pay attention to data extraction from the DataFrame and passing it to the chart.

- **Challenge**: Can you add a custom style to your plots (e.g., colors, fonts)?

---
layout: default
---

## Implementing the Macro

````md magic-move
```rust
macro_rules! plot_data {
    (
        $df:expr,
        $x_col:expr,
        $y_col:expr,
        $chart_type:expr,
        $file_name:expr,
        $title:expr,
        $x_label:expr,
        $y_label:expr
    ) => {
        // Use plotters and other necessary crates

        // Extract data from the DataFrame

        // Set up the drawing area and chart

        // Add different chart types

        // Save the result to file
    };
}
```
```rust
plot_data!(
    &df, // DataFrame
    "year-monh", // X-axis column
    "count", // Y-axis column
    "line", // Chart type
    "earthquake_frequency.png", // Output file name
    "Earthquake Frequency Over Time", // Chart title
    "Month", // X-axis label
    "Number of Earthquakes" // Y-axis label
);
```
````
