## Slide 1

Kia Ora, everyone! Pegah here. Welcome to the first tutorial in our series. We'll be building many tools together, starting with a tool to retrieve earthquake information from the USGS API.

## Slide 2

In this tutorial, we'll cover:

* **Structs:**  Modeling our earthquake data.
* **Traits:** Defining standardized interfaces for data sources.
* **Crates:** Using `reqwest` to handle network requests.

## Slide 3

The `EarthquakeEvent` struct is our blueprint for representing a single earthquake:

* **`Debug`:** Easily print earthquake details for troubleshooting.
* **`Serialize`:** Convert earthquake data to JSON format (used by APIs).
* **`Deserialize`:** Convert JSON data from APIs back into `EarthquakeEvent` objects.

## Slide 4

The `EarthquakeDataSource` trait creates a standard way to fetch earthquake data from any source:

* **`type Error`:** Each data source defines its own error type.
* **`fetch_earthquake_data`:** The core function to get earthquake data:
    * **Parameters:**  Data format, time range, minimum magnitude.
    * **Returns:**
        * `Ok(Vec<EarthquakeEvent>)`:  Success – a list of earthquakes.
        * `Err(Self::Error)`:  An error occurred.

## Slide 5

Let's bring the `EarthquakeDataSource` trait to life with a concrete implementation: the `UsgsDataSource` struct.

This struct is our specialist for working with the USGS API. Its job is to take the parameters you give it (format, time range, minimum magnitude) and do the heavy lifting:

1. **Build the API URL:**  It constructs the exact web address needed to get the right earthquake data from the USGS.
2. **Make the Request:** It sends an HTTP request to that URL, asking the USGS for the data.
3. **Parse the Response:** The USGS sends back the data in JSON format.  The `UsgsDataSource` unpacks this JSON into a tidy list of `EarthquakeEvent` objects that we can use.
4. **Handle Errors:** If anything goes wrong (like a network issue or incorrect parameters), it'll catch the error and let us know.

The implementation details for these steps are in the `impl` block associated with the `UsgsDataSource` struct. We'll dive deeper into that in the next slide.

## Slide 6

Now the heart of the `UsgsDataSource` implementation: the `fetch_earthquake_data` function.

This function is where the actual interaction with the USGS API happens:

1. **Construct API URL:**  Based on the input parameters, it creates the precise URL to query the USGS for earthquakes within the specified time range, minimum magnitude, and format. We print the URL for debugging purposes.

2. **Make the Request:** Using the `reqwest` library, we send a GET request to the constructed URL. The `blocking::get` method is used here to simplify this example.

3. **Handle the Response:** Here's where it gets interesting:

    * **Check Status:** We first examine the HTTP status code of the response. If it's 200 OK, we proceed. Otherwise, we return an error indicating an unexpected status code.
    * **Parse GeoJSON:**  If the request is successful, the USGS sends back data in GeoJSON format.  We parse this into a `GeoJsonData` structure.
    * **Transform to `EarthquakeEvent`:** The `GeoJsonData` contains a list of features, each representing an earthquake. We extract the relevant information from each feature and create an `EarthquakeEvent` instance.
    * **Collect Results:**  We gather all the `EarthquakeEvent` objects into a vector (a list) and return it as a successful result.

4. **Error Handling:**  If any step encounters an issue (like network failure, incorrect data format, etc.), we'll return an appropriate error from our custom `Errors` enum. This allows us to gracefully handle problems and inform the user.

By implementing this `fetch_earthquake_data` function, the `UsgsDataSource` fulfills its promise as a specialized data source for retrieving earthquake information from the USGS API.


You might be wondering why we used `reqwest::blocking::get` here. This blocking method makes the code simpler to understand, especially for this tutorial. It essentially pauses the current thread of execution until the network request completes and we receive the response.

However, in a real-world application, you'll likely want to switch to an asynchronous implementation. Asynchronous requests allow your program to continue doing other work while waiting for the network response, making it more efficient and responsive, especially when dealing with multiple requests or when network latency is high. But we will leave this to the next tutorials.

