# Payment Saga Design: Orchestration, Pause/Resume, and Compensation

John wants a room at [xyz.com](http:/xyz.com). He picks a range of dates, and in turn, the platform holds the room for John for a period of 10 minutes. Now, within this period of 10 minutes, John decides to click on one of the available Payment providers that the booking platform [xyz.com](http:/xyz.com) is providing. It could be either Stripe, Paystack, or Flutterwave. Once the Payment webhook event arrives, platform [xyz.com](http:/xyz.com) validates this webhook event and will then permanently reserve that room for that period for John.

Now you might ask, but there are no issues. Which, to me, is a fair question, since the platform business constraints still apply: booking, payment, calendar, and ledger logic are all sharing the same database transactional client. So either all operations (it can be either write or read) across the domains (I am implying the ledger, booking, payment, calendar) all succeed, or they will all definitely roll back together, which is what makes it safe.

Now there lies a problem, as most organizations tend to grow or expand in size. The various departments can then be split into teams, and these teams are responsible for taking ownership and releasing independently. Conway's Law can be our guide here. Software built should reflect how the team organization is built. So, this implies that we can then have booking, payment, audit, ledger, and notification as loosely coupled services. This then leads to the problem where these services no longer share the same transactional context that the database offers. Note, each of these services can definitely make use of ACID transactions locally, but it still cannot provide a single atomic commit across multiple services.

Irrespective of these changes, the business has to work perfectly. John still needs a confirmed stay after a successful payment, reliable check-in and check-out, and correct money movement. Operationally, that means:

1. Booking service can hold inventory and release it when a payment never completes
2. Payment can create a payment intent, verify webhooks, and stay idempotent under retries
3. Ledger can record balanced entries for money that actually moved

The whole essence of this post is how to achieve a system where we can use a technique called Saga to coordinate all the work done across booking, payment, audit, and ledger services so they can succeed together, and in the case of failure in one of these services, the work done internally by the various services can then be compensated without any need for a shared ACID transaction.

## **A Closer Look at SAGA.**

Think about Saga as a way to manage multiple business operation steps that move across multiple services, where each of these steps cannot share a single ACID transaction guarantee. In a Saga operation, each service within itself can still perform ACID transactions, where either all read and write operations have to be successful, or the operation will be rolled back. So if a later step fails, all services that have previously committed their changes will have to run a compensating action (for example, it could be releasing a booking hold, refunding a Payment to a client).

A Saga does not provide a single distributed rollback. A Saga also does not provide strong consistency where the most recent write must be reflected within the most recent read. Still, a Saga gives you the ability to coordinate multiple business operations across various services, compensate for failures, and also provide eventual consistency (the most recent write will, at a later time, be reflected in the most recent read operations).

We have established what Saga looks like, but we have not yet answered the main question: why do we need Saga?

## What Must the System Guarantee?

For us to appreciate why we need Saga, we need to understand what the system we are building is trying to accomplish. If we cannot measure what the system is trying to achieve, we cannot observe the effect or importance of Saga. Here are some key features that the system has to do:

1. As a guest, I should be able to reserve a room for a short time window (In this case, 10 minutes). If, as a guest, I fail to complete payment within this window, the room reservation should be released.
2. As a guest, I should be able to pay for my reserved room, and the system should ensure that I do not double-pay in the case of retries and double-clicking.
3. As the platform, the system should be able to resume and pause the workflow while waiting for webhook confirmation without losing payment and booking information
4. As the platform, if one of the chains that make up the steps fails (timeout, guest abandons payment, late payment against policies), the previously executed steps should implement a compensating action to roll back the changes.

## What are the non-functional requirements?

The 4 requirements we have stated describe what our platform should offer, but they do not answer how the system should behave under load, retries, bad network, crashes, and concurrent tenants. So, the non-functional requirements answer these questions.

For the System to be able to implement the above 4 requirements for Saga under retries and failures, these non-functional requirements have to be true:

1. The system should tolerate at-least-once delivery of events without any duplication of events.
2. The System should be highly auditable, in the sense that every action and state machine progress should be recorded
3. The system should offer high observability over Saga progress, such as step state, paused state, completed, and compensated states.
4. There should be no leakage of data in the system. Booking and Payment data should have policies where data are only visible to the appropriate tenants.

## The system core entities:

