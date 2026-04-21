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