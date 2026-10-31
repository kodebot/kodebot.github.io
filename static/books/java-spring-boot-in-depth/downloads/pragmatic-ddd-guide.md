# Pragmatic DDD Guide

Oct 31, 2026 · @kodebot

This guide says how we design the inside of a microservice. We use a light form of Domain-Driven Design (DDD). DDD means two things here. Write the code in the business's own words. Put each business rule inside the object it is about.

The guide has two aims. Every business rule lives in exactly one place. A design mistake can be fixed inside one feature of one microservice, without changing the others.

The guide is about design, not about any framework or library. The code examples are plain Java and stand on their own. Every rule has the same shape: the situation you are in, the code most people write first, what goes wrong with it, the code we want, and why. Read the walkthrough and the rules in order once. After that, use the headings as a checklist.

## Terms

| Term | Meaning |
| --- | --- |
| Microservice | An application deployed on its own, with its own database |
| Feature | One business area inside a microservice, in its own folder. Example: claims, notes |
| Business action | One change the user asks for with one button, such as "close claim" |
| Entity | A business object with its own ID. Its details change; its identity does not. Example: `Claim` |
| Value object | A value with no ID, defined by its contents, never changed after it is created. Example: `Money` |
| Aggregate | A group of objects always loaded and saved together, as one unit. The top object is the aggregate root. Example: a `Claim` with its `EvidenceItem`s |
| Repository | The one class that loads and saves one kind of aggregate. Example: `ClaimRepository` |
| Application service | A class with one method per business action. Each method runs that action's steps: load, check the version, get facts, call the entity, save, audit. Example: `ClaimService` with `open`, `addEvidence`, `close`. A class per action (`CloseClaim` with one method) is also fine; pick one shape per microservice |
| Domain service | A rare class for a calculation that belongs to no single entity. Example: `EntitlementCalculator` |
| Adapter | The one class that talks to another microservice or system. Example: `EligibilityClient` |
| Domain event | A record that something happened inside our microservice, named in the past tense. Example: `ClaimClosed` |
| Integration event | A versioned event that other microservices receive. Example: `ClaimClosedV1` |
| Audit record | A line kept for ever after every change: who changed which record, how, and when. Written by the application service as its last step, never by the entity |
| Design group | The lead engineer, two or three senior engineers and the business experts |
| Design note | A one-page description of a feature that the design group approves before coding starts |

## The example

A **caseworker** records a **claim** for a **claimant**, adds **evidence** (documents such as payslips) and marks each item verified. The caseworker writes **notes**; a note can be marked "needs action". At the end, the caseworker closes the claim. The **eligibility** microservice says whether a person may claim. The **payment** microservice pays claimants and tells us when it has paid.

The business rules: a claim is created only if the claimant is eligible and the claim type is active. A claim has at most 10 evidence items, each verified once. A claim is closed only when all evidence is verified and no note needs action. A closed claim cannot be changed. A payment replaces the claim's last payment only if its payment date is later.

A claim and its evidence are one aggregate. Each note is its own aggregate that holds the claim's ID. Rule 7 explains why.

## Closing a claim, end to end

This is the whole path of one business action. Every rule in the guide points back at one of these five classes.

The controller receives the request and checks the permission. It does nothing else.

```java
// POST /claims/{id}/close
public void close(ClaimId id, CloseRequest body, User user) {
    permissions.require(user, CLOSE_CLAIM);
    claimService.close(id, body.version());
    // reply 204, no body
}
```

The application service runs the steps. It holds no business rule.

```java
public class ClaimService {
    public void close(ClaimId id, long expectedVersion) {
        Claim claim = claims.get(id);                        // 1. load, or "not found"
        claim.requireVersion(expectedVersion);               // 2. the version the user saw
        int open = notes.countNeedingAction(id);             // 3. a fact the claim does not hold
        claim.close(open, clock.today());                    // 4. the entity decides
        claims.save(claim);                                  // 5. save; events go out with it
        audit.record("claim closed", id);                    // 6. audit
    }
}
```

The entity decides. All the rules for closing are in this one method.

```java
public class Claim {
    private final ClaimId id;
    private ClaimStatus status;
    private final List<EvidenceItem> evidence = new ArrayList<>();
    private LocalDate closedOn;
    private long version;
    private final List<DomainEvent> events = new ArrayList<>();

    public void close(int notesNeedingAction, LocalDate today) {
        if (status == CLOSED) throw new ClaimAlreadyClosed(id);
        if (evidence.stream().anyMatch(e -> !e.isVerified())) throw new EvidenceNotVerified(id);
        if (notesNeedingAction > 0) throw new ClaimHasOpenActions(id);
        status = CLOSED;
        closedOn = today;
        events.add(new ClaimClosed(id, today));
    }
}
```

The repository saves the whole aggregate and its events in one transaction. The events are stored in an outbox table; a background sender publishes them after the commit.

```java
public class ClaimRepository {
    // one transaction
    public void save(Claim claim) {
        int rows = db.save(claim);                           // writes version + 1 WHERE version = the loaded one
        if (rows == 0) throw new ClaimChangedSinceYouLoadedIt(claim.id());   // two saves raced; the second loses
        outbox.add(claim.pullEvents());                      // same transaction; pullEvents() also empties the list
    }
}
```

The event tells other microservices what happened. The outbox sender turns the domain event `ClaimClosed` into the published `ClaimClosedV1`. It carries IDs, not personal data.

```java
public record ClaimClosedV1(String eventId, String claimId, LocalDate closedOn) {}
```

## Part A. Where a rule lives

### 1. Put a business rule in the entity method that makes the change, not in a service

You are adding "close claim". The rule: a claim closes only when every evidence item is verified and no note needs action.

**Bad** — the rule sits in the service:

```java
public class ClaimService {
    public void close(ClaimId id) {
        Claim claim = claims.get(id);
        if (claim.getStatus() == CLOSED) throw new IllegalStateException();
        if (!claim.getEvidence().stream().allMatch(EvidenceItem::isVerified)) throw new IllegalStateException();
        if (notes.countNeedingAction(id) > 0) throw new IllegalStateException();
        claim.setStatus(CLOSED);
        claims.save(claim);
    }
}
```

