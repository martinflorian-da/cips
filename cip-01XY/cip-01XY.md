# CIP-0121

<pre>
Number: CIP-0121
Title: Remove IP Whitelists from the Global Synchronizer
Author(s): Nicu Reut
Type: Standards Track
Status: Draft
Created: 2026-08-20
License: CC0-1.0
Requires: CIP-0045
</pre>


## Abstract

Access to the public endpoints of the Global Synchronizer (Scan and the sequencers) is today limited by IP whitelisting rules maintained separately each Super Validator (SV) node. Every new validator, address change, or additional egress range requires a request-and-approval step, coordinated by Canton Foundation operations,
before that validator can reach the network. This CIP proposes to remove the IP whitelist so that validators can join the network without any whitelisting requirements.

The whitelist cannot simply be dropped: it currently provides two protections incidentally, a bound on who can consume node resources, and a bound on who can reach the synchronizer. 
This CIP therefore makes removal conditional on two prerequisites being in place first: enforced rate limiting (at both the application and infrastructure layers) and a traffic-based onboarding gate. 


## Specification

The focus is on removing the IP whitelisting for the public endpoints of Scan and the sequencers. 

### Preconditions for Removal

1. **Traffic-based onboarding is in place.** 
   1. Synchronizer access must be secured so that Validators incur a traffic cost to onboard, and malicious nodes can be penalized by the protocol. 
   2. "Traffic-based onboarding will be covered in a separate CIP. This CIP focuses on the remaining preconditions for whitelist removal.
2. **Application-level rate limiting is enforced** (see *Application-Level Rate Limiting*).
3. **Infrastructure-level rate limiting and DDoS protection are enforced** (see *Infrastructure Requirements*).
4. **Verification has passed** (see *Verification*).

### Application-Level Rate Limiting

**The Global Synchronizer's Canton Coin Scan app must provide:**

- Global limits: a maximum number of requests per configurable window (default 60s), plus a short-window burst allowance (default 1s).
- The same type of limits per source IP to avoid a single client consuming all the global allowance.
- The same type of limits, configurable per OpenAPI operation, allowing for more restrictive rate limits for certain operations.
- Bounded per-IP-address-range overrides, so that a known high-volume consumer can be granted a higher limit without being exempted from limiting.


**The sequencer must provide:**

- Concurrency caps on expensive endpoints.
- Per-member and global transaction limits.
- Global and per-IP request-rate limits on the critical endpoint subset, equivalent to the HTTP limits above. 

**Configuration.** The limit *values* for Scan and the sequencer must be maintained in the shared, version-controlled configuration repository and applied by all SVs, so that limits are identical across SVs and tunable network-wide without needing a new release. 

### Infrastructure Requirements

In front of its public endpoints, every SV must operate:

- Global rate limiting across all Scan and all sequencer endpoints.
- Global per-source-IP rate limiting across the same endpoints.
- DDoS protection (for example a cloud provider's network-layer DDoS protection or an equivalent service).
- The ability to add a temporary limit or block per path and/or IP address range, as an incident-response measure.
- Alerting on proximity to, and breach of, the configured limits, based on the metrics exposed by the rate-limiting layer.

These requirements will be documented more in depth in the public documentation available to the SVs. 

### Scope

The removal of the whitelists applies only to the validator-facing public endpoints of Scan and the sequencers. 
No changes are made to the non-public endpoints (endpoints accessible only on the SVs internal network), or to the whitelists that handle traffic that occurs only between SV nodes.

### Verification

A sanity-check tool will be provided to supervalidators to verify that the preconditions for whitelist removal are met. It will check that:

- the configured global, per-IP, and per-operation limits are in effect and match the network configuration;
- throttled requests receive the expected response;
- client IPs are correctly identified and limited, even when forwarded through a trusted proxy;

#### Audit

An internal audit of the APIs that will be exposed to the public will be performed to ensure that they are not vulnerable to abuse or denial-of-service attacks.

## Motivation

The goal is a public network, where a validator can join and reach Scan and the sequencers on its own, without going through a whitelisting process.
The Scan APIs are public and any client interested in consuming the data should be able to do so without a central approval step. 

## Rationale

With a traffic-based onboarding gate in place, the IP whitelist is no longer needed to have control over the participants that can connect to the synchronizer (the traffic-based onboarding gate has no impact on Scan). 
The rate limiting and DDoS protection provide a bound on resource consumption and protect against denial-of-service attacks, so that the network can remain available even when under load. 

## Backwards Compatibility

No compatibilty concerns. 

## Reference Implementation

- Splice implements HTTP rate limiting for its apps, providing global, per-IP, and per-OpenAPI-operation limits with configurable windows, burst allowances, and a bounded LRU cache of tracked addresses. Outstanding for this CIP: per-IP-address-range overrides, and sourcing the limit values from the shared configuration repository.
- The sequencer implements per-endpoint concurrency caps and per-member/global traffic limits. Outstanding: global and per-IP request-rate limits on the critical endpoint subset, and propagation of the real client address as gRPC metadata.
- Infrastructure-level global and per-IP rate limiting, ban thresholds, and DDoS protection are implemented in the reference deployment tooling, together with alerting on the rate-limit metrics.
- The verification tool is to be delivered alongside the enforcement rollout.

## Copyright

This CIP is licensed under [CC0-1.0: Creative Commons CC0 1.0 Universal](https://creativecommons.org/publicdomain/zero/1.0/).
