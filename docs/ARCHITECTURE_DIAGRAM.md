# Architecture Diagram: Stability & Reliability Features

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Trellis API Platform                             │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                    1. Synthetic Monitoring                      │    │
│  │                                                                 │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐│    │
│  │  │ Authentication  │  │   Portfolio     │  │    Payment     ││    │
│  │  │   Journey       │  │   Management    │  │   Settlement   ││    │
│  │  │   (5 steps)     │  │   Journey       │  │   Journey      ││    │
│  │  └────────┬────────┘  │   (5 steps)     │  │   (4 steps)    ││    │
│  │           │            └────────┬────────┘  └────────┬───────┘│    │
│  │           │                     │                    │         │    │
│  │           └─────────────────────┼────────────────────┘         │    │
│  │                                 │                              │    │
│  │                          ┌──────▼────────┐                     │    │
│  │                          │  Ownership    │                     │    │
│  │                          │   Transfer    │                     │    │
│  │                          │   Journey     │                     │    │
│  │                          │  (5 steps)    │                     │    │
│  │                          └───────────────┘                     │    │
│  │                                                                 │    │
│  │  Runs every 5 minutes • Read-only probes • Alerts on failure   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │              2. Problem-Details Exception Filter                │    │
│  │                                                                 │    │
│  │  HTTP Request                                                   │    │
│  │       │                                                          │    │
│  │       ▼                                                          │    │
│  │  ┌─────────┐    Exception    ┌──────────────────────┐          │    │
│  │  │ Controller├───────────────▶│ ProblemDetailsFilter │          │    │
│  │  └─────────┘                  └──────────┬───────────┘          │    │
│  │                                          │                       │    │
│  │                               ┌──────────▼───────────┐          │    │
│  │                               │  Map to Error Code   │          │    │
│  │                               │  • VALIDATION_ERROR  │          │    │
│  │                               │  • UNAUTHORIZED      │          │    │
│  │                               │  • DATABASE_ERROR    │          │    │
│  │                               └──────────┬───────────┘          │    │
│  │                                          │                       │    │
│  │                               ┌──────────▼───────────┐          │    │
│  │                               │ RFC 7807 Response    │          │    │
│  │                               │ • type, title, status│          │    │
│  │                               │ • remediation        │          │    │
│  │                               │ • retryable          │          │    │
│  │                               │ • correlationId      │          │    │
│  │                               └──────────────────────┘          │    │
│  │                                                                 │    │
│  │  Global exception handler • Stable error codes • Retry guidance │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │              3. Ownership Transfer Service                      │    │
│  │                                                                 │    │
│  │  ┌──────────────────────────────────────────────────────────┐  │    │
│  │  │                  Transfer Lifecycle                       │  │    │
│  │  │                                                            │  │    │
│  │  │   ┌─────────┐   accept    ┌──────────┐                   │  │    │
│  │  │   │ PENDING ├────────────▶│ ACCEPTED │                   │  │    │
│  │  │   └────┬────┘              └──────────┘                   │  │    │
│  │  │        │                                                   │  │    │
│  │  │        ├── cancel ──────▶ ┌───────────┐                  │  │    │
│  │  │        │                   │ CANCELLED │                  │  │    │
│  │  │        │                   └───────────┘                  │  │    │
│  │  │        │                                                   │  │    │
│  │  │        ├── reject ──────▶ ┌──────────┐                   │  │    │
│  │  │        │                   │ REJECTED │                   │  │    │
│  │  │        │                   └──────────┘                   │  │    │
│  │  │        │                                                   │  │    │
│  │  │        └── auto-expire ─▶ ┌─────────┐                    │  │    │
│  │  │                            │ EXPIRED │                    │  │    │
│  │  │                            └─────────┘                    │  │    │
│  │  │                                                            │  │    │
│  │  └────────────────────────────────────────────────────────────┘  │    │
│  │                                                                 │    │
│  │  ┌────────────────────────────────────────────────────────────┐│    │
│  │  │               Authorization Matrix                          ││    │
│  │  │                                                              ││    │
│  │  │  Action    │ Current Owner │ Proposed Owner │ Other         ││    │
│  │  │  ─────────┼───────────────┼────────────────┼──────         ││    │
│  │  │  Initiate  │      ✓        │       ✗        │   ✗           ││    │
│  │  │  Accept    │      ✗        │       ✓        │   ✗           ││    │
│  │  │  Cancel    │      ✓        │       ✗        │   ✗           ││    │
│  │  │  Reject    │      ✗        │       ✓        │   ✗           ││    │
│  │  │  Read      │      ✓        │       ✓        │   ✗           ││    │
│  │  │                                                              ││    │
│  │  └────────────────────────────────────────────────────────────┘│    │
│  │                                                                 │    │
│  │  Features: Expiry • Idempotency • Audit Trail • Authorization   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │           4. Deployment Rollback Verification                   │    │
│  │                                                                 │    │
│  │  Deployment Failed (v2.1.0) → Rollback to v2.0.5               │    │
│  │                                                                 │    │
│  │  ┌──────────────────────────────────────────────────────────┐  │    │
│  │  │              Verification Checks                          │  │    │
│  │  │                                                            │  │    │
│  │  │  1. [version]  ──────▶  Verify v2.0.5 deployed            │  │    │
│  │  │  2. [health]   ──────▶  Service healthy                   │  │    │
│  │  │  3. [schema]   ──────▶  DB schema compatible              │  │    │
│  │  │  4. [dependency] ────▶  Stellar/Redis reachable           │  │    │
│  │  │  5. [auth]     ──────▶  Auth guards fail closed (401/403) │  │    │
│  │  │                                                            │  │    │
│  │  └──────────────────────┬───────────────────────────────────┘  │    │
│  │                         │                                       │    │
│  │              ┌──────────▼──────────┐                            │    │
│  │              │ All Checks Pass?    │                            │    │
│  │              └──────────┬──────────┘                            │    │
│  │                         │                                       │    │
│  │           ┌─────────────┴─────────────┐                         │    │
│  │           │                           │                         │    │
│  │      ┌────▼────┐                 ┌────▼────┐                   │    │
│  │      │   YES   │                 │   NO    │                   │    │
│  │      └────┬────┘                 └────┬────┘                   │    │
│  │           │                           │                         │    │
│  │  ┌────────▼────────┐       ┌──────────▼──────────┐             │    │
│  │  │ Resume Traffic  │       │ Block Traffic       │             │    │
│  │  │ ✓ Safe          │       │ ✗ Not Safe          │             │    │
│  │  └─────────────────┘       │ + Remediation Steps │             │    │
│  │                            └─────────────────────┘             │    │
│  │                                                                 │    │
│  │  Actor: RELEASE_MANAGER • Retry: 3 attempts • Backoff: 100ms   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

