# Day 5 - Advanced topics: Monitoring, Security

Welcome to Day 5. This day focuses on two production-critical concerns: **observability** and **security**.

## What you will be able to do by the end

- Explain the difference between probes and monitoring, and the RED signals: rate, errors, duration.
- Instrument a Python service with a counter, a gauge and a histogram, and keep the label cardinality low.
- Make Prometheus scrape a service with a ServiceMonitor.
- Write PromQL queries for rate, error ratio and p95 latency, and use them in Grafana.
- Read a shared Grafana dashboard for your namespace, and see an outage on it.
- Explain the difference between authentication and authorisation, and between `401` and `403`.
- Explain the idea of public key cryptography (key pairs, signatures, key exchange), what TLS protects and what it does not, and read a certificate and its chain of trust.
- Follow cert-manager while it gets a certificate from Let's Encrypt, and explain renewal and why you never re-create a certificate to "try again".
- Explain what a Traefik middleware is, and protect an HTTPS endpoint with HTTP Basic Auth in Traefik, without changing the application.
- Explain what OpenID Connect and signed tokens (JWT) add to passwords, and read the claims of a token.

## Sections

| # | Section | What it covers |
|---|---|---|
| 1 | [Monitoring with Prometheus, Grafana, and Python SDK](01-monitoring-prometheus-grafana-python-sdk.md) | RED signals, metric types, PromQL, cardinality, ServiceMonitor, Grafana dashboards; three hands-ons |
| 2 | [Security fundamentals for cloud-native workloads](02-security-fundamentals.md) | Threat model, authentication and authorisation, public key cryptography, HTTPS and TLS, cert-manager and Let's Encrypt step by step, Traefik middlewares and Basic Auth, OpenID Connect and JWT (theory); two hands-ons |
| 3 | [Group project: package and serve a model](03-group-project.md) | Use the knowledge from all five days: package a machine learning model and serve it in the cloud, in groups |


