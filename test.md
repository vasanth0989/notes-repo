Hi Joe, here is the split of Capability 2 and Capability 10 stories by FIS sandbox dependency.

"Without sandbox" = can be built and accepted against our WireMock stub, the FIS sample messages we already have, and the Code Connect gateway as it is today (token works, no backend). "Needs sandbox" = cannot complete until FIS signs the SOW and configures the OPF backend for Discover.

CAPABILITY 10 - Wire II pain.001 Generation (ENTPMTS-143224), 18 open stories

Without sandbox (17 stories, 67 pts):
- ENTPMTS-144884 Implement wire-ii-pain001-mapper to convert the Payment Object to pain.001 V11 XML (5, in progress)
- ENTPMTS-143562 Implement XML-to-JSON conversion for the generated pain.001.001.11 message (5)
- ENTPMTS-143564 Implement Business Application Header wrapping per the OPF API contract (3)
- ENTPMTS-143560 Wire the pain.001 mapper into the Wire Rail II service as the generation step (3)
- ENTPMTS-143577 Implement direct submission to the OPF API (3) - verified against the stub; first real call is the Capability 2 sandbox story
- ENTPMTS-143578 Implement the Bank Wire Rail II API response contract for the PayX integration service (3)
- ENTPMTS-143580 Implement DynamoDB transaction record persistence for Wire II submissions (3, ready)
- ENTPMTS-145620 Provision the DynamoDB table and access for Wire Rail II (3)
- ENTPMTS-145619 Merge the FIS client prototype into the scaffolded Wire Rail II service and get the pipeline green (5)
- ENTPMTS-145516 Stand up a WireMock stub of the FIS OPF API for development (5, in progress)
- ENTPMTS-145515 Add field-level validation to the Bank Wire Rail II API contract (5)
- ENTPMTS-144081 Registering Wire II API in developer exchange with an Open API spec (5, in review)
- ENTPMTS-144735 Complete data standardization for Payx-Wire-Payment-Processor API attributes (3, in progress)
- ENTPMTS-145622 Align the message with the FIS sample requests and the data-mapping elaboration (3) - samples in hand; the data-mapping elaboration with FIS comes after the SOW
- ENTPMTS-145621 Add the QA environment to the Wire Rail II pipeline and deploy to QA (5)
- ENTPMTS-143585 Observability for Wire Rail II and automated end-to-end test against the stub (5)
- ENTPMTS-143579 Verify in QA that the PayX integration service produces the clearing stages from the Bank Wire Rail II response (3)

Needs sandbox (1 story, 5 pts):
- ENTPMTS-145623 End-to-end test in UAT: PayX integration service to Bank Wire Rail II to the OPF sandbox (5)

CAPABILITY 2 - OPF API Integration (ENTPMTS-143223), 9 open stories

Without sandbox (6 stories, 22 pts):
- ENTPMTS-144736 Registering for external API consumption for FIS OPF, non-production (3, in progress)
- ENTPMTS-145332 Complete the Exchange registration of the FIS OPF API and obtain the security approvals (3)
- ENTPMTS-145334 Secure the OPF API connection: credentials, transport and data-protection exception (5)
- ENTPMTS-145335 Open the network path from the hosting ASV to FIS Code Connect (5)
- ENTPMTS-145337 Build the shared OPF API client configuration for the Wire II service (3)
- ENTPMTS-145624 Move Wire Rail II from the PIE ASV to the Wire II ASV (3)

Needs sandbox / FIS (3 stories, 9 pts):
- ENTPMTS-145333 Establish FIS Code Connect connectivity to the OPF sandbox (3)
- ENTPMTS-145625 Prove the authenticated connection from the Wire Rail II UAT runtime to the OPF sandbox (3)
- ENTPMTS-145336 Obtain production OPF API credentials and configure the connection per environment (3) - issued by FIS after the SOW

Bottom line: 23 of 27 open stories (89 pts) proceed without the sandbox. The 4 that need it (14 pts) are at the end of the plan: sandbox first call in sprint 26.4.6 (10-24 Dec), production credentials, UAT connection proof and the UAT end-to-end test in 27.1.1 (Jan). So the sandbox does not block the team before mid-December, provided the SOW is signed and the backend is configured by early December; if the SOW slips, those four move to late January.