The map function is a powerful tool in Rust that allows us to transform one collection of values into another. In this case, we're transforming a collection of feature objects (representing earthquakes in the GeoJSON data) into a collection of EarthquakeEvent structs.

Here's how it works:

Iteration: The map function iterates over each feature in the features list.
Transformation: For each feature, it creates a new EarthquakeEvent struct. It extracts the relevant data from the feature's properties and geometry fields and assigns them to the corresponding fields in the Ear**thquakeEvent.
Collection: The newly created EarthquakeEvent objects are gathered into a new vector (Vec<EarthquakeEvent>), which we ultimately return as the successful result of our function.
In essence, the map function acts like an assembly line, taking each raw feature from the GeoJSON data and transforming it into a neatly packaged EarthquakeEvent object ready for our use. This makes our data much easier to work with and analyze within our Rust program.

## Slide 7

Error handling is crucial when working with external APIs.

Let's look at the `Errors` enum we've defined:

* **`UnexpectedStatusCode`:** This variant is used when the USGS API responds with a status code other than 200 OK, indicating that something went wrong with the request.
* **`OtherError`:** This variant captures any other errors that might occur during the process, such as network connectivity issues or problems parsing the response. It leverages the `thiserror` crate to automatically convert errors from the `reqwest` library into our own `Errors` type, making error handling more convenient.

By returning a `Result` from the `fetch_earthquake_data` function, we provide a way for the calling code to gracefully handle these potential errors.  If the result is `Ok`, we know the data was fetched successfully. If it's an `Err`, we can inspect the specific error variant to understand what went wrong and take appropriate action.

In the `match` statement, we handle these errors by:

* **Printing the Error:** We print a user-friendly error message using the `println!` macro. This helps during development and debugging.
* **Returning Early:** If there's an error, we immediately return from the function, preventing further execution and potential cascading failures.

This error-handling strategy ensures that our program can gracefully handle unexpected situations, making it more robust and reliable.

## Slide 8

(Slide 8 Voiceover)

Now, let's tie everything together with the `run_fetch` function. This function acts as the main entry point for retrieving earthquake data using our `UsgsDataSource`.

1. **Create `UsgsDataSource`:** We start by creating an instance of the `UsgsDataSource` struct. This is our specialized tool for interacting with the USGS API.

2. **Set the Format:** We specify that we want the earthquake data in "geojson" format. This is a common format for geospatial data.

3. **Fetch Earthquake Data:** The heart of the function is this line:

Here, we call the `fetch_earthquake_data` method on our `usgs_data_source` instance, passing in the parameters we discussed earlier (format, time range, minimum magnitude). This triggers the entire process of constructing the API URL, making the request to the USGS, parsing the response, and handling errors.

4. **Handle the Result:** The `fetch_earthquake_data` method returns a `Result` that could either be `Ok` (containing the earthquake data) or `Err` (containing an error).

*   **If `Ok`:** If the data fetch was successful, the `Ok` variant will contain a vector of `EarthquakeEvent` structs, representing the earthquakes that match our criteria.
*   **If `Err`:** If there was an error during the fetching process, the `Err` variant will hold information about the error (e.g., an unexpected status code, a network problem, etc.).

In this example, we're simply returning the `Result` directly from the `run_fetch` function. This means that the code that calls `run_fetch` will need to handle the `Result` and decide what to do in case of success or error.

By encapsulating the data fetching process within the `run_fetch` function, we make our code more organized and reusable. It provides a convenient way for other parts of our application to retrieve earthquake data without having to worry about the underlying implementation details.

## slide 9

Now, let's explore the `main` function, the starting point of our earthquake information retrieval program.

1. **Imports:** We begin by importing the necessary modules and functions. The `anyhow` crate provides convenient error handling capabilities, and we import the `run_fetch` function we defined earlier.

2. **Set Parameters:** We set the parameters for our earthquake search:
    * `start_time` and `end_time`: Define the date range for the earthquakes we want (January 1st to February 1st, 2014, in this example).
    * `min_magnitude`: Specifies that we only want earthquakes with a magnitude of 3 or higher.

