# Avoid Large Records with a Wildcard Owner Name

**Author:** Peng Zuo (CNNIC)  
**Email:** zuopeng@cnnic.cn  
**Date:** October 2025  
**Area:** Internet Area  
**Workgroup:** DNS Operation Group  
**Intended Status:** Best Current Practice
---

## Abstract

As DNS hosting becomes increasingly centralized—with many zones hosted by the same providers and sharing common name servers—the risk of DNS amplification has grown even more critical. Attackers can exploit these shared authoritative servers to launch large-scale DDoS amplification attacks.

This document provides operational guidance for DNS hosting providers to mitigate DDoS risks arising from amplification potential of responses derived from a wildcard owner name.

---

## 1. Introduction

Due to the small query–large response potential of the DNS system, it is easy to yield great amplification towards the victims. [RFC 5358] points out that both recursive and authoritative DNS can be abused as amplifiers and recommends restricting recursive lookup services to intended clients to prevent default-configured recursive servers from acting as reflectors in denial-of-service attacks. [RFC 8482] recommends that DNS servers return minimal-sized responses for queries with `QTYPE=ANY`.

Nowadays, the risk of DNS amplification remains critical. On one hand, DNS hosting has become increasingly centralized—many zones are now hosted by the same providers and share the same name servers. If a single hosted zone is exploited for DDoS amplification, all zones on those name servers may be affected. On the other hand, the number of open recursive servers worldwide continues to grow, providing more potential reflectors for attackers.

This document offers guidance to help DNS hosting providers mitigate DDoS risks arising from maliciously crafted DNS data.

---

## 2. Problem Description

An effective DNS DDoS amplification attack requires at least three conditions:

1. Very large responses to DNS queries.  
2. Queries that consistently bypass recursive DNS caches.  
3. Low cost or effort for the attacker.  

These conditions can be satisfied by configuring oversized DNS records with wildcard owner name (for example, very large TXT records) on a shared DNS hosting platform. In this case, an attacker can generate small queries with random labels—while discarding the responses—to induce excessive traffic between recursive resolvers and authoritative name servers. The use of wildcards causes queries for random names to bypass resolver caches and be repeatedly forwarded to upstream authoritative servers.

Below is an example of how an attacker could launch a DDoS attack to exhaust the outbound capacity from the victim authoritative server:

1. Identify the name server that hosts the victim domain.  
2. Publish a domain controlled by the attacker on the same name server.  
3. Create very large records with a wildcard owner name (for example, oversized TXT records).  
4. Identify open recursive resolvers worldwide by scanning IP space.  
5. Use packet generation tools to send DNS queries for random names (e.g., `{random}.attack TXT`) to open recursive resolvers worldwide.  
6. The outbound capacity from the authoritative server authoritative server hosting the victim domain will be exhausted.

Attackers can also use compromised hosts (e.g. launched from a botnet) using the configured system resolver to launch the attack. Such an attack could be launched using DNS queries triggered by other protocols, e.g. a web ad campaign that incorporates a reference to an URL including a target domain and a random sublabel, or a small compromise of a popular web page that includes an equivalent (invisible) defacement.

This is an efficient attack because a large response can easily be suppressed by the originating stub resolver, e.g. by using UDP transport without EDNS(0) which will trigger a truncated response from the open resolver (TC=1). This means the large responses are never sent to the originating host, and the bandwidth consumed is isolated to the path between the open resolver and the authoritative server. The use of UDP without EDNS(0) is not much of a fingerprint, and it is a stretch to imagine a mitigation based on just that signal.

#### 2.1. Attack Model
In large-scale amplification scenarios, the total bandwidth impact grows multiplicatively with the number of attack sources and the average response size.  To help operators reason about the practical effect of different response size limits, a simple model is described below.
Let:

* *N* be the number of attack sources (or open resolvers being exploited);
* *q* be the per-source query rate (queries per second);
* *Q* be the query packet size (bytes);
* *S* be the authoritative response size (bytes); 
* *R* be the fraction of queries that result in cache misses and thus generate upstream traffic (typically close to 1 for randomized names).

The total query rate is *N·q*.
The approximate bandwidth at different points in the resolution path is:

* Attacker upstream bandwidth:    *N·q·Q* bytes/s
* Authoritative server outbound bandwidth: *N·q·R·S* bytes/s
* Resolver total bandwidth (receive + send): *(N·q·Q + N·q·Q·R + N·q·R·S)* bytes/s