## Integration Flow

```
┌────────────────────────────────────────────────────────────────┐
│                      Request Flow                               │
└────────────────────────────────────────────────────────────────┘

1. Client Request
   │
   ▼
2. NestJS Middleware
   │
   ▼
3. Controller
   │
   ├─▶ Success ─────────────────▶ Response
   │
   └─▶ Exception ──┐
                   │
                   ▼
              ┌────────────────────────┐
              │ ProblemDetailsFilter   │ ◀── Feature 2
              │ • Maps error           │
              │ • Adds remediation     │
              │ • Sets retry guidance  │
              └────────────────────────┘
                   │
                   ▼
              RFC 7807 Response
              {
                "type": "...",
                "errorCode": "VALIDATION_ERROR",
                "retryable": false,
                "remediation": "..."
              }
```

```
┌────────────────────────────────────────────────────────────────┐
│              Synthetic Monitoring Flow                          │
└────────────────────────────────────────────────────────────────┘

Scheduler (every 5 min)
   │
   ▼
Run All Journeys ◀── Feature 1
   │
   ├─▶ Authentication Journey
   ├─▶ Portfolio Journey
   ├─▶ Payment Journey
   └─▶ Ownership Journey
   │
   ▼
Collect Results
   │
   ├─▶ All Passed ────▶ Log Success
   │
   └─▶ Any Failed ────▶ Alert Ops Team
                          │
                          └─▶ PagerDuty / Slack / Email
```

```
┌────────────────────────────────────────────────────────────────┐
│             Ownership Transfer Flow                             │
└────────────────────────────────────────────────────────────────┘

User A (Current Owner)
   │
   ▼
Initiate Transfer ◀── Feature 3
   │
   ├─▶ Create PENDING transfer
   ├─▶ Set expiry (72h)
   └─▶ Record audit event
   │
   ▼
Notify User B (Proposed Owner)
   │
   ├─▶ User B Accepts
   │   │
   │   ├─▶ Verify User B is proposed owner
   │   ├─▶ Change status to ACCEPTED
   │   └─▶ Record audit event
   │
   ├─▶ User B Rejects
   │   │
   │   └─▶ Change status to REJECTED
   │
   ├─▶ User A Cancels
   │   │
   │   └─▶ Change status to CANCELLED
   │
   └─▶ Auto-Expire (after 72h)
       │
       └─▶ Change status to EXPIRED
```

