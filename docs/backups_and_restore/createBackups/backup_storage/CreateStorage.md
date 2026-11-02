# Create backup storage

Before working with backups, create a backup storage location as a backup destination for creating and storing your backup artifacts.

A backup storage is a bucket in an object store, optionally with a prefix within that bucket, together with the settings and credentials needed to reach it. Every backup storage names a **backup storage class**: the object storage technology it uses. OpenEverest ships one class, `s3` (shown as **S3-compatible object storage**), which covers [Amazon S3](https://aws.amazon.com/s3/) and S3-compatible stores such as [MinIO](https://min.io/), Ceph RGW, SeaweedFS and the Google Cloud Storage (GCS) interoperability endpoint.

!!! note alert alert-primary "Storage class and implementation"
    The storage class is the technology: the API OpenEverest and the database engines use to talk to the store. For the `s3` class, the **Implementation** setting says which product serves that S3 endpoint: Amazon S3, MinIO, Ceph RGW, GCS or another S3-compatible store. Engines use it to pick the right client settings, and not every engine supports every implementation. See [Engine support](../../../crd/backup_storage_config.md#engine-support).

Each instance writes its backups under its own path in the bucket, so many instances and namespaces can share one bucket. See [How backups are laid out in the bucket](../../../crd/backup_storage_config.md#how-backups-are-laid-out-in-the-bucket).

!!! caution alert alert-warning "Important"

     - Currently, OpenEverest does not support S3 buckets with S3 Object Lock. Make sure your backup destination you are registering does not have S3 Object Lock enabled.

     - Make sure you have read/write/delete permissions to the S3 bucket.

     - The storage class, the bucket and the prefix can't be changed once the backup storage is created. To write backups to another bucket or prefix, create a new backup storage.

## Create backup storage from the UI

To create a backup storage location:
{.power-number}

1. Go to Backup storage settings:

    Navigate to OpenEverest home page and select <i class="uil uil-cog"></i> **Settings > Backup storages**. Then click **Add backup storage**.

2. Fill in the general fields:

      - **Name**: Enter a name using only lowercase letters, numbers, or hyphens.
      - **Namespace**: Select where this backup storage will be available. Only instances in this namespace can use it, and the Secrets holding its credentials live in this namespace too.

3. Select the storage class and the location:

      - **Class**: Select **S3-compatible object storage**.
      - **Bucket**: Provide the bucket name where backups will be stored.
      - **Prefix**: (Optional) Enter a path within the bucket to write backups under, for example `team-a`, without a leading or trailing slash.

4. Fill in the connection settings:

      - **Region**: Specify the bucket's region (e.g., us-east-1).
      - **Endpoint URL**: Enter the URL of the S3 API (e.g., https://s3.us-east-1.amazonaws.com). An `http://` URL means plain HTTP, without TLS.
      - **Implementation**: (Optional) Select the software that serves the endpoint: **Amazon S3**, **MinIO**, **Ceph RGW**, **Google Cloud Storage (S3 interoperability)** or **Other S3-compatible**. If you leave it empty, Amazon S3 is assumed.
      - **Force path-style URLs**: Enable this if your store addresses the bucket in the URL path instead of the host name, as most S3-compatible stores require.

5. Set up authentication:

      - **Authentication mode**: Select **AccessKey**: an access key ID and secret access key pair.
      - Choose where the credentials come from:
          - **Enter credentials**: Enter the **Access key ID** and the **Secret access key**. OpenEverest saves them in a new Secret named `backup-storage-<name>-credentials` in the namespace.
          - **Use an existing Secret**: Select a Secret in the namespace that holds the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` keys.

6. (Optional) Expand **Advanced** and choose how the certificate of the store is verified under **TLS certificate verification**:

      - **System trust store**: The default. The certificate must be issued by a certificate authority (CA) the system trusts.
      - **Custom CA**: For a store whose certificate is issued by a private CA. Either select **Paste a PEM bundle** and paste the CA certificates, which OpenEverest saves in a new Secret named `backup-storage-<name>-ca`, or select **Use an existing Secret** and pick the Secret and the **Key** that holds the PEM bundle.
      - **Skip certificate verification**: The certificate isn't verified, which leaves connections to the store open to man-in-the-middle attacks. Use it for testing only.

7. Click **Add** to save your configuration.

The new backup storage appears in the list. Its **Status** turns **Ready** once OpenEverest has checked it against its storage class. Otherwise the status shows the reason, for example `CredentialsNotFound`; hover over it for details. This check covers the settings and the Secrets, not whether the store is reachable. For the list of reasons, see [Status](../../../crd/backup_storage_config.md#status).

When you set up backups for an instance, OpenEverest offers only the backup storages the instance's backup class can write to.

!!! note alert alert-primary "Permissions"
    Besides `create` on `backup-storages`, creating a backup storage requires `read` on its backup storage class (`backup-storage-classes`) and on every Secret it names (`secrets`). To enter credentials or paste a CA bundle, you also need `create` on `secrets`. See [Role-based access control](../../../administer/rbac.md#backup-storages-and-secrets).

## Create backup storage with everestctl

!!! note alert alert-primary ""
    `everestctl` is not published as a binary for the developer previews. Build it from source with `make build-cli` in the [openeverest repository](https://github.com/openeverest/openeverest).

To create a backup storage with `everestctl`:
{.power-number}

1. Put the credentials in a file of `KEY=VALUE` lines, for example `s3.env`:

    ```text
    AWS_ACCESS_KEY_ID=<access key id>
    AWS_SECRET_ACCESS_KEY=<secret access key>
    ```

2. Create the backup storage:

    ```sh
    everestctl backup-storage create --name s3-prod -n team-a \
      --bucket my-backups --prefix team-a \
      --region us-east-1 --endpoint-url https://s3.us-east-1.amazonaws.com \
      --credentials-file ./s3.env
    ```

    `everestctl` creates the Secret `backup-storage-s3-prod-credentials` from the file and never prints its values. To use a Secret that already exists in the namespace, pass `--credentials-secret <secret-name>` instead of `--credentials-file`.

3. Check that the backup storage is accepted:

    ```sh
    everestctl backup-storage list -n team-a
    ```

    The `ACCEPTED` column is `True` once the backup storage is valid for its class. Add `--json` to see the reason when it isn't.

The `create` command takes the following flags:

| Flag | Description |
| ---- | ----------- |
| `--name` | Name of the backup storage. Required. |
| `-n`, `--namespace` | Namespace of the backup storage. Required. |
| `--class` | Backup storage class. Defaults to `s3`. |
| `--bucket` | Bucket name. Required. |
| `--prefix` | Path within the bucket to write backups under, without a leading or trailing slash. |
| `--region`, `--endpoint-url`, `--implementation`, `--force-path-style` | Set the `region`, `endpointURL`, `implementation` and `forcePathStyle` parameters of the `s3` class. `--implementation` is one of `aws`, `minio`, `ceph`, `gcs` or `other`. |
| `--parameter key=value` | Set any parameter of the class. Repeatable. `true`, `false` and numbers keep their JSON type. |
| `--auth-mode` | Authentication mode. Defaults to `AccessKey`. |
| `--credentials-file` | File of `KEY=VALUE` lines to create the Secret `backup-storage-<name>-credentials` from. |
| `--credentials-secret` | Name of an existing Secret holding the credentials. |
| `--ca-file` | PEM CA bundle to create the Secret `backup-storage-<name>-ca` from. |
| `--ca-secret NAME[:KEY]` | Existing Secret holding the PEM CA bundle. `KEY` defaults to `ca.crt`. |
| `--insecure-skip-tls-verify` | Don't verify the certificate of the store. |

Use exactly one of `--credentials-file` and `--credentials-secret`. If the backup storage can't be created, `everestctl` deletes the Secrets it created for it.

??? example "MinIO with a private CA"

    ```sh
    everestctl backup-storage create --name minio -n team-a --bucket backups \
      --region us-east-1 --endpoint-url https://minio.minio.svc:9000 \
      --implementation minio --force-path-style \
      --credentials-file ./minio.env --ca-file ./minio-ca.pem
    ```

To create a backup storage with `kubectl` instead, see [Configure backup storage](../../../crd/backup_storage_config.md).

## Next steps

[Create backups :material-arrow-right:](../../createBackups/CreateOnDemand.md){.md-button}
[Enable PITR :material-arrow-right:](../../createBackups/EnablePITR.md){.md-button}