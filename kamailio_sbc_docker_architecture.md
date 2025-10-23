🧭 Kamailio SBC – Full Dockerized Architecture & Monitoring Stack
Date: 2025-10-23

Author: GPT‑5

Document Type: Technical Implementation Manual

Version: 1.0

🔷 Table of Contents
Initial Design Summary
Security Mechanisms & Layers
Advanced Identification Mechanisms
Centralized Monitoring & Logging
Full Docker Architecture
Docker Compose File
Monitoring Components
Security in Production
Launch Commands
Architecture Diagrams
Summary
1. Initial Design Summary
Base architecture for SBC deployment:

Element	Description
Users (Internet)	Softphones, SIP Clients behind NAT, or SIP Trunks
SBC (Kamailio)	Deployed in DMZ, handles SIP/TLS, NAT, authentication, and routing
Asterisk Servers	Two internal VoIP servers (192.168.10.10, 192.168.10.11) for voice handling
RTPengine	Handles media relay and SRTP bridging
Monitoring/Analytics Stack	Homer Collector, ELK (Elastic Stack), Prometheus/Grafana for metrics
Traffic flow:

User Agent (Internet)

│ (TLS / UDP)

▼

Kamailio SBC (DMZ)

│ (Load Balancing SIP)

▼

Asterisk #1, #2 ── RTP handled via RTPengine

2. Security Mechanisms & Layers
Implemented security layers:

Layer	Purpose	Kamailio Module
Digest Authentication	Authenticates each user via database	auth_db, auth
Pike	Rate limiting to prevent brute force	pike
Permissions (IP Blacklist)	Blocks known malicious IPs	permissions
User-Agent Filtering	Reject suspicious SIP devices	Custom logic
TLS	Encrypted signaling transport	tls
SRTP via RTPengine	Encrypted media between endpoints	rtpengine
3. Advanced Identification Mechanisms
Since client IP whitelisting is not practical (users from the internet):

Intermediate Method:

Require custom SIP header such as

X-Company-Signature: company123KEY

in SIP REGISTER and INVITE requests.

Advanced (Enterprise) Method:

Use Mutual TLS (mTLS) + Digest Authentication, where both server and client have certificates.

4. Centralized Monitoring & Logging
Central monitoring stack components:

Function	Tool	Detail
SIP Trace & Call Flow Visualization	Homer / HEP	Kamailio sends live SIP traffic to Homer via siptrace module
Log Aggregation	ELK Stack	Filebeat pushes /var/log/kamailio.log to ELK
Metrics and Statistics	Prometheus + Grafana	Kamailio exposes metrics via xhttp_prom
User Registration Tracking	REST API Hook	Kamailio uses exec() in REGISTER path to send events to external API
Call CDR Storage	ACC & DIALOG modules	Saves call duration, caller, callee in SQL or text format
5. Full Docker Architecture

content_copy
text
┌─────────────────────────── Docker Network: sbc_net ────────────────────────────┐
│                                                                                │
│  ┌────────────┐      ┌────────────┐       ┌────────────┐                       │
│  │ Asterisk_1 │<---->│ Kamailio  │<----->│  Clients   │                       │
│  │ 10.5.0.10  │      │ 10.5.0.2  │       │ @Internet  │                       │
│  ├────────────┤      │───────────│       └────────────┘                       │
│  │ Asterisk_2 │      │  RTPengine│                                             │
│  │ 10.5.0.11  │──────│ 10.5.0.4  │                                             │
│  │ Filebeat   │────► │ ELK Stack │                                             │
│  │ 10.5.0.6   │      │ 10.5.0.8  │                                             │
│  │ Homer HEP  │────► │ Collector │                                             │
│  │ 10.5.0.9   │      │  9060/udp │                                             │
│  └────────────┘      └────────────┘                                             │
└────────────────────────────────────────────────────────────────────────────────┘
6. Docker Compose File

content_copy
yaml
version: '3.8'

networks:
  sbc_net:
    ipam:
      config:
        - subnet: 10.5.0.0/24

