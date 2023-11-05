# Data on Kubernetes Database Operator Security and Hardening Guide

Version 0.1.0 - 5 November 2023

## Introduction

Among the many valuable workloads on Kubernetes, databases hold pride of
place. Kubernetes orchestrates compute, storage, and networking to run
container-based applications. Modern databases are complex, distributed
applications that map well to the solution Kubernetes provides. Most
databases that can be containerized–from transaction processing to
NoSQL to analytic systems–now run on Kubernetes.

The [Operator pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
has played a key role in this success. Database
operators are Kubernetes extensions that reduce database configuration
to a single [custom resource definition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) 
mapped to Kubernetes resources. Operators take care of change management,
for example database upgrade, and also enable integration of external
functions like backup. Without operators, Kubernetes would be a far less
hospitable environment for database operation.

Database operators are thus the natural vehicle to implement security
protections on databases and the information they contain. This guide
provides principles, a threat model, and most importantly a list of
practical features that operators can and should provide to protect
data effectively.

The goal of this guide is to define baseline operator features that
simplify security management for databases in the same way that operators
have simplified database operation in general. We hope in this way
to improve the security of all databases that run on Kubernetes, and
simplify Kubernetes security as a whole.

## Current State of the Guide

This document is an initial release to provide quick guidance to users of
operators and stimulate further discussion with the Kubernetes community
as a whole. Eventually we hope the guide will expand to provide additional
guidance that integrates with the following:

* The overall Kubernetes security model
* Software supply chain security model(s)
* Global authentication and authorization mechanisms aka single sign-on

## Principles

The approach to operator security is guided by a set of principles
that draw on robust practices from the fields of database management,
Kubernetes, and cloud native computing.

### Security by default

The best security starts with default settings that provide out-of-box
security for data. Examples include providing passwords for default
logins and restricting network access to such accounts. Operators should
come with settings that protect data automatically but can be relaxed
in easy-to-understand, visible ways when operating in development or
other environments.

### Best practice database administration

Databases have a long history of protecting data using mechanisms
ranging from in-flight and at-rest encryption to role-based access
control. Operators should provide easy, well-documented mechanisms to
implement these practices.

### Integration with Kubernetes security model

Kubernetes has a well-developed security model as well as a rich
set of resources to implement that model. Database operator security
should as far as possible build on existing resources like Secrets
and adopt new ones (like those for implementing zero-trust security)
as they stabilize. Moreover, database operators should ensure their
security features conform to practices of Kubernetes as a whole, such
as following the principle of least privilege in service accounts and
leaning on security mechanisms like IAM policies that tie into underlying
cloud security models.

### Adherence to principles of cloud native applications

Cloud native computing provides a set of guidelines for building
scalable systems using containers, microservices, DevOps, and continuous
integration/continuous deployment (CI/CD). Database operators must
integrate well with the practices used to implement and deploy cloud
native applications. It’s especially important for operators to support
modern GitOps and infrastructure-as-code practices, in which Kubernetes
applications are deployed from configurations stored in Git repositories
located in GitHub and GitLab.

### Auditability and monitoring

The best security regime makes security problems as visible as possible as
early as possible. It also provides a clear history of actions that affect
security as well as the actors who performed them. Database operators
should install observability tools and ensure there are rich audit logs.

### Dynamic response to security threats

Users must respond quickly to security incidents as well as knowledge of
critical CVEs. Database operators should make it easy to change security
configuration safely and efficiently in response to such situations. This
includes deploying new versions of the operator itself. Moreover,
database operators themselves must be developed and released in a way
that allows them to adopt new security practices efficiently and make
them available to users.

## Threat Model

For database operators to protect data effectively we need a clear
threat model. The following threat model factors database management
on Kubernetes into three overlapping problems, as illustrated in the
following diagram.

[Kubernetes Database Operator Threat Model](database-operator-threat-model.png))

### Protect the database

We start by concentrating on features related to the database itself,
which means concentrating on the Kubernetes resources that implement
it. Database protection comprises topics like user accounts, in-flight
data (network), at-rest data (storage), intra-cluster communication,
and the like. It also includes appropriate use and monitoring of every
Kubernetes resource that the database operator manages.