```
┌────────────────────────────────────────────────────────────────┐
│           Deployment Rollback Verification Flow                 │
└────────────────────────────────────────────────────────────────┘

CI/CD Pipeline
   │
   ▼
Deployment Failed (v2.1.0)
   │
   ▼
Trigger Rollback
   │
   ├─▶ Apply rollback artifact (v2.0.5)
   ├─▶ Wait for pods to restart
   └─▶ Run Verification ◀── Feature 4
       │
       ├─▶ Check: API Version = 2.0.5
       ├─▶ Check: Health = OK
       ├─▶ Check: DB Schema Compatible
       ├─▶ Check: Dependencies Reachable
       └─▶ Check: Auth Fails Closed
       │
       ▼
   ┌────────────────┐
   │ All Pass?      │
   └───────┬────────┘
           │
      ┌────┴────┐
      │         │
   ┌──▼──┐   ┌──▼──┐
   │ YES │   │ NO  │
   └──┬──┘   └──┬──┘
      │         │
      ▼         ▼
   Resume    Block
   Traffic   Traffic
      │         │
      │         └─▶ Alert: Remediation Steps
      │
      └─▶ Success: Traffic Resumed
```

## Data Flow

```
┌────────────────────────────────────────────────────────────────┐
│                   Database Entities                             │
└────────────────────────────────────────────────────────────────┘

ownership_transfers
├── id (uuid)
├── resource_type (varchar)
├── resource_id (varchar)
├── current_owner_id (varchar)
├── proposed_owner_id (varchar)
├── status (enum: PENDING/ACCEPTED/CANCELLED/REJECTED/EXPIRED)
├── expires_at (timestamp)
├── accepted_at (timestamp, nullable)
├── cancelled_at (timestamp, nullable)
├── rejected_at (timestamp, nullable)
├── idempotency_key (varchar, nullable, indexed)
├── created_at (timestamp)
└── updated_at (timestamp)

ownership_transfer_audit_events
├── id (uuid)
├── transfer_id (varchar, indexed)
├── action (enum: INITIATED/ACCEPTED/CANCELLED/REJECTED/EXPIRED/...)
├── actor_id (varchar)
├── from_status (enum, nullable)
├── to_status (enum, nullable)
├── reason (varchar, nullable)
├── metadata (jsonb, nullable)
└── occurred_at (timestamp)
```

## Monitoring & Observability

```
┌────────────────────────────────────────────────────────────────┐
│                        Metrics                                  │
└────────────────────────────────────────────────────────────────┘

synthetic_journey_duration_ms
├── journey_name
└── passed (true/false)

api_errors_total
├── error_code
├── domain
├── retryable (true/false)
└── status

ownership_transfers_total
├── status
└── resource_type

rollback_verifications_total
├── environment
├── passed (true/false)
└── deployment_id

rollback_check_duration_ms
├── check_name
├── check_kind
└── ok (true/false)
```

```
┌────────────────────────────────────────────────────────────────┐
│                        Logging                                  │
└────────────────────────────────────────────────────────────────┘

Structured JSON Logs
{
  "level": "error",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "context": "OwnershipTransferService",
  "message": "Unauthorized access attempt",
  "correlationId": "req_123_abc",
  "transferId": "ot-123",
  "actorId": "user-3",
  "expectedActor": "user-2"
}
```

---

## Summary

All four features integrate seamlessly:

1. **Synthetic Monitoring** → Probes all features end-to-end
2. **Problem-Details** → Provides consistent error responses for all features
3. **Ownership Transfer** → Core business logic with full observability
4. **Rollback Verification** → Ensures safe deployment of all features

Each feature:
- ✅ Is self-contained and testable
- ✅ Follows NestJS patterns
- ✅ Includes comprehensive error handling
- ✅ Provides observability (metrics, logs, traces)
- ✅ Maintains audit trails
- ✅ Enforces authorization
- ✅ Documents edge cases and recovery

---

*Architecture documented: 2024-01-15*
