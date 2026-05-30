# webMethods Developer — Mock Interview Set 
---

## How to practise with this

For each question you'll see three things:

- **The question** — what they ask.
- **A strong answer covers** — the points that make an answer sound senior, so you can check yourself.
- **Follow-up probes** — the "okay, but how / why / what if" questions they drill into after your first answer. These are where interviews are won or lost. Practise these out loud, because real interviewers rarely accept the first answer — they keep digging until they hit either bedrock or air.

A "Watch out" note appears where there's a common trap or a place people over-claim.

Work through one section per sitting. Say answers aloud, not just in your head. If a probe stumps you, that's the gold — that's the thing to go refresh tonight.

---

## Section 1: Integration Server and Flow fundamentals

**Q1.1 — Walk me through what happens, end to end, when an HTTP request hits the Integration Server and triggers a Flow service.**

A strong answer covers: the request comes in on a port (e.g. 5555), the server matches it to a service via the URL and the invoke directive, checks the ACL/authentication, places the inputs into the pipeline, runs the Flow steps in order, and returns the output matching the service signature. Mention content handlers turning the HTTP body into pipeline data.

Follow-up probes:
- How does the server decide which service to run from the URL? *(invoke directive `/invoke/folder.name:service`, or a URL alias, or a REST resource mapping.)*
- Where does the incoming JSON or XML body actually land in the pipeline, and what converts it? *(content handlers; for JSON the body maps to a document; for XML you often get a node you parse.)*
- How would you change the HTTP response code or content type the service returns? *(`pub.flow:setResponseCode`, `pub.flow:setResponse`, setting the Content-Type.)*
- What ACL controls who can run that service, and where do you set it?

---

**Q1.2 — Explain the pipeline. Then tell me how pipeline scope works when one service invokes another.**

A strong answer covers: the pipeline is the in-memory data of the running service; each step reads from and writes to it; outputs are whatever matches the signature at the end. On invoke, by default the entire pipeline is passed into the called service, and its outputs merge back.

Follow-up probes:
- How do you stop a called service from seeing the whole pipeline? *(The **Scope** property on the INVOKE step — restrict input to a named document, so only that goes in.)*
- Why would you bother restricting scope? *(Avoid name collisions, keep the called service clean and reusable, reduce memory, avoid accidental data leakage.)*
- What's the risk of never dropping pipeline variables in a long flow? *(Memory bloat, accidental reuse of stale values, confusing debugging, larger audit logs.)*
- You map a value, then call a service that outputs a variable of the same name — what wins? *(The later write overwrites; understanding order matters.)*

Watch out: many people describe the pipeline correctly but freeze on the **Scope** probe. Know it cold.

---

**Q1.3 — Take me through every Flow step and when you'd reach for each.**

A strong answer covers: INVOKE (call a service), MAP (move/transform/drop/set, run transformers), BRANCH (switch on a value or evaluate labels as expressions), LOOP (iterate a list, build an output list), REPEAT (retry N times on success or failure), SEQUENCE (group steps; build try/catch), EXIT (leave flow/loop/parent, signal success or failure).

Follow-up probes:
- BRANCH: difference between switching on `/order/type` versus using "Evaluate labels"? *(Switch matches the field's value to a label; Evaluate labels treats each label as a boolean `%expression%` — needed for ranges, AND/OR, null checks.)*
- LOOP: what do you set so the loop builds an output array correctly, and what's a common off-by mistake? *(Input array path and output array path; forgetting the output array gives you only the last iteration's value.)*
- REPEAT: how do you implement "retry 3 times with a delay on a transient failure"? *(REPEAT on FAILURE, count 3, with a delay; or a SEQUENCE-based retry.)*
- Can a MAP step run multiple services? *(Yes — transformers inside a MAP run together, which is cleaner than separate INVOKEs when transforming several fields.)*
- EXIT: how do you exit just the current loop versus the whole flow? *(`$loop` / `$iteration` vs `$flow` / `$parent`.)*

---

**Q1.4 — How do you implement error handling in a Flow service?**

A strong answer covers: nested SEQUENCEs — an outer "exit on DONE" try block, then an "exit on FAILURE" catch block that only runs if the try failed. In the catch, call `pub.flow:getLastError`, then log, alert, route to an error queue, or rethrow.