Before we see how each of the requirements works, we need to look at the core actors that make up the system.

1. Booking: The booking entity owns the reservation workflow and state transitions. So, it is the service that handles reserving a room for a user, John, exclusively for a short time (10 minutes), confirms the booking after payment, and releases the hold if payment never completes. It owns the bookings table and the availability calendar.
2. Payment: The payment entity creates the payment intent for the booking, talks to the payment providers (Stripe, Paystack or Flutterwave), verifies webhook signatures, stays idempotent under retries, and guards payment state transitions (pending, paid, cancelled, refunded). It is also responsible for issuing refunds when a compensating action asks for one.
3. Ledger: The ledger entity records money movement as balanced debit and credit entries (double entry), and only for money that actually moved. When a payment is captured or refunded, the ledger is what lets us prove where the money went.
4. Orchestrator: The orchestrator coordinates the multi-step business operation that spans services. Once John clicks "Pay Now", it creates the saga, calls the payment-service to create the intent, waits for the webhook, and then asks the booking-service to confirm. If a later step fails, it runs compensating actions for the steps that already completed, such as refunding the payment or releasing the hold. It stores its progress (current step, status, context) in its own database, so it can pause while waiting for a webhook and resume after a crash.
5. API Gateway: The API Gateway is the simply just the only entry point for the guest and the payment providers. It basically handles rate limiting, authentication and request routing, and forwards requests to the Main API. Internal service-to-service endpoints are never exposed through it.
6. Availability: Availability is really not a separate service. It is the inventory of rooms per room type and date (the availability_calendar table), and it belongs to the booking-service. Reserving a room decrements the count, and releasing or expiring a hold restores it, both in the same local transaction as the booking change. This is what stops two guests from reserving the last room
7. Workers: Background isolated processes that simply just do the time-based and reliability work. The booking-expiration worker releases unpaid holds, the outbox worker publishes events written in the same transaction as the data change, and the sweeper reclaims sagas whose orchestrator crashed.
8. Audit and Notification: The audit service records every action and state change, and the notification service tells John and the platform what happened (hold expired, payment confirmed, refund issued). Both are fed by events from the outbox rather than called directly.

## How does the system data operate?

We have answered what the functions that the platform offers, but we have not explained how each of these functional requirements is meant to be implemented. So let us look at each of the functional requirements data flow.\\

### 1\. FR1: As a guest, I should be able to reserve a room for a short time window

The core data flow for this requirement will be:

1. The guest, John, selects a date for a particular room and then clicks "Reserve Now"
2. The API gateway informs the <mark>booking-service</mark> to reserve that room by acquiring a temporary lock that expires after 10 minutes
3. The <mark>booking-service,</mark> in turn, reduces the availability count for that date
4. The <mark>booking-service</mark> then returns a 201 response, and the booking ID + expiry to the user
5. If the user fails to pay within the next 10 minutes, a background process runs to release the temporary lease and restore the availability, allowing future users to book the apartment
6. The <mark>booking-service</mark> then informs John that his booking has expired, either by email notification or SMS notification

   ![As a guest, I should be able to reserve a room for a short time window](./diagrams/Booking-Blog-Seq-1.png)

### 2\. FR2: As a guest, I should be able to pay for my reserved room

The main data flow for this requirement will go like this:

1. John gets to the booking details page, and then clicks "Pay Now"
   1. The API gateway forwards the request to <mark>orchestrator-service</mark> to start a Saga intent
2. The <mark>orchestrator-service</mark> then forwards the request with the idempotency key to the <mark>payment-service</mark> to create a booking payment intent.
3. The <mark>payment-service</mark> then returns a 201 response, and the redirect URL to the payment page + the transaction ID of the payment intent
4. John then gets redirected to the Payment Provider's secure checkout page to complete payment with their credit card

   ![FR2: As a guest, I should be able to pay for my reserved room](./diagrams/Booking-Blog-Seq-2.png)

### 3\. FR3: As the platform, we can pause and resume while waiting for webhook confirmation without losing booking–payment correlation

So the core data flow for this requirement will be:

1. Once John payment has been completed, we are expecting the Payment provider to send a webhook notification to our API Gateway.
2. The API Gateway, once it has received the notification, forwards the webhook notification request to <mark>orchestrator-service,</mark> and the saga instance resumes with the booking metadata and payment metadata.
3. The <mark>orchestrator-service</mark> then forwards the Saga Payment instance metadata to the <mark>payment-service</mark> to verify the webhooks, and if the webhook is valid, it then updates the Payment state.
4. The <mark>payment-service</mark> then informs the <mark>orchestrator-service</mark> that its step has been completed and can then move to the next step, which is the <mark>booking-service</mark>
5. The <mark>orchestrator-service</mark> then moves to update the status of the booking to Confirmed
6. The Saga is then marked completed.

   ![FR3: As the platform, we can pause and resume while waiting for webhook confirmation without losing booking–payment correlation](./diagrams/Booking-Blog-Seq-3.png)

### 4\. FR4: As the platform, if a later step fails, earlier steps run compensating actions

The last three data flow where concerned about the happy path, but what if something fails how do we handle it? How will the data flow be if John fails to complete his payment within 10 minutes, or what if John does not have sufficient money to complete the Payment? So this data flow answers these questions:

1. Once the orchestrator-service detects a failed path, which can be a hold that expired with no payment, a "payment failed" webhook from the provider (for example, insufficient funds), or a confirm step that failed permanently.
2. The orchestrator-service will then mark the saga as `compensating` and saves it, so that if it crashes midway, another instance knows compensation is in progress.
3. The orchestrator-service proceeds to then asks the payment-service to cancel or refund, using a step key so a repeat of this request is harmless. The payment-service then takes one of two paths:
   1. **The intent is still pending** (abandoned checkout or failed payment): it cancels the intent at the provider and marks the payment `cancelled`. No money moved, so there is no refund.
   2. **The payment was already paid** (late payment or a failed confirm): it marks the payment `refund_pending`, writes the ledger entry, and calls the provider to refund, using the payment ID as the idempotency key so a retry never refunds twice.
4. The payment-service reports back that it is done, and the orchestrator-service saves that step. If the orchestrator crashes after this point, the replacement resumes from the booking release instead of repeating the payment step.
5. The orchestrator-service asks the booking-service to release the hold. In a single transaction, the booking-service marks the booking `expired` (or `cancelled`) and restores the availability count. If the expiry worker already released it, the call does nothing.
6. The orchestrator-service marks the saga as `compensated` and writes an outbox event, so the notification and audit services can tell John what happened and record the action.
7. If compensation keeps failing after the maximum number of retries, the saga moves to `failed` and an alert goes to a person, because John's money may still be held.

   ![4. FR4: As the platform, if a later step fails, earlier steps run compensating actions](./diagrams/Booking-Blog-Seq-4.png)

We have been able to look at how the 4 functional requirement data flows work. So in the next section, we should establish the core API Design or contract that satisfies the 4 functional requirements

## Saga API Design

For us to implement the 4 Saga functional requirements, we need to design a contract either between the client and the server or between the internal servers in the application, which does not have to be public.

### Public APIs

1. POST /api/v1/bookings: It satisfies functional requirement 1, which allows John to be able to reserve a room that is held for a short period of time (we agreed on 10 minutes)
2. POST /api/v1/payments/initialize: This API addresses functional requirement 2 by allowing John to create a booking payment intent.
3. POST /api/v1/webhooks/paystack: The payment provider uses this endpoint to send a webhook payment notification when John has paid for his payment intent, and the service in turn carries out signature verification.

### Internal APIs

For the internal APIs, the guest cannot call them, since they are not exposed to the API Gateway. Services that reside on the private network can be allowed to communicate via the internal API to these services. For example, the main API call communicates to the <mark>orchestrator-service</mark> via the internal API, and the <mark>orchestrator-service</mark> can also communicate with the <mark>payment-service</mark> and the <mark>booking-service</mark>

In production, we can also go a little bit further by restricting access at the network layer rather than solely depending on the application layer. On the network layer, we can make use of adding these internal services to a private subnet. We can also use AWS security groups, which help us define the ingress and egress rules. That way, a browser cannot hit these internal services even by guessing their endpoints. On EKS, we have what we call IRSA (IAM Role for Service Accounts). We can use it to give temporary privileges to services without the services sending any internal secret keys.

