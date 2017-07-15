# Deckhand Design

## Purpose

Deckhand is a document-based configuration storage service built with
auditability and validation in mind.

## Essential Functionality

* layering - helps reduce duplication in configuration while maintaining
  auditability across many sites
* substitution - provides separation between secret data and other
  configuration data, while allowing a simple, file-like interface for
  clients
  * These documents are designed to present consumer services with ready-to-use
    documents which may include secrets.
* revision history - improves auditability and enables services to provide
  functional validation of a well-defined collection of documents that are
  meant to operate together
* validation - allows services to implement and register different kinds of
  validations and report validation errors

## Documents

All configuration data is stored entirely as structured documents, for which
schemas must be registered.

### Document Format

The document format is modeled loosely after Kubernetes practices. The top
level of each document is a dictionary with 3 keys: `schema`, `metadata`, and
`data`.

* `schema` - Defines the name of the JSON schema to be used for validation.
  Must have the form: `<namespace>/<kind>/<version>`, where the meaning of
  each component is:
  * `namespace` - Identifies the owner of this type of document. The
    values `deckhand` and `metadata` are reserved for internal use.
  * `kind` - Identifies a type of configuration resource in the namespace.
  * `version` - Describe the version of this resource, e.g. "v1".
* `metadata` - Defines details that Deckhand will inspect and understand. There
  are multiple schemas for this section as discussed below. All the various
  types of metadata include a `name` field which must be unique for each
  document `schema`.
* `data` - Data to be validated by the schema described by the `schema`
  field. Deckhand only interacts with content here as instructed to do so by
  the `metadata` section. The form of this section is considered to be
  completely owned by the `namespace` in the `schema`.

#### Document Metadata

There are 3 supported kinds of document metadata. Documents with `Document`
metadata are the most common, and are used for normal configuration data.
Documents with `Control` metadata are used to customize the behavior of
Deckhand. Documents with `Tombstone` metadata are used to delete pre-existing
documents with either `Document` or `Control` metadata.

##### schema: metadata/Document/v1

This type of metadata allows the following metadata hierarchy:

* `name` - string, required - Unique within a revision for a given `schema`.
* `storagePolicy` - string, required - Either `cleartext` or `encrypted`.
* `layeringDefinition` - dict, required - Specifies
  * `abstract` - boolean, required - An abstract document is not expected to
    pass schema validation after layering and substitution are applied.
    Non-abstract (concrete) documents are.
  * `layer` - string, required - References a layer in the `LayeringPolicy`
    control document.
  * `parentSelector` - labels, optional - Used to construct document chains for
    executing merges.  See the Layering section below for details.
  * `actions` - list, optional - A sequence of actions to apply this documents
    data during the merge process.
    * `method` - string, required - How to layer this content.
    * `path` - string, required - What content in this document to layer onto
      parent content.
* `substitutions` - list, optional - A sequence of substitutions to apply. See
  the Substitutions section for additional details.
  * `dest` - dict, required - A description of the inserted content destination.
    * `path` - string, required - The JSON path where the data will be placed
      into the `data` section of this document.
  * `src` - dict, required - A description of the inserted content source.
    * `schema` - string, required - The `schema` of the source document.
    * `name` - string, required - The `metadata.name` of the source document.
    * `path` - string, required - The JSON path from which to extract data in
      the source document relative to its `data` section.

Here is a fictitious example of a complete document which illustrates all the
valid fields in the `metadata` section.

```yaml
---
schema: some-service/ResourceType/v1
metadata:
  schema: metadata/Document/v1
  name: unique-name-given-schema
  storagePolicy: cleartext
  labels:
    genesis: enabled
    master: enabled
  layeringDefinition:
    abstract: true
    layer: region
    parentSelector:
      labels: parents
      must: have
    actions:
      - method: merge
        path: .path.to.merge.into.parent
      - method: delete
        path: .path.to.delete
  substitutions:
    - dest:
        path: .substitution.target
      src:
        schema: another-service/SourceType/v1
        name: name-of-source-document
        path: .source.path
data:
  path:
    to:
      merge:
        into:
          parent:
            foo: bar
          ignored:  # Will not be part of the resultant document after layering.
            data: here
  substitution:
    target: null  # Paths do not need to exist to be specified as substitutiond destinations.
```