Follow-up probes:
- What exactly is inside the `lastError` document? *(`error` message, `errorType`, `errorDump` / stack, the `pipeline` at the moment of failure, the service name.)*
- What's the difference between SEQUENCE exit-on-SUCCESS, FAILURE, and DONE, and which goes on the try versus the catch? *(Try = exit on DONE so it stops at first failure; catch = exit on FAILURE.)*
- How do you rethrow so the caller knows it failed? *(`pub.flow:throwExceptionForRetry` for retryable, or EXIT signalling FAILURE with a message, or re-raise the error.)*
- How would you design org-wide error handling — not just one service? *(A common error-handler service, route failures to an error/dead-letter queue or DB, alert, support resubmission via Monitor.)*
- What's the danger of swallowing the error in the catch and continuing? *(Silent data loss; the caller thinks it succeeded.)*

---

**Q1.5 — How does service result caching work, and when would you use it?**

A strong answer covers: you can cache a service's output keyed by its inputs, with an expiry, so repeated identical calls skip execution. Good for reference data that rarely changes.

Follow-up probes:
- What's the risk of caching a service that takes a large or highly variable input? *(Huge cache, low hit rate, memory pressure.)*
- What happens to cached data on a server restart or in a cluster? *(Per-server unless using a shared/distributed cache; stale data risk.)*
- When would caching be flat-out wrong? *(Anything transactional or time-sensitive — balances, stock levels.)*

---

## Section 2: Web services — SOAP and REST in webMethods

**Q2.1 — How do you expose a Flow service as a SOAP web service in webMethods?**

A strong answer covers: create a **provider Web Service Descriptor (WSD)** from the service, which generates the WSDL; choose the SOAP style; deploy and share the WSDL URL with consumers.

