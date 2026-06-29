---
title: "Ghost in the Object — How OOP Design Patterns Become Attack Patterns"
date: 2026-06-29
permalink: /writeups/ghost-in-the-object/
excerpt: "Mass assignment is just the beginning. A framework for five OOP misuse patterns that turn clean object design into exploitable business logic bugs."
header:
  image: /assets/images/ghost-in-the-object-cover.jpg
  og_image: /assets/images/ghost-in-the-object-cover.jpg
  teaser: /assets/images/ghost-in-the-object-cover.jpg
categories:
  - writeups
tags:
  - bug-bounty
  - business-logic
  - mobile
  - android
  - ios
  - oop
---

Most developers think in features. An attacker thinks in objects.

Every backend system built in an OOP language — Java, Python, Ruby, Kotlin — models the real world as objects with properties and behaviors. A `Hotel` has an `allowSpecifyRoom` flag. A `Booking` has a `roomNumber` field. A `User` has a `role`.

The vulnerability isn't always in the code that handles input. Sometimes it's in the **assumptions baked into the object model itself** — assumptions the UI enforces, but the backend never validates.

This is a class of bugs broader than mass assignment. Broader than IDOR. It lives at the intersection of **business logic and object design.**

---

## The Mental Model

A booking app has hotels. Each hotel is an object with different capabilities:

```
Hotel A → normal room only
Hotel B → normal room + breakfast
Hotel C → normal room + choose your room number
```

Each hotel is an instance of the same `Hotel` class — or a subclass. Each booking flow hits a different endpoint. Each endpoint may or may not validate the hotel's capabilities before processing your input.

That gap — between what the object *says it supports* and what the *flow actually checks* — is the attack surface.

---

## Pattern 1: Capability Bleed

### The Concept

Objects carry capabilities as properties. The UI respects them. The backend — across multiple flows — often doesn't.

```python
class Hotel:
    def __init__(self):
        self.allowSpecifyRoom = False
        self.includesBreakfast = False
```

Hotel A has `allowSpecifyRoom = False`. The UI hides the room picker. You never see the field.

But the backend booking handler looks like this:

```python
# Vulnerable flow
def book(userInput):
    booking = Booking(userInput)  # blindly maps all fields
    db.save(booking)
```

No check against `hotel.allowSpecifyRoom`. The object's capability flag exists — it's just never consulted.

### The Attack

Intercept the booking request for Hotel A. Inject:

```json
{
  "hotelId": "HotelA_123",
  "checkIn": "2025-08-01",
  "checkOut": "2025-08-05",
  "roomNumber": "101"
}
```

If the flow doesn't validate the capability, `roomNumber` gets mapped and saved.

### Why Multiple Flows Matter

The same `Booking` class is reused across flows. The premium flow validates `allowSpecifyRoom`. The standard flow — built six months earlier by a different team — doesn't. Same class. Different instantiation logic. Different validation.

### Impact

Ranges from cosmetic (getting a preferred floor) to critical (booking reserved rooms, out-of-service rooms, or rooms at incorrect prices).

> This is similar to mass assignment but broader. Mass assignment is about *what fields exist*. Capability Bleed is about *whether this object instance is allowed to use those fields at all*.

---

## Pattern 2: Polymorphism Abuse

### The Concept

Polymorphism lets subclasses extend base classes. In a well-designed system, a `PremiumBooking` extends `Booking` and adds fields like `roomPreference` or `upgradeEligible`.

The vulnerability: the backend accepts a base class type but you serialize a subclass payload. The validator only knows about the base class — it ignores fields it doesn't recognize, or worse, it deserializes them anyway.

### The Attack

Normal booking payload:

```json
{
  "hotelId": "123",
  "type": "standard",
  "checkIn": "2025-08-01"
}
```

You send a subclass payload:

```json
{
  "hotelId": "123",
  "type": "premium",
  "checkIn": "2025-08-01",
  "upgradeEligible": true,
  "loyaltyTier": "platinum"
}
```

If the backend deserializes based on `type` without validating your account's actual tier, you've self-upgraded.

### Why It Works

Polymorphic deserialization is hard to secure. Libraries like Jackson (Java) or Marshmallow (Python) can be configured to deserialize into subclasses based on a type discriminator field — a field *you control*.

### Impact