Operators should manage Kubernetes resources with a light touch that
does not obscure security features built into resources like Services,
Persistent Volume Claims, Pods, or Secrets. Where possible users should
be able to invoke those features directly, for example to configure
Services to set specialized network properties.

### Protect Kubernetes

The second layer includes protecting the Kubernetes cluster. This
principally consists of following best practices for cluster management,
for example by using secure, managed Kubernetes platforms like AWS EKS
or Google GKE. However, it also means that database operators should
not use mechanisms that can be used to attack Kubernetes itself, such
as requiring administrator access in database operator service accounts
or allowing databases to become attack vectors to Kubernetes as well as
other applications running on Kubernetes.  ### Protect external data

Modern databases integrate with and export data to an increasing number
of external data stores. These may be in other namespaces of Kubernetes
but often live in different cloud accounts that are controlled by third
parties. Here are some common examples.

* Data sources - Databases commonly integrate with Kafka or similar upstream
data sources. The database may hold credentials that can become attack
vectors.

* Log management services - It is common to ship message logs to
ElasticSearch, DataDog, or other third party services. Such messages
can leak database user logins and user data.

* Table data in object storage - Many databases store and read table
data from object storage. They may hold credentials, which are potential
attack vectors. Attackers can also bypass database protections by reading
data directly in object storage.

* Backups - Backups are by necessity stored in external locations. If
an attacker obtains access to backup data, there’s no need to attack
the database.

External data represents the most diffuse threat and varies widely
across database types. Database operators should avoid becoming a part
of the problem by creating new attack vectors. They should also filter
or encrypt exported data in order to protect it from outside attacks.

Finally, database operators should help users avoid unnecessary export
of data in the first place. Solving the problem by not having it is a
time-honored engineering approach that has special relevance in this case.

## Best Practices

The sections that follow describe best practices for data protection
in summary form. Each section offers a context for the practices, which
are presented in tabular form.

(Note: This is in checklist format for now–we can investigate ways
to dig more deeply into example code or add more context without adding
bloat to this guide.)

## Database Management

#### User Management

All databases have built-in user management. Default accounts are a
special area of risk for initial setup, along with passwords.

| Key | Name | Description |
| ----- | ----- | ----- |
| UM-1 | Default User Password | Automatically generate passwords for default logins. |
| UM-2 | Default User Network | Automatically restrict default logins to localhost or IP addresses within the cluster. |
| UM-3 | Prevent Insecure Passwords | Disable storage of plain text user passwords (if relevant) and enable strong password policies.  |
| UM-4 | Local User Accounts | Provide built-in mechanisms to define and manage local user accounts securely. This can include deferring to database RBAC mechanisms. |
| UM-5 | Authentication Provider Support | Provide means to configure LDAP or other external authentication providers, where applicable. Note: this belongs to a deeper issue of single sign-on integration. |
| UM-6 | Intra-Cluster Authentication | Automatically configure authentication between database cluster nodes through shared secrets or other secure mechanisms. |
| UM-7 | Secure Local User Storage | Ensure that locally defined accounts are stored securely on the file system. |

#### Credential and Key Management

Databases depend on credentials both for internal users as well as access
to external services. Operators should seek to eliminate use of explicit
credentials wherever possible and protect the remainder carefully.

| Key | Name | Description |
| ----- | ----- | ----- |
| CM-1 | Secure Key Management | Support integration with secure key management services like AWS KMS or Hashicorp Vault.|
| CM-2 | Database  Account Credentials | Use secrets to transfer database credentialsEven better, use secrets to define database accounts fully so that the accounts themselves are not externally visible. |
| CM-3 | External Credentials | Use secrets to transfer credentials for external services.|
| CM-4 | Pod Integration | Allow users to specify Kubernetes pod properties directly, so that they can extend operator behavior and use custom methods to inject secrets into pods. |
| CM-5 | Cloud Resource Access | Enable use of Kubernetes service accounts instead of explicit credentials to access cloud resources. Example: use IAM to access object storage.  |

#### Network Management

Protection of in-flight data is one of the most important responsibilities
of any database server. Operators can help in a number of key
ways. Recommended policies apply to both IPv4 and IPv6.

