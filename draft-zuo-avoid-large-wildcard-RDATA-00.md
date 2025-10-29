# Avoid Large Records with a Wildcard Owner Name

**Author:** Peng Zuo (CNNIC)  
**Email:** zuopeng@cnnic.cn  
**Date:** October 2025  
**Area:** Internet Area  
**Workgroup:** DNS Operation Group  

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

This document provides operational guidance for DNS hosting providers to mitigate DDoS risks arising from amplification potential of responses derived from a wildcard owner name.

To be specific, an attacker could launch a DDoS attack as follows:

1. Identify the name server that hosts the victim domain.  
2. Publish a domain controlled by the attacker on the same name server.  
3. Create very large records with a wildcard owner name (for example, oversized TXT records).  
4. Identify open recursive resolvers worldwide by scanning IP space.  
5. Use packet generation tools to send DNS queries for random names (e.g., `{random}.attack TXT`) to open recursive resolvers worldwide.  
6. The authoritative server hosting the victim domain will receive a massive volume of traffic and suffer a DDoS amplification.

---

## 3. Recommendations

This section describes recommended best practices for keeping DNS data at reasonable sizes and reducing the risk that an attacker will abuse a shared name server. Because such attacks rely on shared authoritative name servers, these recommendations are primarily aimed at DNS hosting providers.

In general, operators should enforce size limits on large records—especially those with wildcard owner names—and apply restrictive controls where records also have very short TTLs. Exact threshold values should be chosen by each operator based on their environment and risk tolerance.

### Recommended Practices

1. **Apply size limits to large records with wildcard owner names.**  
   Enforce maximum size thresholds for DNS records defined under wildcard owner names to prevent oversized responses from being used for amplification.

2. **Apply size limits to large records with very small TTLs.**  
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

In our recent tests, some known DNS hosting providers allow users to configure super large records with a wildcard owner name. Although the response may exceed the standard UDP packet size limit, it consequently triggers TCP fallback and allows responses to reach approximately 64 KB. This results in an amplification factor exceeding 1000×.

### Observations

1. Cloudflare imposes a limit of 8192 bytes for jumbo TXT records.  
2. Microsoft’s DNS service sets a limit of 4096 bytes.  
3. GoDaddy **has no limit** for jumbo TXT records.  
4. Alibaba Cloud and DNSPod set limits after we reported this risk to them.

---

## References

- [RFC 5358] — *Preventing Use of Recursive Nameservers in Reflector Attacks*  
- [RFC 8482] — *Providing Minimal-Sized Responses to DNS Queries That Have QTYPE=ANY*