1. POST /api/v1/internal/sagas/booking-payment: The main API service calls the <mark>orchestrator-service</mark> via this API to start the booking payment SAGA. It always requires an idempotency key to enforce no duplicate SAGA history.
2. POST /api/v1/internal/sagas/booking-payment/:bookingId/resume: The main API service calls the <mark>orchestrator-service</mark> via this API to start the booking payment SAGA. It will always require an idempotency key to enforce no duplicate SAGA history.
3. GET /api/v1/internal/bookings/:bookingId: The <mark>orchestrator-service</mark> simply uses this API to call the <mark>booking-service</mark> to fetch the booking metadata
4. POST /api/v1/internal/bookings/reserve: The <mark>orchestrator-service</mark> simply uses this API to call the <mark>booking-service</mark> to reserve a booking
5. POST /api/v1/internal/bookings/:bookingId/confirm: The <mark>orchestrator-service</mark> simply uses this API to call the <mark>booking-service</mark> to confirm a booking (in the sense that it changes the status from payment_pending to booked)
6. POST /api/v1/internal/bookings/:bookingId/release: When John fails to complete the booking payment, the <mark>orchestrator-service</mark> will use this API to call the <mark>booking-service</mark> to release a booking (in the sense that it moves the status from reserved to available)
7. POST /internal/payments/initiate: Orchestrator creates the provider intent/redirect (idempotent).

We have now defined the minimum API routes that define our core functional requirements, but we have not defined what the structure of the data will look like, the database choice, or the relationships between the various tables. We will look at these gaps in the next section

## Saga DB Design.

We will be making use of PostgreSQL since it enforces structured data, and it has a seamless ACID guarantee integration. Other pains of schema migration and table bloat still exist, but we will use it since our application cannot tolerate any data inconsistency or financial incorrectness.

### 1\. Booking service

Bookings

```sql
CREATE TABLE IF NOT EXISTS bookings (
    id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_ref        VARCHAR(20)    NOT NULL UNIQUE,
    tenant_id          UUID NOT NULL,
    property_id        UUID NOT NULL,
    room_type_id       UUID NOT NULL,
    guest_user_id      UUID NOT NULL,
    rooms_count        INT            NOT NULL DEFAULT 1,
    check_in           DATE           NOT NULL,
    check_out          DATE           NOT NULL,
    nights             INT            GENERATED ALWAYS AS (check_out - check_in) STORED,
    guest_count        INT            NOT NULL DEFAULT 1,
    total_amount_ngn   NUMERIC(12,2)  NOT NULL,
    platform_fee_ngn   NUMERIC(12,2)  NOT NULL,
    host_payout_ngn    NUMERIC(12,2)  NOT NULL,
    status             booking_status NOT NULL DEFAULT 'pending_payment',
    cancellation_reason TEXT,
    cancelled_at       TIMESTAMPTZ,
    special_requests   TEXT,
    metadata           JSONB          NOT NULL DEFAULT '{}',
    created_at         TIMESTAMPTZ    NOT NULL DEFAULT now(),
    updated_at         TIMESTAMPTZ    NOT NULL DEFAULT now(),
    CONSTRAINT chk_checkout_after_checkin CHECK (check_out > check_in),
    CONSTRAINT chk_rooms_positive CHECK (rooms_count > 0)
  );
```

Availability Schema:

```sql
CREATE TABLE IF NOT EXISTS availability_calendar (
    id                 UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    room_type_id       UUID  NOT NULL,
    tenant_id          UUID NOT NULL,
    date               DATE          NOT NULL,
    available_count    INT           NOT NULL,
    price_override_ngn NUMERIC(12,2),
    is_blocked         BOOLEAN       NOT NULL DEFAULT false,
    created_at         TIMESTAMPTZ   NOT NULL DEFAULT now(),
    updated_at         TIMESTAMPTZ   NOT NULL DEFAULT now(),
    CONSTRAINT uq_availability UNIQUE(room_type_id, date)
  );
  CREATE INDEX IF NOT EXISTS idx_avail_room_date ON availability_calendar(room_type_id, date);
```

### 2\. Payment service

Payment Schema:

