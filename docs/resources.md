# PipelineResources

`PipelinesResources` in a pipeline are the set of objects that are going to be
used as inputs to a [`Task`](task.md) and can be output by a `Task`.

A `Task` can have multiple inputs and outputs.

For example:

- A `Task`'s input could be a GitHub source which contains your application
  code.
- A `Task`'s output can be your application container image which can be then
  deployed in a cluster.
- A `Task`'s output can be a jar file to be uploaded to a storage bucket.

---

- [Syntax](#syntax)
  - [Declared resources](#declared-resources)
  - [Pipeline Tasks](#pipeline-tasks)
    - [From](#from)
- [Examples](#examples)

## Syntax

To define a configuration file for a `PipelineResource`, you can specify the
following fields:

- Required:
  - [`apiVersion`][kubernetes-overview] - Specifies the API version, for example
    `pipeline.knative.dev/v1alpha1`.
  - [`kind`][kubernetes-overview] - Specify the `PipelineResource` resource
    object.
  - [`metadata`][kubernetes-overview] - Specifies data to uniquely identify the
    `PipelineResource` object, for example a `name`.
  - [`spec`][kubernetes-overview] - Specifies the configuration information for
    your `PipelineResource` resource object.
    - [`type`](#resource-types) - Specifies the `type` of the `PipelineResource`
- Optional:
  - [`params`](#resource-types) - Parameters which are specific to each type of
    `PipelineResource`

[kubernetes-overview]:
  https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/#required-fields

### Resource Types

The following `PipelineResources` are currently supported:

- [Git resource](#git-resource)
- [Image resource](#image-resource)
- [Cluster resource](#cluster-resource)
- [Storage resource](#storage-resource)

#### Git Resource

Git resource represents a [git](https://git-scm.com/) repository, that contains
the source code to be built by the pipeline. Adding the git resource as an input
to a Task will clone this repository and allow the Task to perform the required
actions on the contents of the repo.

To create a git resource using the `PipelineResource` CRD:

```yaml
apiVersion: pipeline.knative.dev/v1alpha1
kind: PipelineResource
metadata:
  name: wizzbang-git
  namespace: default
spec:
  type: git
  params:
    - name: url
      value: https://github.com/wizzbangcorp/wizzbang.git
    - name: revision
      value: master
```

Params that can be added are the following:

1. `url`: represents the location of the git repository, you can use this to
   change the repo, e.g. [to use a fork](#using-a-fork)
1. `revision`: Git
   [revision](https://git-scm.com/docs/gitrevisions#_specifying_revisions)
   (branch, tag, commit SHA or ref) to clone. You can use this to control what
   commit [or branch](#using-a-branch) is used. _If no revision is specified,
   the resource will default to `latest` from `master`._

##### Using a fork

The `Url` parameter can be used to point at any git repository, for example to
use a GitHub fork at master:

```yaml
spec:
  type: git
  params:
    - name: url
      value: https://github.com/bobcatfish/wizzbang.git
```

##### Using a branch

The `revision` can be any
[git commit-ish (revision)](https://git-scm.com/docs/gitrevisions#_specifying_revisions).
You can use this to create a git `PipelineResource` that points at a branch, for
example:

```yaml
spec:
  type: git
  params:
    - name: url
      value: https://github.com/wizzbangcorp/wizzbang.git
    - name: revision
      value: some_awesome_feature
```

To point at a pull request, you can use
[the pull requests's branch](https://help.github.com/articles/checking-out-pull-requests-locally/):

```yaml
spec:
  type: git
  params:
    - name: url
      value: https://github.com/wizzbangcorp/wizzbang.git
    - name: revision
      value: refs/pull/52525/head
```

#### Image Resource

An Image resource represents an image that lives in a remote repository. It is
usually used as [a `Task` `output`](concepts.md#task) for `Tasks` that build
images. This allows the same `Tasks` to be used to generically push to any
registry.

Params that can be added are the following:

1. `url`: The complete path to the image, including the registry and the image
   tag
2. `digest`: The
   [image digest](https://success.docker.com/article/images-tagging-vs-digests)
   which uniquely identifies a particular build of an image with a particular
   tag.

For example:

```yaml
apiVersion: pipeline.knative.dev/v1alpha1
kind: PipelineResource
metadata:
  name: kritis-resources-image
  namespace: default
spec:
  type: image
  params:
    - name: url
      value: gcr.io/staging-images/kritis
```

#### Cluster Resource

Cluster Resource represents a Kubernetes cluster other than the current cluster
the pipeline CRD is running on. A common use case for this resource is to deploy
your application/function on different clusters.

The resource will use the provided parameters to create a
[kubeconfig](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)
file that can be used by other steps in the pipeline Task to access the target
cluster. The kubeconfig will be placed in
`/workspace/<your-cluster-name>/kubeconfig` on your Task container

The Cluster resource has the following parameters:

- Name: The name of the Resource is also given to cluster, will be used in the
  kubeconfig and also as part of the path to the kubeconfig file
- URL (required): Host url of the master node
- Username (required): the user with access to the cluster
- Password: to be used for clusters with basic auth
- Token: to be used for authentication, if present will be used ahead of the
  password
- Insecure: to indicate server should be accessed without verifying the TLS
  certificate.
- CAData (required): holds PEM-encoded bytes (typically read from a root
  certificates bundle).

Note: Since only one authentication technique is allowed per user, either a
token or a password should be provided, if both are provided, the password will
be ignored.

The following example shows the syntax and structure of a Cluster Resource:

```yaml
apiVersion: pipeline.knative.dev/v1alpha1
kind: PipelineResource
metadata:
  name: test-cluster
spec:
  type: cluster
  params:
    - name: url
      value: https://10.10.10.10 # url to the cluster master node
    - name: cadata
      value: LS0tLS1CRUdJTiBDRVJ.....
    - name: token
      value: ZXlKaGJHY2lPaU....
```

For added security, you can add the sensitive information in a Kubernetes
[Secret](https://kubernetes.io/docs/concepts/configuration/secret/) and populate
the kubeconfig from them.

For example, create a secret like the following example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: target-cluster-secrets
data:
  cadatakey: LS0tLS1CRUdJTiBDRVJUSUZ......tLQo=
  tokenkey: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbX....M2ZiCg==
```

and then apply secrets to the cluster resource

```yaml
apiVersion: pipeline.knative.dev/v1alpha1
kind: PipelineResource
metadata:
  name: test-cluster
spec:
  type: cluster
  params:
    - name: url
      value: https://10.10.10.10
    - name: username
      value: admin
  secrets:
    - fieldName: token
      secretKey: tokenKey
      secretName: target-cluster-secrets
    - fieldName: cadata
      secretKey: cadataKey
      secretName: target-cluster-secrets
```

Example usage of the cluster resource in a Task:

```yaml
apiVersion: pipeline.knative.dev/v1alpha1
kind: Task
metadata:
  name: deploy-image
  namespace: default
spec:
  inputs:
    resources:
      - name: workspace
        type: git
      - name: dockerimage
        type: image
      - name: testcluster
        type: cluster
  steps:
    - name: deploy
      image: image-wtih-kubectl
      command: ["bash"]
      args:
        - "-c"
        - kubectl --kubeconfig
          /workspace/${inputs.resources.testCluster.Name}/kubeconfig --context
          ${inputs.resources.testCluster.Name} apply -f /workspace/service.yaml'
```

#### Storage Resource

Storage resource represents blob storage, that contains either an object or
directory. Adding the storage resource as an input to a Task will download the
blob and allow the Task to perform the required actions on the contents of the
blob. Blob storage type
[Google Cloud Storage](https://cloud.google.com/storage/)(gcs) is supported as
of now.

##### GCS Storage Resource

GCS Storage resource points to
[Google Cloud Storage](https://cloud.google.com/storage/) blob.

To create a GCS type of storage resource using the `PipelineResource` CRD:

```yaml
apiVersion: pipeline.knative.dev/v1alpha1
kind: PipelineResource
metadata:
  name: wizzbang-storage
  namespace: default
spec:
  type: storage
  params:
    - name: type
      value: gcs
    - name: location
      value: gs://some-bucket
```

Params that can be added are the following:

1. `location`: represents the location of the blob storage.
2. `type`: represents the type of blob storage. Currently there is
   implementation for only `gcs`.
3. `dir`: represents whether the blob storage is a directory or not. By default
   storage artifact is considered not a directory.
   - If artifact is a directory then `-r`(recursive) flag is used to copy all
     files under source directory to GCS bucket. Eg:
     `gsutil cp -r source_dir gs://some-bucket`
   - If artifact is a single file like zip, tar files then copy will be only 1
     level deep(no recursive). It will not trigger copy of sub directories in
     source directory. Eg: `gsutil cp source.tar gs://some-bucket.tar`.

Private buckets can also be configured as storage resources. To access GCS
private buckets, service accounts are required with correct permissions. The
`secrets` field on the storage resource is used for configuring this
information. Below is an example on how to create a storage resource with
service account.

1. Refer to
   [official documentation](https://cloud.google.com/compute/docs/access/service-accounts)
   on how to create service accounts and configuring IAM permissions to access
   bucket.
2. Create a Kubernetes secret from downloaded service account json key

   ```bash
   $ kubectl create secret generic bucket-sa --from-file=./service_account.json
   ```

3. To access GCS private bucket environment variable
   [`GOOGLE_APPLICATION_CREDENTIALS`](https://cloud.google.com/docs/authentication/production)
   should be set so apply above created secret to the GCS storage resource under
   `fieldName` key.

   ```yaml
   apiVersion: pipeline.knative.dev/v1alpha1
   kind: PipelineResource
   metadata:
     name: wizzbang-storage
     namespace: default
   spec:
     type: storage
     params:
       - name: type
         value: gcs
       - name: location
         value: gs://some-private-bucket
       - name: dir
         value: "directory"
     secrets:
       - fieldName: GOOGLE_APPLICATION_CREDENTIALS
         secretName: bucket-sa
         secretKey: service_account.json
   ```

---

Except as otherwise noted, the content of this page is licensed under the
[Creative Commons Attribution 4.0 License](https://creativecommons.org/licenses/by/4.0/),
and code samples are licensed under the
[Apache 2.0 License](https://www.apache.org/licenses/LICENSE-2.0).
