# CIP-01XY

<pre>
Number: CIP-01XY
Title: Remove IP Whitelists from the Global Synchronizer
Author(s):
  Martin Florian
  Nicu Reut
  Pasindu Tennage
  Moritz Kiefer
Type: Standards Track
Status: Draft
Created: 2026-xx-xx
Approved: 2026-xx-xx
License: CC0-1.0
</pre>


## Abstract

Validator access to the Global Synchronizer - i.e., access to Scan and sequencer APIs - is currently restricted by explicit IP whitelisting rules: connecting new validators to the network requires requesting access in a process coordinated by the Canton Foundation.
Additionally, an established Super Validator (SV) must explicitly "sponsor" each new validator onboarding via manually issuing an onboarding secret.

This CIP proposes the removal of IP whitelisting requirements as well as changes to the validator onboarding process so that new validators can join the network in a self-service way.

The whitelisting requirement cannot simply be dropped: it currently protects the network from malicious actors and excessive load.
For example, the whitelisting requirement makes it straightforward to exclude and block nodes that have been identified as malicious.

This CIP proposes a new onboarding flow that grants synchronizer access automatically once a minimum amount of traffic has been purchased.
The traffic purchase requirement makes it costly for malicious actors to reconnect after getting blocked and effectively limits Sybil attacks.

This CIP furthermore makes "dropping the whitelist" conditional on the enforcement of rate limits on both the application and (SV) infrastructure layers.

## Specification

### Overview and Scope

This CIP focuses on removing the whitelist requirement for the validator-facing public endpoints of Scan and the sequencers.
No changes are made to the access requirements for endpoints that are only used by other SVs as well as any access requirements enforced by individual SVs and validators for their operators and users.

In order to safely remove the IP whitelisting requirement for the public endpoints of Scan and the sequencers, the following prerequisites must be met:

1. Scan and sequencer APIs are audited and hardened. (see *API Security*)
2. Traffic-based onboarding is in place. (see *Traffic-based Validator Onboarding*).
3. Rate limiting and DoS protection is enforced. (see *Rate Limiting and DoS Protection*).
4. For MainNet and TestNet: Testing period on DevNet has passed (see *Rollout Plan*).

### API Security

Audits of the APIs that will be exposed to the public will be performed to ensure that the exposed endpoints are not vulnerable to abuse or denial-of-service attacks.
Audit results will be made available to SVs and other key network stakeholders.

### Traffic-based Validator Onboarding

#### Onboarding Flow

The existing validator onboarding model relies on a single sponsor SV to unilaterally onboard a new validator by generating an onboarding secret. This CIP replaces the sponsor model with a decentralized flow.

Instead of secrets, new validator onboarding is now driven by traffic purchases. The high-level onboarding flow works as follows:

- A prospective validator operator spins up their validator node to generate their cryptographic keys and obtain a unique *participant ID*.

- An existing party on the network purchases traffic for that new participant ID (using Canton Coin; see also *Easier Traffic Purchases* below.)

- This traffic purchase automatically triggers the onboarding process, allowing the validator to connect to the Global Synchronizer.

Concretely, when the `MemberTraffic` contract is created with sufficient traffic (as publicly defined on ledger), SV automation observes this contract and automatically submits a `ParticipantSynchronizerPermission` topology transaction for the validator's participant ID. Once confirmed by a majority of SVs, the validator's connection is accepted.

"Sufficient traffic" here refers to the *total traffic purchased*, not the current traffic balance.
The specific minimum amount will be configurable via an on-ledger governance vote.
This CIP sets 10MB as a starting default, which at the current MainNet rate implies that each validator must spend a total of $600 USD (or more) for traffic purchases in order to be able to connect.

In the event that SVs detect network abuse by a validator, SVs can collectively decide to blacklist the offending validator via a governance vote.
SVs use a new `ValidatorBlacklist` contract to vote on revoking a validator's synchronizer access.
The `ValidatorBlacklist` contract supports two modes of revocation:

- Temporary: Suspend the validator's access until a specific time by setting the `loginAfter` parameter on the `ParticipantSynchronizerPermission`.
- Unlimited: Fully revoke the `ParticipantSynchronizerPermission` topology state.

Even in the latter case, a blacklisted validator can still be whitelisted again at a later time:
SVs may vote to issue a new `ParticipantSynchronizerPermission`.

