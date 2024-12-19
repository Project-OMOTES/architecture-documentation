# Feature: Time series database cleanup

## Problem

Omotes 'forgets' about jobs when they are completed and the results have been retrieved.
Except for the time series data which are stored in a database, and the results are kept
indefinitely. This will cause the database to grow without an option to clean up, since omotes does
not know which data is still relevant.
Most front-ends will have an option to delete omotes runs which should also remove the time series
data. Currently, this is possible by directly accessing the times series database. This is a bit
cumbersome since every frontend will need to implement this, knowledge of the database structure is
needed, and it will be database type dependent.

## Proposal

If omotes deals with removing the time series data, only omotes code needs to be updated in the case
of:

- an update of the database structure
- use of a different type of time series database

The proposal is to create a `job_deletions` queue to which an SDK can publish. The orchestrator will
subscribe to this queue and remove the data from the time series database.  
Also on a `job_cancellations` message, the orchestrator should, after termination of the celery job,
remove any time series data, if present.

### Omotes-rest

A `/job/<job_id>/delete` endpoint will be added, similar to the `/job/<job_id>/cancel` endpoint.
