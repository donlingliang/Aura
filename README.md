
# Mobile System Design

### What To Expect
- Being asked to implement a feature/features (extremely vague)
- Remember that this is a discussion with time limits (~1 hour).
- Remember to mention alternatives and tradeoffs before making a technical decision.
- Remember to follow a certain pattern that works for you (I use **RADCHEB**). 


[RADCHEB](https://davescommutebloghome.wpcomstaging.com/2019/08/27/system-design-interview/) 
-   **R**equirements Clarification (6 mins)
-   **A**PI (10 mins) 
-   **D**ata Model (5 mins)
-   **C**alculate Usage Estimations (5 mins)
-   **H**igh Level Design (10 mins)
-   **E**xhaustive Design (Deep Dive) (8 mins)
-   **B**ottle Necks (10 mins)

### Requirements Clarification
``Functionals``
- Who are the actors in our system?
- Does the user need to be logged in to use the system?
- How data presents in the system (filter/sort/etc..)

``Non-functionals`` ( ask clarifying questions)
- **Scalability/Performance** (Exponential Backoff, API Rate-Limiting, Support Emerging Markets (caching for network requests, optimal bandwidth and CPU/Battery usage) TODO)
- **Engineers Resources Limitation** (How big the engineering team is(Modularization for features), help on testing at our later stages)
- **Availability** (Offline support, OS Versions, Phone Tablet Support and etc.)
- **Security** (Android specific)

``Out of scope``
- Login/Authentication.
- Analytics.
- Real-time notifications.

**Client-side only**  -  just a client-app: you have the backend and APIs available.
**Client-side + API**  -  likely choice for most interviews: you need to design a client app, data models & APIs.
**Client-side + API + Backend**  -  less likely choice since most mobile engineers would not have a proper backend experience. If the interviewer asks server-side questions - let them know that you're most comfortable with the client-side and don't have hands-on experience with backend infrastructure. It's better to be honest than trying to fake your knowledge. If they still persist - let them know that everything you know comes from sources, for backend positions do the same thing.

---
### DataModel
I usually start with the Data Model first because you’ll often want to reference different data models in the API. And then defining an interface between the client and the server is an important part of the system design process to be involved in.

Entity Relationship Diagram
- Each independent object (object name, field name & type, it is PK/FK or not)
	- make sure the data type covers the real usecases.
- Relationship tables (Name1_Name2, FK-FK)
Alternative: Another Representation instead of ERD (OpenAPI spec, not required: one advantage to the Open API Specification is that it includes a standard framework for representing API requests and responses as well as models and relationships between models which is the specialty of the ERD.)
```
definitions:
  Order:
    type: "object"
    properties:
      id:
        type: "integer"
        format: "int64"
  User:
      etc...
```
---
### API
Generally, this is expressed via a REST API.
**Simple Implementation**
We are not going to talk about attached the user-id when make request because we already had the token, but need to route back to pagination section， path(variable) vs query param)  (Response code: 2** ok/3** redirect/ 4**, 401 not authorized, 404 not there/500 server [Ref_link](https://www.ruanyifeng.com/blog/2018/10/restful-api-best-practices.html)
``@GET /address-book?page,limit -> [AddressBookItem]``
``@POST /address-book?user_id,first_name,family_name,phone -> AddressBookItem``
``@PUT /address-book?usert_id,first_name,family_name,phone -> AddressBookItem``
``@DELETE /address-book?user_id``

**Protocols** However if you’re familiar with other alternatives than this would be a good place to discuss the tradeoffs between REST, GraphQL, gRPC or even… SOAP.

- REST (A text-based stateless protocol - most popular choice for CRUD (Create, Read, Update, and Delete) operations).
	- pros: (easy to learn, understand, and implement. / easy to cache using built-in HTTP caching mechanism. / loose coupling between client and server.)
	- cons: (less efficient on mobile platforms since every request requires a separate physical connection. / schemaless - it's hard to check data validity on the client. / stateless - needs extra functionality to maintain a session. / additional overhead - every request contains contextual metadata and headers.)

- GraphQL (A query language for working with API - allows clients to request data from several resources using a single endpoint (instead of making multiple requests in traditional RESTful apps).)
	- pros: (schema-based typed queries - clients can verify data integrity and format. / highly customizable - clients can request specific data and reduce the amount of HTTP-traffic. / bi-directional communication with GraphQL Subscriptions (WebSocket based).)
	-   cons: (more complex backend implementation. / "leaky-abstraction" - clients become tightly coupled to the backend. / the performance of a query is bound to the performance of the slowest service on the backend (in case the response data is federated between multiple services).)

- WebSocket (Full-duplex communication over a single TCP connection.)
	- pros: (real time bi-directional communication. / provides both text-based and binary traffic.)
	- cons: (requires maintaining an active connection - might have poor performance on unstable cellular networks. / schemaless - it's hard to check data validity on the client. / the number of active connections on a single server is limited to 65k.)

- gRPC (Remote Procedure Call framework which runs on top of HTTP/2. Supports bi-directional streaming using a single physical connection.)
	- pros: (lightweight binary messages (much smaller compared to text-based protocols). / schema-based - built-in code generation with Protobuf. / provides support of event-driven architecture: server-side streaming, client-side streaming, and bi-directional streaming / multiple parallel requests.)
	- cons: (limited browser support. / non human-readable format. / steeper learning curve.)

**Pagination** 
Endpoints that return a list of entities must support pagination. Without pagination, a single request could return a huge amount of results causing excessive network and memory usage.
- **Offset Pagination**  
    Provides a  `limit`  and an  `offset`  query parameters. e.g.:  `GET /feed?offset=100&limit=20`.
    -   pros: (easiest to implement - the request parameters can be passed directly to a SQL query. / stateless on the server.)
    -   cons: (bad performance on large offset values (the database needs to skip  `offset`  rows before returning the paginated result). / inconsistent when adding new rows into the database (Page Drift). )
-   **Keyset Pagination**  
    Uses the values from the last page to fetch the next set of items. Example:  `GET /feed?after=2021-05-25T00:00:00&limit=20`.
    -   pros: (translates easily into a SQL query. / good performance with large datasets. / stateless on the server.)
    -   cons: ("leaky abstraction" - the pagination mechanism becomes aware of the underlying database storage. / only works on fields with natural ordering (timestamps, etc).)
-   **Cursor/Seek Pagination** （cursor is a tag for each data record）  
    Operates with stable ids which are decoupled from the database SQL queries (usually, a selected field is encoded using base64 and encrypted on the backend side). Example:  `GET /feed?next_id=hellowword&limit=20`.
    -   pros: (decouples pagination from SQL database. / consistent ordering when new items are inserted. )
    -   cons: (more complex backend implementation. / does not work well if items get deleted (ids might become invalid).)
```
{
	"data": {
		"items": [{},{},{}]
	}
	"cursor": {
		"count": 20, "next_id": "hellowword", "prev_id": null
	}
}
```

**Realtime Notification**
- Push VS Pull Model (Potential model for data update)
- Push Notification:
	- Pros: GCM / FCM, can wake the app in background
	- Cons: message delay and relying on 3rd service
- Http-Polling
	- Short Polling
		- Pros: simple and keep persistent polling to get the notification back
		- Cons: periodically ask server will burden the serve, overhead on handshake
	- Long Polling
		- Pros: instant notification
		- Cons: keep the connection till the result get back
- Server-Sent Events
	- Pros: Real time traffic with a single connection
	- Cons: keeps a persistent connection
- Web-sockets:
	- Pros: Bi-directional communication between client and server
	- Cons: Will kinda waste resources if there is no message for a long time but keeping the persistent connection.

**Bonus Point**(iterate apis/versioning/error-handling...)

---
### High Level Design
- **High Level System Design**
- **High Level Client Design** (Deep dive into one of the flows)

---
### Server Exhaustive Design (CAP, Consistency/Availability/Partition Tolerance)
- Network error handling, rate-limiting
- Offline support(Privacy/Security) vs Cloud based
- State management/Modularization
- Cache
	- Strategies(FIFO, LRU, LFU)
	- AccessPattern
		- write through cache (return success after writing both cache and db, fast read, but will have write latency)
		- write around cache (write in db, and cache read the result from db, this increasing the read burden when get a faster write)
		- write back cache (write db when async write to cache, this will lead a failure when caching layer dies)
	- Master-slave (Back up when the server is down)
	- Read and write (spilt the load for read and write)
- MultipleService

---
### Data Bottle Necks (CAP)
- Database Sharding
	- Basic Sharding (Vertical shard separate the columns and Horizontal separate by rows) (H%S, H is numeric hash amd S is the number of shards, but when shards has to be increases, H%(S+1) will be expensive for relocate the data) (other sharding are like Range, List, Round-Robin)
	- Consistent Hashing, take hash as the basic one, but hashing the cache/db as well and do clockwise for the next available shard, cons sometimes we need to balance the load
	- Advance Consistent Hashing(add virtual node on it to balancing the storage though)
	- [RefLink](http://afghl.github.io/2016/07/04/consistent-hashing.html)
- Highly Available Database
- Highly Consistent Database

---
### Reference
- [The System Design Interview For Mobile Developers](https://davescommutebloghome.wpcomstaging.com/2019/08/27/system-design-interview/)
- [Grokking the Mobile System Design interview](https://artem-goncharov.medium.com/grokking-the-mobile-system-design-interview-6a06fa94491b)
- [My Android Engineer Interview Process](https://dev.to/mjpasx/my-android-engineer-interview-process-2kn8)
- [Take-Home Android Interviews](https://medium.com/android-news/technical-interview-tests-2b4794aa0070)
- [How I cracked Job Interviews for Google, Facebook and UBER](https://blog.usejournal.com/how-i-cracked-job-interviews-for-google-facebook-and-uber-4c180458f571)
- [Caching: system design interview concepts (6 of 9)](https://igotanoffer.com/blogs/tech/caching-system-design-interview)
- [The anatomy of a horizontally scalable distributed application](https://luanjunyi.medium.com/the-anatomy-of-a-horizontal-scalable-distributed-application-25745df8badb)
- [weeeBox-mobile-system-design](https://github.com/weeeBox/mobile-system-design)
- [donnermartin-system-design-primer](https://github.com/donnemartin/system-design-primer)

> Written with [StackEdit](https://stackedit.io/).