#### Network Transition

In order for traffic-based onboarding to offer effective protection, each network must undergo a 3-step transition process initiated by an on-chain vote:

1. SVs vote to initiate the switch to traffic-based onboarding. The vote has effective time T1 and commits to a switch over at time T2, i.e., `DsoRulesConfig.svOperationsSwitchOverTimes[trafficBasedOnboarding]=T2`.
2. Between T1 and T2, SVs automatically submit `ParticipantSynchronizerPermission` topology transactions for all SVs and all existing validators that hold a valid `MemberTraffic` contract with sufficient traffic.
3. At time T2, the network automatically switches over to `RestrictedOpen` mode.

Once the network switches over (step 3), existing validators whose total past traffic purchases is below the minimum traffic requirement lose access to the synchronizer.
Access can be restored by purchasing sufficient traffic (via an active validator) to meet the traffic requirement (defined on ledger and made public via Scan).

#### Easier Traffic Purchases

In order for a new validator to onboard, an existing party on the network must purchase traffic for that validator's participant ID (see *Onboarding Flow* above).
Any existing party that holds sufficient Canton Coin may perform this purchase on behalf of the new validator.
For example, dedicated services may emerge that offer traffic purchases in exchange for fiat currency payments.

To enable regular wallet users to purchase traffic for new validators, traffic purchases will be supported through token standard v1 compatibility mode:
The `ExternalPartyAmuletRules` will be extended so that a transfer to an address of the form `cip-<xxx>_traffic-purchase::1220...abcd` with an appropriately formatted memo tag `memberId=<member>&synchronizerId=<synchronizer>&migrationId=<int>&trafficAmount=<int>` will have the same effective outcome for the referenced member ID (typically a participant ID) as purchasing traffic via `AmuletRules_BuyMemberTraffic`.
For more details see the reference implementation of this feature at: https://github.com/canton-network/splice/pull/7427

To make it easier to deploy validators on DevNet, SVs will expose a new DevNet-only endpoint: `/v0/devnet/onboard/validator/purchase-traffic`.
This endpoint uses an SV's own (DevNet) coin holdings to generate `MemberTraffic` for joining validators.
To prevent denial-of-service attacks on this free (DevNet-only) onboarding mechanism, aggressive IP-based rate limiting is applied to the new endpoint.

### Rate Limiting and DoS Protection

In order to safely remove the IP whitelisting requirement for the public endpoints of Scan and the sequencers on a given network,
effective rate limiting and DoS protection must be in place for that network.
All of the following prerequisites must be met:

1. Application-level rate limiting is enforced (see *Application-Level Rate Limiting*).
2. Infrastructure-level rate limiting and DDoS protection are enforced (see *Infrastructure Requirements*).
3. Verification has passed (see *Verification*).

#### Application-Level Rate Limiting

The Global Synchronizer's Canton Coin Scan app must provide:

- Global limits: a maximum number of requests per configurable window (default 60s), plus a short-window burst allowance (default 1s).
- The same type of limits per source IP to avoid a single client consuming all the global allowance.
- Global and per source IP limits configurable per endpoint (more specifically, per operation defined in the OpenAPI spec), allowing for more restrictive rate limits for certain operations.
- Bounded per-IP-address-range overrides, so that a known high-volume consumer can be granted a higher limit without being exempted from limiting.

The sequencer must provide:

- Per-member and global transaction limits (based on synchronizer-wide load; also known as "sequencer caps").
- Global and per-IP request-rate limits (based on sequencer-local load, equivalent to the HTTP limits above).
- Concurrency caps on expensive endpoints. (Some sequencer endpoints invoke long-running operations, making it difficult to manage their overhead via rate limits alone.)

The limit values for Scan and the sequencer must be maintained in a shared, version-controlled configuration repository and applied by all SVs, so that limits are identical across SVs and tunable network-wide without needing a new release.
*All* SVs *must* adopt the agreed upon rate limiting configuration in a timely fashion and *may not* override rate limits with individual settings or exceptions (to ensure fairness and consistent quality of service).

#### Infrastructure Requirements

In front of its public endpoints, every SV must implement as part of their ingress setup:

