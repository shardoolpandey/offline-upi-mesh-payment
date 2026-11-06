# Interview Question Bank: UPI Offline Mesh

Last synced: 2026-11-05

**Levels:** `L1` = up to ~10 LPA (what it is, what the code does) · `L2` = ~10-15 LPA (why, internals, gotchas) · `L3` = ~15-20 LPA (trade-offs, production, redesign)

## Contents
1. [Project story](#1-project-story)
2. [Request flow and architecture](#2-request-flow-and-architecture)
3. [Cryptography](#3-cryptography)
4. [Idempotency and concurrency](#4-idempotency-and-concurrency)
5. [Settlement, JPA and transactions](#5-settlement-jpa-and-transactions)
6. [Spring Boot concepts used](#6-spring-boot-concepts-used)
7. [Mesh simulator and dashboard](#7-mesh-simulator-and-dashboard)
8. [Testing](#8-testing)
9. [Security and production readiness](#9-security-and-production-readiness)
10. [Roadmap (NOT implemented)](#10-roadmap-not-implemented)

---

## 1. Project story

### Walk me through your project in two minutes.
`level: L1` · `sources: README.md, service/BridgeIngestionService.java, service/MeshSimulatorService.java`

**Answer:** It is a Spring Boot backend for deferred UPI settlement over an offline mesh. A sender phone with no internet encrypts a `PaymentInstruction` into a `MeshPacket`. Nearby phones copy it between themselves (gossip) until a phone with 4G, the bridge, POSTs it to `/api/bridge/ingest`. The server hashes the ciphertext, claims the hash to drop duplicates, decrypts, checks freshness and settles in one DB transaction. The Bluetooth mesh is simulated by `MeshSimulatorService`; the server side is the real part.

**Follow-ups:**
1. *Is the Bluetooth part real?* No. `VirtualDevice` objects held in a `ConcurrentHashMap` stand in for phones. Real BLE would be an Android client, which this repo does not contain.
2. *What was the hardest part?* Making sure the same packet arriving via several bridges at once settles exactly once. I solved it with an atomic `putIfAbsent` claim on the ciphertext hash, with a unique DB index as a backstop.
3. *What would you call this honestly?* Mesh-routed deferred settlement, not real-time offline UPI, because the receiver cannot confirm funds until the server settles.

### What problem does this solve, and what does it not solve?
`level: L2` · `sources: README.md, service/SettlementService.java`

**Answer:** It lets a payment instruction travel through untrusted offline devices and settle safely later. It does not solve fund verification: the receiver gets an IOU, and if the sender's balance is too low at settlement, `SettlementService` records a `REJECTED` transaction. It also cannot stop offline double-spending, since two packets spending the same balance race to the server and the first wins.

**Follow-ups:**
1. *How does real offline UPI handle this?* UPI Lite uses a pre-funded, hardware-backed wallet so the amount is already reserved offline.
2. *Could you reduce the double-spend risk?* Yes, with per-device offline spending limits and a pre-reserved balance (an escrow). Neither is implemented here.

### What would you do differently if you rebuilt it?
`level: L3` · `sources: service/BridgeIngestionService.java, service/IdempotencyService.java, crypto/ServerKeyHolder.java`

**Answer:** I would add sender signatures so the server can authenticate who created a packet, store the idempotency outcome rather than just a claim, move the cache to Redis and the keys to KMS, add mutual TLS and rate limiting on the ingest endpoint, and use a real database with migrations instead of `create-drop` H2.

**Follow-ups:**
1. *Which first?* Sender authentication, because today nothing proves who created a packet.
2. *Why store the outcome with the claim?* So a retry after a transient failure can be told apart from a true duplicate (see section 4).

---

## 2. Request flow and architecture

### What are the layers and what does each class do?
`level: L1` · `sources: controller/ApiController.java, service/*.java, crypto/*.java, model/*.java`

**Answer:** `ApiController` is the REST layer. Services hold the logic: `BridgeIngestionService` (pipeline), `IdempotencyService` (duplicate gate), `SettlementService` (ledger update), `DemoService` (simulated sender and account seeding), `MeshSimulatorService` and `VirtualDevice` (mesh). `crypto` has `HybridCryptoService` and `ServerKeyHolder`. `model` has JPA entities, repositories and the DTOs `MeshPacket` and `PaymentInstruction`.

**Follow-ups:**
1. *Which endpoint is the "real" one?* `POST /api/bridge/ingest`. The `/api/mesh/*` and `/api/demo/*` endpoints exist only for the simulator.
2. *How does the dashboard get updates?* `dashboard.html` calls the REST endpoints with `fetch` and refreshes every 3 seconds via `setInterval`.

### Trace one payment from click to ledger entry.
`level: L2` · `sources: controller/ApiController.java, service/DemoService.java, service/MeshSimulatorService.java, service/BridgeIngestionService.java, service/SettlementService.java`

**Answer:** `POST /api/demo/send` calls `DemoService.createPacket`, which builds a `PaymentInstruction` (PIN hash, random nonce, `signedAt`), encrypts it, and wraps it in a `MeshPacket` with TTL 5. `MeshSimulatorService.inject` puts it on `phone-alice`. `POST /api/mesh/gossip` copies it to all devices with TTL minus 1. `POST /api/mesh/flush` collects packets from internet-enabled devices and calls `BridgeIngestionService.ingest` in a parallel stream. `ingest` runs hash, claim, decrypt, freshness, then `SettlementService.settle`.

**Follow-ups:**
1. *Where does the hop count come from?* In `meshFlush` it is `5 - packet.getTtl()`. A packet copied once has TTL 4, so hops = 1, matching the ledger in the dashboard. The 5 is hardcoded, so it breaks if a different TTL is passed to `/demo/send`.
2. *What does the real bridge send instead?* It sends the `X-Hop-Count` and `X-Bridge-Node-Id` headers, which `ingest` reads with defaults `0` and `unknown`.

### Why does the pipeline run in this order?
`level: L3` · `sources: service/BridgeIngestionService.java`

**Answer:** Cheapest checks first. Hashing is cheap, the idempotency claim is an in-memory atomic op, and RSA decryption is expensive, so duplicates never reach it. Freshness runs after decryption because `signedAt` is inside the encrypted payload. Settlement is last because it touches the DB.

**Follow-ups:**
1. *Is there a downside to claiming before decrypting?* Yes. Junk or tampered ciphertext still consumes a cache entry, so an attacker can grow the map until eviction. Rate limiting and a size cap would mitigate it.
2. *Could this pipeline be async?* Yes. The endpoint could enqueue after the claim and return 202, with settlement in a worker. The trade-off is you then need a way to report the outcome back.

---

## 3. Cryptography

### What is hybrid encryption and why did you use it?
`level: L1` · `sources: crypto/HybridCryptoService.java`

**Answer:** AES-256-GCM encrypts the payload, which is fast and unlimited in size. RSA-2048 with OAEP/SHA-256 encrypts only the one-time AES key. RSA alone is too slow and can encrypt only about 245 bytes. AES alone has no safe way to share its key. The wire format is `[256-byte RSA-encrypted key][12-byte IV][ciphertext + 16-byte GCM tag]`, base64 encoded.

**Follow-ups:**
1. *Where does the 256 come from?* A 2048-bit RSA key produces 256-byte output. `RSA_ENCRYPTED_KEY_BYTES` is hardcoded to that, so changing the key size would break decryption.
2. *Is this the same as TLS?* Same idea (asymmetric for key exchange, symmetric for data), but TLS also adds certificates, forward secrecy and a handshake. This is a one-way, single-message scheme.

### Why AES-GCM and not CBC?
`level: L2` · `sources: crypto/HybridCryptoService.java`

**Answer:** GCM is authenticated encryption. The tag makes `doFinal` throw if any bit changes, so a malicious intermediate cannot alter the packet undetected. CBC gives only confidentiality and would need a separate MAC.

**Follow-ups:**
1. *What happens if you reuse a GCM IV with the same key?* It is catastrophic, leaking plaintext relationships and allowing forgery. Here each packet gets a fresh random 12-byte IV and a fresh AES key, so reuse does not happen.
2. *What does OAEP add over plain RSA?* Randomized padding, so encrypting the same key twice gives different output, and protection against padding-oracle style attacks that hit PKCS#1 v1.5.
3. *What exception does tampering cause?* `AEADBadTagException` from `doFinal`, which `BridgeIngestionService` catches and returns as `INVALID` with reason `decryption_failed`.

### Why hash the ciphertext for the idempotency key rather than packetId or the plaintext?
`level: L2` · `sources: crypto/HybridCryptoService.java, model/MeshPacket.java`

**Answer:** `packetId` is in the outer, unauthenticated part, so an intermediate can rewrite it and make a duplicate look new. The plaintext needs an RSA decryption first, which defeats the point of cheap dedup. The ciphertext is covered by the GCM tag, so changing it invalidates the packet.

**Follow-ups:**
1. *The README says identical ciphertexts happen because "AES is deterministic". Is that right?* Not quite. `encrypt` uses a random key and IV each call, so two separate encryptions of the same payment differ. Copies match only because gossip copies the same ciphertext string. A sender who re-encrypts and resends creates a different hash, which is why the nonce matters for intent, not for dedup.
2. *Any weakness in `hashCiphertext`?* It calls `getBytes()` without a charset. For ASCII base64 it is harmless, but `StandardCharsets.UTF_8` is safer.
3. *Could someone mutate the ciphertext to get a fresh hash and a second settlement?* Any modification fails the GCM tag, so it returns `INVALID`. Only the original bytes can settle.

### How does the project prevent replay?
`level: L2` · `sources: service/BridgeIngestionService.java, service/IdempotencyService.java, resources/application.properties`

**Answer:** Two layers. `signedAt` and the nonce are inside the encrypted payload, so they cannot be edited. `ingest` rejects packets older than `upi.mesh.packet-max-age-seconds` (86400) or more than 300 seconds in the future. A byte-identical replay is caught by the idempotency claim. The two windows are both 86400 on purpose: the cache must remember a hash at least as long as the packet stays valid, or a replay could slip through after eviction.

**Follow-ups:**
1. *What if someone changed only the freshness config?* If `packet-max-age-seconds` exceeded `idempotency-ttl-seconds`, a replay could pass after the hash is evicted from the cache. The DB unique index would still block double settlement for settled and rejected packets.
2. *Why a 300-second future tolerance?* Sender phone clocks can be ahead of the server. Offline phones cannot sync time, so some skew is expected.

---

## 4. Idempotency and concurrency

### What is idempotency and how do you implement it?
`level: L1` · `sources: service/IdempotencyService.java`

**Answer:** Idempotency means processing the same request many times has the same effect as once. `IdempotencyService.claim(hash)` calls `ConcurrentHashMap.putIfAbsent` and returns true only if there was no previous entry. The first caller settles; later callers get `DUPLICATE_DROPPED`.

**Follow-ups:**
1. *Why is `putIfAbsent` safe under concurrency?* It is atomic in `ConcurrentHashMap`, so exactly one of N simultaneous callers sees `null`.
2. *Why not `containsKey` then `put`?* That is check-then-act, so two threads can both see "absent" and both proceed.
3. *What is the production equivalent?* Redis `SET key value NX EX 86400`, which is atomic and shared across servers.

### What are the limits of the in-memory cache?
`level: L3` · `sources: service/IdempotencyService.java, model/Transaction.java`

**Answer:** It is JVM-local, so it fails with multiple instances (each has its own map) and is lost on restart. The `@Scheduled` `evictExpired` method (every 60 seconds) removes entries past the TTL. The safety net is the unique index on `transactions.packet_hash`: a second settlement attempt for the same hash fails at the database.

**Follow-ups:**
1. *What happens on the DB fallback path?* The second insert throws a constraint violation, `ingest` catches it in the outer `catch` and returns `INVALID` with `internal_error`. Money is safe, but the error message is misleading, and the rollback also undoes the balance changes.
2. *How would you scale it?* Redis for the claim, plus the DB constraint. For multi-region you need a single source of truth for the hash, or sticky routing by hash.

### What happens if settlement fails after the claim succeeded?
`level: L3` · `sources: service/BridgeIngestionService.java, service/SettlementService.java`

**Answer:** This is a real gap. The claim is never released. If `settle` throws (unknown VPA, optimistic-lock failure, DB error), the transaction rolls back but the hash stays in the cache, so a legitimate retry of the same packet returns `DUPLICATE_DROPPED` until the TTL expires. The payment is effectively lost. The fix is to release the claim on failure, or store an outcome (`PROCESSING`, `SETTLED`, `FAILED`) with the hash so retries can resume.

**Follow-ups:**
1. *Is a REJECTED (insufficient funds) packet retryable?* No by design. `recordRejected` saves a row with the same unique `packetHash`, so even after the sender is funded the same packet cannot settle later.
2. *How would storing an outcome change the API?* A duplicate could return the original result (same transaction ID) instead of a bare `DUPLICATE_DROPPED`, which is what payment APIs like Stripe do.

### Two different packets from the same sender arrive at the same moment. What happens?
`level: L3` · `sources: model/Account.java, service/SettlementService.java, service/BridgeIngestionService.java`

**Answer:** Different packets have different hashes, so both pass the claim and both run `settle` concurrently. Both read the same `Account` version. The first commit increments `@Version`; the second commit fails with an optimistic-locking exception, which rolls back its transaction. That prevents a lost update, so the balance stays correct. But `ingest` turns the exception into `INVALID: internal_error` and the claim stays held, so the second payment is dropped rather than retried.

**Follow-ups:**
1. *How would you retry it?* Catch `ObjectOptimisticLockingFailureException` in the ingest layer, release the claim, and retry a few times. Or use a pessimistic lock (`SELECT ... FOR UPDATE`) on the sender account.
2. *Optimistic vs pessimistic here?* Optimistic is good when conflicts are rare. A hot sender account with many in-flight packets favors pessimistic locking or a per-account queue.
3. *Is this scenario tested?* No. The existing test covers the same packet three times, not different packets on one account.

---

## 5. Settlement, JPA and transactions

### What does `@Transactional` do on `settle`?
`level: L1` · `sources: service/SettlementService.java`

**Answer:** It wraps the method in one database transaction. The debit, the credit and the ledger insert all commit together, or all roll back if an exception is thrown.

**Follow-ups:**
1. *Does it roll back on every exception?* By default only on unchecked exceptions (`RuntimeException` and `Error`). `settle` throws `IllegalArgumentException`, which is unchecked, so it rolls back.
2. *Why does it work when called from `BridgeIngestionService`?* Spring applies it through a proxy, and the call comes from another bean. A call from inside the same class (self-invocation) would bypass the proxy.

### What is `@Version` on `Account`?
`level: L2` · `sources: model/Account.java`

**Answer:** It enables optimistic locking. Hibernate adds the version to the `UPDATE ... WHERE id=? AND version=?`. If another transaction already changed the row, zero rows update and Hibernate throws an optimistic-lock exception. It prevents lost updates without holding database locks.

**Follow-ups:**
1. *Why is it needed if idempotency already exists?* Idempotency stops the same packet twice. It does not stop two different packets from racing on one account.
2. *Why `BigDecimal` for money?* Floating-point types cannot represent decimals exactly. The column is `precision = 19, scale = 2`.

### How is the uniqueness of a transaction enforced?
`level: L2` · `sources: model/Transaction.java`

**Answer:** `packetHash` has `unique = true` and a unique index `idx_packet_hash`. It is the last line of defense if the cache layer fails. `Status` is an enum stored as a string (`SETTLED` or `REJECTED`) via `@Enumerated(EnumType.STRING)`.

**Follow-ups:**
1. *Why `EnumType.STRING` over ordinal?* Ordinal breaks if someone reorders the enum values.
2. *Why `GenerationType.IDENTITY`?* The DB assigns the ID; it works well with H2 and auto-increment columns.

### What would you change about the persistence setup for production?
`level: L3` · `sources: resources/application.properties`

**Answer:** `ddl-auto=create-drop` wipes the schema on shutdown and H2 is in-memory, so all data is lost on restart. I would use PostgreSQL, Flyway migrations, `validate` for DDL, and disable the H2 console (`spring.h2.console.enabled=true` exposes the DB at `/h2-console`).

**Follow-ups:**
1. *Isolation level?* The default (READ COMMITTED on most DBs). With `@Version` that is enough for balance updates; stricter levels would add retries.
2. *How would you reconcile the ledger?* A periodic job summing transactions versus balances, plus alerts on mismatch.

---

## 6. Spring Boot concepts used

### What does `@SpringBootApplication` and `@RestController` do here?
`level: L1` · `sources: UpiMeshApplication.java, controller/ApiController.java`

**Answer:** `@SpringBootApplication` combines configuration, auto-configuration and component scanning. `@RestController` makes `ApiController` a bean whose return values are serialized to JSON by Jackson. `@RequestMapping("/api")` sets the base path.

**Follow-ups:**
1. *How does `@RequestBody` work?* Jackson converts the JSON body into `MeshPacket` using its no-arg constructor and setters.
2. *How are headers read?* `@RequestHeader` with `defaultValue`, as with `X-Bridge-Node-Id` and `X-Hop-Count`.

### The code uses field `@Autowired`. Is that good practice?
`level: L2` · `sources: controller/ApiController.java, service/BridgeIngestionService.java`

**Answer:** It works but constructor injection is preferred: fields can be `final`, dependencies are explicit, and the class can be built in a plain unit test without Spring. Field injection hides dependencies and needs reflection or a Spring context to test.

**Follow-ups:**
1. *What is the DI container doing?* It creates singleton beans and injects dependencies by type at startup.
2. *What are bean scopes?* All services here are singletons, so they must be thread-safe. They are, because state lives in concurrent collections or the DB.

### Are the validation annotations on `MeshPacket` effective?
`level: L2` · `sources: model/MeshPacket.java, controller/ApiController.java`

**Answer:** No. `MeshPacket` has `@NotBlank`, `@NotNull` and `@Min`, and `spring-boot-starter-validation` is on the classpath, but `ingest` takes `@RequestBody MeshPacket packet` without `@Valid`. So the constraints are not enforced. A packet with a null ciphertext reaches `hashCiphertext`, throws a `NullPointerException`, and is caught as `INVALID: internal_error`.

**Follow-ups:**
1. *How do you fix it?* Add `@Valid` to the parameter, and handle `MethodArgumentNotValidException` with a `@ControllerAdvice` to return a 400.
2. *Why is returning HTTP 200 for `INVALID` debatable?* Clients and monitoring cannot distinguish failures by status code. A 4xx for invalid packets is more conventional, but a bridge uploading many packets might prefer a 200 with per-packet results.

### What do `@PostConstruct` and `@Scheduled` do in this project?
`level: L2` · `sources: service/DemoService.java, crypto/ServerKeyHolder.java, service/IdempotencyService.java, config/AppConfig.java`

**Answer:** `@PostConstruct` runs once after a bean is built: `DemoService` seeds four accounts, `ServerKeyHolder` generates the RSA keypair. `@Scheduled(fixedDelay = 60_000)` runs `evictExpired` a minute after the previous run finishes. It needs `@EnableScheduling`, which `AppConfig` provides.

**Follow-ups:**
1. *`fixedDelay` vs `fixedRate`?* `fixedDelay` waits after completion; `fixedRate` starts on a fixed clock regardless.
2. *Consequence of the keypair being regenerated at startup?* Packets encrypted for the old public key cannot be decrypted after a restart. Production needs a persistent key from KMS.

---

## 7. Mesh simulator and dashboard

### How does the gossip algorithm work?
`level: L1` · `sources: service/MeshSimulatorService.java, service/VirtualDevice.java`

**Answer:** `gossipOnce` takes a snapshot of what every device holds. For each packet with TTL greater than 0, it copies the packet to every other device that does not already hold that `packetId`, with TTL decreased by 1. `VirtualDevice.hold` uses `putIfAbsent` keyed by packet ID, so a device never stores a packet twice.

**Follow-ups:**
1. *Why snapshot first?* Otherwise a packet could cascade through every device within a single round and the hop count would be wrong.
2. *What does TTL do?* It bounds propagation so packets cannot circulate forever. At TTL 0 a packet stays put.
3. *How does it differ from real BLE?* Here everyone is in range of everyone. Real gossip is pairwise and depends on physical proximity and battery constraints.

### Why `ConcurrentHashMap` in `MeshSimulatorService` and `VirtualDevice`?
`level: L2` · `sources: service/MeshSimulatorService.java, service/VirtualDevice.java, controller/ApiController.java`

**Answer:** Requests arrive on multiple threads, and `meshFlush` uses `parallelStream`. A plain `HashMap` is not thread-safe. Concurrent maps give safe reads and writes without a global lock.

**Follow-ups:**
1. *Why `synchronized (results)` in `meshFlush`?* The `results` list is an `ArrayList`, which is not thread-safe, and parallel stream threads add to it. A `CopyOnWriteArrayList` or collecting via `map` and `collect` would be cleaner.
2. *Is `gossipOnce` itself thread-safe?* Individual maps are, but the round as a whole is not atomic. Two simultaneous gossip calls could interleave, which is acceptable for a demo.

---

## 8. Testing

### What do the tests cover?
`level: L2` · `sources: src/test/java/com/demo/upimesh/IdempotencyConcurrencyTest.java`

**Answer:** Three tests in a `@SpringBootTest`. `encryptDecryptRoundTrip` checks that encrypt then decrypt returns the same instruction. `tamperedCiphertextIsRejected` flips one base64 character and expects `INVALID`. `singlePacketDeliveredByThreeBridgesSettlesExactlyOnce` runs three threads released together by a `CountDownLatch` and asserts one `SETTLED`, two `DUPLICATE_DROPPED`, and that Alice and Bob's balances changed by exactly ₹100.

**Follow-ups:**
1. *Why a `CountDownLatch`?* It holds all threads at the start line so they hit `ingest` at the same instant, maximizing contention.
2. *Weaknesses?* The context and H2 database are shared, and only the idempotency cache is cleared in `@BeforeEach`, so tests depend on seed balances being sufficient. There are no controller tests, no test for stale or future packets, none for insufficient funds, and none for two different packets racing on one account.
3. *Is the tamper test reliable?* Flipping one character can occasionally produce an invalid base64 string instead of a bad tag; both end as `INVALID`, so it passes either way, but it does not prove specifically that GCM caught it.

---

## 9. Security and production readiness

### What security gaps does the current code have?
`level: L3` · `sources: controller/ApiController.java, service/DemoService.java, service/SettlementService.java, crypto/ServerKeyHolder.java`

**Answer:** There is no sender authentication: anyone with the public key (served at `/api/server-key`) can build a valid packet naming any `senderVpa`. The PIN hash is stored but never verified, and a SHA-256 of a 4-digit PIN is trivially brute-forced. The ingest endpoint has no auth or rate limit, the H2 console is enabled, and the RSA private key is in memory and regenerated on restart.

**Follow-ups:**
1. *How would you authenticate senders?* Each device holds a key pair registered with the bank. The payload is signed with the device private key, and the server verifies the signature against the registered public key, in addition to a PIN verified by the bank.
2. *How would you secure the bridge endpoint?* Mutual TLS or signed bridge certificates, plus per-node rate limits.
3. *What should be monitored?* Spikes of `INVALID`, `DUPLICATE_DROPPED` ratio, settlement latency, and rejected-for-funds rates, with structured logs to a SIEM.

### Is it a problem that errors return `internal_error: ` plus the exception message?
`level: L2` · `sources: service/BridgeIngestionService.java`

**Answer:** Yes. `invalid("?", "internal_error: " + e.getMessage())` returns internal exception text to the caller, which can leak implementation details. Log it server-side and return a generic code.

**Follow-ups:**
1. *Why is the hash `"?"` there?* If hashing itself fails there is no hash to report.

---

## 10. Roadmap (NOT implemented)

These come from the README's production table and from gaps in the code. None of them exist yet; do not claim them.

### How would you take this from demo to production?
`level: L3` · `sources: README.md`

**Answer:** Not implemented. The plan would be: PostgreSQL replacing H2; Redis `SET NX EX` replacing the map; keys in KMS or an HSM; an Android client that does the real BLE or Wi-Fi Direct gossip and runs the encryption code; mutual TLS and rate limiting on ingest; and integration with a real bank core instead of the self-owned ledger.

**Follow-ups:**
1. *What is the hardest part?* The Android mesh layer: background BLE is throttled on Android and iOS peripheral mode is restricted.
2. *How would you handle the funds problem?* A pre-funded offline wallet in the style of UPI Lite, which the current design does not have.
