# Post-Call Processing Pipeline — Design Document

**Author:** PALLAVI C O
**Date:** 27 June 2026

---

## 1. Assumptions

_State every assumption you made about the business, system, or environment. Be specific. These will be discussed in the follow-up._

1.  Multiple customers can run campaigns simultaneously.
2. Each completed call generates a transcript before post-call processing starts.
3. The LLM provider enforces strict request-per-minute and token-per-minute limits.
4. Redis is available for caching and rate-limit tracking.
5. Exotel recordings may become available anytime between 10 and 120 seconds after a call ends.
6. Dashboard users expect near real-time analysis results.
7. A failed downstream action (CRM, WhatsApp, etc.) should not prevent the analysis result from being stored.
 

---

## 2. Problem Diagnosis

_Before designing anything: what is actually broken, and why does it break at scale? In your own words._


1.The current system works for small workloads but does not scale to large campaigns. Every completed call follows the same sequential pipeline, causing unnecessary delays. The recording step blocks the LLM analysis even though both tasks are independent.

2.The system also sends every transcript directly to the LLM    without checking available capacity. When many calls finish together, this results in LLM rate-limit errors (429 responses). The circuit breaker reacts by freezing the dialer instead of slowing processing gradually.

3.Another issue is that all customers share the same processing queue and LLM quota. A single high-volume customer can delay processing for every other customer. Recording failures, retry handling, and logging are also limited, making production issues difficult to diagnose.

## 3. Architecture Overview

_End-to-end flow from call-end webhook to completed analysis. Include a diagram._

```
The redesigned architecture separates independent tasks and introduces rate-limit-aware processing. Instead of executing all operations sequentially, recording retrieval and transcript analysis are handled independently. This reduces overall processing time and prevents unnecessary delays.

                  Call Ends
                        │
                        ▼
              Receive End Webhook
                        │
                        ▼
              Create Processing Task
                        │
          ┌─────────────┴─────────────┐
          │                           │
          ▼                           ▼
 Recording Poller              LLM Processing Queue
 (Retry + Backoff)                    │
                                      ▼
                           Rate Limiter & Token Budget
                                      │
                                      ▼
                              LLM Analysis
                                      │
                                      ▼
                         Store Analysis Result
                                      │
                                      ▼
                CRM Update / Dashboard / Signal Jobs


```

### Key design decisions

1. Recording retrieval and LLM analysis are separated because they are independent tasks.
2. A rate limiter checks token availability before sending requests to the LLM.
3. Customer-specific token budgets prevent one customer from consuming all available capacity.
4. Recording retrieval uses retry with exponential backoff instead of waiting a fixed 45 seconds.
5. Structured logging is added to simplify debugging and monitoring.
---

## 4. Rate Limit Management

_This is the primary problem. How does your system respect LLM rate limits across 100K calls?_

The system uses a Redis-based rate limiter to track token usage before sending requests to the LLM.

### How you track rate limit usage

- Store token usage in Redis.
- Track both global token usage and per-customer token usage.
- Reserve estimated tokens before making the LLM request.
- Update actual token usage after receiving the response.

### How you decide what to process now vs. defer

If sufficient token capacity is available, the request is processed immediately. Otherwise, the request waits in the queue until capacity becomes available.

Priority calls such as confirmed bookings can be processed before lower-priority calls if required.

### What happens when the limit is hit (recovery, not crash)

Instead of stopping the dialer, new LLM requests wait in the processing queue until token capacity becomes available again. This prevents 429 errors while allowing the system to continue operating.

## 5. Per-Customer Token Budgeting

_If total capacity is N tokens/min and K customers are active simultaneously:_

- How do you allocate capacity across customers?
- What guarantees does a customer with a pre-allocated budget receive?
- What happens when a customer exceeds their budget?
- What happens to unallocated headroom?

The platform serves multiple customers simultaneously, so token usage must be shared fairly.

- Each customer is assigned a token budget based on the total available LLM tokens per minute.
- Before processing a request, the system checks both the global token limit and the customer's allocated budget.
- If a customer reaches their budget, their requests are temporarily queued until tokens become available.
- Any unused capacity from inactive customers can be temporarily used by active customers to improve overall resource utilization.
- This approach prevents a single customer from consuming all available LLM capacity while maintaining fair access for everyone.
---

## 6. Differentiated Processing

_Some call outcomes are time-sensitive. Some can wait. How do you determine which is which?_

_What mechanism do you use — is it a classification step, a flag set by the business, something else? Justify your choice._

---

Not every call requires the same level of analysis.

The system first performs a lightweight classification to determine the importance of the call.

Examples:

- High Priority
  - Appointment confirmed
  - Customer requested callback
  - Customer interested in the product

- Low Priority
  - Wrong number
  - No answer
  - Call disconnected
  - Not interested