Next month someone adds "close claims from the nightly job" and copies these three `if`s. Then one copy changes and the other does not.

**Good** — the rule sits in `Claim.close`. The service fetches the one fact the claim does not hold (the note count) and passes it in:

```java
public class Claim {
    public void close(int notesNeedingAction, LocalDate today) {
        if (status == CLOSED) throw new ClaimAlreadyClosed(id);
        if (evidence.stream().anyMatch(e -> !e.isVerified())) throw new EvidenceNotVerified(id);
        if (notesNeedingAction > 0) throw new ClaimHasOpenActions(id);
        status = CLOSED;
        closedOn = today;
        events.add(new ClaimClosed(id, today));
    }
}

public class ClaimService {
    public void close(ClaimId id, long expectedVersion) {
        Claim claim = claims.get(id);
        claim.requireVersion(expectedVersion);
        int open = notes.countNeedingAction(id);
        claim.close(open, clock.today());
        claims.save(claim);
        audit.record("claim closed", id);
    }
}
```

Nobody can call `close` without supplying the note count, so the check cannot be forgotten. The nightly job calls the same `Claim.close`.

### 2. Change an entity's fields only through its own methods; no setters

You need to mark an evidence item verified.

**Bad** — a setter:

```java
item.setVerified(true);
item.setVerifiedOn(today);
```

Nothing checks that the item was not verified already, and no `EvidenceVerified` event is recorded. Any class anywhere can do this.

**Good** — a method with the business name, on the entity:

```java
public class Claim {
    public void verifyEvidence(EvidenceId evidenceId, LocalDate today) {
        if (status == CLOSED) throw new ClaimAlreadyClosed(id);
        EvidenceItem item = find(evidenceId);
        item.markVerified(today);                            // refuses a second time
        events.add(new EvidenceVerified(id, evidenceId));
    }
}
```

All fields are private, and a build check fails on any public setter in an entity. A method checks the rules and records the event; a setter skips both.

### 3. A service method only runs the steps: load, check the version, get facts, call the entity, save, audit

You are writing `ClaimService.addEvidence`.

**Bad** — the service makes decisions:

```java
public void addEvidence(ClaimId id, EvidenceRequest request) {
    Claim claim = claims.get(id);
    if (claim.getEvidence().size() >= 10) throw new TooMuchEvidence(id);
    if (request.fileName() == null) throw new IllegalArgumentException();
    claim.getEvidence().add(new EvidenceItem(request.fileName()));
    claims.save(claim);
}
```

The limit of 10 is now in the service. The next service that adds evidence (an import job, say) does not know about it.

**Good** — the same six steps as `close`, and nothing else:

```java
public void addEvidence(ClaimId id, long expectedVersion, FileName fileName) {
    Claim claim = claims.get(id);                            // 1
    claim.requireVersion(expectedVersion);                   // 2
                                                             // 3: no outside facts needed
    claim.addEvidence(fileName, clock.today());              // 4: the entity checks the limit
    claims.save(claim);                                      // 5
    audit.record("evidence added", id);                      // 6
}
```

A service method contains no `if` about status, amounts, dates or counts. If you want to write one, it belongs in the entity (rule 1) or in a value object (rule 6). The database transaction covers steps 1 to 6; anything that calls another microservice happens before it opens (rule 17). Take the time from one injected `clock`, so tests can set the date.

### 4. Name classes, methods and errors with the business's own words

The business says "close a claim" and "a claim has open actions".

**Bad**:

```java
claim.updateStatus(3);
throw new ValidationException("invalid state");
```

A caseworker reading the log cannot tell what happened, and a developer must look up what 3 means.

**Good**:

```java
claim.close(openActions, today);
throw new ClaimHasOpenActions(id);
```

The design note, the code, the error message and the conversation with the business then use the same words. One shared error handler turns `ClaimHasOpenActions` into the HTTP answer `422` with the code `CLAIM_HAS_OPEN_ACTIONS`; the API Design Developer Guide, rule 26, shows it. The entity never knows about HTTP.

### 5. Put a calculation that spans several objects in a domain service, and only when it has worked examples

You must calculate a claimant's weekly entitlement from the claim type, the rate table and the number of children.

**Bad** — a "policy" class that holds one `if`:

```java
public class ClaimClosurePolicy {
    public boolean canClose(Claim claim) {
        return claim.evidenceAllVerified();
    }
}
```

This is a rule about closing a claim. Hidden in a helper class, it is one more place to look, and it will be bypassed.

**Good** — a stateless calculation with a table of worked examples in the design note:

```java
public class EntitlementCalculator {
    public Money weeklyEntitlement(ClaimType type, RateTable rates, int children) {
        Money base = rates.baseFor(type);
        return base.plus(rates.perChild().times(children));
    }
}
```

| Claim type | Children | Weekly entitlement |
| --- | --- | --- |
| Standard | 0 | £84.80 |
| Standard | 2 | £146.20 |

A domain service never loads or saves. If you cannot write a table like this, the logic is one or two `if`s and belongs in the entity (rule 1).

### 6. Give every ID and every business value its own type, and check its format once, where the value is created

An amount must not be negative. A file name must not be blank. A claimant ID must never be passed where a claim ID is expected.

**Bad** — checks sprinkled along the way:

```java
// in the controller
if (body.amount() < 0) return badRequest();
// in the service
if (amount < 0) throw new IllegalArgumentException();
// in the entity
if (amount < 0) throw new IllegalArgumentException();

public void close(String claimId, String claimantId, BigDecimal amount) { ... }   // three bare values
```

Three copies of the check, and the import job has none. And `close(claimantId, claimId, ...)` compiles, with the two IDs swapped.

**Good** — every ID and every business value is its own small type, and it refuses to exist with a bad value:

```java
public record ClaimId(String value) {
    public ClaimId { if (!value.matches("[0-9a-f]{4,}")) throw new InvalidClaimId(value); }
}

public record Money(BigDecimal amount) {
    public Money {
        if (amount.signum() < 0) throw new NegativeAmount(amount);
        if (amount.scale() > 2) throw new TooManyDecimalPlaces(amount);
        amount = amount.setScale(2);
    }
}

public void close(ClaimId id, int notesNeedingAction, LocalDate today) { ... }    // the compiler checks the types
```

`ClaimId`, `ClaimantId`, `NoteId`, `Money`, `NoteText`, `FileName`, `Nino`: never a bare `String` or `BigDecimal` inside the microservice. A bare string arrives only at the edges (a request body, a message, a database row) and becomes a type there. Once a `Money` exists, it is valid, everywhere, and two IDs cannot be swapped by mistake. A check that a code exists in another feature, such as a valid claim type, is not a format check: ask that feature's application service (rule 14).

### Where does this code go? Stop at the first yes

1. Does it decide whether the entity may change, from its current state? Entity method (rule 1).
2. Is it a calculation across several things, with worked examples? Domain service (rule 5).
3. Does it load, save, call another microservice, start a transaction or write audit? Application service (rule 3).
4. Does it check the format of one value? Value object or `create` (rule 6).
5. Does it check a code from another feature? That feature's application service (rule 14).

