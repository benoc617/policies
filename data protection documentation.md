# Policy
## Executive Summary
This document defines Graylog Cloud backup and recovery systems. This is including but not limited to: retention definitions, alerting operations, security and encryption, data availability guarantees, recovery point and recovery time objectives, and the technical details of how the systems work.  It is both the "why" and the "what" for Graylog Cloud's backup and recovery.

This policy is designed to protect data within Graylog Cloud to be sure it is backed up, retained, and can be recovered in situations such as:
* equipment and service provider failures
* intentional or accidental destruction of data
* forced migration
* other scenarios leading to data loss or corruption

## Scope
This policy applies to all data and configuration within the Graylog Cloud service residing within systems owned/leased and operated by Graylog, Inc., as would be required to completely reconstruct any or all customers' services and data in the covered situations outlined below.

## Requirements
### General
These requirements are default least restrictive. They can be overriden by stricter requirements given to specific components of Graylog Cloud. They can also be overridden for specific customers or classes of customers with stricter requirements as would be fully documented in those customer agreements and contracts.

The supported backup and recovery scenarios listed below must be tested weekly.

### Availability
- Log and configuration data must be recoverable during an AWS Availability Zone (AZ) outage
- Log data must be recoverable during an AWS full region outage (e.g. us-east-1)

### Security
- All log data and configuration data in backups must be subject to same or stricter access controls as production data
- All log data in backups must be encrypted in transit and at rest

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
Any recovery of customer data must include all data and configuration up to 30 minutes prior to service outage. (30 minutes max recovery data loss)

### Alerting
Responsible engineering staff must be presented with emergency alerts in any of the following situations that pertain to this policy:
- Failed backups causing a violation of the above recovery point objective
- Failed periodic recovery tests

# Technical Documentation
## Backups
### Overview
In order to restore a customer's hosted system in the case of a disaster, we potentially need to reconstruct and restore the AWS OpenSearch Service which contains all retained and indexed log data, as well as the MongoDB database cluster which contains the customer's graylog configuration
### AWS OpenSearch Backend
The following AWS document outlines the functionality of AWS OpenSearch snapshots:  
https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-snapshots.html

Note: the creation of snapshots do not interrupt operations on the OpenSearch cluster, but do slightly adversely affect performance, and we have factored in this required overhead in the sizing of these clusters.
#### Automatic OpenSearch Snapshots
AWS OpenSearch Service automatically takes incremental snapshot series every hour via internal mechanisms. It only retains them for 14 days, and stores them in a single-region S3 bucket that is not generally accessible except by the OpenSearch Service's internal and proprietary restore mechanisms (which we will go over below, and are outlined in the AWS documentation linked above).

Since these backups do not meet our RPO due to their low frequency, they can only be used for recoveries and/or parallel restores where it is known that new data has not been ingested in the past hour, or only data older than one hour is required (e.g. a parallel restore to analyze or compare historical data).
#### Scheduled Manual OpenSearch Snapshots
In addition to the automatic snapshots, we have manually-defined snapshots scheduled to execute on all OpenSearch Service clusters. These snapshots are created every 15 minutes and store to an S3 bucket per customer that is replicated to an alternate AWS region.  e.g. if customer is provisioned in us-east-1, the customer's snapshot bucket will be replicated between us-east-1 and us-west-1. These manual snapshots are retained for 90 days default, and longer for customers that have longer general data retention (as per above retention requirement).

These backups are generally the main source of data for customer recoveries and parallel restores, since they satisfy our 30 minute RPO.  
### MongoDB
We make use of Mongo Cloud Manager to execute scheduled snapshots of each customer's MongoDB cluster as documented here:
https://www.mongodb.com/docs/cloud-manager/tutorial/nav/backup-use/

The Mongo Cloud Manager is configured to execute: 
- a full snapshot backup weekly
- an incremental snapshot every 6 hours
- a checkpoint every 15 minutes between snapshots 

These are retained for 90 days default, and longer for customers that have longer general data retention (as per above retention requirement).

## Restoring
### Overview
In a recovery scenario, the customer's existing data stores would be re-provisioned, and restoration of AWS OpenSearch and MongoDB would be done in-place.

In a parallel restore scenario, a new set of data stores and service would be provisioned, and restoration of AWS OpenSearch and MongoDB would be done into those new data stores.

### AWS OpenSearch Backend
AWS documentation on restoring snapshots:  https://docs.aws.amazon.com/opensearch-service/latest/developerguide/managedomains-snapshots.html#managedomains-snapshot-restore

For all recoveries or parallel restores, we restore all indices from the scheduled manual OpenSearch snapshot at the desired recovery point. Restoring from the automatic OpenSearch snapshot is an option as long as a potentially one-hour older recovery point is acceptable.

`opensearchrestore` is a script to perform snapshot restoration. It accepts, at a minimum, the OpenSearch domain endpoint, snapshot repository name, and snapshot name as parameters.  

### MongoDB
Mongo Cloud Manager documentation on restoring snapshots:  https://www.mongodb.com/docs/cloud-manager/tutorial/nav/backup-restore-deployments/

For all recoveries or parallel restores, we restore service configuration into MongoDB by using the Mongo Cloud Manager and documentation linked above.

`mongodbrestore` is a script to perform MongoDB restoration using the Mongo Cloud Manager API.  It accepts, at a minimum, the snapshot ID and the MongoDB cluster ID to restore the data into as parameters.

## Testing
We maintain multiple provisioned test customers that receive and index test log flow continuously. On a weekly schedule, a parallel restore of one test account, and a recovery of another test account will run, and both restored test instances will be brought online automatically.  Once online, smoke tests will be run automatically to ensure the test instances are running, and the restores were successful according to some limited data comparison, and index size/count expectations.

If any part of these automated restores and verifications fail, an alert will be sent to responsible engineers to repair the issue, re-run all backups if necessary, and re-test restores.

In addition to full restore testing, we execute backup check scripts hourly for each customer. This backup check verifies that OpenSearch and MongoDB snapshots are present for the entire retention period as expected for the given customer, and that ISM (or Curator) for OpenSearch and Mongo Cloud Manager for MongoDB are configured according to specification.

If any error is determined in configuration, or snapshots are not present for the required range, an alert will be sent to responsible engineers to repair the issue, re-run all backups if necessary, and re-run the check script.

## Alerting
### Prometheus Monitoring and Metrics
Prometheus is a time series database (TSDB) that gathers metrics and telemetry from our graylog servers, MongoDB clusters, and AWS OpenSearch cluster via standard prometheus exporters.

This allows us to monitor critical infrastructure metrics, such as disk space, cpu saturation, server reachability, loadbalancer health, etc.  It also allows us to monitor critical service metrics, such as JVM garbage collection rates, heap utilization, server performance, log throughput, OpenSearch indexing throughput and health, etc.

### Alerts
Within prometheus, we have several alerts defined based on the above metrics, and these forward to a pager rotation service to reach engineers via prometheus alertmanager.

One that pertains specifically to this data protection policy is the health of AWS OpenSearch. When the AWS OpenSearch cluster is `RED` an alert will be sent to responsible engineers to repair the issue, and re-run all backups if necessary, and verify backups are functioning. When OpenSearch clusters are not healthy, snapshots do not run, and it puts us at risk of missing our RPO of 30 minutes.

In addition, the weekly automated backup and restore tests, and backup configuration and retention check, will also send alerts to prometheus alertmanager if they encounter any failures or inconsistencies.