```sql
CREATE TABLE IF NOT EXISTS payments (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    booking_id              UUID NOT NULL,
    tenant_id               UUID NOT NULL,
    guest_user_id           UUID NOT NULL,
    gateway                 payment_gateway NOT NULL,
    transaction_id          VARCHAR(150)   UNIQUE,
    amount_ngn              NUMERIC(12,2)  NOT NULL,
    status                  payment_status NOT NULL DEFAULT 'pending',
    channel                 VARCHAR(50),
    paid_at                 TIMESTAMPTZ,
    refunded_at             TIMESTAMPTZ,
    idempotency_key         VARCHAR(128)   NOT NULL UNIQUE,
    metadata                JSONB          NOT NULL DEFAULT '{}',
    created_at              TIMESTAMPTZ    NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ    NOT NULL DEFAULT now()
  );
  CREATE INDEX IF NOT EXISTS idx_payments_booking     ON payments(booking_id);
  CREATE INDEX IF NOT EXISTS idx_payments_transaction ON payments(transaction_id);
  CREATE INDEX IF NOT EXISTS idx_payments_idempotency ON payments(idempotency_key);
  CREATE INDEX IF NOT EXISTS idx_payments_status      ON payments(status, created_at DESC);

```

IdempotencyKey Schema:

```sql
  DO $$ BEGIN
    CREATE TYPE idempotency_status AS ENUM ('processing', 'completed', 'failed');
  EXCEPTION WHEN duplicate_object THEN NULL; END $$;

  CREATE TABLE IF NOT EXISTS idempotency_keys (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    request_hash   VARCHAR(64)         NOT NULL UNIQUE,
    endpoint       VARCHAR(150)        NOT NULL,
    user_id        UUID,
    status         idempotency_status  NOT NULL DEFAULT 'processing',
    status_code    INT,
    response_body  JSONB,
    failure_reason TEXT,
    expires_at     TIMESTAMPTZ         NOT NULL,
    created_at     TIMESTAMPTZ         NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ         NOT NULL DEFAULT now()
  );
  CREATE INDEX IF NOT EXISTS idx_idempotency_hash    ON idempotency_keys(request_hash);
  CREATE INDEX IF NOT EXISTS idx_idempotency_expires ON idempotency_keys(expires_at);
  CREATE INDEX IF NOT EXISTS idx_idempotency_status  ON idempotency_keys(status, created_at);
  
```

### 3\. Orchestrator service

Saga Schema:

```sql
CREATE TABLE IF NOT EXISTS saga_instances (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    saga_type        TEXT NOT NULL,
    status           TEXT NOT NULL DEFAULT 'running' CHECK (status IN ('running','completed','compensating','compensated','failed')),
    current_step     INT NOT NULL DEFAULT 0,
    context          JSONB NOT NULL DEFAULT '{}',
    booking_id       UUID,
    created_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    claimed_by       UUID,
    lease_expires_at TIMESTAMPTZ,
    idempotency_key UUID,
    last_error UUID,
    forward_attempts INT NOT NULL DEFAULT 0
);
CREATE UNIQUE INDEX IF NOT EXISTS uq_saga_instances_active_booking
  ON saga_instances (saga_type, booking_id)
  WHERE booking_id IS NOT NULL AND status IN ('running', 'compensating');

```

We have defined each of the core Tables that make up the core functional requirements. Yet, they still do not show us how these functional requirements follow the happy and failure paths. In our next section, I will be showcasing how each of these paths for the high-level flow works.

## Saga High-Level Design

### 1\. FR1: As a guest, I should be able to reserve a room for a short time window

![FR1 reserve flow](./diagrams/Booking-Blog-1.png)


The diagram above is more of a representation of the functional requirement 1 we discussed earlier. The idea in the diagram is that John asks for a room to hold for a short period of time. The Main API sends the requests to the <mark>booking-service</mark> which acquires a lock, writes a <mark>pending_payment</mark> status, and then decreases the availability count for that room.

### 2\. FR2: As a guest, I should be able to pay for my reserved room

![FR2: Pay for the reserved room](./diagrams/Booking-Blog-2.png)

So when John clicks on "Pay Now", the Main API receives the Payment request with an idempotency key to prevent a duplicate Saga instance, which could occur during double clicks or network retries. The main API then forwards the request to the orchestrator service. The Orchestrator service then calls the Payment service to create the Payment intent. Once the intent has been created, the user is then forwarded to the Payment Provider page.

### 3\. FR3: As a guest, I should be able to pay for my reserved room

![FR3: As a guest, I should be able to pay for my reserved room](./diagrams/Booking-Blog-3.png)

### 4\. FR4: As a guest, I should be able to pay for my reserved room

