# 🧭 Kamailio SBC – Full Dockerized Architecture & Monitoring Stack

**Date:** 2025‑10‑23  
**Author:** GPT‑5  
**Version:** 1.0  

---

## 🔷 1. Initial Design Summary

**Topology Overview**
```
Internet Users → Kamailio SBC (DMZ: PublicIP 1.2.3.4)

↓

├── Asterisk #1 (192.168.10.10)

└── Asterisk #2 (192.168.10.11)

Kamailio Role: Security, Load Balancing, NAT handling
RTPengine: Media relay (SRTP, NAT Fix)
Asterisk Cluster: Call Control and Media handling
```

## 🛡️ 2. Security Layers (Multi‑Layer Defense)

|  Layer | Tool / Module	  | Description  |
|---|---|---|
| Anti‑Flood	  | pike  | Limits requests rate per IP  |
| Static Blacklist	  |permissions	   |  Blocking fixed IPs in MySQL address table |
| SIP User‑Agent Filtering	  | sanity / custom route	  | Blocks scanners like friendly‑scanner  |
|  Authentication	 |  auth, auth_db	 |Digest Auth via credential DB   |
| Custom Header Auth	  |X‑Company‑Signature	   | Additional check for official clients  |
|  Transport Security	 | TLS / mTLS	  | Optional certificate‑based authentication  |
| SRTP	  | rtpengine  | Encrypted media path  |