- Global rate limiting across all Scan and all sequencer endpoints.
- Global per-source-IP rate limiting across the same endpoints.
- DDoS protection (for example a cloud provider's network-layer DDoS protection or an equivalent service).
- The ability to add a temporary limit or block per path and/or IP address range, as an incident-response measure.
- Alerting on proximity to, and breach of, the configured limits, based on the metrics exposed by the rate-limiting layer.
- Correct client identification: the ingress layer must correctly set client IPs in HTTP headers before forwarding to backends, to allow reliable application-level per-IP rate limiting.

The specific requirements will be documented in depth in the public documentation available to the SVs.

Like for the application-level rate limiting, core rate limiting parameters (requests per minute, ...) will be maintained in a shared, version-controlled configuration repository available to all SVs.
SV operators must ensure that their individual deployment systems can parse and apply the shared configuration.
*All* SVs *must* adopt the agreed upon rate limiting configuration in a timely fashion and *may not* override rate limits with individual settings or exceptions (to ensure fairness and consistent quality of service).

#### Verification

A sanity-check tool will be provided to SVs to verify that the preconditions for whitelist removal are met. It will check that:

- the configured global, per-IP, and per-operation limits are in effect and match the network configuration;
- throttled requests receive the expected response;
- client IPs are correctly identified and limited, even when forwarded through a trusted proxy;

SVs are encouraged to test both their and their peers' setups prior to the removal of the whitelisting requirement.
All tests involving higher loads (to effectively test rate limits) should be coordinated within the SV operators group.

### Rollout Plan

The whitelisting requirements can be dropped on a network once all prerequisites above are fulfilled for that network,
most notably once the network has transitioned to traffic-based onboarding and all SVs have made the necessary adjustments to their deployments to support effective rate limiting.

Additionally, the whitelist requirement should only be dropped on TestNet and MainNet after a testing period on DevNet of at least 4 weeks.
More specifically, the opening schedule should be no more condensed than:

- Week 0: Whitelist requirement dropped on DevNet
- Week 4: Whitelist requirement dropped on TestNet
- Week 6: Whitelist requirement dropped on MainNet

## Motivation

### A Public Network

A decentralized network must be public and open, and the participation should not require manual, off-ledger gatekeeping like IP whitelisting. This CIP ensures anyone can join the network seamlessly based on protocol-level rules rather than manual administrative approval.

### Governance

The existing secret-based onboarding model makes a single sponsor SV responsible for admitting a new validator. This CIP ensures that onboarding and offboarding validators is a decentralized process.

### Operational Overhead

The manual generation of onboarding secrets by a sponsor SV is a time-consuming process that does not scale as the network expands.
Furthermore, maintaining static IP whitelists creates a significant operational bottleneck, severely delaying the speed at which new validators can join the network.

## Rationale

- To prevent attackers from misusing the Global Synchronizer, joining the network must have a cost. Since validators already purchase `MemberTraffic` to transact, we reuse this existing financial requirement as the economic barrier to entry rather than inventing a new mechanism.

- With a traffic-based onboarding gate in place, the IP whitelist is no longer needed to have control over the participants that can connect to the synchronizer (the traffic-based onboarding gate has no impact on Scan).

- The rate limiting and DDoS protection provide a bound on resource consumption and protect against denial-of-service attacks, so that the network can remain available even when under load.

## Backwards Compatibility

- Once a network initiates the transition to traffic-based onboarding, new validators wishing to join the network may be required to use a sufficiently recent version of Splice to be able to onboard.
- Once a network completes the transition to traffic-based onboarding, existing validators that haven't purchased sufficient traffic (in total, over their whole lifetime, as per the state recorded in `MemberTraffic` contacts) to cover the minimal required traffic requirement will experience synchronizer downtime until sufficient traffic is purchased (via an active validator) for their participant ID.

## Reference Implementation

- An advanced reference implementation for traffic-based onboarding is available at: https://github.com/canton-network/splice/tree/feature-public-sequencer-and-scan
- Advanced reference implementations for both application-level and infrastructure-level rate limiting are available on Splice `main`: https://github.com/canton-network/splice/
- Support to traffic purchase through token standard v1 compatibility mode: https://github.com/canton-network/splice/pull/7427/

## Copyright

This CIP is licensed under [CC0-1.0: Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).

## Changelog

- 2026-XX-XX: Approved
- 2026-XX-XX: Initial draft v1 (based on merging two previous CIP drafts)
