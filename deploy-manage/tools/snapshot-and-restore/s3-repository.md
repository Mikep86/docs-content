---
mapped_pages:
  - https://www.elastic.co/guide/en/elasticsearch/reference/current/repository-s3.html
applies_to:
  deployment:
    self:
products:
  - id: elasticsearch
---

# S3 repository [repository-s3]

You can use AWS S3 as a repository for [Snapshot/Restore](../snapshot-and-restore.md).

::::{note}
If you are looking for a hosted solution of {{es}} on AWS, visit [https://www.elastic.co/cloud/](https://www.elastic.co/cloud/).
::::

See [this video](https://www.youtube.com/watch?v=ACqfyzWf-xs) for a walkthrough of connecting an AWS S3 repository.

## Getting started [repository-s3-usage]

To register an S3 repository, specify the type as `s3` when creating the repository. The repository defaults to using [ECS IAM Role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html) credentials for authentication. You can also use [Kubernetes service accounts](#iam-kubernetes-service-accounts) for authentication.

The only mandatory setting is the bucket name:

```console
PUT _snapshot/my_s3_repository
{
  "type": "s3",
  "settings": {
    "bucket": "my-bucket"
  }
}
```


## Client settings [repository-s3-client]

The client that you use to connect to S3 has a number of settings available. The settings have the form `s3.client.CLIENT_NAME.SETTING_NAME`. By default, `s3` repositories use a client named `default`, but this can be modified using the [repository setting](#repository-s3-repository) `client`. For example, to use a client named `my-alternate-client`, register the repository as follows:

```console
PUT _snapshot/my_s3_repository
{
  "type": "s3",
  "settings": {
    "bucket": "my-bucket",
    "client": "my-alternate-client"
  }
}
```

Most client settings can be added to the [`elasticsearch.yml`](/deploy-manage/stack-settings.md) configuration file with the exception of the secure settings, which you add to the {{es}} keystore. For more information about creating and updating the {{es}} keystore, see [Secure settings](../../security/secure-settings.md).

For example, if you want to use specific credentials to access S3 then run the following commands to add these credentials to the keystore.

```sh
bin/elasticsearch-keystore add s3.client.default.access_key
bin/elasticsearch-keystore add s3.client.default.secret_key
# a session token is optional so the following command may not be needed
bin/elasticsearch-keystore add s3.client.default.session_token
```

If you do not configure these settings then {{es}} will attempt to automatically obtain credentials from the environment in which it is running:

* Nodes running on an instance in AWS EC2 will attempt to use the EC2 Instance Metadata Service (IMDS) to obtain instance role credentials. {{es}} supports both IMDS version 1 and IMDS version 2.
* Nodes running in a container in AWS ECS and AWS EKS will attempt to obtain container role credentials similarly.

You can switch from using specific credentials back to the default of using the instance role or container role by removing these settings from the keystore as follows:

```sh
bin/elasticsearch-keystore remove s3.client.default.access_key
bin/elasticsearch-keystore remove s3.client.default.secret_key
# a session token is optional so the following command may not be needed
bin/elasticsearch-keystore remove s3.client.default.session_token
```

Define the relevant secure settings in each node’s keystore before starting the node. The secure settings described here are all [reloadable](../../security/secure-settings.md#reloadable-secure-settings) so you may update the keystore contents on each node while the node is running and then call the [Nodes reload secure settings API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-nodes-reload-secure-settings) to apply the updated settings to the nodes in the cluster. After this API completes, {{es}} will use the updated setting values for all future snapshot operations, but ongoing operations may continue to use older setting values.

The following list contains the available client settings. Those that must be stored in the keystore are marked as "secure" and are **reloadable**; the other settings belong in the [`elasticsearch.yml`](/deploy-manage/stack-settings.md) file.

`region`
:   Specifies the region to use. When set, determines the signing region and regional endpoint to use, unless the endpoint is overridden via the `endpoint` setting. If not set, {{es}} will attempt to determine the region automatically using the AWS SDK.

`access_key` ([Secure](/deploy-manage/security/secure-settings.md), [reloadable](../../security/secure-settings.md#reloadable-secure-settings))
:   An S3 access key. If set, the `secret_key` setting must also be specified. If unset, the client will use the instance or container role instead.

`secret_key` ([Secure](/deploy-manage/security/secure-settings.md), [reloadable](../../security/secure-settings.md#reloadable-secure-settings))
:   An S3 secret key. If set, the `access_key` setting must also be specified.

`session_token` ([Secure](/deploy-manage/security/secure-settings.md), [reloadable](../../security/secure-settings.md#reloadable-secure-settings))
:   An S3 session token. If set, the `access_key` and `secret_key` settings must also be specified.

`endpoint`
:   The S3 service endpoint to connect to. This defaults to the regional endpoint corresponding to the configured `region`, but the [AWS documentation](https://docs.aws.amazon.com/general/latest/gr/rande.html#s3_region) lists alternative S3 endpoints. If you are using an [S3-compatible service](#repository-s3-compatible-services) then you should set this to the service’s endpoint. The endpoint should specify the protocol and host name, e.g. `https://s3.ap-southeast-4.amazonaws.com`, `http://minio.local:9000`.

    When using HTTPS, this repository type validates the repository’s certificate chain using the JVM-wide truststore. Ensure that the root certificate authority is in this truststore using the JVM’s `keytool` tool. If you have a custom certificate authority for your S3 repository and you use the {{es}} [bundled JDK](../../deploy/self-managed/installing-elasticsearch.md#jvm-version), then you will need to reinstall your CA certificate every time you upgrade {{es}}.

`protocol`
:   The protocol to use to connect to S3. Valid values are either `http` or `https`. Defaults to `https`. Note that this setting is deprecated since 8.19 and is only used if `endpoint` is set to a URL that does not include a scheme. Users should migrate to including the scheme in the `endpoint` setting. 

`proxy.host`
:   The host name of a proxy to connect to S3 through.

`proxy.port`
:   The port of a proxy to connect to S3 through.

`proxy.scheme`
:   The scheme to use for the proxy connection to S3. Valid values are either `http` or `https`. Defaults to `http`. This setting allows to specify the protocol used for communication with the proxy server

`proxy.username` ([Secure](/deploy-manage/security/secure-settings.md), [reloadable](../../security/secure-settings.md#reloadable-secure-settings))
:   The username to connect to the `proxy.host` with.

`proxy.password` ([Secure](/deploy-manage/security/secure-settings.md), [reloadable](../../security/secure-settings.md#reloadable-secure-settings))
:   The password to connect to the `proxy.host` with.

`read_timeout`
:   ([time value](elasticsearch://reference/elasticsearch/rest-apis/api-conventions.md#time-units)) The maximum time {{es}} will wait to receive the next byte of data over an established, open connection to the repository before it closes the connection. The default value is 50 seconds.

`max_connections`
:   The maximum number of concurrent connections to S3. The default value is `50`.

`max_retries`
:   The number of retries to use when an S3 request fails. The default value is `3`.

`path_style_access`
:   Whether to force the use of the path style access pattern. If `true`, the path style access pattern will be used. If `false`, the access pattern will be automatically determined by the AWS Java SDK (See [AWS documentation](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/s3/AmazonS3Builder.html#setPathStyleAccessEnabled-java.lang.Boolean-) for details). Defaults to `false`.

::::{note}
:name: repository-s3-path-style-deprecation

In versions `7.0`, `7.1`, `7.2` and `7.3` all bucket operations used the [now-deprecated](https://aws.amazon.com/blogs/aws/amazon-s3-path-deprecation-plan-the-rest-of-the-story/) path style access pattern. If your deployment requires the path style access pattern then you should set this setting to `true` when upgrading.
::::


`disable_chunked_encoding`
:   Whether chunked encoding should be disabled or not. If `false`, chunked encoding is enabled and will be used where appropriate. If `true`, chunked encoding is disabled and will not be used, which may mean that snapshot operations consume more resources and take longer to complete. It should only be set to `true` if you are using a storage service that does not support chunked encoding. See the [AWS Java SDK documentation](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/s3/AmazonS3Builder.html#disableChunkedEncoding--) for details. Defaults to `false`.


## Repository settings [repository-s3-repository]

The `s3` repository type supports a number of settings to customize how data is stored in S3. These can be specified when creating the repository. For example:

```console
PUT _snapshot/my_s3_repository
{
  "type": "s3",
  "settings": {
    "bucket": "my-bucket",
    "another_setting": "setting-value"
  }
}
```

The following settings are supported:

`bucket`
:   (Required) Name of the S3 bucket to use for snapshots.

    The bucket name must adhere to Amazon’s [S3 bucket naming rules](https://docs.aws.amazon.com/AmazonS3/latest/dev/BucketRestrictions.html#bucketnamingrules).


`client`
:   The name of the [S3 client](#repository-s3-client) to use to connect to S3. Defaults to `default`.

`base_path`
:   Specifies the path to the repository data within its bucket. Defaults to an empty string, meaning that the repository is at the root of the bucket. The value of this setting should not start or end with a `/`.

    ::::{note}
    Don’t set `base_path` when configuring a snapshot repository for {{ECE}}. {{ECE}} automatically generates the `base_path` for each deployment so that multiple deployments may share the same bucket.
    ::::


`chunk_size`
:   ([byte value](elasticsearch://reference/elasticsearch/rest-apis/api-conventions.md#byte-units)) The maximum size of object that {{es}} will write to the repository when creating a snapshot. Files which are larger than `chunk_size` will be chunked into several smaller objects. {{es}} may also split a file across multiple objects to satisfy other constraints such as the `max_multipart_parts` limit. Defaults to `5TB` which is the [maximum size of an object in AWS S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/qfacts.html).

`compress`
:   When set to `true` metadata files are stored in compressed format. This setting doesn’t affect index files that are already compressed by default. Defaults to `true`.

`max_restore_bytes_per_sec`
:   (Optional, [byte value](elasticsearch://reference/elasticsearch/rest-apis/api-conventions.md#byte-units)) Maximum snapshot restore rate per node. Defaults to unlimited. Note that restores are also throttled through [recovery settings](elasticsearch://reference/elasticsearch/configuration-reference/index-recovery-settings.md).

`max_snapshot_bytes_per_sec`
:   (Optional, [byte value](elasticsearch://reference/elasticsearch/rest-apis/api-conventions.md#byte-units)) Maximum snapshot creation rate per node. Defaults to `40mb` per second. Note that if the [recovery settings for managed services](elasticsearch://reference/elasticsearch/configuration-reference/index-recovery-settings.md#recovery-settings-for-managed-services) are set, then it defaults to unlimited, and the rate is additionally throttled through [recovery settings](elasticsearch://reference/elasticsearch/configuration-reference/index-recovery-settings.md).

`readonly`
:   (Optional, Boolean) If `true`, the repository is read-only. The cluster can retrieve and restore snapshots from the repository but not write to the repository or create snapshots in it.

    Only a cluster with write access can create snapshots in the repository. All other clusters connected to the repository should have the `readonly` parameter set to `true`.

    If `false`, the cluster can write to the repository and create snapshots in it. Defaults to `false`.

    ::::{important}
    If you register the same snapshot repository with multiple clusters, only one cluster should have write access to the repository. Having multiple clusters write to the repository at the same time risks corrupting the contents of the repository.

    ::::


`server_side_encryption`
:   When set to `true` files are encrypted on server side using AES256 algorithm. Defaults to `false`.

`buffer_size`
:   ([byte value](elasticsearch://reference/elasticsearch/rest-apis/api-conventions.md#byte-units)) Minimum threshold below which the chunk is uploaded using a single request. Beyond this threshold, the S3 repository will use the [AWS Multipart Upload API](https://docs.aws.amazon.com/AmazonS3/latest/dev/uploadobjusingmpu.html) to split the chunk into several parts, each of `buffer_size` length, and to upload each part in its own request. Note that setting a buffer size lower than `5mb` is not allowed since it will prevent the use of the Multipart API and may result in upload errors. It is also not possible to set a buffer size greater than `5gb` as it is the maximum upload size allowed by S3. Defaults to `100mb` or `5%` of JVM heap, whichever is smaller.

`max_multipart_parts`
:   (integer) The maximum number of parts that {{es}} will write during a multipart upload of a single object. Files which are larger than `buffer_size × max_multipart_parts` will be chunked into several smaller objects. {{es}} may also split a file across multiple objects to satisfy other constraints such as the `chunk_size` limit. Defaults to `10000` which is the [maximum number of parts in a multipart upload in AWS S3](https://docs.aws.amazon.com/AmazonS3/latest/userguide/qfacts.html).

`canned_acl`
:   The S3 repository supports all [S3 canned ACLs](https://docs.aws.amazon.com/AmazonS3/latest/dev/acl-overview.html#canned-acl) : `private`, `public-read`, `public-read-write`, `authenticated-read`, `log-delivery-write`, `bucket-owner-read`, `bucket-owner-full-control`. Defaults to `private`. You could specify a canned ACL using the `canned_acl` setting. When the S3 repository creates buckets and objects, it adds the canned ACL into the buckets and objects.

`storage_class`
:   Sets the S3 storage class for objects written to the repository. Values may be `standard`, `reduced_redundancy`, `standard_ia`, `onezone_ia` and `intelligent_tiering`. Defaults to `standard`. See [S3 storage classes](#repository-s3-storage-classes) for more information.

`delete_objects_max_size`
:   (integer) Sets the maxmimum batch size, betewen 1 and 1000, used for `DeleteObjects` requests. Defaults to 1000 which is the maximum number supported by the [AWS DeleteObjects API](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObjects.html).

`max_multipart_upload_cleanup_size`
:   (integer) Sets the maximum number of possibly-dangling multipart uploads to clean up in each batch of snapshot deletions. Defaults to `1000` which is the maximum number supported by the [AWS ListMultipartUploads API](https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListMultipartUploads.html). If set to `0`, {{es}} will not attempt to clean up dangling multipart uploads.

`throttled_delete_retry.delay_increment`
:   ([time value](elasticsearch://reference/elasticsearch/rest-apis/api-conventions.md#time-units)) This value is used as the delay before the first retry and the amount the delay is incremented by on each subsequent retry. Default is 50ms, minimum is 0ms.

`throttled_delete_retry.maximum_delay`
:   ([time value](elasticsearch://reference/elasticsearch/rest-apis/api-conventions.md#time-units)) This is the upper bound on how long the delays between retries will grow to. Default is 5s, minimum is 0ms.

`throttled_delete_retry.maximum_number_of_retries`
:   (integer) Sets the number times to retry a throttled snapshot deletion. Defaults to `10`, minimum value is `0` which will disable retries altogether. Note that if retries are enabled in the Azure client, each of these retries comprises that many client-level retries.

`get_register_retry_delay`
:   ([time value](elasticsearch://reference/elasticsearch/rest-apis/api-conventions.md#time-units)) Sets the time to wait before trying again if an attempt to read a [linearizable register](#repository-s3-linearizable-registers) fails. Defaults to `5s`.

::::{note}
The option of defining client settings in the repository settings as documented below is considered deprecated, and will be removed in a future version.
::::


In addition to the above settings, you may also specify all non-secure client settings in the repository settings. In this case, the client settings found in the repository settings will be merged with those of the named client used by the repository. Conflicts between client and repository settings are resolved by the repository settings taking precedence over client settings.

For example:

```console
PUT _snapshot/my_s3_repository
{
  "type": "s3",
  "settings": {
    "client": "my-client",
    "bucket": "my-bucket",
    "endpoint": "my.s3.endpoint"
  }
}
```

This sets up a repository that uses all client settings from the client `my_client_name` except for the `endpoint` that is overridden to `my.s3.endpoint` by the repository settings. `


## S3 storage classes [repository-s3-storage-classes]

Amazon S3 supports a variety of  *storage classes*, each of which offers different operational characteristics. For instance, some classes cost less per byte stored per month, but cost more per request, and other classes may vary in terms of their availability guarantees.

You may specify the storage class that {{es}} uses to store data objects with the `storage_class` repository setting.

Changing the `storage_class` setting on an existing repository only affects the storage class for newly created objects, resulting in a mixed usage of storage classes.

You may use an S3 Lifecycle Policy to adjust the storage class of existing objects in your repository, but you must not transition objects to an unsupported class such as the Glacier classes, and you must not expire objects. If you use a Glacier storage class, or another unsupported storage class, or object expiry, then you may permanently lose access to your repository contents.

You may use the `intelligent_tiering` storage class to automatically manage the class of objects, but you must not enable the optional Archive Access or Deep Archive Access tiers. If you use these tiers then you may permanently lose access to your repository contents.

For more information about S3 storage classes, see [AWS Storage Classes Guide](https://docs.aws.amazon.com/AmazonS3/latest/dev/storage-class-intro.html).


## Recommended S3 permissions [repository-s3-permissions]

In order to restrict the {{es}} snapshot process to the minimum required resources, we recommend using Amazon IAM in conjunction with pre-existing S3 buckets. Here is an example policy which will allow the snapshot access to an S3 bucket named "snaps.example.com". This may be configured through the AWS IAM console, by creating a Custom Policy, and using a Policy Document similar to this (changing snaps.example.com to your bucket name).

```js
{
  "Statement": [
    {
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:ListBucketMultipartUploads",
        "s3:ListBucketVersions"
      ],
      "Effect": "Allow",
      "Resource": [
        "arn:aws:s3:::snaps.example.com"
      ]
    },
    {
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Effect": "Allow",
      "Resource": [
        "arn:aws:s3:::snaps.example.com/*"
      ]
    }
  ],
  "Version": "2012-10-17"
}
```

You may further restrict the permissions by specifying a prefix within the bucket, in this example, named "foo".

```js
{
  "Statement": [
    {
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:ListBucketMultipartUploads",
        "s3:ListBucketVersions"
      ],
      "Condition": {
        "StringLike": {
          "s3:prefix": [
            "foo/*"
          ]
        }
      },
      "Effect": "Allow",
      "Resource": [
        "arn:aws:s3:::snaps.example.com"
      ]
    },
    {
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Effect": "Allow",
      "Resource": [
        "arn:aws:s3:::snaps.example.com/foo/*"
      ]
    }
  ],
  "Version": "2012-10-17"
}
```

The bucket needs to exist to register a repository for snapshots. If you did not create the bucket then the repository registration will fail.


#### Using IAM roles for Kubernetes service accounts for authentication [iam-kubernetes-service-accounts]

If you want to use [Kubernetes service accounts](https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/) for authentication, you need to add a symlink to the `$AWS_WEB_IDENTITY_TOKEN_FILE` environment variable (which should be automatically set by a Kubernetes pod) in the S3 repository config directory, so the repository can have the read access for the service account (a repository can’t read any files outside its config directory). For example:

```bash
mkdir -p "${ES_PATH_CONF}/repository-s3"
ln -s $AWS_WEB_IDENTITY_TOKEN_FILE "${ES_PATH_CONF}/repository-s3/aws-web-identity-token-file"
```

::::{important}
The symlink must be created on all data and master eligible nodes and be readable by the `elasticsearch` user. By default, {{es}} runs as user `elasticsearch` using uid:gid `1000:0`.
::::


If the symlink exists, it will be used by default by all S3 repositories that don’t have explicit `client` credentials.


## AWS VPC bandwidth settings [repository-s3-aws-vpc]

AWS instances resolve S3 endpoints to a public IP. If the {{es}} instances reside in a private subnet in an AWS VPC then all traffic to S3 will go through the VPC’s NAT instance. If your VPC’s NAT instance is a smaller instance size (e.g. a t2.micro) or is handling a high volume of network traffic your bandwidth to S3 may be limited by that NAT instance’s networking bandwidth limitations. Instead we recommend creating a [VPC endpoint](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-endpoints.html) that enables connecting to S3 in instances that reside in a private subnet in an AWS VPC. This will eliminate any limitations imposed by the network bandwidth of your VPC’s NAT instance.

Instances residing in a public subnet in an AWS VPC will connect to S3 via the VPC’s internet gateway and not be bandwidth limited by the VPC’s NAT instance.


## S3-compatible services [repository-s3-compatible-services]

There are a number of storage systems that provide an S3-compatible API, and the `s3` repository type allows you to use these systems in place of AWS S3. To do so, you should set the `s3.client.CLIENT_NAME.endpoint` setting to the system’s endpoint. This setting accepts IP addresses and hostnames and may include a port. For example, the endpoint may be `172.17.0.2` or `172.17.0.2:9000`.

By default {{es}} communicates with your storage system using HTTPS, and validates the repository’s certificate chain using the JVM-wide truststore. Ensure that the JVM-wide truststore includes an entry for your repository. If you wish to use unsecured HTTP communication instead of HTTPS, set `s3.client.CLIENT_NAME.protocol` to `http`.

There are many systems, including some from very well-known storage vendors, which claim to offer an S3-compatible API despite failing to emulate S3’s behavior in full. If you are using such a system for your snapshots, consider using a [shared filesystem repository](shared-file-system-repository.md) based on a standardized protocol such as NFS to access your storage system instead. The `s3` repository type requires full compatibility with S3. In particular it must support the same set of API endpoints, with the same parameters, return the same errors in case of failures, and offer consistency and performance at least as good as S3 even when accessed concurrently by multiple nodes. You will need to work with the supplier of your storage system to address any incompatibilities you encounter. Don't report {{es}} issues involving storage systems which claim to be S3-compatible unless you can demonstrate that the same issue exists when using a genuine AWS S3 repository.

You can perform some basic checks of the suitability of your storage system using the [repository analysis API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-snapshot-repository-analyze). If this API does not complete successfully, or indicates poor performance, then your storage system is not fully compatible with AWS S3 and therefore unsuitable for use as a snapshot repository. However, these checks do not guarantee full compatibility.

::::{admonition} Using MinIO with {{es}}
[MinIO](https://minio.io) is an example of a storage system that provides an S3-compatible API. The `s3` repository type allows {{es}} to work with MinIO-backed repositories as well as repositories stored on AWS S3. The {{es}} test suite includes some checks which aim to detect deviations in behavior between MinIO and AWS S3. Elastic will report directly to the MinIO project any deviations in behavior found by these checks. If you are running a version of MinIO whose behavior deviates from that of AWS S3 then you must upgrade your MinIO installation. If in doubt, please contact the MinIO support team for further information.

The performance, reliability, and durability of a MinIO-backed repository depend on the properties of the underlying infrastructure and on the details of your MinIO configuration. You must design your storage infrastructure and configure MinIO in a way that ensures your MinIO-backed repository has performance, reliability, and durability characteristics which match AWS S3 in order for it to be fully S3-compatible. If you need assistance with your MinIO configuration, please contact the MinIO support team.
::::

Most storage systems can be configured to log the details of their interaction with {{es}}. If you are investigating a suspected incompatibility with AWS S3, it is usually simplest to collect these logs and provide them to the supplier of your storage system for further analysis. If the incompatibility is not clear from the logs emitted by the storage system, configure {{es}} to log every request it makes to the S3 API by [setting the logging level](/deploy-manage/monitor/logging-configuration/update-elasticsearch-logging-levels.md) of the `com.amazonaws.request` logger to `DEBUG`.

To prevent leaking sensitive information such as credentials and keys in logs, {{es}} rejects configuring this logger at high verbosity unless [insecure network trace logging](elasticsearch://reference/elasticsearch/configuration-reference/networking-settings.md#http-rest-request-tracer) is enabled. To do so, you must explicitly enable it on each node by setting the system property `es.insecure_network_trace_enabled` to `true`.

Once enabled, you can configure the `com.amazonaws.request` logger:

```console
PUT /_cluster/settings
{
  "persistent": {
    "logger.com.amazonaws.request": "DEBUG"
  }
}
```

Collect the {{es}} logs covering the time period of the failed analysis from all nodes in your cluster and share them with the supplier of your storage system along with the analysis response so they can use them to determine the problem. See the [AWS Java SDK](https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/java-dg-logging.html) documentation for further information, including details about other loggers that can be used to obtain even more verbose logs. When you have finished collecting the logs needed by your supplier, set the logger settings back to `null` to return to the default logging configuration and disable insecure network trace logging again. See [Logger](elasticsearch://reference/elasticsearch/configuration-reference/miscellaneous-cluster-settings.md#cluster-logger) and [Cluster update settings](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-cluster-put-settings) for more information.


## Linearizable register implementation [repository-s3-linearizable-registers]

The linearizable register implementation for S3 repositories is based on the strongly consistent semantics of the multipart upload API. {{es}} first creates a multipart upload to indicate its intention to perform a linearizable register operation. {{es}} then lists and cancels all other multipart uploads for the same register. {{es}} then attempts to complete the upload. If the upload completes successfully then the compare-and-exchange operation was atomic.