![FR4: As a guest, I should be able to pay for my reserved room](./diagrams/Booking-Blog-4.png)

## How will the System react to Failures?

The flows we have looked at above mostly focus on the system behaviour when everything works fine; that is, it focuses on the happy path. Real-world systems do have their own nuances. A system can be experiencing timeouts, partial failures across the various services, and retries. So we will use this section to look at the failures that John can experience when he tries to pay for a stay, and what is expected of the platform to behave.

### Case 1: Double Click Pay

For this case of failure, John intends to pay for a stay but ends up paying twice because he mistakenly double-clicked on the "Pay Now" button. If we do not address this failure scenario, we could end up losing trust from our paying customers.

To address this issue, we have to ensure that the write path for creating a payment intent is idempotent. An idempotent path ensures that the function along that path does not create a new record but returns a single value safely irrespective of the number of times that function is called by the user.

From the IdempotencyKey table, we could use it to implement idempotency. So I will think of it like this, practically:

1. Once a payment request (POST /payments/initialize) arrives, we will validate the request, ensuring that the "x-idempotency-key" header is being supplied. If it is not supplied, we will reject the request. Practically, we could make use of Joi by creating middleware that sits between the request and response cycle.
2. Since the "x-idempotency-key" is now being supplied, we should create a signature using the key, the booking ID, the payment provider label, and the callback URL.
3. **The first time** we see that signature, we create the payment intent, talk to the provider, persist the payment row, store the idempotency record with the response (redirect URL, transaction ID), and return that payload.
4. **Second time** (double-click/retry) with the same signature: we **do not** create a new payment. We load the saved response from the idempotency table and return the **same** redirect URL and transaction ID.

### Case 2: Duplicate payment webhook

This failure scenario violates our first non-functional requirement, where we demanded that our system should not allow duplicate request processing.

So the way this failure scenario works is that when John completes his payment, the Payment service provider is obliged to send a payment notification to our system, and for reliability's sake, the payment provider is also obligated to send it multiple times to ensure that our system receives the payment notification events. If, in this case, our system the <mark>payment-service</mark> treats every new payment webhook as a new event, we will be performing duplicate actions, like sending a new payment completed notification more than once, duplicate audit events, or duplicate refunds if the webhook was a failed event. So the failure is due to our own system failing to be idempotent along the webhook write path.

Our <mark>payment-service</mark> system can solve these issues via these steps:

1. The service should verify the webhook; if it is invalid, then the event should be rejected in turn.
2. The service, in turn, should acquire a lock to allow only one process of the service to handle the webhook events. We can simply make use of PostgreSQL advisory locks since they can scale across various instances of the server, and they can release the lock when the process dies rather than allowing a lock to be permanent. We can still consider the case of the locks not being scalable across multiple instances of PostgreSQL, in which we can use Redis setnx for this case.
3. We will check the status of the payment if it is successful or it is failed. If true, we will return the result of the payment provider back to the provider.
4. If the payment status is neither successful nor failed, we will move to update the Payment status and then write to an outbox table, where a worker will pick up the event and inform the orchestrator service to resume.
5. The orchestrator should also be idempotent to prevent any duplicates across the resume write path.

### Case 3: Abandon checkout

So, in this failure case, what happens is that John, when he gets redirected to the Payment provider page (Paystack, Stripe, PayPal), might decide to add his credit card information or might just leave the Page due to his personal choices.

So this means that the room is still on hold if John leaves the Payment page without paying. This implies a loss in platform revenue since the seller might not be able to sell that room reservation period to another client. The room gets blocked, and the Saga will also be set to Paused forever.

For us to handle this failure scenario, we can follow these steps:

1. The booking hold is temporary. We store an expiry (about 10 minutes) and treat unpaid holds as reclaimable after that time, not as a permanent reservation.
2. We will have a worker that finds bookings that have these conditions: status is still payment_pending, or status is pending_payment, but the hold has not expired, and the saga is still paused past the pause timeout
3. If the above conditions are true, we will then carry out these actions
   1. We will cancel all pending payment intents
   2. We will also release the booking hold and restore the availability count
   3. In the same context, the orchestrator service will then mark the saga as being compensated with the reason as abandoned, or we can say timeout
