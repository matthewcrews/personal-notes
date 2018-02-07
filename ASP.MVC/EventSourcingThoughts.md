# Event Sourcing Thoughts

This is a collection of my thoughts on Event Sourcing and the implementation difficulties I may have.

## Design Idea

Perhaps the way to do this is to have a ASP.NET api which just exposes the controllers which take requests from the client. The client doesn't actually change the domain but instead emits requests to change the domain and the subscribes to updates. An example would be wanting to add an Order Line to an Order. The Order itself is the Aggregate.  If I moddled the request as an F# record it would look something like this:

```fsharp
let aggregate<'a> = {
    AggregateId : Guid
    Version : int
    Body : 'a
}

let request<'a> = {
    RequestId : Guid
    UserId : string
    TimeStamp : DateTimeOffset
    RequestType : string
    Body : 'a
}


let addOrderLineBody = {
    AggregateId : Guid
    ItemCode : string
    Amount : decimal
}
```

## Client Request Pipeline

### Client Generates Request

The User will be using a Client of some kind whether it be a native app or a web app. The app displays information based on the latest version of the domain. When the user wants to update the domain they will request that an update be processed. This would take the form of creating an order or adding an order line. This does NOT actually change the domain immediately. What happens is that a Request is sent to the API Service.

### Api Service Receives Request

When the Client sends a Request to the API service authentication and authorization occur. This service could be as simple as an ASP.NET backend with a set of controllers for handling the requests. If the Client passes the security checks then a `RequestEvent` is generated and added to the `RequestEventStream`. The API responds with a token which corresponds to the event that was generated. The Client can poll for responses to the request using this token. A user could make the Client actively update the UI when the Request is received. It could query the Aggregate once it received confirmation that the request had been completed.

### Service stores the request in the Request Event Stream

### Request Processor Picks up Request

- Request Processer continually processes new Request events. When the new event appears in the Request Event Stream it will pull them and process them one at a time. If you need to go faster you could either create a separate processor per aggregate or have a deterministic hash which mapped the request to an Aggregate Set handler.
- The Request processor will query for the Aggregate. If none exists then the request must have a Version of 0. This indicates that the request was made at a time that the Aggregate did not exist. If the Aggregate does exist then the Version of the Request and the Version of the Aggregate are compared. If they do not match then an Request Processed Event will be generated with a type of `Version Misalignment Error`. The Client will be polling for Request Processed Events looking for the one with the matching RequestId.
- If the Request Aggegrate Version and the Aggregate Version do match then the Request Body will be processed. This takes the form of a Command in Event Sourcing parlance. If the Request cannot be completed then a `RequestProcessedEvent` will be stored and emitted of the type `BadRequest`. Additional information can be included which identifies exactly what went on.
- If the Request can be succesfully completed then the resulting event will be stored in the corresponding Aggregates Event Stream. Each Aggregate actually has its own separate set of events which make it up. To hydrate the Aggregate you need to pull the history of events and apply them in order. This takes the form of a left fold in functional programming.

---

**Note:** The initial prototype for this will just have one Request Event Processor. This is for simplicity. There is some concern over throughput though. If too many Request Events stack up then the Processor will not be able to keep up. To scale out I can see two strategies 1) spin up a separate processor per Aggregate or 2) implement a hashing function which determinitically routes work to processors based on the AggregateId. The first strategy is again simply but likely not tenable. If there are many Aggregates in the system then a lot of Processors may get spun up. You may be able to handle this though by actively disposing of processors when Aggegrates are no longer seeing traffic. This means that you would need some kind of periodic clean up. The second option requires having a hashing function which can evenly divide the load. The downside here is that you could occasionaly get load which heavily depends on a single node. Solving this problem is beyond the scope of this exercise but it's a place for further thought should the need arrise.

---