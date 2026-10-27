# Capability 3 (Submit Request to OPF API) - ADR-0003

**Problem Statement**

After Capability 10 (Wire II) produces a JSON pain.001.001.11, it must be submitted to the FIS OPF API and the outcome returned to Payment Exchange so the payment can be advanced or stopped.

**Assumptions**

- The connection to the OPF API (auth, CodeConnect, firewall, DevExchange registration) is established by Capability 2 (OPF API integration).
- Capability 10 (Wire II) produces a valid JSON pain.001.001.11 in the same service.
- The UETR received from the Payment Network Gateway is the idempotency key.
- OPF requires clear account numbers; Payment Exchange holds them tokenised.

**Constraints**

- The OPF API returns 202 (no content) on acceptance and synchronous error codes otherwise. No pain.002 over MQ is received in Phase 1.
- Each submission carries the required `uuid` and `idempotency-key` headers per the OPF API contract (Capability 2).
- The call from Payment Exchange is synchronous, so retries inside the service must be short.
- Phase 1 only: retired at the full OPF exit (Phase 2).

**Alternatives**

- **Submit inline within Wire II, synchronous (chosen)**
  - Pros: one service and one deployable for a transitional path; no queue to run, secure or monitor; no callback API or status endpoint; the outcome is known while the request is open.
  - Cons: a slow OPF call holds the Wire II request; long retries depend on PayX.
- **A dedicated SQS consumer (not taken forward after architecture review)**
  - Pros: decoupled and independently retryable.
  - Cons: an extra service, a queue and a second boundary for a throwaway path; more to secure and register.

**Decision**

Submit Request to OPF API is a component of the Wire II service (`bank-wire-ii-api`), not a separate service. It:

1. Detokenises the account numbers through the Turing API. OPF requires clear account numbers; they are never stored in DynamoDB.
2. Calls the OPF API (`POST /customer-credit-transfer-initiations`) over the connection established in Capability 2, with the `uuid` header and `idempotency-key` = UETR. On a 5xx or a connection failure it retries a few times immediately, then stops.
3. Stores the response (the latest status, including failures) against the transaction record in DynamoDB.
4. Returns OPF's response code to the PayX integration service in the same call. There is no callback API and no PayX-facing status endpoint. PayX acts on the code:

| OPF response | Meaning | PayX stage |
|---|---|---|
| 202 Accepted | Accepted for processing by OPF (not settled at the bank) | .OPFCLEARING |
| 400 Bad Request (validation) | Rejected | .OPFCLEARING.STOP |
| 422 Idempotent replay with a different payload | Rejected (duplicate UETR, different content) | .OPFCLEARING.STOP |
| 401 / 403 | Technical failure (credentials or entitlement); alert | .OPFCLEARING.STOP; PayX re-drives after the fix |
| 500 / 503, timeout, connection failure (after the in-service retries) | Technical failure | .OPFCLEARING.STOP; PayX re-drives with the same UETR |

Longer retries belong to PayX: when the clearing stage is not published within its SLA, PayX regenerates the input event and the integration service calls Wire II again with the same UETR. An unmapped response code is a visible failure, never a silent success.

End-to-end flow: see Capability 10 (Wire II). This component's part is steps 4 to 8.

**Justification/Rationale**

Following architecture review, Wire II and Submit Request to OPF API were combined into one service: a separate SQS consumer adds a service, a queue and a security boundary to a path that is retired in Phase 2. Keeping submission in Wire II leaves one owner for the idempotency key, the stored OPF response and the response to PayX, and a synchronous reply removes the need for a callback API or status endpoint. PayX already owns the payment's state, so it owns the long retries; the service only retries briefly on 5xx. The UETR as idempotency key makes a retry safe at FIS.

**Implications**

- Depends on Capability 10 (Wire II) for the JSON pain.001, in the same service, and on Capability 2 for the connection and contract.
- Returns OPF's response code synchronously; the PayX integration service produces .OPFCLEARING on 2xx and .OPFCLEARING.STOP on 4xx/5xx (Pod 2 dependency).
- Calls the Turing API to detokenise account numbers before the OPF call.
- The DynamoDB record must carry a success/failure status PayX can query; retry counts are not persisted (application-level retries only).
- Maps OPF's synchronous response codes to a Payment Exchange outcome. There is no asynchronous pain.002/MQ outcome in Phase 1; bank acknowledgements stay in OPF.
- The 202 means "accepted for processing" by OPF, not settled at the bank. Any later failure inside OPF is handled through OPF operations, not through PayX.

**Technical Debt incurred**

Transitional: a throwaway submission path, retired at the full OPF exit (Phase 2).

**Outstanding Questions**

- Final sign-off of the single synchronous service by architecture (diagram in review).
- Service name: `bank-wire-ii-api` or Wire Rail II (a Wire II service already exists in PIE).
- In-service retry policy: how many attempts, at what spacing, for which codes.
- DynamoDB columns and the success/failure flag PayX will query (with Pod 2).
- Payload-level encryption on the OPF API (pending with FIS); until then account numbers go over TLS only, covered by a UTEP.

**Annual Update**

| Version | Author | Date | Significant Change (Yes/No) | Change Made | Reason |
|---|---|---|---|---|---|
| 1.0 | Cameron Mayor | 2026-09-14 | Yes | Initial Creation | N/A |
| 1.1 | Vasanth Senthamaraikannan | 2026-10-27 | Yes | Rewritten for direct, synchronous submission: component of the Wire II service; SQS, pain.002/MQ and the callback to PayX removed; Turing detokenisation, retries and the response-code table added | Architecture decision of 2026-10-23, refined at the grooming of 2026-10-26 |