##### schema: metadata/Control/v1

This schema is the same as the `Document` schema, except it omits the
`storagePolicy`, `layeringDefinition`, and `substitutions` keys, as these
actions are not supported on `Control` documents.

The complete list of valid `Control` document kinds is specified below along
with descriptions of each document kind.

##### schema: metadata/Tombstone/v1

The only valid key in a `Tombstone` metadata section is `name`.  Additionally,
the top-level `data` section should be omitted.

### Layering

When queried, Deckhand assembles the chain of partial documents in the order
specified in the `deckhand/LayeringPolicy/v1` document. Each chain of partial
documents is made up only of documents using a single `schema`.

<!-- TODO: describe document chain construction w/ diagrams -->

During each layering step, the list of `actions` will be applied in order.
Supported actions are:

* `merge` - a "deep" merge that layers new and modified data onto existing data
* `replace` - overwrite data at the specified path and replace it with the data
  given in this document
* `delete` - remove the data at the specified path

<!-- TODO: Add examples and possibly figures -->

### Substitution

Concrete documents can be used as a source of substitution into other
documents. This substitution is layer-independent.

This is primarily designed as a mechanism for inserting secrets into
configuration documents, but works for unencrypted source documents as well.

<!-- TODO: Add example(s) maybe simple + an example with source layering? -->

### Control Documents

Control documents (documents which have `metadata.schema=metadata/Control/v1`),
are special, and are used to control the behavior of Deckhand at runtime.  Only
the following types control documents are allowed.

#### DataSchema

`DataSchema` documents are used by various services to register new schemas
that Deckhand can use for validation. No `DataSchema` documents with names
beginning with `deckhand/` or `metadata/` are allowed.

<!-- TODO: give valid, tiny schema as example -->

```yaml
---
schema: deckhand/DataSchema/v1
metadata:
  schema: metadata/Control/v1
  name: promenade/Node/v1  # Specifies the documents to be used for validation.
  labels:
    application: promenade
data:  # Valid JSON Schema is expected here.
  $schema: http://blah
```

#### LayeringPolicy

Only one `LayeringPolicy` document can exist within the system at any time.
It is an error to attempt to insert a new `LayeringPolicy` document if it has
a different `metadata.name` than the existing document. If the names match,
it is treated as an update to the existing document.

This document defines the strict order in which documents are merged together
from their component parts. It should result in a validation error if a
document refers to a layer not specified in the `LayeringPolicy`.

```yaml
---
schema: deckhand/LayeringPolicy/v1
metadata:
  schema: metadata/Control/v1
  name: layering-policy
data:
  layerOrder:
    - global
    - site-type
    - region
    - site
    - force
```

#### ValidationPolicy

Unlike `LayeringPolicy`, many `ValidationPolicy` documents are allowed. This
allows services to check whether a particular revision (described below) of
documents meets a configurable set of validations without having to know up
front the complete list of validations.

Deckhand provides validation of all concrete documents given their schema
under the name `deckhand-schema-validation`. Since validations may indicate
interactions with external and changing circumstances, an optional
`expiresAfter` key may be specified for each validation as an ISO8601
duration. If no `expiresAfter` is specified, a successful validation does not
expire.

```yaml
---
schema: deckhand/ValidationPolicy/v1
metadata:
  schema: metadata/Control/v1
  name: site-deploy-ready
data:
  validations:
    - name: deckhand-schema-validation
    - name: drydock-site-validation
      expiresAfter: P1W
    - name: promenade-site-validation
      expiresAfter: P1W
    - name: armada-deployability-validation
```

### Provided Utility Document Kinds

These are documents that use the `Document` metadata schema, but live in the
`deckhand` namespace.

#### Certificate

```yaml
---
schema: deckhand/Certificate/v1
metadata:
  schema: metadata/Document/v1
  name: application-api
  storagePolicy: cleartext
data: |-
  -----BEGIN CERTIFICATE-----
  MIIDYDCCAkigAwIBAgIUKG41PW4VtiphzASAMY4/3hL8OtAwDQYJKoZIhvcNAQEL
  ...snip...
  P3WT9CfFARnsw2nKjnglQcwKkKLYip0WY2wh3FE7nrQZP6xKNaSRlh6p2pCGwwwH
  HkvVwA==
  -----END CERTIFICATE-----
```