Only high-priority calls receive full LLM analysis with entity extraction and summarization. Low-priority calls are classified with minimal processing, reducing token usage and improving throughput.

## 7. Recording Pipeline

_Replacement for `asyncio.sleep(45s)`. How does it work? What does a failure look like to the on-call engineer?_

---

Instead of waiting a fixed 45 seconds, the new recording pipeline continuously checks whether the recording is available.

The process is:

1. Call ends.
2. Immediately request the recording URL.
3. If unavailable, retry using exponential backoff (5s, 10s, 20s, 40s...).
4. Stop after the maximum retry limit.
5. Record every attempt in the application logs.

Advantages:
- Recordings that become available quickly are processed immediately.
- Late recordings are still retrieved successfully.
- Every failure is visible to the operations team through structured logs and alerts.

## 8. Reliability & Durability

_How do you ensure no analysis result is permanently lost?_

---
To prevent data loss:

- Every completed interaction is stored before processing begins.
- Failed processing requests are retried automatically.
- LLM failures and downstream service failures are handled independently.
- Processing status is maintained so unfinished work can be resumed.
- Structured logging allows failed interactions to be replayed and investigated later.

## 9. Auditability & Observability

_How would you debug a specific failed interaction 3 days after the fact?_

Every interaction should be traceable from start to finish.

### What you log (and what fields every log event includes)

Each log entry includes:

- interaction_id
- customer_id
- campaign_id
- lead_id
- call_stage
- processing_step
- tokens_used
- latency_ms
- timestamp
- error_message (if any)

These logs make it easy to trace a single interaction and identify where processing failed.

### Alert conditions

Alerts should be generated when:

- LLM rate limit exceeds 80% of capacity.
- Recording retrieval fails after all retry attempts.
- Processing queue grows beyond the configured threshold.
- A large number of LLM requests fail with 429 errors.
- Downstream services such as CRM or WhatsApp repeatedly fail.

## 10. Data Model

_Schema changes required. Show the SQL._

```sql
-- Your schema additions/changes here
```

---
The following schema changes are required.

```sql
ALTER TABLE interactions
ADD COLUMN processing_status VARCHAR(30) DEFAULT 'pending';

ALTER TABLE interactions
ADD COLUMN tokens_used INT DEFAULT 0;

ALTER TABLE interactions
ADD COLUMN recording_status VARCHAR(30) DEFAULT 'pending';

CREATE INDEX idx_processing_status
ON interactions(processing_status);

CREATE INDEX idx_customer_id
ON interactions(customer_id);
```

## 11. Security

_What data in this system is sensitive? How do you protect it at rest and in transit?_

---
The system processes sensitive customer information and call transcripts.

Security measures include:

- Encrypt sensitive data at rest.
- Use HTTPS/TLS for all external communication.
- Store API keys securely using environment variables.
- Restrict database access using role-based permissions.
- Avoid logging personally identifiable information (PII).
- Encrypt recordings stored in S3.

## 12. API Interface

_Did you change the API contract (`POST /session/.../end`)? If yes, explain why. If no, explain why you kept it._

---
No changes are required to the existing API endpoint.

The endpoint `POST /session/.../end` remains unchanged because all improvements are implemented internally within the post-call processing pipeline.

This keeps the API backward compatible while improving scalability and reliability.

## 13. Trade-offs & Alternatives Considered

| Option | Why Considered | Why Rejected / What You Chose Instead |

| Fixed 45-second wait for recordings | Simple to implement | Replaced with polling and exponential backoff to reduce delays and improve success rate. |
| Stop the dialer using a circuit breaker | Prevent LLM overload | Replaced with a rate limiter because it slows processing instead of stopping the entire system. |
| Single processing queue | Easy to manage | Introduced priority-based processing to prevent important calls from waiting behind low-priority calls. |
| Process every call with full LLM analysis | Simple implementation | Added differentiated processing to reduce token usage for low-value calls. |

---

## 14. Known Weaknesses

_What are the gaps in your design? What would you address next?_

---
Although the proposed design improves scalability and reliability, some limitations still remain:

- Customer token budgets are allocated using a simple strategy and could be improved with dynamic allocation.
- Priority rules may need adjustment based on changing business requirements.
- Queue management can become complex when handling a very large number of simultaneous campaigns.
- Additional monitoring dashboards would further improve operational visibility.

## 15. What I Would Do With More Time

_Specific, prioritised list — not a generic wishlist._

1. Implement dynamic customer token allocation based on real-time demand.
2. Add monitoring dashboards using Prometheus and Grafana.
3. Introduce dead-letter queues for permanently failed tasks.
4. Improve retry strategies using provider-specific retry information.
5. Add automated performance and load testing for large campaign simulations.