services:

  kamailio:
    image: kamailio/kamailio:latest
    container_name: kamailio_sbc
    restart: always
    networks:
      sbc_net:
        ipv4_address: 10.5.0.2
    ports:
      - "5060:5060/udp"
      - "5060:5060/tcp"
      - "5061:5061/tls"
    volumes:
      - ./kamailio/kamailio.cfg:/etc/kamailio/kamailio.cfg
      - ./kamailio/dispatcher.list:/etc/kamailio/dispatcher.list
    environment:
      - DBURL=mysql://kamailio:pass@db/kamailio
    depends_on:
      - rtpengine

  asterisk1:
    image: andrius/asterisk:latest
    container_name: asterisk_1
    networks:
      sbc_net:
        ipv4_address: 10.5.0.10
    volumes:
      - ./asterisk/conf1:/etc/asterisk

  asterisk2:
    image: andrius/asterisk:latest
    container_name: asterisk_2
    networks:
      sbc_net:
        ipv4_address: 10.5.0.11
    volumes:
      - ./asterisk/conf2:/etc/asterisk

  rtpengine:
    image: sipwise/rtpengine:mr10.5
    container_name: rtpengine
    privileged: true
    networks:
      sbc_net:
        ipv4_address: 10.5.0.4
    command: >
      --interface=10.5.0.4
      --listen-ng=22222
      --log-level=6

  homer:
    image: sipcapture/heplify-server:latest
    container_name: homer
    networks:
      sbc_net:
        ipv4_address: 10.5.0.9
    ports:
      - "9060:9060/udp"
      - "9080:9080"
    environment:
      - HEP_SERVER_HEPLIFY=1
      - HOMER_PROMETHEUS_PORT=9096

  elk:
    image: sebp/elk:latest
    container_name: elk
    networks:
      sbc_net:
        ipv4_address: 10.5.0.8
    ports:
      - "5601:5601"
      - "9200:9200"
      - "5044:5044"
    volumes:
      - ./elk/logstash.conf:/etc/logstash/conf.d/logstash.conf

  filebeat:
    image: docker.elastic.co/beats/filebeat:7.17.0
    container_name: filebeat
    networks:
      sbc_net:
        ipv4_address: 10.5.0.6
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    networks:
      sbc_net:
        ipv4_address: 10.5.0.12
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    networks:
      sbc_net:
        ipv4_address: 10.5.0.13
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
7. Monitoring Components
Service	Port	Function
Homer	9080	View live SIP call flows, register messages
Kibana (ELK)	5601	Search and visualize Kamailio & Asterisk logs
Grafana	3000	View system-level and Kamailio metrics
Prometheus	9090	Collect metrics from Kamailio
Kamailio SBC	5060/5061	SIP signaling over TLS
RTPengine	22222	Media relay
Asterisk Nodes	Internal network only	SIP media handling
8. Security in Production
Best practices when deploying this Docker stack in production:

Use valid TLS certificates for Kamailio and Homer.
Restrict Kamailio’s exposed ports with firewall/Nginx reverse proxy.
Add HTTP Basic Auth for ELK/Kibana and Grafana dashboards.
Keep monitoring tools on a separate internal network (monitoring_net).
Rotate logs and use persistent volumes for CDR and database.
9. Launch Commands

content_copy
bash
cd /opt/sbc-stack
docker-compose up -d
Check logs:


content_copy
bash
docker logs kamailio_sbc
View systems:

Homer: http://localhost:9080
ELK Stack (Kibana): http://localhost:5601
Grafana: http://localhost:3000
Prometheus: http://localhost:9090
10. Architecture Diagrams
ASCII Diagram

content_copy
text
                    ┌────────────────────────────┐
                    │   Internet Users (SIP)    │
                    └───────────────┬────────────┘
                                    │
                            (TLS/UDP 5060)
                                    │
                        ┌────────────────────────────┐
                        │       Kamailio SBC        │
                        │   🔸 Auth + Pike + TLS     │
                        │   🔸 UA Filters            │
                        │   🔸 rtpengine proxy       │
                        │   🔸 Dispatcher LB         │
                        │   🔸 siptrace → Homer      │
                        │   🔸 logs → Filebeat → ELK │
                        │   🔸 xhttp_prom → Prometheus│
                        └──────────────┬─────────────┘
                                       │
           ┌───────────────────────────┼────────────────────────┐
           │                           │                        │
   ┌─────────────┐            ┌─────────────┐         ┌───────────┐
   │ Asterisk #1 │            │ Asterisk #2 │         │ RTPengine │
   │ 10.5.0.10   │            │ 10.5.0.11   │         │ 10.5.0.4  │
   └─────────────┘            └─────────────┘         └───────────┘

Kamailio exports events → Homer (HEP)
                logs → ELK
            metrics → Prometheus → Grafana
Mermaid Diagram
```

                    ┌────────────────────────────┐
                    │   Internet Users (SIP)    │
                    └───────────────┬────────────┘
                                    │
                            (TLS/UDP 5060)
                                    │
                        ┌────────────────────────────┐
                        │       Kamailio SBC        │
                        │   🔸 Auth + Pike + TLS     │
                        │   🔸 UA Filters            │
                        │   🔸 rtpengine proxy       │
                        │   🔸 Dispatcher LB         │
                        │   🔸 siptrace → Homer      │
                        │   🔸 logs → Filebeat → ELK │
                        │   🔸 xhttp_prom → Prometheus│
                        └──────────────┬─────────────┘
                                       │
           ┌───────────────────────────┼────────────────────────┐
           │                           │                        │
   ┌─────────────┐            ┌─────────────┐         ┌───────────┐
   │ Asterisk #1 │            │ Asterisk #2 │         │ RTPengine │
   │ 10.5.0.10   │            │ 10.5.0.11   │         │ 10.5.0.4  │
   └─────────────┘            └─────────────┘         └───────────┘

Kamailio exports events → Homer (HEP)
                logs → ELK
            metrics → Prometheus → Grafana
```
نمایش کد


11. Summary
✅ Functional Overview:

Kamailio = Secure SBC entry point.
Asterisk cluster = call control & voice features.
RTPengine = media relay / SRTP proxy.
Homer = live SIP traffic monitoring (HEP/UDP).
ELK = centralized textual logs.
Prometheus + Grafana = real-time metrics.
✅ Security:

Multi-layer (Digest + Pike + TLS + Permissions).
Central monitoring & analytics.
Fully containerized, isolated deployment.
✅ Deployment:

Everything launches automatically with:


content_copy
bash
docker-compose up -d
End of Document

© 2025 – Kamailio SBC Deployment Architecture (Prepared by GPT‑5)
