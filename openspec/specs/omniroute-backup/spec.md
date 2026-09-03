# omniroute-backup Specification

## Purpose

Provides an automated daily off-host backup of OmniRoute's SQLite database to a GitHub private repository, so the proxy's configuration and keys can be recovered if the VPS is lost.

## Requirements

### Requirement: Daily snapshot publication
The system SHALL run a backup worker daily. Each run SHALL select the newest consistent SQLite snapshot produced by OmniRoute's built-in backup mechanism and publish it to the backup repository as a single fixed-named file at the repository root.

#### Scenario: Successful daily backup
- **WHEN** the scheduled backup run executes and a snapshot file exists
- **THEN** the snapshot is copied to the repository as `omniroute-backup.sqlite`
- **AND** the change is committed and pushed to the backup repository

#### Scenario: No new snapshot available
- **WHEN** the backup run executes but no snapshot file is present
- **THEN** the run fails with a logged error
- **AND** the previous published backup is left untouched

### Requirement: Bounded backup history
The backup repository SHALL retain a single current snapshot in its working tree and SHALL periodically rewrite history so the repository size does not grow without bound.

#### Scenario: History rewrite
- **WHEN** the periodic rewrite runs
- **THEN** the repository history is collapsed to a single commit containing only the current snapshot
- **AND** the rewritten history is force-pushed to the remote

### Requirement: Non-disruptive operation
The backup worker SHALL operate without stopping, restarting, or reconfiguring the OmniRoute service. It SHALL read only the snapshot files produced by OmniRoute's own online backup mechanism.

#### Scenario: Service stays up during backup
- **WHEN** the backup worker runs
- **THEN** the OmniRoute service remains running and its traffic is unaffected

### Requirement: Authenticated private repository access
The system SHALL authenticate to the private backup repository using a read/write SSH deploy key. The private key material SHALL be provisioned by Ansible and SHALL NOT be committed to this repository in plaintext.

#### Scenario: Repository access
- **WHEN** the backup worker pushes to the repository
- **THEN** the SSH deploy key authenticates the push
- **AND** no credentials other than the deployed key are required

### Requirement: Missed-run recovery
The scheduler SHALL run the backup at a fixed daily time and SHALL execute a missed run after host downtime.

#### Scenario: Host was down at scheduled time
- **WHEN** the host was offline at the scheduled backup time and comes back up
- **THEN** a backup run is executed shortly after startup

### Requirement: Failure logging
Each backup run SHALL record its outcome (success or failure with reason) to the host's system journal.

#### Scenario: Failed backup is visible
- **WHEN** a backup run fails
- **THEN** the failure and its reason are recorded in the system journal
