# Broker-backed cooperative nack regression (experimental)

This branch is a **candidate validation branch** for upstream PR
[spring-projects/spring-kafka#4980](https://github.com/spring-projects/spring-kafka/pull/4980).
The official PR branch has not been changed.

## Run

Requires a JDK compatible with the Spring Kafka build. From repository root:

```bash
./gradlew --no-daemon --max-workers=2 :spring-kafka:test \
  --tests org.springframework.kafka.listener.NackCooperativeRebalanceIntegrationTests
```

The test starts a real embedded Kafka broker. It does **not** require Docker,
an external broker, or a Kubernetes network failure.

## Intended sequence

1. C1 joins a single consumer group and initially owns both topic partitions.
2. C2 joins the group; the cooperative assignor splits the partitions.
3. C1 receives a record on its retained partition A and nacks it for 20 seconds.
4. C2 exits. C1 retains A and receives an additional partition B.
5. Publish a record to B while A's nack sleep is active.
6. **Patched expectation:** B must not deliver while nack sleep is active; after
   the wake, A must redeliver and B must deliver.
7. **Original expectation:** B may deliver during the active nack sleep,
   demonstrating the missing cooperative-assignment pause.

The test asserts that C1's originally assigned partition was not revoked,
that the new assignment occurred with at least two seconds of nack sleep left,
and that both partitions progress after the nack wake.

## Evidence standard

This is an *experimental*, broker-backed integration test. The two-consumer
cooperative assignment and its timing are asynchronous. Results must be
classified as one of the following:

- **Confirmed baseline regression:** unpatched implementation fails specifically
  because the newly assigned B delivered before the nack deadline.
- **Confirmed fix:** patched implementation passes the same broker-backed test.
- **Inconclusive / environment failure:** allocation did not split as expected,
  original A was revoked, the intended timing window was missed, the broker did
  not start, or compilation/build failed.

An early test failure is not automatically evidence of a bug. CI logs and
JUnit XML must be reviewed before citing the test in the upstream PR.

A separate private CI workflow compares the candidate test on the patched
branch with the same source after reverting the production fix. Until both
results have been inspected, this branch is not presented as validated.
