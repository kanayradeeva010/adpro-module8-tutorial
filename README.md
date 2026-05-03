# 1. What are the key differences between unary, server streaming, and bi-directional streaming RPC (Remote Procedure Call) methods, and in what scenarios would each be most suitable?

- Unary : type of communication protocol that allows client server interactions where client can send a single request to the server and get single response back  from the server (client sends one piece of data to the server and waits for a response).
This type of communication protocol is used if we eant to fetch a single item from a database, authenticate a user, and perform a calculation and obtain the result.

- Server Streaming : communication protocol that allows server to send a stream of responses to the client, this is used when server needs to push a large amount of data or continuous stream of updates to the client, example if we want to get real-time updates like stock market prices, weather updates, or sending a large file in chunks. 

- Bi-directional streaming RPC:communication pattern which both the client and the server can send multiple messages to each other in continuous stream, it allows a real time two way communication between client and server. This pattern is usually used in a chat application where it can send and receive messages in real time between client and server and also for real-time analytics where data is continuously sent from server to client and vice versa. 

# 2. What are the potential security considerations involved in implementing a gRPC service in Rust, particularly regarding authentication, authorization, and data encryption? 
- Data encryption: the grpc uses HTTP/2, which can send data in a binary format that is still readable as "plaintext" if intercepted via sniffing. we must implement TLS (Transport Layer Security), in the tonic library, this involves configuring ServerTlsConfig (in server side) and ClientTlsConfig (in client side) to wrap the communication in an encrypted tunnel. 

- authentication : the server must be able to confirm that client calling a function for example process_payment is a legitimate entity and not an attacker, in rust we can use mutual TLS (mTLS) as a solution where both client and server exchange digital certificates to verify each other's identity. also, we can use token-based (JWT) using json web tokens that inserted to the grpc metadata to prove user's identity.  

- authorization : once an indentity is verified, the server must restrict the access so that users can only call functions that is appropriate for their spesific role. In rust we can implement Role based access control that can be achieved using interceptors in rust. It can act as a middleware to check all the user's permissions before llowing the request to proceed the business logic. 

# 3. What are the potential challenges or issues that may arise when handling bidirectional streaming in Rust gRPC, especially in scenarios like chat applications?
- when a server sends messages faster than the client can process them or vice versa, it can ended up with a memory leak as the message queue fills up indefinitely
- a chat app requires high concurrency, it reads from the network, updates a shared list of users, and write it back to the network all at once.  if we want to use a mutex to protect the shared list and a thread holds the lock while waiting for a network operation that id blocked, it can cause a deadlock. 
- in rust once we move a variable to tokio::spawn block, the original scope no longer owns it, bidirectional streams often need to stay open for a long time, but they also need to access shared data. 

# 4. What are the advantages and disadvantages of using tokio_stream::wrappers::ReceiverStream for streaming responses in Rust gRPC services? 
- advantages
1. simple conversion: it makes the conversion from standard asynchronous channel (tokio::sync::mpsc::Receiver) to a stream that are compatible with gRPC easier.
2. it able us to seperate the logic on data production (where .send() is called to a channel) from gRPC transmission logic.
3. it integrates seamlessly with tokio ecosystem, it enables the use of built in features such as timeouts and buffering 
4. it does not block the server's execution due to its non-blocking nature, making it ideal for high-concurrency applications like chat services

- disadvantages:
1. since ReceiverStream takes full ownership of the Receiver, it can be challenging to manage or manually close the channel from outside the stream
2. if not configured with an appropriate bounded channel, data accumulation within the ReceiverStream can lead to uncontrolled memory usage (memory bloat).
3. integrating error handling from the data production loop into the gRPC stream often requires additional boilerplate code, such as wrapping every message in a Result<T, Status>.
4. ReceiverStream is designed for a single consumer only, we cannot rely on a standard ReceiverStream and would need a different approach like broadcast channel if we require broadcasting the same data to multiple streams.

# 5.  In what ways could the Rust gRPC code be structured to facilitate code reuse and modularity, promoting maintainability and extensibility over time? 
- Separate each service into its own module/file instead of putting everything in one file like main.rs, seperate them based on their function.
- Create one folder/module spesifically that contains functions that commonly used  such as how to handle error in gPRC or logging user's activity.
- Use traits in rust to define a service's interface independently of its implementation 
- Keep proto definitions organized and versioned for easier maintenance.

