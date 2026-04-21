## Reflection 1

When a browser connects to our server, the `handle_connection` read what it sends. The method buffers the data, reads it line by line until the function hit a blank space, pulls out the text, and prints it to the screen to see what the browser wants.

* **Buffering the Stream:** First, the function wraps our network `stream` in a `BufReader`. Instead of reading the incoming data byte-by-byte (which is inefficient), `BufReader` pulls in chunks of data at once. It automatically manages the low-level reading mechanics under the hood.
* **Setting up Storage:** the function needs a place to put the incoming text, so the function sets up a variable called `http_request`. By giving it the `Vec<_>` type annotation, the function's telling Rust to collect all the incoming lines into a Vector. 
* **Reading Line by Line:** Because `BufReader` implements the `BufRead` trait, the function gets access to a super helpful method called `.lines()`. This method reads the data stream and automatically splits it into chunks every time it hits a newline character. 
* **Handling the Results (or Ignoring the Errors):** The `.lines()` method doesn't just hand us raw text; it hands us an iterator of `Result` types, just in case the text is garbled (invalid UTF-8) or the connection drops. To keep our code simple right now, the function uses `.map` and `.unwrap()` to force out the clean `String`. If something goes wrong, the program will just crash. (In a production app, you'd want to handle this gracefully, but for a simple server, it does the trick!)
* **Knowing When to Stop:** How do the function knows when the browser is finished sending its request? By rule, HTTP requests always end with a completely blank line. So, the function tells our code to keep pulling lines until it hits an empty string, and then stop.
* **Taking a Look:** Once the function's collected all those lines into our Vector, the function prints the whole thing out using Rust's pretty debug formatting. This gives us a nice, readable look at the exact instructions the browser just sent to our server. 

## Reflection 2
![Commit 2 screen capture](/assets/img/refl2.png)
This commit completes the HTTP request-response cycle. The previous version received and parsed the request but never wrote anything back, leaving the browser hanging. I learned that receiving a connection and sending a response are two separate, explicit steps: read the request and write response. The server must actively write the response back down the socket, otherwise the client gets nothing.

## Reflection 3
![Commit 3 screen capture](/assets/img/refl3.png)
This section focuses on building a complete request-response cycle by reading the first line of the HTTP request to identify what the client wants, then responding with either the correct HTML page or a 404 error. 

Refactoring becomes necessary because both the success and error branches do the exact same work (read a file, format a response, write it to the stream) with only two values differing which are the status line and the filename. Duplicating that logic means any future change must be made in two places, risking inconsistency. By extracting just the differing values into a tuple from the if/else and sharing the rest of the code, there is now one place to update if the response format ever changes.

## Reflection 4
Adding the `/sleep` route makes the single-threaded design fail in a visible way. When one browser requests `127.0.0.1:7878/sleep`, `handle_connection` calls `thread::sleep(Duration::from_secs(10))` and blocks the only thread handling all work. During those 10 seconds, the server cannot move on to the next connection in `listener.incoming()`, so a second browser requesting `127.0.0.1:7878` has to wait even though that route itself is fast.

The bottleneck is not file reading or HTTP parsing rather the fact that the entire server processes requests sequentially on one thread. Each request must fully It does not scale because one slow request delays unrelated users. The server needs a way to handle multiple connections independently so slow work on one request does not block all others which is concurrency.

## Reflection 5
This section changes the design of the server more than the HTTP logic itself. Previously, `main` accepted a `TcpStream` and immediately called `handle_connection(stream)`, so the same thread that listened for new connections was also responsible for doing all the work. Now `main` creates `ThreadPool::new(4)` and passes each connection into `pool.execute(...)`, which means accepting connections and processing connections are no longer tied to the same single execution path.

The most important idea I learned here is that a thread pool is not just “many threads" rather a fixed set of worker threads that are created ahead of time and kept alive, waiting for jobs. In this project, each incoming request is wrapped up as a closure, boxed into a `Job`, and sent through a channel. The pool holds the sender, while the workers share the receiver. Each worker blocks on `recv()`, wakes up when a job arrives, prints which worker picked it up, and then runs the closure.

Another key thing I learned is why `Arc<Mutex<mpsc::Receiver<Job>>>` is needed. The sender side of the channel is easy to keep in the pool, but the receiver cannot simply be copied into every worker because Rust channels are multiple producer, single consumer. Since all workers need access to the same queue, the receiver has to be shared safely. `Arc` gives multiple workers shared ownership, and `Mutex` makes sure only one worker at a time locks the receiver and pulls the next job off the queue.

The project also stopped being only a binary crate and became both a binary and a library crate. `src/lib.rs` now holds the reusable `ThreadPool` abstraction, while `src/main.rs` focuses on the web server behavior. That separation makes the code easier to reason about because the server logic does not need to know how workers, channels, and thread management are implemented internally.

The bigger lesson is that concurrency is not the same as creating unlimited threads. Spawning a new thread for every request might feel simple at first, but it would scale badly and could exhaust system resources. A fixed-size pool gives controlled concurrency: up to four requests can be handled at the same time, and extra work waits in the queue instead of creating unbounded thread growth. That makes the server much more practical and explains why the `/sleep` route no longer blocks every other request behind it.
