# DoK Operator Security & Hardening Guide

This directory is the home of materials for operator security and
hardening including the [DoKC Operator Security and Hardening Guide](guide/DoKc-Operator-Security-and-Hardening.md) 
is in this repo.

## Project goals and definition

Kubernetes operators are essential to manage cloud native databases. We
propose a guide for hardening database operators that will form a baseline
for secure management of data on Kubernetes.

The guide will identify common attack surfaces for databases running
on Kubernetes. It will define a set of best practices for securing them
using operators.

Best practices will cover a range of topics including software supply
chain, operator features, and integration with Kubernetes itself. They
may include any or all of the following.

* Required Kubernetes privileges
* Data in flight and at rest protection
* User management and handling of credentials
* Certificate management
* Secure exchange of data with database clients
* Secure exchange of data within clusters
* Backup/DR data protection
* Operator build process, e.g., signing, testing, scanning
* CVE management

It is our goal to keep the Operator Security & Hardening Guide simple and
practical. We will focus on operator security for databases only. Where
possible we will build upon rather than repeat existing best practices
for Kubernetes, cloud environments, etc.

This proposal is inspired by the well-known OWASP guidelines as well as
practical efforts for specific operators such as the ClickHouse Kubernetes
Operator and many others.

## Audience

* Operator producers - Developers creating or maintaining operators for data services
* Operator consumers - Developers seeking secure operators for data services

## How to contribute

Please join the #sig-operator channel on the DoKC Slack Workspace. You
can also file issues for problems. We also accept pull requests. Please
DM Robert Hodges in the DoKC Slack Workspace. 

## Contacts

Robert Hodges, Altinity 