# 6. In the MyPaymentService implementation, what additional steps might be necessary to handle more complex payment processing logic? 
- input validation to check whether user_id is valid, amount is greater than 0, and required fields are not empty before processing
- database integration : connect to a real database to store transaction records and check user balance
- error handling :  return meaningful gRPC status codes like INVALID_ARGUMENT for bad input or NOT_FOUND if the user doesn't exist instead of always returning success: true
- ensure that if the same request is sent twice, the user doesn't get charged twice (handled via unique transaction IDs)
- verify that the requester is a valid and authorized user before processing any payment

# 7. What impact does the adoption of gRPC as a communication protocol have on the overall architecture and design of distributed systems, particularly in terms of interoperability with other technologies and platforms? 
- systems shift from a resource-based design (REST) to an action/procedure-based design, where each RPC method represents a specific operation (example ProcessPayment, GetTransactionHistory)
- Services become more tightly coupled through shared .proto contracts, it enforces consistency but requires more coordination 
- encourages icroservices architecture where each service has clearly defined responsibilities and interface
- since protobuf generates code for many languages, a rust server can easily communicate with a Java or Python client without manual serialization logic
-  protobuf supports backward-compatible schema evolution, making it easier to manage multiple versions of a service compared to REST.

# 8. What are the advantages and disadvantages of using HTTP/2, the underlying protocol for gRPC, compared to HTTP/1.1 or HTTP/1.1 with WebSocket for REST APIs? 
- HTTP/2 (gRPC) Advantages:
1. multiple requests and responses simultaneously over a single connection, reducing latency.
2. header compression using HPACK reduces overhead.
3. built-in support for server, client, and bidirectional streaming.
4. binary protocol makes data transfer faster and more efficient.

- HTTP/2 (gRPC) disadvantages:
1. binary protocol is not human-readable, making it harder to debug.
2. cannot be used directly from browsers without gRPC-Web proxy.
3. overkill for simple request-response use cases.

- HTTP/1.1 (REST) Advantages:
1. simple and human-readable text-based format, easy to debug.
2. 3niversally supported by all browsers and platforms.
4. no strict schema required, flexible and easy to iterate.

- HTTP/1.1 (REST) Disadvantages:
1. one request per connection (no multiplexing), causing higher latency.
2. no built-in streaming support, requires polling or long-polling for real-time data.
3. large header overhead on every request.

- HTTP/1.1 with WebSocket (REST) Advantages:
1. enables real-time, full-duplex communication over a single persistent connection.
2. browser compatible natively without additional proxy.
3. lower latency than polling for real-time use cases.

- HTTP/1.1 with WebSocket (REST) Disadvantages:
1. no built-in message framing or protocol structure, must be designed manually.
2. no code generation unlike gRPC.
3. no header compression, higher overhead compared to HTTP/2.

# 9. How does the request-response model of REST APIs contrast with the bidirectional streaming capabilities of gRPC in terms of real-time communication and responsiveness? 
REST follows a strict client-initiated request-response model where the client always sends a request and waits for a single response. this makes real-time communication difficult. gRPC's bidirectional streaming allows both client and server to send and receive messages independently and simultaneously over a persistent connection, making it far more suitable for real-time applications like chat, live dashboards, or collaborative tools where low latency and continuous data flow are critical

# 10. What are the implications of the schema-based approach of gRPC, using Protocol Buffers, compared to the more flexible, schema-less nature of JSON in REST API payloads? 
1. protobuf (gRPC)
- in protobuf (grpc) all fields and types are strictly defined in.proto files, all the meanig type mismatches and missing fields are caught at compile time, reducing runtime errors significantly
- binary serialization results in smaller payload sized and faster serialization/deserialization compared to json
- protobuf automatically generates client and server code from .proto files, reducing boilerplate and ensuring consistency across service
- protobuf supports backward-compatible changes (adding new fields without breaking old clients) through field numbering
- any schema change must be synchronized across all services that share the .proto file

2. JSON(REST)
- No strict schema required,  fields can be added, removed, or changed freely without breaking existing code, making it easier to iterate quickly
- text-based format makes it easy to inspect, debug, and test payloads directly using tools like Postman or browser DevTools.
- natively supported by all browsers, languages, and platforms without additional tooling.
- type errors and missing fields are only discovered at runtime, increasing the risk of bugs in production.
- text-based format results in bigger payloads compared to protobuf, which can impact performance in high-throughput systems.
- unlike protobuf, JSON has no standard way to automatically generate client/server code from a schema.