These relationships are linear in the response size *S*, illustrating that a modest reduction in response size directly reduces the required bandwidth at all other participants.
The model is simplified and ignores retransmissions, protocol overhead, and TCP fallback, but it provides a practical basis for comparing response size caps.

---

## 3. Recommendations

This section describes recommended best practices for keeping DNS data at reasonable sizes and reducing the risk that an attacker will abuse a shared name server. Because such attacks rely on shared authoritative name servers, these recommendations are primarily aimed at DNS hosting providers.

In general, operators should enforce size limits on large records—especially those with wildcard owner names—and apply restrictive controls where records also have very short TTLs. Exact threshold values should be chosen by each operator based on their environment and risk tolerance.

#### 3.1. Recommended Practices

1. **Apply size limits to large records with wildcard owner names.**
   Enforce maximum size thresholds for DNS records defined under wildcard owner names to prevent oversized responses from being used for amplification.

2. **Apply size limits to large records with very small TTLs. （can be removed）**  
   Excessively small TTL values increase cache-miss frequency and consequently the number of forwarded queries.

3. **Monitor for abnormal traffic patterns.**  
   Implement comprehensive logging and real-time alerting mechanisms to detect abnormal high query volumes or other signs of attack activity.

4. **Rate-limit queries that generate very large responses.**  
   Employ per-source, per-prefix, or query-type–aware rate limiting to reduce the impact of amplification and prevent overload on authoritative servers.

5. **Restrict and periodically review wildcard usage.**  
   Require clear justification, periodic review, or explicit approval for wildcard records containing large RDATA to avoid unintended amplification exposure.

6. **Instrument and test mitigation controls.**  
   Regularly test monitoring, rate-limiting, and record-size enforcement mechanisms under realistic load conditions, and tune thresholds to ensure effective protection without disrupting legitimate traffic.

Applying these measures will reduce the attack surface for DNS amplification attacks while allowing operators to choose limits that balance availability and safety for their user base.

---

## 4. Implementation Experience

#### 4.1. Some observations
In our recent tests, some known DNS hosting providers allow users to configure super large records with a wildcard owner name. Although the response may exceed the standard UDP packet size limit, it consequently triggers TCP fallback and allows responses to reach approximately 64 KB. This results in an amplification factor exceeding 1000×.

1. Cloudflare sets a limit of 8192 bytes for jumbo TXT records.  
2. Microsoft’s DNS service sets a limit of 4096 bytes.  
3. GoDaddy HAS NO LIMIT for jumbo TXT records.  
4. Alibaba Cloud and DNSPod set limits after we reported this risk to them.

#### 4.2. Aggregated bandwidth impact on the various actors at different Response Size Caps

For illustration, consider two nominal attack-source distributions:

| Parameter                   | Case A   | Case B   |
| --------------------------- | -------- | -------- |
| Number of sources (*N*)     | 1,000    | 50,000   |
| Query rate per source (*q*) | 1 qps    | 1 qps    |
| Query size (*Q*)            | 60 bytes | 60 bytes |
| Cache miss ratio (*R*)      | 1.0      | 0.8      |

The table below shows the approximate aggregate bandwidths at different response size caps based on the Model described in 2.1.

| Response Size<br> (bytes) |     Attacker Upstream<br> (Case A / Case B) | Authoritative Outbound<br> (Case A / Case B) |      Resolver Total<br> (Case A / Case B) | 
| --------------------: | --------------------------------------: | ---------------------------------------: | ------------------------------------: | 
|            **65,535** | 0.480 Mbps / 24.0 Mbps  |      524.3 Mbps / 21.0 Gbps |  525.2 Mbps/ 21.1 Gbps    | 
|             **8,192** | 0.480 Mbps / 24.0 Mbps  |      65.5 Mbps / 2.62 Gbps  |   66.5 Mbps / 2.66 Gbps   | 
|             **4,096** | 0.480 Mbps / 24.0 Mbps  |       32.8 Mbps / 1.3 Gbps  |   33.7 Mbps / 1.35 Gbps   |
|             **1,024** | 0.480 Mbps / 24.0 Mbps  |       8.2 Mbps  / 0.32 Gbps |   9.1 Mbps  / 0.37 Gbps   |
|               **512** | 0.480 Mbps / 24.0 Mbps  |       4.1 Mbps  / 0.16 Gbps |   5.1 Mbps   / 0.21 Gbps  |




---

## References

- [RFC 5358] — *Preventing Use of Recursive Nameservers in Reflector Attacks*  
- [RFC 8482] — *Providing Minimal-Sized Responses to DNS Queries That Have QTYPE=ANY*
