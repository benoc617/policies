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
- Reccovery:  Full data recovery of either individual or multiple customers, replacing into their existing running service(s) and account(s).
  - e.g. for the purposes of emergency recovery after accidental data deletion, deprovisioning, corruption
- Parallel Restore:  Full data recovery of either individual or multiple customers, into separate service(s) and account(s).
  - e.g. for the purposes of data migration, out of band data analysis, testing, or temporary partial data access outside of existing running service.

### Retention
Data must be available for the full time that a customer retains data with us. For example, if a customer's service agreement includes 90 days of log retention, we must retain functional backups covering 90 days into the past.

### Recovery Time Objective (RTO)
In the worst case scenario, a customer's full data set and configuration must be able to be recovered and back online within 12 hours. (12 hour max recovery downtime)
### Recovery Point Objective (RPO)
Any recovery of customer data must include all data up to 30 minutes prior to service outage. (30 minutes max recovery data loss)
### Alerting
Responsible engineering staff must be presented with emergency alerts in any of the following situations that pertain to this policy:
- Failed backups causing a violation of the above recovery point objective
- Failed periodic recovery tests

# Technical Documentation
## Backups
### Overview
In order to restore a customer's hosted system in the case of a disaster, we need to reconstruct and restore the AWS OpenSearch Service which contains all retained and indexed log data, as well as the MongoDB database cluster which contains the service configuration and graylog index data
### AWS OpenSearch Backend
The following AWS document outlines the functionality of AWS OpenSearch snapshots:  
https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-snapshots.html

The creation of snapshots do not interrupt operations on the OpenSearch cluster, but do slightly adversely affect performance, and we have factored in this required overhead in the sizing of these clusters.
#### Automatic OpenSearch Snapshots
AWS OpenSearch Service automatically takes incremental snapshot series every hour via internal mechanisms. It only retains them for 14 days, and stores them in a single-region S3 bucket that is not generally accessible except by the OpenSearch Service's internal and proprietary restore mechanisms (which we will go over below, and are outlined in the AWS documentation linked above).

Since these backups do not meet our RPO due to their low frequency, they can only be used for recoveries and/or parallel restores where it is known that new data has not been ingested in the past hour, or only data older than one hour is required (e.g. a parallel restore to analyze or compare historical data).
#### Manual OpenSearch Snapshots
In addition to the automatic snapshots, we have manual snapshots scheduled to execute on all OpenSearch Service clusters. These snapshots are created every 15 minutes and store to an S3 bucket per customer that is replicated to an alternate AWS region.  e.g. if customer is provisioned in us-east-1, the customer's snapshot bucket will be replicated between us-east-1 and us-west-1. These manual snapshots are retained for 90 days minimum, and longer for customers that have longer general data retention (as per above retention requirement).

These backups are generally the main source of data for customer recoveries and parallel restores, since they satisfy our 30 minute RPO.  
### MongoDB
## Recovery
### Overview
### AWS OpenSearch Backend
### MongoDB
## Alerting
### Overview
### Prometheus Monitoring and Metrics
### Alerts