![How the building blocks handle one request: the application service runs the steps, and the aggregate decides. A controller or a message listener calls the application service. It asks other microservices through an adapter, loads and saves through the repository, and uses a domain service for a calculation. The Claim root checks the rules and raises events. The repository saves the whole aggregate, with its outbox entry, in one transaction, and the outbox sends the events to the message broker after the commit.](https://kodebot.com/books/java-spring-boot-in-depth/downloads/fig-ddd-guide-flow.png)

## Part B. Aggregates

### 7. Keep a child inside the aggregate only when the parent's rules need its details, it has a maximum count, and it changes with the parent; otherwise give it its own aggregate

You are deciding where evidence items and notes live.

**Bad** — everything a claim has, inside the claim:

```java
public class Claim {
    private final List<EvidenceItem> evidence = new ArrayList<>();
    private final List<Note> notes = new ArrayList<>();          // grows for years
    private final List<Payment> payments = new ArrayList<>();    // grows for years
}
```

Two caseworkers add a note to the same claim at the same time. The whole claim is one record, so the second save is refused as a conflict (rule 11). After three years the claim record holds 4,000 notes and the save fails on size.

**Good** — ask the four questions in the table about each child. Four times yes: inside. Any no: its own aggregate, holding the parent's ID.

| Child | Rule needs details? | Maximum? | Changes with parent? | Meaningless alone? | Decision |
| --- | --- | --- | --- | --- | --- |
| Evidence item | Yes: closing needs every item verified | 10 | Yes | Yes | Inside the claim |
| Note | No: closing needs only a count | None | No: two caseworkers at once | No | Own aggregate |
| Payment | No: only the latest | None | No: the payment microservice sends them | No | Own aggregate |

```java
public class Claim {
    private final List<EvidenceItem> evidence = new ArrayList<>();   // inside
}

public class Note {
    private final NoteId id;
    private final ClaimId claimId;                                   // points at the parent
    private final NoteText text;
    private boolean needsAction;
    private long version;
}
```

Inside one aggregate a rule is always true, because the whole aggregate is saved in one transaction. Between aggregates a rule is true only at the moment it is checked, which is why `close` asks for the note count at that moment. Copying the old system's tables one to one as aggregates is the most common way to get this wrong.

### 8. Store another aggregate's ID, never the object itself

A note belongs to a claim.

**Bad**:

```java
public class Note {
    private Claim claim;
}
```

Loading a note now loads the claim and its evidence. Saving a note saves the claim. You have one large aggregate again, by accident.

**Good**:

```java
public class Note {
    private final ClaimId claimId;
}
```

When the note feature needs something about the claim, it asks `ClaimService` with the ID (rule 14).

### 9. Save one aggregate per transaction

A nightly job must mark 10,000 claims as "review due".

**Bad**:

```java
// one transaction
public void markReviewDue(LocalDate today) {
    for (Claim claim : claims.findReviewDue(today)) {
        claim.markReviewDue(today);
        claims.save(claim);
    }
}
```

One transaction for 10,000 records holds locks for minutes, blocks caseworkers, and if record 9,999 fails, nothing is saved.

**Good** — one transaction per record, and skip records already done:

```java
public void markReviewDue(LocalDate today) {
    for (ClaimId id : claims.findIdsReviewDue(today)) {
        markOne(id, today);                                  // its own transaction
    }
}

// one transaction
void markOne(ClaimId id, LocalDate today) {
    Claim claim = claims.get(id);
    if (claim.isReviewDue()) return;                         // already done; safe to rerun
    claim.markReviewDue(today);
    claims.save(claim);
    audit.record("review due", id);
}
```

Saving two aggregates in one transaction needs an approved exception in the design note. Example: creating a claim together with its first note.

### 10. Give every list inside an aggregate a maximum size

A claim holds evidence items.

**Bad**:

```java
public void addEvidence(FileName fileName, LocalDate today) {
    evidence.add(new EvidenceItem(fileName, today));
}
```

A stored record has a size limit. One day a claim with 600 scanned pages fails to save, in production, for one user.

**Good** — the design note says "at most 10", and the entity refuses the eleventh:

```java
private static final int MAX_EVIDENCE = 10;

public void addEvidence(FileName fileName, LocalDate today) {
    if (status == CLOSED) throw new ClaimAlreadyClosed(id);
    if (evidence.size() >= MAX_EVIDENCE) throw new TooMuchEvidence(id, MAX_EVIDENCE);
    EvidenceItem item = EvidenceItem.create(fileName, today);
    evidence.add(item);
    events.add(new EvidenceAdded(id, item.id()));             // the ID, not the file name (rule 26)
}
```

A list that cannot have a maximum is not a child; it is its own aggregate (rule 7).

### 11. Refuse a save that was made from an old version of the record

Two caseworkers open the same claim. One adds evidence and saves. The other, still looking at the old screen, closes the claim.

**Bad** — the second save wins:

```java
Claim claim = claims.get(id);
claim.close(open, today);
claims.save(claim);                                          // overwrites the new evidence
```

The first caseworker's evidence is gone and nobody is told.

**Good** — every change carries the version the user saw, and the entity refuses an old one:

```java
public class Claim {
    public void requireVersion(long expected) {
        if (version != expected) throw new ClaimChangedSinceYouLoadedIt(id, expected, version);
    }
}
```

The screen shows "this claim was changed by someone else; reload". `requireVersion` is step 2 of every change (rule 3) and gives the friendly error. The repository repeats the check in the database (`WHERE version = ?`, see the walkthrough), because two saves in the same millisecond can both pass the entity's check. Never remove either check or write to one field directly to get around them.

### 12. Make every change safe to repeat: a repeated request must not add or count anything twice

The payment microservice sends `PaymentMadeV1`. The broker delivers it twice.

**Bad**:

```java
public void recordPayment(Money amount) {
    totalPaid = totalPaid.plus(amount);                      // twice is wrong
}
```

**Good** — the entity keeps what it already has, and ignores a repeat:

```java
public void recordPayment(PaymentId paymentId, Money amount, LocalDate paidOn) {
    if (lastPayment != null && !paidOn.isAfter(lastPayment.paidOn())) return;   // older or same: ignore
    lastPayment = new Payment(paymentId, amount, paidOn);
    events.add(new PaymentRecorded(id, paymentId));
}
```

A request is repeated after a timeout, because the caller cannot tell whether the first one landed. Three things together make a repeat harmless: the version check for changes from a screen (rule 11), an idempotency key for create requests (a repeat with the same key returns the first result), and a record of processed message IDs for messages (rule 21).

### 13. Store money as an exact decimal and time in UTC, and count days in UK dates

A claim closed on 1 March can be reopened within 28 days.

**Bad**:

```java
double amount = 0.1 + 0.2;                                   // 0.30000000000000004
Instant deadline = closedAt.plus(28 * 24, ChronoUnit.HOURS); // an hour out after the clocks change
```

**Good**:

```java
public record Money(BigDecimal amount) { ... }               // two decimal places, exact (rule 6)

static final ZoneId UK = ZoneId.of("Europe/London");         // one constant, used everywhere

Instant closedAt = clock.instant();                          // stored in UTC
LocalDate closedOn = closedAt.atZone(UK).toLocalDate();
LocalDate deadline = closedOn.plusDays(28);                  // a date, in UK time
```

Floating point gives penny errors. Local timestamps are an hour wrong twice a year. A rule that counts days works in dates, not hours.

## Part C. Features, microservices and events

### 14. Use another feature only through its application service, passing IDs and simple values

The claim feature needs to know how many notes need action.

**Bad** — reaching into the other feature's storage:

```java
public class ClaimService {
    private final NoteRepository notes;                      // another feature's repository
    ...
    long open = notes.findByClaimId(id).stream().filter(Note::needsAction).count();
}
```

When the notes team changes how notes are stored, the claim feature breaks. The "needs action" rule now lives in two features.

**Good** — ask the other feature's application service for the answer:

```java
public class ClaimService {
    private final NoteService notes;                         // the other feature's front door
    ...
    int open = notes.countNeedingAction(id);
}
```

Pass IDs, numbers and small records, never entities or repositories. Calls go one way: claims may call notes, so notes must not call claims; notes use events for the other direction (rule 19). A build check enforces this.

### 15. Check permissions in the controller, never in the service or the entity

Only a caseworker with `CLOSE_CLAIM` may close a claim.

**Bad**:

```java
public class Claim {
    public void close(User user, int open, LocalDate today) {
        if (!user.hasRole("CASEWORKER")) throw new Forbidden();
        ...
    }
}
```

A message listener also closes claims (when the payment microservice says the last payment was made). There is no user there, so someone passes a fake system user, and the check means nothing.

**Good** — the controller knows the user, so the controller checks:

```java
// POST /claims/{id}/close
public void close(ClaimId id, CloseRequest body, User user) {
    permissions.require(user, CLOSE_CLAIM);
    claimService.close(id, body.version());
    // reply 204, no body
}
```

A test proves that a user without the permission gets a refusal. Application services and entities never see a user.

### 16. Talk to another system only through one adapter class that translates its data into our types

Creating a claim needs the eligibility microservice's answer.

**Bad** — their data leaks into our entity:

```java
Map<String, Object> response = httpClient.get("/eligibility/" + nino);
if ("ELIGIBLE".equals(response.get("status"))) { ... }       // their word, inside our code
```

When the eligibility team renames `status` to `outcome`, we hunt through our whole codebase.

**Good** — one adapter class, returning our own type and our own errors:

```java
public class EligibilityClient {
    public Eligibility check(ClaimantId claimantId) {
        EligibilityResponse theirs = http.get("/eligibility/" + claimantId.value(), EligibilityResponse.class);
        return switch (theirs.status()) {
            case "ELIGIBLE" -> Eligibility.ELIGIBLE;
            case "NOT_ELIGIBLE" -> Eligibility.NOT_ELIGIBLE;
            default -> throw new EligibilityUnknown(claimantId);
        };
    }
}
```

Nothing from the other system appears outside its adapter. Nobody reads another microservice's database.

### 17. Call another microservice before you open the transaction, never inside it

Creating a claim asks eligibility, then saves.

**Bad**:

```java
// one transaction
public ClaimId open(ClaimantId claimantId, ClaimType type) {
    Eligibility eligibility = eligibilityClient.check(claimantId);   // a network call, inside the transaction
    Claim claim = Claim.open(claimantId, type, eligibility, clock.today());
    claims.save(claim);
    return claim.id();
}
```

Eligibility takes four seconds today. For four seconds the transaction holds its locks, other caseworkers wait, and the database's transaction time limit is close.

**Good** — the call first, then the transaction:

```java
public ClaimId open(ClaimantId claimantId, ClaimType type) {
    Eligibility eligibility = eligibilityClient.check(claimantId);   // outside
    return saveNew(claimantId, type, eligibility);
}

// one transaction
ClaimId saveNew(ClaimantId claimantId, ClaimType type, Eligibility eligibility) {
    Claim claim = Claim.open(claimantId, type, eligibility, clock.today());
    claims.save(claim);
    audit.record("claim opened", claim.id());
    return claim.id();
}
```

Get every outside fact first (step 3 in rule 3), then open the transaction and save.

### 18. Retry a read; never retry a change

The eligibility check timed out.

**Bad** — retry everything the same way:

```java
retry(3, () -> addressClient.changeAddress(claimantId, newAddress));
```

The first call reached the other system and changed the address; only the reply was lost. The retry changes it again, and the audit trail there now shows two changes we cannot undo.

**Good** — reads may retry; a change is sent once:

```java
Eligibility eligibility = retry(3, () -> eligibilityClient.check(claimantId));   // a read: safe

try {
    addressClient.changeAddress(claimantId, newAddress);                          // a change: once
} catch (Unreachable e) {
    throw new AddressNotChanged(claimantId);              // nothing happened; tell the user
} catch (TimedOut e) {
    throw new AddressChangeUnknown(claimantId);           // maybe it happened; the user checks first
}
```

Stop calling a system that keeps failing, for a short time (a circuit breaker). A business action that sends a change to another system makes exactly one call and saves nothing else of its own.

### 19. Save an event in the same transaction as the change it reports, and send it afterwards

Closing a claim must tell the payment microservice.

**Bad** — publish from the service:

```java
claims.save(claim);
broker.publish(new ClaimClosedV1(id));                   // the broker is down: saved, never sent
```

Or the other way round: published first, then the save fails. Payments stop for a claim that is still open.

**Good** — the entity records the event; the repository stores it in the same transaction (an outbox); a background sender publishes it:

```java
// in Claim.close
events.add(new ClaimClosed(id, today));

// in ClaimRepository.save, one transaction (see the walkthrough)
db.save(claim);
outbox.add(claim.pullEvents());

// a background sender, every second
for (OutboxEntry entry : outbox.unsent()) {
    broker.publish(entry.toIntegrationEvent());          // ClaimClosedV1
    outbox.markSent(entry);                              // retried next time if the broker is down
}
```

A saved change always sends its event, and an unsaved change never does. Nobody publishes any other way.

### 20. Only add fields to a published event; to remove or rename one, publish a new version

The payment team reads `ClaimClosedV1.closedOn`. You want to call it `closureDate`.

**Bad**:

```java
public record ClaimClosedV1(String eventId, String claimId, LocalDate closureDate) {}   // renamed in place
```

The payment microservice's listener reads `closedOn`, gets null, and stops payments on the wrong date.

**Good** — add freely; for a rename, add a version and send both until every receiver has moved:

```java
public record ClaimClosedV1(String eventId, String claimId, LocalDate closedOn, String closedByUserId) {}   // a new field: fine
public record ClaimClosedV2(String eventId, String claimId, LocalDate closureDate, String closedByUserId) {} // sent beside V1
```

Every integration event has a written schema. Tell the receiving teams before the release. A domain event, used only inside our microservice, can change at any time.

### 21. Write every message listener so a message that arrives late, twice or out of order does no harm

The payment microservice sends `PaymentMadeV1`. The broker may deliver it twice, or deliver March's payment after April's.

**Bad**:

```java
// called by the broker for each PaymentMadeV1
public void on(PaymentMadeV1 event) {
    Claim claim = claims.get(event.claimId());
    claim.setLastPayment(event.amount(), event.paidOn());    // the last one to arrive wins
    claims.save(claim);
}
```

**Good** — an inbox skips repeats, and the entity ignores anything older than what it holds:

```java
// called by the broker for each PaymentMadeV1
// one transaction
public void on(PaymentMadeV1 event) {
    if (inbox.alreadyProcessed(event.eventId())) return;    // seen it: skip
    claimService.recordPayment(                              // translate their strings into our types
        new ClaimId(event.claimId()), new PaymentId(event.paymentId()),
        new Money(event.amount()), event.paidOn());
    inbox.record(event.eventId());                           // same transaction as the change
}

// in Claim.recordPayment (rule 12)
if (lastPayment != null && !paidOn.isAfter(lastPayment.paidOn())) return;
```

The listener only translates the message and calls an application service. A message that fails repeatedly goes to a dead-letter queue, where someone finds the cause before it is re-sent. No message broker delivers exactly once and in order.

### 22. Change several microservices as a chain of steps joined by events, with a named step for failure

Closing a claim must also stop its payments, which live in the payment microservice.

**Bad** — one transaction across two microservices:

```java
// one transaction
public void close(ClaimId id, long version) {
    ...
    claims.save(claim);
    paymentClient.stopPayments(id);                          // another microservice, inside our transaction
}
```

A transaction cannot cover two databases. If `stopPayments` succeeds and our commit fails, payments are stopped for an open claim.

**Good** — each step changes one microservice and sends an event that starts the next:

```
1. Claim microservice:   Claim.close  → saves → sends ClaimClosedV1
2. Payment microservice: on ClaimClosedV1 → stops payments → sends PaymentsStoppedV1
3. Claim microservice:   on PaymentsStoppedV1 → Claim.markPaymentsStopped
   If step 2 fails for good: payment microservice sends PaymentsStopFailedV1,
   and we create a task for a caseworker.
```

A failed step cannot be rolled back, so the design names what happens instead, and that is a new business action. Each such process needs design group approval and an architecture decision record (ADR). Do not use events to keep a rule true across two aggregates; that is a sign the two should be one (rule 7).

## Part D. Process

### 23. Get the design note approved before you code a feature with business rules

You are about to build the notes feature.

**Bad**: start coding, and discover in review that notes should have been inside the claim, or the other way round. Moving children across an aggregate boundary means a data migration.

**Good**: a one-page design note, approved by the design group first:

```
Purpose:     caseworkers record notes against a claim; a note can need action
Aggregates:  Note (own aggregate, holds claimId); no maximum
States:      OPEN -> ACTIONED (markActioned)
Rules:       a claim cannot close while any note needs action
             enforced in Claim.close, with the count passed in
             example: claim C-1 has 2 notes needing action -> close refused
Events:      NoteNeedsActionV1, NoteActionedV1; used by the claim feature
Talks to:    nothing outside the microservice
Exceptions:  none
Open questions: can a closed claim get a note?
```

The design group decides what each aggregate contains, the rules with worked examples, and any transaction that saves two aggregates. Engineers decide endpoints, classes and tests. Which microservices exist is an ADR. A wrong aggregate boundary is the most expensive mistake to fix later.

### 24. Mark a feature Simple (data with no rules) or Rich (states and rules) in its design note

You are adding a list of claim types that the business maintains.

**Bad**: a `ClaimType` aggregate with version checks, events and an outbox, for a list of 12 codes that changes twice a year.

**Good**: mark it **Simple** in the design note. A Simple feature is data with no business rules: lookup lists, user preferences. Its entity has fields and getters, and checks may sit in the application service.

```java
public record ClaimType(String code, String label, boolean active) {}
```

Claims, payments and awards are **Rich**: states and rules. Everything in this guide applies to them. The moment a Simple feature gains a rule ("a claim type cannot be deactivated while claims use it"), it becomes Rich and gets an entity method for that rule.

### 25. Fix a mistake with a safe step today and the proper fix by a date

You find that `ClaimService` holds an `if` on status that should be in `Claim`.

Do the safe step now. Tell the design group the same day. Create a ticket with a date. Then do the proper fix and add a test, so the same mistake fails the build next time. Rules 14 and 20 are what make this possible: a feature can change inside without the others noticing.

| You find | Today | Proper fix | Never |
| --- | --- | --- | --- |
| Two aggregates share a rule that must always be true | Save both in one transaction; add the exception to the design note | Merge them; a migration moves the children inside the parent | Keep the rule true with events |
| An aggregate gives frequent conflicts or keeps growing | Keep the version check; make the screen say "reload" | Move the part that changes often into its own aggregate | Turn off the version check |
| A rule sits in a service, controller or listener | Ticket for this sprint | Move it into the entity method; add a test | Copy the `if` elsewhere |
| A feature uses another feature's repository | Record today's violations so only new ones fail the build | Add a method on the other feature's application service | Turn the build check off |
| A change sent no event | Add the missing event to the entity method | A one-off job sends the missed events, agreed with receivers | Let other teams read our database |
| Messages are in the dead-letter queue | Find the cause with the trace ID and fix it | Re-send them; the inbox skips duplicates | Delete them before you understand them |
| Our data and another microservice's disagree | Use both audit logs to find which side is right | A job that reports differences; fix through entity methods | A job that silently overwrites one side |
| Data was repaired by writing to the database | Find the affected records | Repair through entity methods or a reviewed migration | Repair by hand again |

### 26. Keep personal data out of logs, error messages and events

A claim is refused because the National Insurance number is wrong.

**Bad**:

```java
throw new IllegalArgumentException("Invalid NINO: " + nino);     // lands in the logs
public record ClaimOpenedV1(String claimId, String nino, String fullName) {}
```

Logs and events are copied to monitoring tools and other teams' systems, which the data protection rules for our database do not cover.

**Good**:

```java
public record Nino(String value) {
    @Override public String toString() { return "Nino(****" + value.substring(5) + ")"; }
}
throw new InvalidNino(claimantId);                        // the ID, not the number
public record ClaimOpenedV1(String eventId, String claimId, String claimantId) {}
```

Value objects hide personal data when printed. Events carry IDs, not personal details. If personal data reached production logs, report it through the data protection process.

## Part E. Situations you will meet

### 27. When one button must save two aggregates, first ask whether they are one; then react to an event; only then save both in one approved transaction

The Create claim screen ends with a Check answers page that creates the claim and its first note.

**Bad** — two saves in a service method, with no decision made:

```java
public ClaimId open(ClaimantId claimantId, ClaimType type, NoteText firstNote) {
    Claim claim = Claim.open(claimantId, type, eligibility, clock.today());
    claims.save(claim);                                      // transaction 1
    Note note = Note.create(claim.id(), firstNote, clock.today());
    notes.save(note);                                        // transaction 2; the process dies here: claim, no note
    return claim.id();
}
```

Half the user's work is saved and the screen shows an error, so the user creates the claim again.

**Good** — three answers, in this order. First, is the second thing really a child? If yes, it goes inside (rule 7) and there is one save. Second, can the second thing follow a moment later? Then the first aggregate raises a domain event and a listener in our own microservice creates the second:

```java
// in Claim.open, when a first note was given
events.add(new ClaimOpened(id));
events.add(new FirstNoteRequested(id, firstNote));           // firstNote is a NoteText value object

// a listener inside the claim microservice, its own transaction
void on(FirstNoteRequested e) {
    if (inbox.alreadyProcessed(e.eventId())) return;
    noteService.add(e.claimId(), e.firstNote());             // the note feature's front door (rule 14)
    inbox.record(e.eventId());
}
```

Third, when the user must see both at once and neither answer fits, save both in one transaction, with the exception written in the design note:

```java
// one transaction; approved exception "create claim with first note" in the design note
public ClaimId open(ClaimantId claimantId, ClaimType type, NoteText firstNote) {
    Eligibility eligibility = eligibilityClient.check(claimantId);        // before the transaction (rule 17)
    return transaction(() -> {
        Claim claim = Claim.open(claimantId, type, eligibility, clock.today());
        Note note = Note.create(claim.id(), firstNote, clock.today());
        claims.save(claim);
        notes.save(note);
        audit.record("claim opened with first note", claim.id());
        return claim.id();
    });
}
```

The event answer is the normal one. The two-save answer is allowed because both aggregates are in our database; it is never allowed across microservices (rule 22).

### 28. Keep a rule that spans all records of a kind true with a unique database index, and turn the index error into a business error

The rule: a claimant has at most one open claim.

**Bad** — check, then save:

```java
if (claims.hasOpenClaimFor(claimantId)) throw new ClaimantAlreadyHasOpenClaim(claimantId);
claims.save(Claim.open(claimantId, ...));                    // two caseworkers do this at the same second
```

Both checks pass, both saves succeed, and the claimant has two open claims.

**Good** — the database holds the rule; the repository translates its error:

```sql
CREATE UNIQUE INDEX one_open_claim_per_claimant ON claim (claimant_id) WHERE status = 'OPEN';
```

```java
public void save(Claim claim) {
    try {
        int rows = db.save(claim);                           // as in the walkthrough
        if (rows == 0) throw new ClaimChangedSinceYouLoadedIt(claim.id());
        outbox.add(claim.pullEvents());
    } catch (UniqueIndexViolation e) {
        if (e.index().equals("one_open_claim_per_claimant")) throw new ClaimantAlreadyHasOpenClaim(claim.claimantId());
        throw e;
    }
}
```

An aggregate can only guarantee rules about itself. A rule about all claims is a database rule, and the index is the only check that two users cannot race past. Keep the `if` as well, for a friendly error in the common case.

### 29. Create an entity through a named factory method that checks the creation rules and raises the event; keep the constructor private

You are writing the code that creates a claim.

**Bad**:

```java
Claim claim = new Claim();
claim.setClaimantId(claimantId);
claim.setStatus(OPEN);
claims.save(claim);
```

Nothing checks eligibility or the claim type, and no `ClaimOpened` event is sent.

**Good**:

```java
public class Claim {
    private Claim(ClaimId id, ClaimantId claimantId, ClaimType type, LocalDate openedOn) { ... }

    public static Claim open(ClaimantId claimantId, ClaimType type, Eligibility eligibility, LocalDate today) {
        if (eligibility != ELIGIBLE) throw new ClaimantNotEligible(claimantId);
        if (!type.active()) throw new ClaimTypeNotActive(type);
        Claim claim = new Claim(ClaimId.next(), claimantId, type, today);
        claim.events.add(new ClaimOpened(claim.id));
        return claim;
    }
}
```

Creation is a business action like any other, so it has a business name and its rules live in one place. The only `new Claim(...)` is inside `Claim` and inside the repository when it loads.

### 30. Read for a screen through a query class that reads the tables directly; never build a screen out of aggregates

The claim list shows the claim ID, status, claimant name, number of open notes and last payment date, for 50 claims a page.

**Bad** — load aggregates and pick fields out:

```java
List<ClaimRow> rows = new ArrayList<>();
for (Claim claim : claims.findOpen(page)) {                  // 50 full aggregates with evidence
    int open = notes.countNeedingAction(claim.id());         // 50 more queries
    rows.add(new ClaimRow(claim.id(), claim.status(), open, claim.lastPayment()));
}
```

Fifty-one database round trips for one page, and the aggregate code now has getters that exist only for screens.

**Good** — a query class, one SQL statement, a flat record:

```java
public class ClaimListQuery {
    public Page<ClaimRow> openClaims(int page) {
        return db.query("""
            SELECT c.id, c.status, c.claimant_name, c.last_paid_on,
                   (SELECT COUNT(*) FROM note n WHERE n.claim_id = c.id AND n.needs_action) AS open_notes
            FROM claim c WHERE c.status = 'OPEN'
            ORDER BY c.opened_on DESC LIMIT 50 OFFSET ?""", page * 50, ClaimRow::from);
    }
}
```

Aggregates and repositories exist to change data safely. Reads have no rules to protect, so they go straight to the tables, join across aggregates freely, and return exactly what the screen needs. A query class never changes anything, and a rule never lives in a query.

### 31. Search and reports are queries too; give a slow one its own read table filled by events

The business wants a report of claims closed per month with total payments.

**Bad** — compute it from aggregates at request time:

```java
for (Claim claim : claims.findAll()) { ... }                 // 2 million claims
```

**Good** — a query over the tables (rule 30). When that is too slow, keep a separate read table that the domain events fill:

```java
// a listener in our own microservice
void on(ClaimClosed e) {
    monthlyClosures.increment(YearMonth.from(e.closedOn()));
}
```

```sql
SELECT month, closed_count, total_paid FROM monthly_closure_report ORDER BY month;
```

The read table is a copy, built from events, and can be rebuilt by replaying them. Nothing is ever changed through it.

### 32. Do not store on a parent what a child aggregate knows; ask for it when reading, or copy it through an event and accept a short delay

The claim page shows "has notes needing action".

**Bad** — a flag on the claim, set by the note feature:

```java
// in NoteService.markNeedsAction
Claim claim = claims.get(note.claimId());
claim.setHasOpenNotes(true);                                 // another feature changes our aggregate
claims.save(claim);
```

Two aggregates are saved in one action, the flag drifts when a note is actioned on another path, and the note feature now depends on the claim's internals.

**Good** — the normal answer: ask when reading (rule 30, the `open_notes` column). When the flag must be on the claim itself, for a rule or for speed, copy it through an event:

```java
void on(NoteNeedsAction e) {
    Claim claim = claims.get(e.claimId());
    claim.noteNeedsAction(e.noteId());                       // a method, not a setter; records the note ID
    claims.save(claim);
}
```

The copy is a second behind, and the design note says so. For the real rule, `close` still asks for the count at the moment of closing (rule 1), because a copy is never the truth.

### 33. Never delete a business record; add a business action that marks it, with a reason

A caseworker created a claim for the wrong claimant.

**Bad**:

```java
claims.delete(claimId);
```

The audit trail, the notes that point at the claim and the events already sent now point at nothing, and the payment microservice may still act on it.

**Good**:

```java
public void cancel(CancellationReason reason, LocalDate today) {
    if (status == CLOSED) throw new ClaimAlreadyClosed(id);
    status = CANCELLED;
    cancelledOn = today;
    events.add(new ClaimCancelled(id, reason, today));
}
```

Lists hide cancelled claims by default. A deletion required by the data protection rules is a separate, approved process that removes personal data and keeps the record's skeleton.

### 34. Run a rule that depends on time from a scheduled job that calls the same entity method, one record per transaction

A claim with no activity for 90 days is closed automatically.

**Bad** — a job that updates the table:

```sql
UPDATE claim SET status = 'CLOSED' WHERE last_activity_on < now() - interval '90 days';
```

No rule check, no event, no audit; the payment microservice never hears about it.

**Good**:

```java
public void closeInactiveClaims(LocalDate today) {
    for (ClaimId id : claims.findIdsInactiveSince(today.minusDays(90))) {
        closeOne(id, today);                                 // one transaction each (rule 9)
    }
}

void closeOne(ClaimId id, LocalDate today) {
    Claim claim = claims.get(id);
    int open = notes.countNeedingAction(id);
    claim.closeForInactivity(open, today);                   // the entity decides; refuses if notes are open
    claims.save(claim);
    audit.record("claim closed for inactivity", id);
}
```

The job is just another caller of the entity, like the controller and the message listener. The time comes from the injected clock, so a test can set today to any date.

### 35. When a process has its own steps and deadlines, give it its own aggregate

Closing a claim with payments takes three steps across two microservices (rule 22), and the business wants to see which step each claim is at, and chase one that is stuck for two days.

**Bad** — flags on the claim:

```java
private boolean closeRequested, paymentsStopped, paymentsStopFailed, letterSent;
```

The claim's methods fill up with process `if`s, and nobody can see the whole process in one place.

**Good** — a `ClaimClosure` aggregate that holds the state of the process and reacts to the events:

```java
public class ClaimClosure {
    private final ClaimId claimId;
    private ClosureStep step;                                // REQUESTED, PAYMENTS_STOPPED, LETTER_SENT, DONE, STUCK
    private LocalDate stepStartedOn;                         // a UK date (rule 13)

    public void paymentsStopped(LocalDate today) {
        if (step != REQUESTED) return;                       // late or repeated event: ignore (rule 21)
        step = PAYMENTS_STOPPED;
        stepStartedOn = today;
        events.add(new ClosureLetterRequested(claimId));
    }

    public void checkDeadline(LocalDate today) {
        if (step == DONE || step == STUCK) return;
        if (today.isAfter(stepStartedOn.plusDays(2))) {
            step = STUCK;
            events.add(new ClosureStuck(claimId, step));
        }
    }
}
```

The claim keeps its own rules; the process keeps its steps. This is the "directing microservice" of the API guide, at the scale of one aggregate.

### 36. Change an aggregate's shape behind a flag, with a migration that uses the entity's own methods

Notes were inside the claim; the design group decided they are their own aggregate (rule 7).

**Bad** — a SQL script on release night:

```sql
INSERT INTO note (id, claim_id, text) SELECT ... FROM claim_notes;
```

Half the claims are migrated when the script fails, the old code reads the rest, and the two sets drift.

**Good**:

1. Release 1: the new `Note` aggregate and repository exist; new notes go there; reads check both places behind a feature flag.
2. A migration job, one claim per transaction (rule 9), creates the `Note` aggregates through `Note.create` and marks the claim as migrated; it can be rerun.
3. Release 2: the flag is on for everyone; the old list is read by nobody.
4. Release 3: the old column and the flag are removed.

The migration goes through the entity's methods so that every rule and event still holds. Rule 25's "proper fix" column is this list.

### 37. Test an entity with no database, a service with a fake repository, and the repository alone against a real database

You are writing the tests for "close claim".

**Bad** — every test starts the whole microservice and a database:

```java
void closeRefusedWhenNotesOpen() {
    db.insert(claimRow, noteRow);                            // 40 lines of setup
    http.post("/claims/9f1c/close");
    assertEquals(422, response.status());
}
```

Ten seconds per test, and the rule being tested is buried under HTTP and SQL.

**Good** — three layers, each tested for what it owns:

```java
// the entity: the rule, in memory, in milliseconds
void closeRefusedWhenNotesOpen() {
    Claim claim = Claim.open(claimantId, ClaimTypes.standard(), ELIGIBLE, date("2026-10-01"));
    assertThrows(ClaimHasOpenActions.class, () -> claim.close(2, date("2026-10-31")));
}

// the service: the steps, with a fake repository and a fixed clock
void closeSavesAndAudits() {
    service.close(claimId, 7);
    assertTrue(fakeClaims.saved(claimId));
    assertEquals("claim closed", fakeAudit.last());
}

// the repository: the mapping, against a real database, one test class
void saveAndLoadRoundTrip() { ... }
```

Most tests are entity tests, because that is where the rules are. One test per endpoint proves the permission check (rule 15). A test that sets the clock to 29 March proves the date rule (rule 13).

### 38. Show two aggregates on one screen and save them with two buttons; one button that saves two things is a design smell to raise

The claim page shows the claim and lets the caseworker edit contact details and add a note.

**Bad** — one Save button that sends both:

```java
public void saveClaimPage(ClaimId id, long version, ContactDetails contact, NoteText noteText) {
    Claim claim = claims.get(id);
    claim.requireVersion(version);
    claim.changeContact(contact);
    claims.save(claim);
    notes.save(Note.create(id, noteText, clock.today()));    // two aggregates, one button (rule 27)
}
```

**Good** — each part of the screen saves its own aggregate with its own button and its own version:

```
[Contact details]   Save contact details  -> ClaimService.changeContact(claimId, version, contact)
[Add a note]        Add note              -> NoteService.add(claimId, noteText)
```

When a screen keeps needing one button for two aggregates, say so to the design group: either the boundary is wrong (rule 7) or the action deserves a name of its own (rule 27).

## Who to ask

Ask a senior engineer the same day when you do not know where code belongs. Ask the design group, before you code, for a new or changed aggregate, a two-aggregate transaction or a new link between microservices. Tell the receiving teams before you change a published event. Report a security or personal data problem through the incident process at once.

Still to fill in: design group members, team channel, design session time, incident process link.