| Key | Name | Description |
| ----- | ----- | ----- |
| NM-1 | Secure Ports | Disable insecure network ports and wire protocols. For example, HTTP access can be disabled in favor of HTTPS. |
| NM-2 | IP Address Filtering | Allow configuration of IP-based whitelisting/blacklisting mechanisms for specific database logins.  |
| NM-3 | Server Certificate Verification | Force certificate verification on all encrypted communications from the database to other databases, ZooKeeper, event streams, etc.   |
| NM-4 | Encrypted Client Connections | Configure best practice TLS-based encryption of database client connections including acceptable TLS versions, ciphers, and mTLS (mutual authentication based on client and server certificates). |
| NM-5 | Encrypted Intra-cluster Communications | Same as above but for intra-database communication like replication and distributed queries. |
| NM-6 | Encrypted External Service Communications | Same as above but for communication to external services like Kafka. |
| NM-7 | Server X509 Certificate and Keys | Install, rotate, and remove database X509 certificates and keys. |
| NM-8 | Root X509 Certificates | Install, rotate, and remove root CA certificates. This is necessary to enable local certificate authorities, a common security practice in data centers.  |
| NM-9 | Chained certificates | Support chained certificates in which the database server cert is signed by an intermediate certificate instead of the root CA certificate. |
| NM-10 | FIPS Compatibility | Where available, allow users to enable FIPS-compatible crypto for network communications.  |
| NM-11 | Service Integration | Allow users to specify Kubernetes service properties directly so that they can control (for example) Ingress properties implemented by Kubernetes provider. |

### Storage Management

Protection of data in storage is another key responsibility of databases.

| Key | Name | Description |
| ----- | ----- | ----- |
| SM-1 | Disk Encryption | Automatically enable encryption of storage, example using built-in encryption provided by CSI storage driver.   |
| SM-2 | Application Level Encryption | Allow users to enable built-in storage encryption supplied by the database, if available. |
| SM-3 | Database Key Management | Store keys for database level encryption in secure key management services like AWS KMS or Hashicorp Vault, so that they cannot be lost or compromised.  |
| SM-4 | Persistent Volume Claim Integration | Allow users to specify Persistent Volume Claim properties directly so they can take advantage of additional storage class features.  |
| SM-5 | Storage Class Integration | Allow users to select storage classes directly so they can implement and use custom storage provisioning policies. |

#### External Data

External data comes in many forms. The specific protections here are
suggestions that should be applied where appropriate. The best guideline
is to use common sense and try not to forget anything obvious.

| Key | Name | Description |
| ED-1 | Encrypted Backups | Encrypt backups using keys stored in a secure key management facility.  |
| ED-2 | Log Message Filtering | Prevent export of log messages that contain sensitive information such as credentials or user data. This problem can also be solved by enabling local storage of data.  |
| ED-3 | Table data | Encrypt exported table data using keys stored in a security key management facility.  |

#### Monitoring, Alerting, and Auditing

Many exploits against databases show up as changes in database
behavior, such as a jump in CPU load average due to parasitic crypto
mining. Monitoring and alerts help identify exploits quickly. Auditing
preserves the history of operations that affect security.

| Key | Name | Description |
| MAA-1 | Database Monitoring | Provide comprehensive Day 2 database monitoring using standard tools like Prometheus for metric storage and Grafana for operational dashboards.  |
| MAA-2 | Default Security Alerts | Provide alerts on security-related events like login failures.  |
| MAA-3 | Default Audit Logging | Turn on available audit logs by default. Set other logs to the minimum level needed to diagnose security issues.  |

#### Kubernetes Security Integration

Database operators need to build on Kubernetes and avoid creating new
loopholes.

| Key | Name | Description |
| KSI-1 | No Cluster Admin Access | Default to running operator tasks without access to Cluster Admin role. This includes installation which should be possible to do in a single namespace. Cluster role binding–if necessary–should be explicitly enabled by the user.    |
| KSI-2 | Namespace Scoping | Restrict operator scope to a single name space by default. Add mechanisms to explicitly expand scope to a list of namespaces or Kubernetes cluster as a whole.  |
| KSI-3 | Service Accounts | Restrict service account(s) for operators to a single namespace and minimize granted privileges.  |
| KSI-4 | IAM Policy Integration | Allow database pods to integrate with provider IAM policies instead of using explicit credentials.  |
| KSI-5 | External Network Access | Ensure database operators do not silently expose Kubernetes network externally. For example, database operators should not create load balancers with public Internet access unless explicitly configured by the operator user.  |
| KSI-6 | Safe Client Calls to Network | Disable defaults that allow database users to use database operations to access IP addresses outside the database cluster itself. These capabilities should be explicitly enabled. Examples include the ability to use SQL to make HTTP requests to random hosts. |

