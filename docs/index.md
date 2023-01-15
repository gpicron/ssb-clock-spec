# Introduction

## Abstract

The ssb-clock feature is a set of protocols and data structures aimed at
improving the security, reliability, and scalability of Secure Scuttlebutt (SSB)
network. It introduces a new data structure, the Bloom clock, which allows
agents to express the causality between messages in potentially different feeds,
and also improves the efficiency and security of the gossip protocol. The
feature also includes protocols for efficiently resolving collaboratively a fork
in a feed and informing the owner of the feed, sharing status and synchronizing
feeds between connected peers, and establishing a consensus on elapsed time.
These protocols aim to improve the accuracy of determining the order of messages
in a feed and the state of the system. The protocol for establishing a consensus
on elapsed time can be used for several potential use cases such as timestamps,
scheduling, time-based events, network synchronization and proof of elapsed
time. The protocols are based on the combination of cryptographic techniques
such as digital signatures and hash chains to ensure that each increment to the
global counter is valid and that sufficient time has elapsed between increments.

## Status of this memo


| name      | ssb-clock-spec  |
|-----------|-----------------|
| Version   | 0.1.0           |
| Status    | Draft           |
| Author(s) | Geoffrey Picron |
| Category  | Experimental    |
| Relations |                 |
| License   | CC-BY-SA-4.0    |


This document is not an SSB specification; it is published for examination,
experimental implementation, and
evaluation.

This document defines an Experimental Protocol for the SSB community.
This is a contribution to the SSB specification series. The Specification
author has chosen to publish this document at its discretion and makes no
statement about its value for implementation or deployment.

## Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in [@rfc2119].

## Changelog

| Version | Changes |
|---------|---------|
| 0.1.0   | Notes   |