Follow-up probes:
- Document/Literal versus RPC/Encoded — which do you use and why? *(Document/Literal (wrapped) is the standard and interoperable; RPC/Encoded is legacy/deprecated.)*
- How do you consume an external SOAP service? *(Create a **consumer WSD** by importing the partner's WSDL; it generates the connector services.)*
- How do you add WS-Security — say, signing or a username token? *(WS-Security policies on the WSD, message-level signing/encryption, handlers; certificates in the keystore.)*
- How do you send a binary attachment with SOAP? *(MTOM.)*
- Where do you handle SOAP headers for routing or auth? *(SOAP handlers / processing on the WSD.)*

---

**Q2.2 — How have you built REST APIs in webMethods? Walk me through both the old and the newer way.**

A strong answer covers: the older **REST resource** approach (a folder with `_get`, `_post` etc. services and the `restv2` / `rest` directive), and the newer **REST API Descriptor** that's tied to an OpenAPI/Swagger definition with proper resource and method mapping.

Follow-up probes:
- How do you map a path parameter like `/orders/{id}` to a pipeline variable?
- How do you set the HTTP status code and return a proper error body for a REST call? *(`setResponseCode`, build an error document, set Content-Type to application/json.)*
- How do you convert between JSON and a webMethods document both ways? *(`pub.json:jsonStringToDocument` and `pub.json:documentToJSONString`.)*
- How would you version a REST API without breaking existing clients? *(URI versioning `/v1/`, or via API Gateway; keep old version live during migration.)*
- How do you secure a REST API you expose externally? *(Front it with API Gateway — OAuth/JWT/API key — not the raw IS.)*

---

**Q2.3 — A partner says your SOAP service returns a SOAP fault intermittently. How do you investigate?**

A strong answer covers: reproduce, check the server and error logs around the timestamp, inspect the inbound payload that fails, check whether it's a validation fault versus a back-end failure versus a timeout, look at the SOAP processor and any handler.

Follow-up probes:
- How do you tell a client-side fault (bad request) from a server-side fault? *(SOAP fault code: Client/Sender versus Server/Receiver.)*
- Where would you turn on more detail without flooding production? *(Targeted service logging / audit on that one service, raise log level briefly.)*
- "Intermittent" — what's your first hypothesis? *(Timeout to a downstream system, connection pool exhaustion, or one malformed message type.)*

---

## Section 3: Messaging, pub/sub, and triggers

**Q3.1 — Explain publish/subscribe in webMethods and the role of triggers.**

A strong answer covers: a service publishes a publishable document to Universal Messaging; it sits on a channel; a **trigger** on a (possibly different) Integration Server subscribes to that document type and runs a service to process it. Decouples sender from receivers.

Follow-up probes:
- Universal Messaging versus the old Broker — what changed and what stayed the same? *(UM replaced Broker; pub/sub concepts identical; Broker is end-of-life.)*
- Guaranteed versus volatile delivery — when do you choose each? *(Guaranteed = persisted, survives restart, slower; volatile = in-memory, fast, can be lost — fine for non-critical, high-volume.)*
- A trigger's processing mode: serial versus concurrent — what's the trade-off? *(Serial preserves order, lower throughput; concurrent is faster but order isn't guaranteed.)*
- How do you control how many documents a trigger processes at once? *(Max execution threads / processing settings on the trigger.)*

---

**Q3.2 — How do you guarantee exactly-once processing so a duplicate publish doesn't create two orders?**

A strong answer covers: enable **exactly-once / duplicate detection** on the trigger using document history (a database) or a document resolver service that checks a unique key (UUID) before processing.

Follow-up probes:
- Where is the duplicate-detection state stored, and what breaks if it's not shared in a cluster? *(Document history DB; per-server state means duplicates slip through across nodes.)*
- What's a "document resolver" and when would you write a custom one? *(A service that decides new/duplicate/in-doubt based on your own key/business logic.)*
- What happens to a document the trigger can't process after all retries? *(Goes to error/dead-letter handling; you decide resubmit strategy.)*
- Difference between retry-on-transient-error and a genuine failure? *(Transient = retryable like a brief DB outage; permanent = bad data, don't retry endlessly.)*

---

**Q3.3 — When would you use JMS instead of native webMethods messaging?**

A strong answer covers: JMS for interoperability with other JMS providers and standards-based messaging; native UM messaging (publishable docs/triggers) when you're inside the webMethods world. JMS triggers versus webMethods messaging triggers.

Follow-up probes:
- Queue versus topic — pick one for "process each order once" and one for "notify three systems of a price change." *(Queue for once-only consumption; topic for broadcast.)*
- What's a durable subscriber and why does it matter? *(Receives messages published while it was down.)*
- How do JMS transactions and acknowledgement modes affect reliability? *(Transacted sessions, AUTO vs CLIENT acknowledge — CLIENT lets you ack only after successful processing.)*

---

## Section 4: Adapters and back-end connectivity

**Q4.1 — Walk me through the JDBC adapter: connections, services, and transactions.**

A strong answer covers: configure a connection (driver, URL, credentials, pool min/max), build adapter services (select, insert, update, delete, custom SQL, batch, stored procedure), and choose the transaction type per connection.

Follow-up probes:
- The three JDBC connection transaction types — name them and when you'd use each. *(NO_TRANSACTION, LOCAL_TRANSACTION, XA_TRANSACTION — XA when you need a single commit across two resources, e.g. DB plus JMS.)*
- You need two inserts to either both succeed or both fail. How do you do it in Flow? *(Explicit transaction with `pub.art.transaction:startTransaction` / `commitTransaction` / `rollbackTransaction`, both services on the same connection in LOCAL_TRANSACTION, or XA.)*
- Connection pool is exhausting under load — what do you check? *(Pool max size, connections not being released, long-running queries holding connections, expire timeout.)*
- Difference between a parameterised select adapter service and a dynamic SQL service — what's the SQL-injection consideration? *(Templates parameterise safely; dynamic SQL built from input is an injection risk — validate/escape.)*
- What's a JDBC notification and how does it detect changes? *(A buffer table plus a database trigger; the adapter polls the buffer and publishes a document.)*

Watch out: the transaction-type probe (NO / LOCAL / XA) is the classic senior filter. Be ready.

---

**Q4.2 — Describe a SAP integration you've built.**

A strong answer covers: use the SAP adapter; receive/send **IDocs**, call **BAPIs / RFCs**; mention inbound versus outbound IDoc flows and the listener.

Follow-up probes:
- IDoc versus BAPI versus RFC — when each? *(IDoc for asynchronous, structured business documents; BAPI/RFC for synchronous function calls and reads.)*
- tRFC and qRFC — what problem do they solve? *(Transactional and queued RFC for reliable, ordered delivery so you don't lose or reorder calls.)*
- How does webMethods receive an IDoc pushed from SAP? *(RFC listener / IDoc routing; the IDoc is parsed into a document and processed/published.)*
- How do you handle an IDoc that fails mapping — do you ack SAP or not? *(Careful handling so SAP doesn't think it failed to deliver; route the bad IDoc to error handling.)*

---

**Q4.3 — A partner drops files via SFTP. Design the pickup and walk me through the pitfalls.**

A strong answer covers: a file polling port or scheduled task watching the directory, picking up files, processing, archiving; secure transfer over SFTP with keys.

Follow-up probes:
- How do you avoid grabbing a file that's still being uploaded? *(Wait for a trigger/marker file like `.done`, or the partner uploads with a temp name then renames, or check file size stability.)*
- What do you do with a file after success and after failure? *(Archive on success; move to an error folder and alert on failure — never just delete.)*
- Two server nodes both poll the same folder — what goes wrong and how do you fix it? *(Both grab the same file; use a single scheduled task target, locking, or rename-on-pickup.)*
- Huge file — how do you avoid loading it all into memory? *(Stream/process in chunks, use large-file handling / flat file iterator rather than parsing the whole thing.)*

---

## Section 5: B2B — Trading Networks, EDI, AS2

**Q5.1 — Walk me through an inbound B2B flow end to end: partner sends an EDI purchase order over AS2.**

A strong answer covers: AS2 message arrives over HTTPS → decrypt and verify signature → return signed MDN → Trading Networks recognises the document (sender, type, e.g. X12 850) → a processing rule fires → service splits/maps the EDI to your canonical/internal format → routes to back-end (e.g. SAP IDoc) → generate and return a 997/CONTRL acknowledgement.

Follow-up probes:
- How does Trading Networks "recognise" the document and identify the partner? *(TN document types and recognition criteria — ISA/GS/ST for X12; sender/receiver IDs; external IDs on the profile.)*
- A processing rule isn't firing for a partner — what do you check, and does rule order matter? *(Criteria match — sender, receiver, doc type; yes, rule order matters, first match can win; check it's enabled.)*
- How do you split a single interchange containing 500 transaction sets into individual documents? *(EDI splitting / TN large-document handling; process per transaction set.)*
- What's the difference between a 997, a 999, and a TA1? *(997 functional ack, 999 implementation ack with stricter validation, TA1 interchange-level ack.)*
- Where do control numbers come from and why do they matter? *(ISA/GS/ST control numbers for tracking and duplicate detection; out-of-sequence or duplicate numbers flag problems.)*

---

**Q5.2 — Explain AS2 properly. What gives you non-repudiation?**

A strong answer covers: AS2 is secure transport over HTTP(S) using S/MIME — encryption, digital signatures, and a **signed MDN** receipt. The signed MDN is the non-repudiation proof.

Follow-up probes:
- Synchronous versus asynchronous MDN — when would a partner insist on async? *(Async when processing takes time or for reliability on their side; the MDN comes back as a separate inbound message.)*
- Which certificates are involved and which key does what? *(Your private key signs and decrypts; partner's public cert verifies their signature and encrypts to them. Keystores/truststores.)*
- A partner says they never got your MDN — how do you trace it? *(TN/EDIINT logs, MDN status, retry settings; check async MDN delivery URL and certs.)*
- What happens if the signature verification fails on an inbound AS2 message? *(Reject; don't process; alert — possible tampering or wrong cert.)*

---

**Q5.3 — How do you handle a malformed EDI document from a partner in production?**

A strong answer covers: TN flags it on recognition/validation; route to error handling; notify the partner with the appropriate negative acknowledgement; never silently process bad data.

Follow-up probes:
- Structural error versus business-data error — handled the same way? *(Structural caught at parse/validation → reject with ack; business error → may need manual review or a different rejection.)*
- How do you reprocess once the partner resends? *(Resubmit via TN console; or they send a corrected interchange with a new control number.)*
- How do you alert the right people without spamming? *(Targeted alerts, error queue with severity, avoid alerting on every routine rejection.)*

---

## Section 6: API Gateway

**Q6.1 — What does API Gateway do that the Integration Server doesn't, and what policies have you applied?**

A strong answer covers: it's the secure, governed front door — authentication/authorisation, threat protection, rate limiting/throttling, routing, transformation, logging and analytics, plus a Developer Portal. IS runs the logic; Gateway governs access.

Follow-up probes:
- Name the policy categories and give one policy from each. *(Identify & access: API key/OAuth/JWT; traffic: rate limit/throttle; security: payload/threat protection; routing: straight-through/load-balanced; transformation; logging.)*
- Difference between rate limiting and throttling, and how would you protect a fragile back-end? *(Rate limit caps calls per consumer over time; throttling shapes/queues bursts; combine to protect the back-end.)*
- How do you do protocol mediation — accept REST at the front, call SOAP at the back? *(Routing + transformation policies on the API.)*
- API Gateway versus the older Mediator/CentraSite setup — what replaced what? *(Gateway is the modern replacement; CentraSite was the registry/repository.)*
- Where does API Gateway store its data and config? *(Its own data store — Elasticsearch-based — separate from IS.)*

---

## Section 7: Deployment, environments, and CI/CD

**Q7.1 — How do you promote code from Dev to QA to Production?**

A strong answer covers: code in Git; build assets with the **Asset Build Environment (ABE)** into a composite; deploy with **Deployer** or **Command Central**; keep environment-specific values external so the same build runs everywhere; never edit code in production.

Follow-up probes:
- How do you handle settings that differ per environment — DB URLs, partner endpoints, credentials? *(Variable substitution / configuration variables templates, global variables, externalised properties; never hard-code.)*
- Repository-based versus runtime-based Deployer projects — what's the difference? *(Repository-based pulls from a build repository (fits CI/CD); runtime-based copies directly between live servers.)*
- How would you wire this into Jenkins? *(Checkout from Git → ABE build → automated tests → Deployer/Command Central push → smoke test.)*
- How do you roll back a bad deployment? *(Redeploy the previous composite/build; keep versioned artifacts; have a rollback plan before you ship.)*
- How is package source controlled when it historically lived only on the server? *(Local Service Development with VCS/Git integration; treat packages as code in a repo.)*

---

**Q7.2 — How do you test a Flow service, and how would you build regression testing into a pipeline?**

A strong answer covers: unit-test services with a test framework, mock external dependencies, assert outputs for known inputs; run the suite automatically in CI before deploy.

Follow-up probes:
- How do you test a service that calls SAP or a partner without hitting the real system? *(Stub/mock the adapter or external call; test data sets.)*
- What do you assert beyond the happy path? *(Error handling, edge cases, malformed input, boundary values.)*

---

## Section 8: Performance and troubleshooting (scenario-heavy)

**Q8.1 — Production is slow this morning. Services are timing out. Walk me through your diagnosis, live.**

A strong answer covers: check server health (CPU, memory/heap, GC), the server thread pool usage, the service stats (`stats.log`), recent deployments, and the health of downstream systems. Narrow from "is it us or them."

Follow-up probes:
- The main port (5555) is unresponsive — how do you get in to look? *(The **diagnostic port** — separate port, dedicated thread, lets you in when the main port is saturated.)*
- Heap is climbing and won't come down — what do you capture and what causes it? *(Heap dump; causes: unbounded cache, large pipelines not dropped, leak in a Java service, document backlog.)*
- Thread pool is maxed — what's typically holding the threads? *(Blocked on slow downstream calls / DB; no timeouts on outbound HTTP/JDBC; deadlocks.)*
- Where do outbound timeouts get set so one slow partner doesn't take the server down? *(HTTP/SOAP client timeouts, JDBC query timeout, watt/server config parameters.)*

Watch out: "how do you get into a hung server" — the **diagnostic port** answer instantly signals real operational experience.

---

**Q8.2 — A nightly batch that used to finish in 30 minutes now takes 3 hours. How do you find the cause?**

A strong answer covers: compare what changed (data volume, code, infra), profile where the time goes (which service/step), look for row-by-row DB work, missing indexes, synchronous calls that should be batched or async.

Follow-up probes:
- Row-by-row inserts in a LOOP — how do you fix it? *(Batch insert adapter service; reduce round trips.)*
- The DB query got slow as data grew — whose problem and how do you confirm? *(Check the execution plan with the DBA; missing/!used index; not always your code.)*
- Would caching help here, and where could it hurt? *(Cache static reference lookups; don't cache changing transactional data.)*

---

**Q8.3 — Where do you look first for any failed transaction, and how do you safely turn up detail?**

A strong answer covers: webMethods Monitor (service and document audit, pipeline at failure, resubmit), Trading Networks console for B2B, server/error logs by timestamp, API Gateway analytics for API issues. Raise logging on the specific service briefly rather than globally.

Follow-up probes:
- What's the risk of `savePipeline` / pipeline debug to file left on in production? *(Writes potentially sensitive data to disk, fills disk, performance hit — clean it up.)*
- How do you correlate a single business transaction across multiple services and systems? *(A correlation/transaction ID carried through and logged; document tracking.)*
- In a regulated client (pharma), what must you be careful about in logs? *(No sensitive/regulated data in logs; keep audit trails intact; access controls.)*

---

## Section 9: Architecture and design judgement

**Q9.1 — Design an integration between an e-commerce site and SAP for new orders. Talk me through your choices.**

A strong answer covers: decide synchronous versus asynchronous, define a canonical order model, validate input, map to SAP (IDoc/BAPI), handle errors and retries, acknowledge the site, and make it observable. Reusable service layering.

Follow-up probes:
- Sync or async, and why? *(Async/queue so a slow or down SAP doesn't fail the customer's checkout; the site gets an immediate "received," SAP processes when ready.)*
- Why a canonical model instead of mapping the site format straight to SAP? *(Add new front-ends or back-ends without N×N mappings; isolate change.)*
- What happens if SAP is down for two hours? *(Messages persist in the queue/guaranteed delivery; process on recovery; alert if backlog grows.)*
- How do you make sure one order isn't created twice? *(Idempotency key, exactly-once processing, duplicate detection.)*
- How would you layer the services for reuse? *(Adapter layer, utility/transformation layer, business/process layer — don't put SAP calls directly in the public-facing service.)*

---

**Q9.2 — How do you decide between synchronous request/response and asynchronous messaging for a given integration?**

A strong answer covers: sync when the caller needs an immediate answer and the call is fast and reliable; async when you want decoupling, resilience to downtime, load smoothing, or the work is slow.

Follow-up probes:
- Give a real example where you switched a sync integration to async and what improved. *(From your own experience — resilience, throughput.)*
- What do you give up by going async? *(Immediate confirmation, simpler error handling; you now need tracking and reconciliation.)*

---

**Q9.3 — How do you scale and make the Integration Server highly available?**

A strong answer covers: stateless services scale horizontally behind a load balancer; cluster nodes; externalise state; Universal Messaging for shared messaging; shared config via Command Central.

Follow-up probes:
- Why does statelessness matter for clustering? *(Any node can handle any request; no sticky state to lose.)*
- Where does shared state live in a cluster (caching, duplicate detection, scheduling)? *(Distributed cache, shared DB for document history, a single scheduler target to avoid double-running jobs.)*
- A scheduled task is running on every node instead of once — how do you fix it? *(Configure it to run on one node / cluster-aware scheduling.)*

---

## Section 10: Java services and SQL

**Q10.1 — When do you write a Java service instead of Flow, and what's the risk?**

A strong answer covers: Java for things Flow does awkwardly — complex string/regex work, custom algorithms, calling a third-party Java library. Risk: harder to maintain, breaks the graphical visibility, needs proper exception handling and resource cleanup.

Follow-up probes:
- How does a Java service read from and write to the pipeline? *(`IData` / `IDataCursor` — `pipelineCursor.getValue` / `insertAfter`.)*
- How do you make sure a Java service doesn't leak resources? *(Close cursors, streams, connections in `finally`.)*
- Where do you put a shared Java utility used by many services? *(Shared/jcode or a shared static method / library in the package, referenced via the classpath.)*

---

**Q10.2 — Quick SQL on the spot: get all orders over €1,000 for a given customer, newest first. Then: what if some customers have no orders and you still want them listed?**

A strong answer covers: a `SELECT ... WHERE amount > 1000 AND customer_id = ? ORDER BY order_date DESC`; for the second part, a `LEFT JOIN` from customers to orders.

Follow-up probes:
- Inner versus left join in one plain sentence each. *(Inner = only matching rows; left = all left rows plus matches, nulls where none.)*
- Why is `UPDATE`/`DELETE` without a `WHERE` dangerous, and how do you protect yourself? *(Hits every row; test the WHERE as a SELECT first, use transactions.)*
- What does an index buy you and what does it cost? *(Faster reads, slower writes and more storage.)*

---

## Section 11: Experience, behaviour, and the career break

These aren't trick questions. They want a calm, specific human who's done real work.

**Q11.1 — Tell me about the most complex integration you've built.**

Prepare one real story with: the business problem, the systems involved, your design choices, what was hard, and the result. Be ready for probes: *"Why that design?" "What would you do differently now?" "What broke and how did you fix it?"*

**Q11.2 — Tell me about a production incident you owned.**

One real story: how you were alerted, how you diagnosed it, the fix, and what you changed so it wouldn't recur. Probe: *"What was the root cause versus the symptom?"*

**Q11.3 — Tell me about working with business analysts and architects when requirements were unclear.**

Show you ask questions, clarify, and don't just build blindly. Probe: *"Give an example where you pushed back on a requirement."*

**Q11.4 — You've been out for two years. How do you get back up to speed quickly?**

Say it plainly and confidently: you took time for family; the platform is fundamentally the same through 10.x and now under IBM; you've refreshed the core; you ramp up fast and you've done it before. One or two sentences, no apology, move on. Probe they might add: *"What's changed since you last worked on it?"* — answer: ownership moved to IBM, more cloud/SaaS and container-based deployment, but Integration Server, Flow, Trading Networks, and the adapters are unchanged.

Watch out: don't over-explain the gap or sound defensive. The calmer you are about it, the smaller it gets.

---

## Section 12: Curveballs and gotchas (the real-practitioner filters)

These are short, sharp questions that someone who only studied theory tends to miss.

- **What port lets you into a hung Integration Server, and why does it work?** *(Diagnostic port — dedicated thread separate from the main thread pool.)*
- **Name the three JDBC connection transaction types.** *(NO_TRANSACTION, LOCAL_TRANSACTION, XA_TRANSACTION.)*
- **How do you restrict what a called service sees in the pipeline?** *(Scope property on the INVOKE.)*
- **What carries the error details in a catch block?** *(`pub.flow:getLastError`.)*
- **Guaranteed versus volatile messaging — one trade-off each.** *(Guaranteed survives restart but slower; volatile is fast but loses on crash.)*
- **What proves a partner received your AS2 document?** *(The signed MDN.)*
- **What's the standard SOAP binding style today?** *(Document/Literal wrapped.)*
- **Where should environment-specific config live?** *(Externalised — never hard-coded in the package.)*
- **What's the danger of leaving pipeline-save/debug on in production?** *(Sensitive data to disk, disk fill, performance.)*
- **What replaced webMethods Broker?** *(Universal Messaging.)*
- **What's a 997 in EDI?** *(Functional acknowledgement.)*
- **What's the default Integration Server HTTP port?** *(5555.)*
- **How do you stop one slow downstream system from taking the whole server down?** *(Outbound timeouts plus right-sized connection/thread pools; isolate with async where possible.)*
- **What's a canonical data model and the problem it solves?** *(Shared internal format; avoids N×N mappings.)*

---

## A note on delivery

Two habits make these answers land well in the room:

1. **Lead with the direct answer, then add depth.** Don't warm up for thirty seconds. Say the answer, then expand. Interviewers relax when you're clear.
2. **Flag hands-on versus alongside.** "I built this" carries weight — use it where it's true (and with 10.x hands-on, it's true for most of this). For anything you only worked next to, say so and then show you understand it. That mix reads as senior and honest.