#### Cloud Native Development and CI/CD Integration

Database operators and the database they manage are typically just
one part of the user application stack. Best practice cloud native
development uses GitOps combined with infrastructure-as-code to deploy
Kubernetes applications in a controlled, repeatable way. This practice is
also the backbone of security management. Specific features are needed
to allow GitOps using tools like ArgoCD and Terraform to integrate
database operators.
| Key | Name | Description |
| CN-1 | Helm Chart Install | Provide an accessible helm chart for the operator in an accessible public helm repo with documented properties and without explicit namespace labels.  |
| CN-2 | Manifest Install | Provide an accessible manifest-based install with properties defined in forms that can be changed using Kustomize and without namespace labels.  |
| CN-3 | Credential Encryption | Use general mechanisms for credentials such as Secrets that can be encrypted on check in to Git and decrypted for use inside Kubernetes |

#### Software Supply Chain

Database operators run as containers in Kubernetes and pull additional
containers to set up and manage databases. There are a number of important
features related to software supply chain management for operators and
their dependent software.

Note: Supply chain security is a deep subject. It’s worth considering
whether we should include discussion of specifications like SLSA
(Supply-chain Levels for Software Artifacts) here.

| Key | Name | Description
| SSC-1 | Operator Security Test | Operates should include a regression test with cases for each security feature recommended by this guide. 
| SSC-2 | Version Management | Operator releases should adopt a versioning scheme and release management practices to ensure that users only pull fully tested, production-ready software versions. 
| SSC-3 | Operator CVE Scanning | Operators and any sidecar container images that they depend on should be scanned using tools like Trivy and/or Docker Scout before release. All HIGH CVEs should be eliminated prior to release. 
| SSC-4 | Database CVE Scanning | Where appropriate, the operator should lean on database CVE management and only deploy containers that are free of HIGH CVEs. 
| SSC-5 | CVE Tracking and Response | Operators should release new versions to correct CVEs are found in the operator code or dependent images, including database containers.
| SSC-6 | Responsible Exploit Reporting | Database operator projects should provide a way to report new exploits securely. |

#### Documentation

You cannot secure what you do not understand. Readable, up-to-date
documentation is key to enabling secure use of database operators.

| Key | Name | Description |
| D-1 | Operator Doc Set | The database operator should have up-to-date documentation that includes an introductory tutorial, examples of use, and a reference guide that describes all settings.  |
| D-2 | Hardening Guide | The operator should have a comprehensive hardening guide that covers all configurations related to security including assumptions for secure use, an explanation of security-related features with examples, and a list of any features in this guide that are not covered.  |
| D-3 | Release Notes | New operator releases should include release notes that document any changes that affect security including notes on new CVEs.  |

## Editor 

* Robert Hodges

## Acknowledgments

* Alexander Zaitsev 
* Edith Pucilla 
* Victor Lu

## Further Reading

* [Kubernetes Documentation: Overview of Cloud Native Security](https://kubernetes.io/docs/concepts/security/overview/)
* [CNCF TAG Security: Cloud Native Security Whitepaper](https://github.com/cncf/tag-security/blob/main/security-whitepaper/v2/cloud-native-security-whitepaper.md)
* [CNCF TAG:  Security Zero Trust using Cloud Native Platforms (Review)](https://docs.google.com/document/d/10g2390JdCBXmSmzQ_EGHFWrg2JosPsXLaqXaGQ-B9NA/edit )
* [SLSA: Supply-chain Levels for Software Artifacts](https://slsa.dev/spec/v1.0/)
* [Altinity Kuberenetes Operator for ClickHouse Hardening Guide](https://github.com/Altinity/clickhouse-operator/blob/master/docs/security_hardening.md)
* [CloudNativePG Operator Security Documentation](https://cloudnative-pg.io/documentation/1.21/security/ )
