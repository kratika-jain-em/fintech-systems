# Idempotency in Payment Systems

## Context
Idempotency is a property of a system's operation or a request that produces the same results whether it is executed once or many times. In a distributed environment, it is not uncommon for networks to become unavailable at times, or server crash happening mid request. For a payment system, these cases could lead to client retrying the payment request, and without idempotency, the risk is of duplicate processing (a customer charged twice, or a refund processed again etc.).

In these failures, the financial impact is immediate, reconciliation burden is significant and the trust damage could be lasting. 

## Example Problem
Consider a client submitting a payment request to API. The request reaches the server and payment is processed succesfully. But a network unavailability caused the response to never make it back to the client. The client has no idea if the payment is processed or not, and they retries. 

**Without Idempotency**
```
Client                    Load Balancer              Payment Service
  |                            |                           |
  |------- POST /payments ----->|                           |
  |                            |-------- forward ---------->|
  |                            |                           | (payment processed)
  |                            |                           | (network failure - response lost)
  |<------- timeout -----------|                           |
  |                            |                           |
  |------- POST /payments ----->|  (retry)                 |
  |                            |-------- forward ---------->|
  |                            |                           | (DUPLICATE payment processed)
  |                            |<------- 200 OK -----------|
  |<------- 200 OK ------------|                           |
```

The client sees 200 response second time. The issue with duplicate payment isn't picked up until reconciliation or if customer complaints.  

## The Solution - Idempotency Key
The standard, accepted solution is the *idempotency key*, a client-generated unique id attached to each request, that a server uses to recognize and deduplicate the repeated requests. 
