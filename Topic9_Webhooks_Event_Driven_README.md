# Topic 9 - Webhooks & Event-Driven Architectures

## Overview
This n8n training workflow accepts POST webhook events, generates an HMAC-SHA256 digest using the n8n Crypto node and a configured Crypto credential, compares it with the `x-signature-sha256` request header, rejects invalid requests, and routes valid events to simulated handlers. It generates execution-level audit and security-incident records.

## Endpoint and request contract
- HTTP method: `POST`
- Test URL: `https://n8n.tayanasolutions.com/webhook-test/v1/events/inbound`
- Path: `v1/events/inbound`
- Content-Type: `application/json`
- Signature header: `x-signature-sha256: sha256=<hex-hmac>`
- Signature algorithm: HMAC-SHA256 of the exact request body bytes, using the same shared secret in Postman and the n8n Crypto credential.
- Webhook configuration: Raw Body ON; Respond = `Using 'Respond to Webhook' Node`.
- Do not put real secrets in workflow exports, screenshots, or this README.

## Architecture
```
Webhook_InboundEvents
  -> Crypto_GenerateHMAC
  -> Code_VerifyHMACSignature
  -> IF_IsSignatureValid
     |-- false -> Resp_401Unauthorized -> Log_SecurityIncident
     `-- true  -> Resp_202Accepted -> Switch_RouteEventType
                     |-- user.created    -> Code_HandleUserCreated ------|
                     |-- invoice.paid    -> Code_HandleInvoicePaid ------|
                     |-- order.cancelled -> Code_HandleOrderCancelled ---|-> Code_AuditTrail
                     `-- fallback        -> Code_HandleUnknownEvent -----|
```

## Node functions
| Node | Purpose |
| --- | --- |
| Webhook_InboundEvents | Receives the raw POST request and headers. |
| Crypto_GenerateHMAC | Calculates HMAC-SHA256 using a secret stored in an n8n Crypto credential. |
| Code_VerifyHMACSignature | Compares the computed signature and supplied header; emits `isValidSignature`, `event`, and `payload`. |
| IF_IsSignatureValid | Routes verified requests to the accepted branch and other requests to rejection. |
| Resp_401Unauthorized | Sends HTTP 401 JSON response for invalid or missing signatures. |
| Log_SecurityIncident | Generates a security incident record within the workflow execution. |
| Resp_202Accepted | Sends HTTP 202 JSON acknowledgment for valid signatures. |
| Switch_RouteEventType | Routes `user.created`, `invoice.paid`, `order.cancelled`, and unknown events. |
| Code_HandleUserCreated | Simulates user provisioning. |
| Code_HandleInvoicePaid | Simulates invoice ledger update. |
| Code_HandleOrderCancelled | Simulates inventory restocking. |
| Code_HandleUnknownEvent | Safely handles unsupported event names. |
| Code_AuditTrail | Generates an audit entry with event, outcome, reference ID, and timestamp. |

## Testing with Postman
1. Open the n8n Webhook node and click **Listen for test event**. The `/webhook-test/` endpoint is temporary and normally handles one request per listening session.
2. In Postman, select POST, use the test URL, and choose Body -> raw -> JSON.
3. For valid tests, enable a pre-request HMAC script that signs the exact raw JSON body and sets the `x-signature-sha256` header. The script secret must match the Crypto credential.
4. Send the request; expect **202 Accepted** and the matching Switch branch and audit output.
5. For a missing-signature test, disable signing and remove the signature header; expect **401 Unauthorized** and a security incident record.
6. For an invalid-signature test, send an incorrect `x-signature-sha256` value; expect **401 Unauthorized**.
7. Click **Listen for test event** again before each new test. A 404 `webhook is not registered` error indicates the test listener is not active.

### Sample event bodies
```json
{"event":"user.created","userId":"USR-101"}
```
```json
{"event":"invoice.paid","invoiceId":"INV-101"}
```
```json
{"event":"order.cancelled","orderId":"ORD-101"}
```
```json
{"event":"payment.refunded","paymentId":"PAY-101"}
```

## Observed test results (22 October 2026)
| Scenario | Observed evidence | Status |
| --- | --- | --- |
| Valid user.created | Handler branch and audit executed | Pass - n8n |
| Valid invoice.paid | Handler branch and audit executed | Pass - n8n |
| Valid order.cancelled | Handler branch and audit executed | Pass - n8n |
| Valid unknown event | Fallback branch and audit executed | Pass - n8n |
| Valid signed webhook | Postman HTTP 202 Accepted | Pass - HTTP |
| Missing signature | Postman HTTP 401 Unauthorized; security incident output | Pass - HTTP |
| Incorrect signature distinct from missing header | No separate verified Postman screenshot provided | Not separately verified |

## Evidence and outputs
- `Code_AuditTrail` example: `eventHandled: payment.refunded`, `executionOutcome: LOGGED_UNKNOWN_EVENT`, `referenceId: null`.
- `Log_SecurityIncident` example: `incidentType: INVALID_WEBHOOK_SIGNATURE`, `severity: HIGH`, `httpStatus: 401`.
- Refer to the accompanying evidence PDF for screenshots and explanations.

## Limitations and production-readiness notes
- Handler nodes are simulations: they do not create users, update ERP ledgers, or restock inventory.
- Audit and security-incident records are only generated in n8n execution output; durable storage was not demonstrated.
- The Code-node signature comparison was not demonstrated as timing-safe; harden comparison before production.
- Replay protection (timestamp/nonce), request-size limits, secret rotation, and monitoring are not demonstrated.
- HTTP 202 acknowledges acceptance; it does not prove durable queueing or successful downstream processing.
- Export the workflow JSON separately from n8n and review it to ensure credentials and secrets are not embedded.
