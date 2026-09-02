# Backup and Data Protection

The server stores persistent application data on a dedicated SSD. To protect this data against hardware failures and accidental data loss, a second, larger HDD is used as a dedicated backup target.

Backups are performed automatically once per day during a nightly backup process using Restic.

The backup mechanism operates independently from the Docker application stacks. This separates data protection from the availability of the individual services.

## Storage Architecture

The storage setup consists of two external drives with different purposes:

| Storage | Purpose                 | Usage          |
| ------- | ----------------------- | -------------- |
| SSD     | Active application data | Continuous     |
| HDD     | Backup storage          | Nightly backup |

The SSD is used for the live application data of the self-hosted services.

The HDD is dedicated primarily to backups and is normally inactive outside the backup window. This reduces unnecessary operating time of the backup drive.

Conceptually:

```text
                    Server

                      │

                      │
              ┌───────▼───────┐
              │   Active SSD  │
              │               │
              │ Application   │
              │     Data      │
              └───────┬───────┘
                      │
                      │ Nightly backup
                      ▼
                ┌───────────┐
                │   Restic  │
                └─────┬─────┘
                      │
                      ▼
              ┌───────────────┐
              │   Backup HDD  │
              │               │
              │    Restic     │
              │   Repository  │
              └───────────────┘
```

## Restic

Restic is used as the backup tool.

The backup process runs automatically once per day and creates a Restic snapshot containing the selected persistent application data on the dedicated backup storage.

The backup process is separate from the Docker services themselves. A failure or restart of an individual application container therefore does not directly prevent the host-level backup process from operating.

The backup configuration defines:

* the directories to back up
* excluded data
* the Restic repository
* the required Restic credentials
* retention and pruning rules
* logging

The current backup script uses Restic retention policies to retain a defined number of daily, weekly, and monthly snapshots.

Sensitive credentials and the live backup configuration are kept outside the public repository.

The repository contains only sanitized examples of the backup configuration.

## Backup Scope

The backup is intended to protect the persistent data required to restore the self-hosted applications.

This primarily includes application data stored on the active SSD, such as:

* application configuration
* databases
* uploaded files
* user-generated content
* other persistent application state

The Docker containers themselves are not considered the primary backup target. Containers can generally be recreated from their respective images and configuration, while persistent application data requires explicit protection.

This distinction can be summarized as:

```text
Container / Image
       │
       │ Recreate
       ▼
   Application
       │
       │ Uses
       ▼
Persistent Data
       │
       │ Restore
       ▼
Backup Repository
```

The combination of reproducible container images, documented configuration, and backed-up persistent data provides the basis for rebuilding the application environment after a failure.

## Backup Schedule

A backup is performed once per day during the nightly backup process.

The general workflow is:

```text
1. Application data

        │
        ▼

2. Start nightly backup

        │
        ▼

3. Restic reads selected data

        │
        ▼

4. Create / update Restic snapshot

        │
        ▼

5. Apply retention policy and prune old snapshots

        │
        ▼

6. Record backup status

        │
        ▼

7. Backup storage becomes inactive
```

The backup drive is not intended to operate continuously.

The backup script also records the outcome of the backup process so that failed backup runs can be identified.

## Data Flow

The current data-protection architecture can be summarized as:

```text
                    ┌────────────────────┐
                    │   Hosted Services  │
                    └─────────┬──────────┘
                              │
                              ▼
                    ┌────────────────────┐
                    │     Active SSD     │
                    │                    │
                    │ Persistent Data    │
                    └─────────┬──────────┘
                              │
                         Nightly
                         Restic
                              │
                              ▼
                    ┌────────────────────┐
                    │     Backup HDD     │
                    │                    │
                    │ Restic Repository  │
                    └────────────────────┘
```

## Failure Scenarios

The backup strategy is primarily intended to protect against failures affecting the active storage or the logical state of application data.

### Active SSD Failure

If the active SSD fails, the application data can be restored from the Restic repository.

The expected recovery process is conceptually:

```text
Failed SSD

    │
    ▼

Replace / repair storage

    │
    ▼

Recreate application environment

    │
    ▼

Restore persistent data

    │
    ▼

Resume service operation
```

The exact recovery procedure depends on the affected application and its configuration.

### Accidental Data Deletion

If important data is accidentally deleted from the active storage, a previous Restic snapshot can provide a recovery point.

The exact recovery point depends on the available snapshots and the configured retention policy.

### Server Failure

If the server itself fails while the backup HDD remains intact, the Restic repository can be used as the source for restoring persistent application data to a replacement system.

The application containers can then be recreated using the required container images and documented configuration.

This is one reason the infrastructure configuration and architecture are documented separately from the live server.

## Important Limitation: Local Backup

The current backup system is **not an off-site backup**.

Both the server and backup HDD are located at the same physical location.

This means that a single physical event could affect both copies simultaneously.

Examples include:

* fire
* theft
* major water damage
* other events causing loss of the physical infrastructure

The current architecture therefore provides protection against many forms of logical and hardware failure, but not against complete physical loss of the home infrastructure.

```text
              Same Physical Location
              ───────────────────────

        ┌──────────────┐
        │    Server    │
        │    + SSD     │
        └──────┬───────┘
               │
               │ Backup
               ▼
        ┌──────────────┐
        │   Backup HDD │
        └──────────────┘

                 │
                 │
          Physical disaster
                 │
                 ▼
             Both lost
```

## Planned Improvement: Off-site Backup

An off-site backup is planned as the next major improvement to the data-protection architecture.

The target architecture would provide an additional copy at a physically independent location:

```text
                  Active Data
                      │
                      ▼
                 ┌─────────┐
                 │   SSD   │
                 └────┬────┘
                      │
                ┌─────┴─────┐
                │           │
             Local       Off-site
             Backup       Backup
                │           │
                ▼           ▼
              HDD      Remote Storage
```

This would protect against scenarios in which both the server and the local backup drive are lost.

The off-site solution has not yet been implemented.

## Security of Backup Data

Backup credentials and other sensitive backup configuration are not part of the public infrastructure repository.

In particular, the following types of information are kept private:

* Restic repository credentials
* Restic repository passwords
* environment files containing secrets
* access tokens
* private keys
* other authentication material

The public repository only documents the architecture and contains sanitized examples such as:

```text
restic/
├── backup.sh.example
├── env.example
└── excludes.txt.example
```

The actual Restic repository, password file, environment configuration, logs, and application data are not published.

## Backup Philosophy

The backup strategy follows the principle that **availability and recoverability are separate concerns**.

Running the applications successfully does not guarantee that their data can be recovered after a failure.

The infrastructure therefore separates:

1. **Active storage** — used by the applications during normal operation.

2. **Local backup storage** — used to recover from hardware failures, accidental deletion, and other logical failures.

3. **Future off-site storage** — intended to protect against physical loss of the local infrastructure.

The current setup provides a practical local recovery mechanism while explicitly identifying off-site backup as an important remaining improvement.