4. On the read path, we will not depend on the worker since the worker might have a delay based on the schedule interval, or the Server running the worker might be experiencing an OOM (Out of Memory Error), but regardless of the state of the worker, we can use the lease expiration period we set on the booking locks to allow users to be able to view unreserved rooms

Tradeoffs

1. The timeout period matters more. A short timeout frees booked rooms faster, but may cut off slow payers, which could be due to network issues or health issues (eye issues, elderly). A long timeout might block inventory and cost the seller revenue too.

### Case 4: Payment succeeds after compensation (late webhook)

This failure scenario looks a bit nuanced. So let us address it. The way this failure happens is that a room booking lock could get released since there was no payment notification for the booking.

So let's say John tries to reserve the room at 9:58 am, which means the hold expires at 10:08 am since the expiration period is simply 10 minutes. At 10:07 am, John completes his payment, and the webhook arrives at 10:12 am. The worker tries to compensate at 10:09 am. So this means that the <mark>booking-expiration</mark> worker will mark it as expired and will release the hold.

It fails to account for the fact that a Payment might have been completed, but the Payment provider might be too busy to send the webhook notification due to high traffic in the payment provider's systems. So if the provider later sends the webhook events, how do we address this issue since the platform has already released the booking?

We can handle this failure gracefully via these steps:

1. The <mark>payment-service</mark> will verify the webhook and ensure deduplication
2. In turn, the <mark>payment-service</mark> will try to load the payment and booking relationship and will notice that the booking has already been compensated by the <mark>booking-expiration</mark> worker
3. We will then follow these late payment policy steps:
   1. Depending on the business policy, I will push the event of refunding to an outbox event table to ensure that the write path has minimal critical paths
   2. A worker will pick the event from the outbox events and push it to the RabbitMQ exchange for refund.
   3. The refund consumer will then carry out the refund flow, and the booking will still have the same new state.
   4. We will add an exponential backoff strategy to ensure that we have a network-resilient consumer, and also ensure that the worker is idempotent to prevent duplicate refunds by using the payment ID as the idempotency key too,
   5. In the same transactional client, we can then record the money movement based on a debit and credit on the ledger table.
4. Once the refund worker is completed, we will write to the outbox event for the audit service to record this action
5. For a more product-oriented workflow, we will notify the customer and also the seller of the refund action we took.

### Case 5: Confirm fails after payment success

This specific failures is about us answering the question of how will our system react when a service-to-service request timeout due to one service being unavailable or due to network glitch issues.

So let us say John has completed his payment; the webhook has been verified, and the payment is being marked as successful. The <mark>orchestrator-service</mark> will then try to call the <mark>booking-service</mark> which is the next step to confirm the booking payment and mark the status as "confirmed". That internal API call can fail based on what we mentioned earlier.

In other words, the Money has been captured, but the stay has not been confirmed. John has been charged, but he cannot check in since the booking state is still "payment_pending"

This failure can be resolved by following these steps:

1. The <mark>orchestrator-service</mark> will record the specific step failure, status and error
2. It will retry it again for transient errors (such as a 503 error). We can use an exponential backoff strategy with Jitter as our retry pattern. The booking service will obviously be idempotent in turn
3. If the max retries have been maxed out, we can proceed to move the error metadata to the dead letter queue for manual observation from the platform team.
4. If the error is not a transient error, but a permanent error where the booking hold has been released or a business rule has been violated. We will then start carrying out compensation.
   1. The <mark>payment-service</mark> will carry out refunds to the customer
   2. The booking service will have to release the hold
   3. Saga ends compensated, not “completed.”

### Case 6: Orchestrator crashes mid-step

In this failure case, we are asking a question of how we handle a critical failure that the Saga orchestrator does experience. So you could remember, the <mark>orchestrator-service</mark> handles cross-service operations. For each of these steps, it does 2 separate things: It tells a service, like the <mark>payment-service,</mark> to do work, and the <mark>orchestrator-service</mark> then records the committed result and the step.

The operations that happened in the <mark>payment-service</mark> DB and the <mark>orchestrator-service</mark> DB are 2 operations that are not atomic since they can fail independently, causing them to not be atomic.

So this failure is all about what happens if the orchestrator dies in that gap.

For example, let us walk through how the failure happens when John clicks "Pay Now":

