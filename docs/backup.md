# Backup and Data Protection

The server stores personal application data on a dedicated SSD. To protect this data against hardware failures and accidental data loss, a second, larger HDD is used as a dedicated backup target.

Backups are performed automatically once per day using Restic.

## Storage Architecture

The storage setup consists of two external drives with different purposes:

| Storage | Purpose                 | Usage        |
| ------- | ----------------------- | ------------ |
| SSD     | Active application data | Continuous   |
| HDD     | Backup storage          | Daily backup |

The SSD is used for the live application data of the self-hosted services.

The HDD is primarily used for backups and is normally inactive outside the backup process. This reduces unnecessary operating time for the backup drive.

Conceptually:

```text
                    Server
                      │
                      │
              ┌───────▼───────┐
              │   Active SSD  │
              │               │
              │ Application   │
              │ Data          │
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
              │ Backup HDD    │
              │               │
              │ Restic Backup │
              └───────────────┘
```

## Restic

Restic is used as the backup tool.

The backup process runs automatically once per day and creates a backup of the relevant application data on the dedicated backup storage.

The backup process is separate from the applications themselves. A failure or restart of an individual Docker service therefore does not prevent the backup mechanism from operating independently.

The backup configuration contains information about:

* the directories to back up
* excluded data
* the backup repository
* required Restic credentials
* logging

Sensitive credentials are kept outside the public repository.

## Backup Scope

The backup is intended to protect the persistent data required to restore the self-hosted applications.

This primarily includes application data stored on the active SSD.

The Docker containers themselves are not considered the primary backup target. Containers can generally be recreated from their respective images and configuration, while persistent application data requires explicit protection.

This distinction can be summarized as:

```text
Container / Image
       │
       │ Recreate
       ▼
   Application

Persistent Data
       │
       │ Restore
       ▼
Backup Repository
```

The combination of reproducible container images and backed-up persistent data provides the basis for rebuilding the application environment after a failure.

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
4. Data is stored in backup repository
        │
        ▼
5. Backup process completes
        │
        ▼
6. Backup storage becomes inactive
```

The backup drive is not intended to operate continuously.

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

The backup strategy is primarily intended to protect against failures affecting the active storage.

Examples include:

### Active SSD Failure

If the active SSD fails, the application data can be restored from the Restic backup.

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

### Accidental Data Deletion

If important data is accidentally deleted from the active storage, a previous backup can provide a recovery point.

The exact recovery point depends on the available Restic snapshots and backup retention.

### Server Failure

If the server itself fails while the backup HDD remains intact, the backup repository can be used as the source for restoring the persistent application data to a replacement system.

The application containers can then be recreated using the required container images and sanitized configuration.

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

## Security of Backup Data

Backup credentials and other sensitive backup configuration are not part of the public infrastructure repository.

In particular, the following types of information are kept private:

* Restic repository credentials
* Restic repository passwords
* environment files containing secrets
* access tokens
* private keys
* other authentication material

The public repository only documents the architecture and may contain sanitized examples.

## Backup Philosophy

The backup strategy follows the principle that **availability and recoverability are separate concerns**.

Running the applications successfully does not guarantee that their data can be recovered after a failure.

The infrastructure therefore separates:

1. **Active storage** — used by the applications.
2. **Local backup storage** — used to recover from hardware or logical failures.
3. **Future off-site storage** — intended to protect against physical loss of the local infrastructure.

The current setup provides a practical local recovery mechanism while explicitly identifying off-site backup as an important remaining improvement.
