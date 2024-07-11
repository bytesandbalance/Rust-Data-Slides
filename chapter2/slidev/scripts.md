# Slide 1

Go Async?

Now, let's consider why we might want to "go async" in our Rust earthquake program.


# Slide 2

Asynchronous programming offers several key advantages:

Responsiveness: In our current blocking implementation, the entire application pauses while waiting for the earthquake data to be fetched from the USGS API. This can make the program feel unresponsive, especially if the network connection is slow.By switching to async, we can prevent this blocking and allow the rest of our application to continue running smoothly while we wait for the data.

Efficiency: When dealing with tasks that involve waiting for external resources (like network requests or database queries), asynchronous programming allows us to handle multiple operations concurrently. This means we can fetch earthquake data, process other parts of our application, and even make multiple requests simultaneously, leading to better overall performance.

Modern Standard: Asynchronous programming has become the standard approach for building network and I/O-heavy applications. By adopting async patterns in Rust, you'll be aligning with modern best practices and ensuring your code is well-suited for handling the demands of today's software landscape.

In the next steps of our project, we'll explore how to transform our current blocking implementation into an asynchronous one.

# Slide 3

Now, the `async`/`.await` syntax, the foundation for writing asynchronous code in Rust.

**`async fn`:** When a function is marked with `async`, it becomes a special kind of function that returns a `Future`. This `Future` represents a value that may not be available immediately, like the result of a network request. An `async fn` doesn't execute its code right away; instead, it sets up the steps to be taken to eventually produce a result.

**`.await`:** The `.await` operator is used to pause the execution of an `async` function until the `Future` it returns is ready. This allows other parts of the program to run while we're waiting for the result. Once the `Future` is ready, execution resumes, and we can work with the returned value.

In our example:

1.  The `fetch_earthquake_data` function is marked as `async`, indicating it will return a `Future` representing the earthquake data that will be fetched.
2.  When `fetch_earthquake_data(...)` is called and `.await` is added, the program pauses until the earthquake data is retrieved.

By combining `async` and `.await`, we can write code that appears to run sequentially but actually allows other tasks to execute while we wait for long-running operations to complete. This makes our program more responsive and efficient.

# Slide 4

We should revisit our original fetch_earthquake_data function, which uses a synchronous (blocking) approach:

The key takeaway here is the use of reqwest::blocking::get. This function makes a blocking network request, meaning it halts the execution of your program's current thread until the request is completed and the response is received. While this might be acceptable in some scenarios, it can become a bottleneck when you need your program to remain responsive or when dealing with multiple network requests.

To overcome this limitation, we'll switch to the asynchronous version of the reqwest library, which allows us to make non-blocking requests. This means our program can continue doing other work while waiting for the network response, leading to better performance and responsiveness.

# Slide 5

We can leverage asynchronous programming to create a more dynamic earthquake data retrieval tool:

In this modified function, we've made a few key changes to enable real-time fetching:

Asynchronous Function: The function is marked as async, indicating that it will be executed asynchronously.
.await on Fetch: We use .await after calling fetch_earthquake_data. This pauses the current task until the earthquake data is fetched from the USGS API, allowing other tasks to run concurrently.
Non-Blocking Sleep: Instead of using the blocking thread::sleep function, we've switched to tokio::time::sleep. This function allows us to pause the current task for the specified polling interval without blocking the entire thread.
By combining these changes, we've created a loop that continuously fetches earthquake data from the USGS API at regular intervals. This enables us to keep our application updated with the latest earthquake information without impacting the responsiveness of other parts of our program.

# Slide 6

We need to adapt our EarthquakeDataSource trait and the UsgsDataSource struct to work with asynchronous functions:

In the updated code, we've made two key changes:

async_trait Macro: We've introduced the #[async_trait] macro from the async_trait crate. This macro is essential when you want to define a trait that contains methods that can be asynchronous. Without it, Rust doesn't allow async methods within traits directly.

async fn: We've modified the fetch_earthquake_data method within the trait to be an async fn.  This means that implementations of this method, such as in our UsgsDataSource, can now use .await to pause execution and wait for asynchronous operations like network requests to complete.

By applying these changes, we've transformed our EarthquakeDataSource trait into an async trait. This allows us to seamlessly integrate asynchronous programming into our earthquake data retrieval logic,

# Slide 7

Now that our fetch_earthquake_data function is asynchronous, let's see how to run this updated code.

In this code, the crucial element is the #[tokio::main] attribute placed before our main function.  Let's break down what this does:

Tokio Runtime: This attribute signals to the Rust compiler that we want to use the Tokio runtime to manage our asynchronous tasks. Tokio is a popular asynchronous runtime for Rust that provides the necessary tools and infrastructure to execute asynchronous code efficiently.

Executor Setup: Behind the scenes, the #[tokio::main] attribute sets up a Tokio runtime. This runtime includes an executor, which is responsible for scheduling and driving the execution of our async tasks. The executor takes care of polling futures, handling task completion, and ensuring that our asynchronous code runs smoothly.

Async main Function: By marking the main function with async, we're indicating that it's now an asynchronous function itself. This means it can contain .await calls to pause execution and wait for asynchronous operations like our fetch_earthquake_data_real_time function.
The #[tokio::main] attribute takes care of the complex machinery needed to run asynchronous code in Rust. It abstracts away the details of the runtime, allowing us to focus on writing the logic of our program in a clear and concise way.

Without #[tokio::main], our async code wouldn't run correctly because there would be no executor to manage its execution. So, remember, when working with asynchronous Rust code, you typically need an async runtime like Tokio to make it work!