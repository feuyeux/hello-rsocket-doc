### Reactive Streams
- [Reactive Streams](http://www.reactive-streams.org/) (<http://www.reactive-streams.org>) 
- [Reactive Extensions](http://www.reactivex.io/) (<http://www.reactivex.io>) 
- [Reactor](https://github.com/reactor/reactor) (<https://github.com/reactor/reactor>)

#### API

| merge | map|zip| flatMap |
| ----|--|---|----|
| ![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/mergeFixedSources.svg)| ![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/mapForFlux.svg) |![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/zipIterableSourcesForFlux.svg) |![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/flatMapForFlux.svg)|

| cast |index|
|--|--- |
|![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/castForFlux.svg) |![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/index.svg)|

| skip | window |
| ---- | --- |
|![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/skipWithTimespan.svg)|![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/windowWithTimespan.svg)|

| all | any |
| --- | --- |
| ![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/all.svg) | ![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/any.svg) |

| blockFirst | blockLast |
| --- | --- |
| ![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/blockFirst.svg) | ![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/blockLast.svg) |

| buffer|collect| collectList | collectMap |
| ---|--- | --- | --- |
| ![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/buffer.svg) |![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/collectWithCollector.svg)| ![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/collectList.svg)| ![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/collectMapWithKeyExtractor.svg) |

| cache| replay|limitRate |
| -- | ----|-- |
| ![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/cacheForFlux.svg) |![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/replay.svg)| ![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/limitRate.svg) |



| generate|combineLatest | concat |
| ---- | --- |
| ![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/generate.svg)|![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/combineLatest.svg) | ![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/concatVarSources.svg) |

| delayElements | delaySequence |
| ---- | ---- |
| ![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/delayElements.svg) | ![](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/doc-files/marbles/delaySequence.svg) |



```java
class LettersSubscriber implements Subscriber<String> {
    CountDownLatch latch;

    public LettersSubscriber(CountDownLatch latch) {
        this.latch = latch;
    }

    @Override
    public void onSubscribe(Subscription s) {
        log.info("request(n)=Long.MAX_VALUE");
        s.request(Long.MAX_VALUE);
    }

    @SneakyThrows
    @Override
    public void onNext(String s) {
        TimeUnit.MILLISECONDS.sleep(100);
        log.info(s);
    }

    @Override
    public void onError(Throwable t) {
        log.error("", t);
    }

    @Override
    public void onComplete() {
        log.info("Complete!");
        latch.countDown();
    }
}
```



```java
@SneakyThrows
@Test
public void test() {
    String[] letters = "The quick brown fox jumps over a lazy dog".split("");
    Flux<String> publisher = Flux.fromArray(letters);

    Flux<String> flux1 = publisher
            .publishOn(Schedulers.newParallel("X"))
            .filter(s -> {
                log.debug("filter {}", s);
                return !s.trim().isEmpty();
            })
            .map(s -> {
                log.debug("map {}", s);
                return s.toLowerCase();
            })
            .distinct()
            .sort();

    ParallelFlux<String> flux2 = publisher
            .filter(s -> {
                log.debug("filter {}", s);
                return !s.trim().isEmpty();
            })
            .map(s -> {
                log.debug("map {}", s);
                return s.toUpperCase();
            })
            .distinct()
            .sort()
            .publishOn(Schedulers.newParallel("Y"))
            .parallel(2);

    CountDownLatch latch = new CountDownLatch(1);
    LettersSubscriber subscriber = new LettersSubscriber(latch);

    Flux<String> flux3 = flux1.publishOn(Schedulers.newParallel("Z"))
            .zipWith(flux2, (s1, s2) -> {
                log.debug("zipWith: {},{}", s1, s2);
                return String.format("%s[%d] %s[%d]",
                        s2, (int) s2.charAt(0), s1, (int) s1.charAt(0));
            })
            .limitRate(2)
            .publishOn(Schedulers.newParallel("R"));
    flux3.subscribe(subscriber);

    latch.await();

    StepVerifier.create(flux3)
            .expectNext("A[65] a[97]")
            .thenCancel()
            .verify();
}
```

