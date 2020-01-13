---
title: Reactive Systems
output:
  flexdashboard::flex_dashboard:
    theme: yeti
    orientation: rows
    vertical_layout: scroll
    includes:
      after_body: footer.html

---

> [Reactive Manifesto](http://reactivemanifesto.org/)
> 
>  we want systems that are Responsive, Resilient, Elastic and Message Driven. We call these Reactive Systems.


# Java  

![](img/rr.png){ width=600 }

Row {data-height=240}
-------------------------------------

### Request (request —> <— response)
    
```java
Payload payload = DefaultPayload.create(JSON.toJSONString(helloRequest));
CountDownLatch c = new CountDownLatch(1);
socket.requestResponse(payload)
        .subscribe(p -> {
            HelloResponse response = JSON.parseObject(p.getDataUtf8(), HelloResponse.class);
            log.info("<< [Request-Response] response id:{},value:{}", response.getId(), response.getValue());
            c.countDown();
        });
c.await();
```

### Response (request —> <— response)
    
```java
public Mono<Payload> requestResponse(Payload payload) {
    HelloRequest helloRequest = JSON.parseObject(payload.getDataUtf8(), HelloRequest.class);
    log.info(" >> [Request-Response] data:{}", helloRequest);
    String id = helloRequest.getId();
    HelloResponse helloResponse = getHello(id);
    return Mono.just(DefaultPayload.create(JSON.toJSONString(helloResponse)));
}
```

Row {data-height=280}
-------------------------------------
![](img/rs.png){ width=600 }

Row {data-height=275}
-------------------------------------
    
### Request (request —> <— <— stream)
    
```java
List<String> ids = HelloUtils.getRandomIds(5);
Payload payload = DefaultPayload.create(JSON.toJSONString(new HelloRequests(ids)));
CountDownLatch c = new CountDownLatch(5);
socket.requestStream(payload).subscribe(p -> {
    HelloResponse response = JSON.parseObject(p.getDataUtf8(), HelloResponse.class);
    log.info("<< [Request-Stream] response id:{},value:{}", response.getId(), response.getValue());
    c.countDown();
});
c.await();
```

### Response (request —> <— <— stream)
    
```java
public Flux<Payload> requestStream(Payload payload) {
  HelloRequests helloRequests = JSON.parseObject(payload.getDataUtf8(), HelloRequests.class);
  log.info(">> [Request-Stream] data:{}", helloRequests);
  List<String> ids = helloRequests.getIds();
  return Flux.fromIterable(ids)
          .delayElements(Duration.ofMillis(500))
          .map(id -> {
              HelloResponse helloResponse = getHello(id);
              return DefaultPayload.create(JSON.toJSONString(helloResponse));
          });
}
```

Row {data-height=500}
-------------------------------------
![](img/rc.png){ width=600 }

Row {data-height=420}
-------------------------------------
    
### Request (request channel —> —> <— —> <—)
    
```java
CountDownLatch c = new CountDownLatch(TIMES * 3);

Flux<Payload> send = Flux.<Payload>create(emitter -> {
    for (int i = 1; i <= TIMES; i++) {
        List<String> ids = HelloUtils.getRandomIds(3);
        Payload payload = DefaultPayload.create(JSON.toJSONString(new HelloRequests(ids)));
        emitter.next(payload);
    }
    emitter.complete();
}).delayElements(Duration.ofMillis(1000));

socket.requestChannel(send).subscribe(p -> {
    HelloResponse response = JSON.parseObject(p.getDataUtf8(), HelloResponse.class);
    log.info("<< [Request-Channel] response id:{},value:{}", response.getId(), response.getValue());
    c.countDown();
});
c.await();
```

### Response (request channel —> —> <— —> <—)
    
```java
public Flux<Payload> requestChannel(Publisher<Payload> payloads) {
    final Scheduler scheduler = Schedulers.parallel();

    return Flux.from(payloads)
            .doOnNext(payload -> {
                log.info(">> [Request-Channel] data:{}", payload.getDataUtf8());
            })
            .map(payload -> {
                HelloRequests helloRequests = JSON.parseObject(payload.getDataUtf8(), HelloRequests.class);
                return helloRequests.getIds();
            })
            .flatMap(HelloRSocket::apply)
            .map(id -> {
                HelloResponse helloResponse = getHello(id);
                return DefaultPayload.create(JSON.toJSONString(helloResponse));
            })
            .subscribeOn(scheduler);
}
```

Row {data-height=200}
-------------------------------------
![](img/fnf.png){ width=600 }

Row {data-height=160}
-------------------------------------
    
### Request (fire and forget —>!)
    
```java
HelloRequest helloRequest = new HelloRequest("1");
Payload payload = DefaultPayload.create(JSON.toJSONString(helloRequest));
socket.fireAndForget(payload).block();
```

### Response (fire and forget —>!)
    
```java
public Mono<Void> fireAndForget(Payload payload) {
    HelloRequest helloRequest = JSON.parseObject(payload.getDataUtf8(), HelloRequest.class);
    log.info(">> [FireAndForget] FNF:{}", helloRequest);
    return Mono.empty();
}
```

Row {data-height=200}
-------------------------------------
![](img/mp.png){ width=600 }

Row {data-height=160}
-------------------------------------
    
### Request (meta push —>!)
    
```java
Payload payload = DefaultPayload.create(new byte[]{}, "JAVA".getBytes());
socket.metadataPush(payload).block();
```

### Response (meta push —>!)
    
```java
public Mono<Void> metadataPush(Payload payload) {
    String metadata = payload.getMetadataUtf8();
    log.info(">> [MetadataPush]:{}", metadata);
    return Mono.empty();
}
```

Row {data-height=50}
-------------------------------------
- <https://github.com/feuyeux/hello-rsocket-java/blob/master/ultimate/src/main/java/org/feuyeux/rsocket/RSocketClient.java>
- <https://github.com/feuyeux/hello-rsocket-java/blob/master/ultimate/src/main/java/org/feuyeux/rsocket/ultimate/HelloRSocket.java>

# Go  

![](img/rr.png){ width=600 }

Row {data-height=320}
-------------------------------------

### Request (request —> <— response)
    
```go
client, _ := BuildClient()
defer client.Close()
// Send request
request := &common.HelloRequest{Id: "1"}
json, _ := request.ToJson()
//p := payload.New(json, []byte(Now()))
p := payload.New(json, nil)
result, err := client.RequestResponse(p).Block(context.Background())
if err != nil {
    log.Println(err)
}
data := result.Data()
response := common.JsonToHelloResponse(data)
```

### Response (request —> <— response)
    
```go
rsocket.RequestResponse(func(p payload.Payload) mono.Mono {
    data := p.Data()
    request := common.JsonToHelloRequest(data)
    metadata, _ := p.MetadataUTF8()
    log.Println(">> [Request-Response] data:", request, ", metadata:", metadata)
    id := request.Id
    index, _ := strconv.Atoi(id)
    response := common.HelloResponse{Id: id, Value: helloList[index]}
    json, _ := response.ToJson()
    meta, _ := p.Metadata()
    return mono.Just(payload.New(json, meta))
})
```

Row {data-height=280}
-------------------------------------
![](img/rs.png){ width=600 }

Row {data-height=570}
-------------------------------------
    
### Request (request —> <— <— stream)
    
```go
cli, _ := BuildClient()
defer cli.Close()
ids := RandomIds(5)

request := &common.HelloRequests{}
request.Ids = ids
json, _ := request.ToJson()
p := payload.New(json, []byte(Now()))
f := cli.RequestStream(p)
```

### Response (request —> <— <— stream)
    
```go
rsocket.RequestStream(func(p payload.Payload) flux.Flux {
    data := p.Data()
    request := common.JsonToHelloRequests(data)
    log.Println(">> [Request-Stream] data:", request)

    return flux.Create(func(ctx context.Context, emitter flux.Sink) {
        for i := range request.Ids {
            // You can use context for graceful coroutine shutdown, stop produce.
            select {
            case <-ctx.Done():
                log.Println(">> [Request-Stream] ctx done:", ctx.Err())
                return
            default:
                id := request.Ids[i]
                index, _ := strconv.Atoi(id)
                response := common.HelloResponse{Id: id, Value: helloList[index]}
                json, _ := response.ToJson()
                meta, _ := p.Metadata()
                rp := payload.New(json, meta)
                emitter.Next(rp)
                time.Sleep(500 * time.Millisecond)
            }
        }
        emitter.Complete()
    })
})
```

Row {data-height=500}
-------------------------------------
![](img/rc.png){ width=600 }

Row {data-height=450}
-------------------------------------
    
### Request (request channel —> —> <— —> <—)
    
```go
cli, _ := BuildClient()
defer cli.Close()

send := flux.Create(func(i context.Context, sink flux.Sink) {
    for i := 1; i <= 3; i++ {
        request := &common.HelloRequests{}
        request.Ids = RandomIds(3)
        json, _ := request.ToJson()
        p := payload.New(json, []byte(Now()))
        sink.Next(p)
        time.Sleep(100 * time.Millisecond)
    }
    time.Sleep(1000 * time.Millisecond)
    sink.Complete()
})

f := cli.RequestChannel(send)
```

### Response (request channel —> —> <— —> <—)
    
```go
rsocket.RequestChannel(func(payloads rx.Publisher) flux.Flux {
    return flux.Create(func(i context.Context, sink flux.Sink) {
        payloads.(flux.Flux).
            SubscribeOn(scheduler.Elastic()).
            DoOnNext(func(p payload.Payload) {
                data := p.Data()
                //request := common.JsonToHelloRequest(data)
                request := common.JsonToHelloRequests(data)
                log.Println(">> [Request-Channel] data:", request)
                for _, id := range request.Ids {
                    index, _ := strconv.Atoi(id)
                    response := common.HelloResponse{Id: id, Value: helloList[index]}
                    json, _ := response.ToJson()
                    sink.Next(payload.New(json, nil))
                }
            }).
            Subscribe(context.Background())
        //sink.Complete()
    })
})
```

Row {data-height=200}
-------------------------------------
![](img/fnf.png){ width=600 }

Row {data-height=160}
-------------------------------------
    
### Request (fire and forget —>!)
    
```go
client, _ := BuildClient()
defer client.Close()
request := &common.HelloRequest{Id: "1"}
json, _ := request.ToJson()
client.FireAndForget(payload.New(json, nil))
```

### Response (fire and forget —>!)
    
```go
rsocket.FireAndForget(func(p payload.Payload) {
    data := p.Data()
    request := common.JsonToHelloRequest(data)
    log.Println(">> [FireAndForget] FNF:", request.Id)
})
```

Row {data-height=200}
-------------------------------------
![](img/mp.png){ width=600 }

Row {data-height=160}
-------------------------------------
    
### Request (meta push —>!)
    
```go
client, _ := BuildClient()
defer client.Close()
client.MetadataPush(payload.New(nil, []byte("GOLANG")))
```

### Response (meta push —>!)
    
```go
rsocket.MetadataPush(func(p payload.Payload) {
    meta, _ := p.MetadataUTF8()
    log.Println(">> [MetadataPush]:", meta)
})
```

Row {data-height=50}
-------------------------------------
- <https://github.com/feuyeux/hello-rsocket-golang/blob/master/src/requester/request.go>
- <https://github.com/feuyeux/hello-rsocket-golang/blob/master/src/responder/acceptor.go>


# Rust  

![](img/rr.png){ width=600 }

Row {data-height=340}
-------------------------------------

### Request (request —> <— response)
    
```rust
let request = HelloRequest { id: "1".to_owned() };
let json_data = request_to_json(&request);
let p = Payload::builder()
    .set_data(Bytes::from(json_data))
    .set_metadata_utf8("RUST")
    .build();

let resp: Payload = self.client.request_response(p).await.unwrap();
let data = resp.data();

let hello_response = data_to_response(data);
println!("<< [request_response] response id:{},value:{}", hello_response.id, hello_response.value);
```

### Response (request —> <— response)
    
```rust
fn request_response(&self, req: Payload) -> Mono<Result<Payload, RSocketError>> {
    let request = data_to_request(req.data());
    println!(
        ">> [request_response] data:{:?}, meta={:?}", request, req.metadata()
    );
    let index = request.id.parse::<usize>().unwrap();
    let response = HelloResponse { id: request.id, value: HELLO_LIST[index].to_string() };
    let json_data = response_to_json(&response);
    let p = Payload::builder()
        .set_data(Bytes::from(json_data))
        .set_metadata_utf8("RUST")
        .build();
    Box::pin(future::ok::<Payload, RSocketError>(p))
}
```

Row {data-height=280}
-------------------------------------
![](img/rs.png){ width=600 }

Row {data-height=380}
-------------------------------------
    
### Request (request —> <— <— stream)
    
```rust
let request = HelloRequests { ids: RequestCoon::random_ids(5) };
let json_data = requests_to_json(&request);
let sending = Payload::builder()
    .set_data(Bytes::from(json_data))
    .set_metadata_utf8("RUST")
    .build();
let mut results = self.client.request_stream(sending);
loop {
    match results.next().await {
        Some(v) => {
            let data = v.data();
            let hello_response = data_to_response(data);
            println!("<< [request_stream] response:{:?}", hello_response)
        }
        None => break,
    }
}
```

### Response (request —> <— <— stream)
    
```rust
fn request_stream(&self, req: Payload) -> Flux<Payload> {
    let request = data_to_requests(req.data());
    println!(">> [request_stream] data:{:?}", request);
    let mut results = vec![];
    for _id in request.ids {
        let index = _id.parse::<usize>().unwrap();
        let response = HelloResponse { id: _id, value: HELLO_LIST[index].to_string() };
        let json_data = response_to_json(&response);
        let p = Payload::builder()
            .set_data(Bytes::from(json_data))
            .set_metadata_utf8("RUST")
            .build();
        results.push(p);
    }
    Box::pin(futures::stream::iter(results))
}
```

Row {data-height=500}
-------------------------------------
![](img/rc.png){ width=600 }

Row {data-height=450}
-------------------------------------
    
### Request (request channel —> —> <— —> <—)
    
```rust
let mut sends = vec![];
for _ in 0..3 {
    let request = HelloRequests { ids: RequestCoon::random_ids(3) };
    let json_data = requests_to_json(&request);
    let sending = Payload::builder()
        .set_data(Bytes::from(json_data))
        .set_metadata_utf8("RUST")
        .build();
    sends.push(sending);
}
let iter = stream::iter(sends);
let pin = Box::pin(iter);
let mut resps = self.client.request_channel(pin);

sleep(Duration::from_millis(1000));
while let Some(v) = resps.next().await {
    let data = v.data();
    let hello_response = data_to_response(data);
    println!("<< [request_channel] response:{:?}", hello_response);
}
```

### Response (request channel —> —> <— —> <—)
    
```rust
fn request_channel(&self, mut reqs: Flux<Payload>) -> Flux<Payload> {
    let (sender, receiver) = mpsc::unbounded_channel::<Payload>();
    tokio::spawn(async move {
        while let Some(p) = reqs.next().await {
            let request = data_to_requests(p.data());
            println!(">> [request_channel] data:{:?}", request);
            for _id in request.ids {
                let index = _id.parse::<usize>().unwrap();
                let response = HelloResponse { id: _id, value: HELLO_LIST[index].to_string() };
                let json_data = response_to_json(&response);
                let resp = Payload::builder()
                    .set_data(Bytes::from(json_data))
                    .set_metadata_utf8("RUST")
                    .build();
                sender.send(resp).unwrap();
            }
        }
    });
    Box::pin(receiver)
}
```

Row {data-height=200}
-------------------------------------
![](img/fnf.png){ width=600 }

Row {data-height=160}
-------------------------------------
    
### Request (fire and forget —>!)
    
```rust
let request = HelloRequest { id: "1".to_owned() };
let json_data = request_to_json(&request);
let fnf = Payload::builder().set_data(Bytes::from(json_data)).build();
self.client.fire_and_forget(fnf).await;
```

### Response (fire and forget —>!)
    
```rust
fn fire_and_forget(&self, req: Payload) -> Mono<()> {
    let request = data_to_request(req.data());
    println!(">> [fire_and_forget] FNF:{}", request.id);
    Box::pin(async {})
}
```

Row {data-height=200}
-------------------------------------
![](img/mp.png){ width=600 }

Row {data-height=150}
-------------------------------------
    
### Request (meta push —>!)
    
```rust
let meta = Payload::builder().set_metadata_utf8("RUST").build();
self.client.metadata_push(meta).await;
```

### Response (meta push —>!)
    
```rust
fn metadata_push(&self, req: Payload) -> Mono<()> {
    println!(">> [metadata_push]: {:?}", req);
    Box::pin(async {})
}
```

Row {data-height=50}
-------------------------------------
- <https://github.com/feuyeux/hello-rsocket-rust/blob/master/src/requester/mod.rs>
- <https://github.com/feuyeux/hello-rsocket-rust/blob/master/src/responder/mod.rs>


