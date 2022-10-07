# Policy
## Executive Summary
This document defines Graylog Cloud backup and recovery systems. This is including but not limited to: retention definitions, alerting operations, security and encryption, data availability guarantees, recovery point and recovery time objectives, and the technical details of how the systems work.  It is both the "why" and the "what" for Graylog Cloud's backup and recovery.

This policy is designed to protect data within Graylog Cloud to be sure it is not lost, and can be recovered in situations such as:
* equipment and service provider failures
* intentional or accidental destruction of data
* forced migration
* other scenarios leading to data loss or corruption

## Scope
This policy applies to all data and configuration within the Graylog Cloud service residing within systems owned/leased and operated by Graylog, Inc., as would be required to completely reconstruct any or all customers' services and data in the covered situations.

## Requirements
### General
These requirements are default least restrictive. They can be overriden by stricter requirements given to specific components of Graylog Cloud. They can also be overridden for specific customers or classes of customers with stricter requirements as would be fully documented in those customer agreements and contracts.

The supported backup and recovery scenarios listed below must be tested weekly.

### Availability
Data must be recoverable during an AWS Availability Zone (AZ) outage
Data must be recoverable during an AWS full region outage (e.g. us-east-1)

### Security
Data in backups must be subject to same or stricter access controls as production data
Data in backups must be encrypted in transit and at rest

### Data Recovery Scenarios Supported
- Full data recovery of either individual or multiple customers, replacing into their existing running service(s) and account(s).
  - e.g. for the purposes of emergency recovery after accidental data deletion, deprovisioning, corruption
- Full data recovery of either individual or multiple customers, into separate service(s) and account(s).
  - e.g. for the purposes of data migration, out of band data analysis, testing, or temporary partial data access outside of existing running service.

### Retention
Data must be available for the full time that a customer retains data with us. For example, if a customer's service agreement includes 90 days of log retention, we must retain functional backups covering 90 days into the past.

### Recovery Time Objective
In the worst case scenario, a customer's full data set and configuration must be able to be recovered and back online within 12 hours. (12 hour max recovery downtime)
### Recovery Point Objective
Any recovery of customer data must include all data up to 30 minutes prior to service outage. (30 minutes max recovery data loss)
### Alerting
Responsible engineering staff must be presented with emergency alerts in any of the following situations that pertain to this policy:
- Failed backups causing a violation of the above recovery point objective
- Failed periodic recovery tests

# Technical Documentation
## Backups
### Overview
### AWS OpenSearch Backend
#### Automatic OpenSearch Snapshots
#### Manual OpenSearch Snapshots
### MongoDB
## Recovery
### Overview
### AWS OpenSearch Backend
### MongoDB
## Alerting
### Overview
### Prometheus Monitoring and Metrics
### Alerts