Privilege escalation at the object level. Not a role change. Not an IDOR. You're injecting yourself into a higher-capability object type.

---

## Pattern 3: Lazy Initialization / Null Default Abuse

### The Concept

Many object fields are initialized to `null` or `false` as safe defaults. The backend logic reads:

```python
if booking.roomNumber is not None:
    assign_room(booking.roomNumber)
```

The assumption: only flows that explicitly support room selection will ever populate `roomNumber`. So the check feels safe.

But `roomNumber` being non-null is the *only* guard. There's no capability check on the hotel object itself.

### The Attack

Simply send the field. The backend's null-check passes. The room gets assigned.

```json
{
  "hotelId": "123",
  "roomNumber": "penthouse"
}
```

### Why It Works

Developers write null checks as data-integrity guards, not security controls. The mental model during development is: *"this field will only be set by our premium flow."* The attacker's mental model is: *"this field will be set by me."*

---

## Pattern 4: Shared Constructor, Different Context

### The Concept

The same constructor is called from multiple places in the codebase. Some callers add guards before calling it. Others call it raw.

```python
# Secure caller (premium endpoint)
if hotel.allowSpecifyRoom and user.isPremium:
    booking = Booking(userInput)

# Insecure caller (legacy endpoint)
booking = Booking(userInput)  # no guards
```

The constructor itself doesn't validate. It trusts its callers to do that. One caller doesn't.

### The Attack

Find the legacy or secondary endpoint. It accepts the same `Booking` object. It calls the same constructor. But it skips the guards.

You discover this by:
- Mapping all booking-related endpoints in the app
- Comparing request/response structures across flows
- Sending the same payload to each endpoint and observing differences

### Why Multiple Endpoints Exist

Apps grow. Teams change. A "quick" new booking flow gets shipped. It reuses the existing `Booking` class because that's good engineering. But it doesn't reuse the validation logic because that's in a different service, or nobody remembered it was there.

### Impact

Full bypass of business logic enforced by the secure flow. The insecure endpoint is functionally equivalent but without the guardrails.

---

## Pattern 5: State Machine Skip

### The Concept

Objects have states. A booking moves through:

```
PENDING → CONFIRMED → PAID → CHECKED_IN
```

Methods on the object are designed to be called only in specific states. `generateInvoice()` only makes sense after `CONFIRMED`. `checkIn()` only makes sense after `PAID`.

The vulnerability: endpoints that call these methods don't always verify the object's current state first.

### The Attack

Find an endpoint that triggers a state-specific action. Call it when the object is in the wrong state.

```python
def checkIn(bookingId):
    booking = db.get(bookingId)
    booking.status = "CHECKED_IN"  # no state validation
    db.save(booking)
```

Call this before completing payment. You've checked in without paying.

### Why It Works

State validation is often added as an afterthought. The happy path works. Edge cases — like calling endpoints out of order — aren't tested. The object model *implies* a state machine. The code doesn't *enforce* it.

Real world analogs:
- Completing an order without going through payment
- Accessing premium content before a subscription is confirmed
- Triggering a refund on an order that was never charged

---

## The Meta-Pattern

All five patterns share the same root cause:

**The object model encodes the rules. The code doesn't enforce them.**

Developers trust the UI to prevent invalid states. They trust other developers to call constructors correctly. They trust that if a field is null, it means it wasn't set intentionally.

The attacker breaks every one of those assumptions — not with complex exploits, but with a clear understanding of how the system's objects are supposed to work, and where the enforcement falls short.

---

## How to Hunt This

1. **Map all endpoints** that touch the same resource type (bookings, orders, subscriptions)
2. **Compare parameters** across flows — what does one flow accept that another doesn't expose in the UI?
3. **Identify object attributes** from responses, error messages, or JS source — what fields exist on the object that the UI never sends?
4. **Test capability flags** — inject fields that belong to higher-tier object types and observe backend behavior
5. **Probe state transitions** — call endpoints out of order and see if the backend validates current state

---

## Closing

This isn't a new vulnerability class with a CVE number. It's a lens.

When you look at an app through OOP — objects, capabilities, states, inheritance — you stop looking for individual bugs and start looking for **design assumptions the backend forgot to enforce**.

That's where the real findings live.

---

*Real world examples coming. Have findings that fit these patterns? Reach out.*