1. The orchestrator creates the saga row with `current_step = 0` and `status = 'running'`, then asks payment-service to create a payment intent.
2. Payment-service creates the intent and commits it. The commit will be durable and survives a restart of payment-service.
3. The orchestrator then crashes before it can write "step 1 complete" and set `current_step = 1`.
4. Obviously, we will be running multiple copies of the <mark>orchestrator-service</mark> to aid us in disaster recovery
5. The next <mark>orchestrator-service</mark> will try to read the saga row, and it will see the step to be 0 instead of 1. This will lead to 2 forms of errors:
   1. The first failure is that, if the orchestrator service is not idempotent, if a new "Pay Now" comes again due to John retrying, the orchestrator will call the Payment service, which will create duplicate charges.
   2. A second separate failure will occur in the webhook processing write path. If a new request comes through the orchestrator service, it will not see a state with step 1 but a state with step 0, which will cause it to reject the request since it is violating state transition.
6. The Saga row state created is now dead `status = 'running'` and no <mark>orchestrator-service</mark> instance can recover it.

For us to resolve it, we can also follow these steps:

1. Claim every saga with a lease\
   Before an orchestrator-service instance runs a saga, it has to claim it. The claim is simply writes who owns the saga and when the ownership expires. We can use the database clock (`now()`), not the application clock, so that instances with drifting clocks cannot disagree about whether a lease has expired.\\

   ```sql
   UPDATE saga_instances
   SET claimed_by       = $worker_id,
       lease_expires_at = now() + interval '30 seconds',
       claim_version    = claim_version + 1,
       forward_attempts = forward_attempts + 1
   WHERE id = $saga_id
     AND status IN ('running', 'compensating')
     AND (claimed_by IS NULL OR lease_expires_at < now())
   RETURNING claim_version, current_step, context;
   ```

   So If the update returns no row, someone else owns the saga and this instance backs off. During long steps, such as a slow provider call, the running instance extends its lease with a heartbeat. If the instance dies, the heartbeat stops and the lease expires. This fixes the stuck "running forever" row, because a dead runner can now be told apart from a live one.
2. A little caveat here, some the Sagas operation crashes the process is running which for example can be an out-of-memory error on a bad payload. So even if we reclaim it dynamically via a lease it will be repeating often. So what we can do is to track the number of attempts it has been retried: forward_attempts. So once the forward_attempts maxed out the maximum capacity of retries, we will move it to a dead letter queue and step the Saga retries.

## Tradeoffs

So we are nearly at the close of this article. Every choice choosen for this design gives us a guarantee, but also in turn we are being charged for it somewhere else. When we split the system into services, it truly gave us independent deployments but in turn we had to pay the cost of shared transaction. Our orchestration made the flow easy to debug and set up, but we also have to ensure that the orchestrator service also stays alive

| Decision I am making | What we are to gain | What we are going to pay |
| --- | --- | --- |
| Choosing Saga instead of a distributed transaction | Services will still stay independently deployable, with no cross-service locks | No atomic rollback or strong consistency. We will be writing compensations and accept eventual consistency |
| Outbox pattern with a polling worker | Database writes and event publishing can't diverge due to dual write failures | Extra latency will definitely come from polling. we will have an extra table to clean up, and a relay worker to run and monitor |
| Choosing Orchestration instead of choreography | The flow lives in one place, so it's easy to follow, observe and debug | A lot of failure handling: we need to provision leases, a sweeper and its own recovery story |
| Time-boxed holds (10 minutes) | Unpaid rooms return to inventory automatically | Slow but legitimate payers can lose their room, so we need a late-payment policy |

Every guarantee in this design is bought with extra state, extra workers or extra code. We chose to pay for correctness over simplicity because the system moves money.

## Designing for Failure, Not Around It

We have been able to look at how to design a Saga workflow using specifically an orchestration Saga workflow. What I have been pushing for is to see that failures are inherent in every core scenario. A Saga is not a silver bullet, but it is a tool that comes with its own problems. The network will deliver messages twice, workers will crash halfway through a step, and webhooks will arrive after the room has been released. We can't prevent any of that, so we designed for it: idempotent steps, durable state, and compensations that put things right.

In this post, we have followed John from his reservation to a confirmed stay, and from a failed payment back to a released room. Whichever path he takes, his money and his booking end up in a consistent place. Our platform still keeps his trust, and we make John booking process to be seamless also.