#### CertificateKey

```yaml
---
schema: deckhand/CertificateKey/v1
metadata:
  schema: metadata/Document/v1
  name: application-api
  storagePolicy: encrypted
data: |-
  -----BEGIN RSA PRIVATE KEY-----                                 
  MIIEpQIBAAKCAQEAx+m1+ao7uTVEs+I/Sie9YsXL0B9mOXFlzEdHX8P8x4nx78/T
  ...snip...
  Zf3ykIG8l71pIs4TGsPlnyeO6LzCWP5WRSh+BHnyXXjzx/uxMOpQ/6I=
  -----END RSA PRIVATE KEY-----
```

#### Passphrase

```yaml
---
schema: deckhand/Passphrase/v1
metadata:
  schema: metadata/Document/v1
  name: application-admin-password
  storagePolicy: encrypted
data: some-password
```

## Revision History

Documents will be ingested in batches which will be given a revision index.
This provides a common language for describing complex validations on sets of
documents.

Revisions can be thought of as commits in a linear git history, thus looking
at a revision includes all content from previous revision.

## Validation

Services can report success for validation types for a given revision.

## API

This API will only support YAML as a serialization format. Since the IETF
does not provide an official media type for YAML, this API will use
`application/x-yaml`.

### POST `/documents`

Accepts a multi-document YAML body and creates a new revision which adds
those documents. Updates are detected based on exact match to an existing
document of `schema` + `metadata.name`. Documents are "deleted" by including
documents with the tombstone metadata schema, such as:

```yaml
schema: any-namespace/AnyKind/v1
metadata:
  schema: metadata/Tombstone/v1
  name: name-to-delete
```

This endpoint is the only way to add, update, and delete documents. This
triggers Deckhand's internal schema validations for all documents.

### GET `/revisions/{revision_id}/documents`

Returns a multi-document YAML response containing all the documents matching
the filters specified via query string parameters. Returned documents will be
fully merged and substituted.

* `schema` - string, optional - The top-level `schema` field to select.
* `metadata.name` - string, optional
* `metadata.layeringDefinition.abstract` - string, optional - Valid values are
  the empty string, "true" and "false".  Defaults to "false". Specifying the empty string removes this filter.
* `metadata.label` - string, optional, repeatable - Uses the format
  `metadata.label=key=value`. Repeating this parameter indicates all
  requested labels must apply (AND not OR).

### GET `/revisions`

Lists existing revisions and reports basic details including a summary of
validation status for each `deckhand/ValidationPolicy` that is part of that
revision.

<!-- NOTE: probably need to paginate this -->

Sample response:

```yaml
---
- id: 0
  url: https://deckhand/revisions/0
  createdAt: 2017-07-14T021:23Z
  validationPolicies:
    site-deploy-validation:
      status: failed
...
```

### GET `/revisions/{{revision_id}}`

Get a detailed description of a particular revision.

```yaml
---
id: 0
href: https://deckhand/revisions/0
createdAt: 2017-07-14T021:23Z
validationPolicies:
  site-deploy-validation:
    url: https://deckhand/revisions/0/documents?schema=deckhand/ValidationPolicy/v1&name=site-deploy-validation
    status: failed
    validationNames:
      - deckhand-schema-validation
      - drydock-site-validation
      - promenade-site-validation
      - armada-deployability-validation
validations:
  deckhand-schema-validation:
    url: https://deckhand/revisions/0/validations/deckhand-schema-validation/0
    status: success
  drydock-site-validation:
    status: missing
  promenade-site-validation:
    url: https://deckhand/revisions/0/validations/promenade-site-validation/0
    status: expired
  armada-deployability-validation:
    url: https://deckhand/revisions/0/validations/armada-deployability-validation/0
    status: failed
...
```

### POST `/revisions/{{revision_id}}/validations/{{name}}`

Add the results of a validation for a particular revision.

<!-- TODO: success example -->
<!-- TODO: fail example -->

### GET `/revisions/{{revision_id}}/validations/{{name}}`

Gets the list of validation entry summaries that have been posted.

<!-- TODO: expand -->

### GET `/revisions/{{revision_id}}/validations/{{name}}/entries/{{entry_id}}`

Gets the full details of a particular validation entry, including all posted
error details.

<!-- TODO: expand -->