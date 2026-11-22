## Supported Parameters
| Parameter                          | Description of value                                      |
|------------------------------------|-----------------------------------------------------------|
| fastSnapshotRestoreAvailabilityZones | Comma separated list of availability zones                |
| outpostArn                         | Arn of the outpost you wish to have the snapshot saved to |
| lockMode                   | Lock mode (governance/compliance)                         |
| lockDuration               | Lock duration in days                                     |
| lockExpirationDate         | Lock expiration date (RFC3339 format)                    |
| lockCoolOffPeriod          | Cool-off period in hours (compliance mode only)          | 

The AWS EBS CSI Driver supports [tagging](tagging.md) through `VolumeSnapshotClass.parameters` (in v1.6.0 and later). 
## Prerequisites

- Install the [Kubernetes Volume Snapshot CRDs](https://github.com/kubernetes-csi/external-snapshotter/tree/master/client/config/crd) and external-snapshotter sidecar. For installation instructions, see [CSI Snapshotter Usage](https://github.com/kubernetes-csi/external-snapshotter#usage).

# Fast Snapshot Restores

The EBS CSI Driver provides support for [Fast Snapshot Restores(FSR)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-fast-snapshot-restore.html) via `VolumeSnapshotClass.parameters.fastSnapshotRestoreAvailabilityZones`.


Availability zones are specified as a comma separated list.

- The EBS CSI Driver must be given permission to access the [`EnableFastSnapshotRestores` EC2 API](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_EnableFastSnapshotRestores.html). This example snippet can be used in an IAM policy to grant access to `EnableFastSnapshotRestores`:

**Example**
```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-vsc
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  fastSnapshotRestoreAvailabilityZones: "us-east-1a, us-east-1b"
```

```json
{
  "Effect": "Allow",
  "Action": [
    "ec2:EnableFastSnapshotRestores"
  ],
  "Resource": "*"
}
```

## Failure Mode 

The driver will attempt to check if the availability zones provided are supported for fast snapshot restore before attempting to create the snapshot. If the `EnableFastSnapshotRestores` API call fails, the driver will hard-fail the request and delete the snapshot. This is to ensure that the snapshot is not left in an inconsistent state.

# Snapshot Lock

The EBS CSI Driver supports [EBS Snapshot Lock](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshot-lock.html) via `VolumeSnapshotClass.parameters`. Snapshot locking protects snapshots from accidental or malicious deletion. A locked snapshot can't be deleted.

**Example - Lock in Governance Mode with Specified Duration**
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-vsc-locked
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  lockMode: "governance"
  lockDuration: "7"
```

**Example - Lock in Compliance Mode with Expiration Date and Cool Off Period**
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-vsc-compliance
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  lockMode: "compliance"
  lockExpirationDate: "2030-12-31T23:59:59Z"
  lockCoolOffPeriod: "24"
```

## Failure Mode

If the `LockSnapshot` API call fails, the driver will hard-fail the request and delete the snapshot. This ensures that the snapshot is not left in an unlocked state when locking was explicitly requested.


# Snapshot states

EC2 reports one of five snapshot states. The driver maps them to the CSI `readyToUse` flag as follows:

| EC2 state | `readyToUse` | `CreateSnapshot` behavior | `ListSnapshots` / `GetSnapshot` behavior |
|---|---|---|---|
| `pending` | false | returns the snapshot, not yet ready, no error | listed as a not-ready entry |
| `completed` | true | returns the snapshot, ready | listed as a ready entry |
| `error` | false | **fails the RPC with `Aborted`**, including the EC2 state message in the error | retained as a not-ready entry, no error |
| `recovering` | false | returns the snapshot, not yet ready, no error | listed as a not-ready entry |
| `recoverable` | false | returns the snapshot, not yet ready, no error | listed as a not-ready entry |
| unknown / empty | false | returns the snapshot, not yet ready, no error | listed as a not-ready entry |

When a newly created snapshot reaches the `error` state the driver does **not** delete it. The caller (for example the external-snapshotter sidecar after receiving `Aborted`) is expected to delete the failed snapshot and recreate it. The choice of gRPC code (`Aborted` rather than `Internal`) is intentionally narrow to this path and is not applied as a general error-mapping convention.

# Pagination and listing

## Max entries

- EC2 rejects `MaxResults` values between 1 and 4. The driver returns `InvalidArgument` (`ErrInvalidMaxResults`) for these values.
- A CSI `max_entries` of `0` means "no limit": the driver omits the EC2 `MaxResults` field and EC2 applies its own page size (up to 1000 snapshots per page).

## Starting tokens

- An empty `starting_token` lists the first page.
- An invalid or expired token is rejected by EC2 with `InvalidPaginationToken`; the driver returns gRPC `Aborted`. The client should restart listing from the first page.
- The token returned in each response must be passed unchanged to the next request.

## Listing scope

By default, `ListSnapshots` without a source volume ID describes every snapshot visible to the account, which can include snapshots shared from other accounts. Because the driver never sets `OwnerIds`, account-owned, shared and public snapshots can all appear.

Passing `--list-snapshots-restrict-to-driver-tags=true` scopes these account-wide listings to snapshots carrying the driver tag `ebs.csi.aws.com/cluster=true`, excluding shared, public and foreign snapshots. The flag defaults to `false` to preserve the historical behavior; listings that specify a source volume ID are unaffected (they keep their own `volume-id` filter).

## Malformed snapshot IDs

Snapshot IDs that do not have the `snap-` prefix shape (for example `foo`), or that EC2 rejects with `InvalidSnapshotID.Malformed`, are treated as not found:

- `DeleteSnapshot` succeeds idempotently and logs a warning; no EC2 call is made for IDs rejected locally.
- `ListSnapshots` with such a `snapshot_id` returns an empty response.

This keeps delete idempotency at the cost of potentially masking typoed IDs; the warning log is the signal for those cases.

# Restoring a volume from a snapshot

When `CreateVolume` carries a snapshot content source, the driver first describes the snapshot (through the existing snapshot batcher when batching is enabled) and reconciles the requested capacity against the snapshot size:

| Request (snapshot is 10 GiB) | Result |
|---|---|
| 5 GiB required | `InvalidArgument`, message names both the requested and the snapshot size |
| 10 GiB required | 10 GiB volume created |
| 20 GiB required | 20 GiB volume created; the filesystem is expanded during `NodeStageVolume` |
| no `CapacityRange` | 10 GiB volume created (the snapshot's own size, **not** the 100 GiB driver default) |
| `limitBytes` smaller than the snapshot | `InvalidArgument` |

A request larger than the snapshot is allowed and the larger size is sent to EC2; filesystem expansion happens on the node via `NeedResize`. Encrypted-snapshot-to-unencrypted-volume and cross-availability-zone combinations are not validated locally and remain EC2's responsibility. If the snapshot does not exist, the describe returns `NotFound`; the existing `CreateVolume` `ErrSourceNotFound` mapping remains as a backstop for snapshots deleted between the describe and create calls. Volume clones (content source is another volume) use `CopyVolumes` and do not participate in this snapshot sizing logic.


# Amazon EBS Local Snapshots on Outposts

The EBS CSI Driver provides support for [Amazon EBS local snapshots on Outposts](https://docs.aws.amazon.com/ebs/latest/userguide/snapshots-outposts.html) via `VolumeSnapshotClass.parameters.outpostArn`.



**Example**
```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-vsc
driver: ebs.csi.aws.com
deletionPolicy: Delete
parameters:
  outpostarn: {arn of your outpost}
```