3. **Call `run_fetch`:** We then call our handy `run_fetch` function, passing in the parameters we just defined. This function encapsulates all the logic for fetching the data from the USGS API, including constructing the URL, making the request, parsing the response, and handling errors.

4. **Handle Result and Print Earthquakes:** The `run_fetch` function returns a `Result` that we need to handle.

    * **If `Ok`:** If the data fetching was successful, we iterate through the list of `earthquake_events` and print each event's details to the console using the `{:?}` format specifier (which is enabled by the `Debug` derive attribute on the `EarthquakeEvent` struct).
    * **If `Err`:** If there was an error, the `?` operator will automatically propagate the error up to the `main` function, where it will be handled by the `anyhow` crate and converted into a user-friendly error message.

5. **Return Ok():** Finally, we return `Ok(())` to indicate that the `main` function has executed successfully.

By following these steps, our `main` function provides a clear and concise way to start our program, fetch earthquake data, handle errors, and display the results to the user.

## Slide 10

Taking a closer look at the Result types we used in our code:

1. **`anyhow::Result<()>`:** This is the return type of our `main` function.

- `anyhow::Result` is a type provided by the `anyhow` crate. It's designed for flexible error handling, letting you work with a wide range of error types without needing to explicitly handle each one.

- The `()` (empty tuple) within the angle brackets signifies that the successful outcome of the `main` function doesn't produce a value. The function is primarily focused on side effects (like fetching and printing data).

- If an error occurs within `main`, it's wrapped in the `anyhow::Result` and propagated upward.  The `anyhow` crate provides convenient tools for dealing with these errors, such as printing error messages or logging them.

2. **`Result<Vec<EarthquakeEvent>, Errors>`:** This is the return type of the `run_fetch` function.  It's Rust's standard `Result` type, a core part of Rust's error handling mechanism.

- The `Vec<EarthquakeEvent>` part indicates that a successful result will be a vector (a collection) of `EarthquakeEvent` objects, representing the earthquake data we fetched.

- The `Errors` part is our custom enum defined in the project to represent different error scenarios that could arise during the fetching process.

- By returning a `Result` from `run_fetch`, we clearly communicate that the function might fail and provide a structured way to handle potential errors.  The calling code (in this case, the `main` function) can then use a `match` statement or other error-handling techniques to deal with both success (`Ok`) and error (`Err`) cases gracefully.

## Slide 11

Let's talk about the question mark operator (`?`), a handy tool we've been using to streamline our error handling. It's a concise way to work with `Result` values, which are central to error management in Rust.

Here's a breakdown of what the `?` operator does in our code:

1. **Unwrapping Success:**
    - When you see `?` after a function that returns a `Result`, it's checking whether the result is successful (i.e., an `Ok` variant).
    - If it is, the `?` operator automatically extracts the value inside the `Ok` variant and allows your code to continue using that value.

2. **Early Return on Error:**
    - If the result is an error (i.e., an `Err` variant), the `?` operator takes care of several things:
       - **Error Conversion:** It attempts to convert the error type within the `Err` to match the error type that the current function is supposed to return. For example, in our `main` function, it tries to convert the error into an `anyhow::Error`.
       - **Early Exit:** It immediately returns that converted error from the function. This means the rest of the function's code won't run. Instead, the error is passed back to the calling code, which can then handle it appropriately.

**Key Benefits:**

* **Readability:** The `?` operator makes your code much more concise and easier to read, especially when you're dealing with multiple `Result` values.
* **Error Propagation:** It simplifies the process of propagating errors up the call stack, allowing you to handle them at a higher level in your program where it might be more appropriate.
* **Less Boilerplate:** You don't need to write repetitive `match` statements every time you work with a `Result`, saving you time and effort.

In our example, we use `?` in the `main` function and `run_fetch` to gracefully handle errors that might occur during the earthquake data fetching process. This helps us write more robust and error-resistant code.
