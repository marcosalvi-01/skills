# EPOS DB-API reference

## Overview

The db-api library provides entity-specific APIs for persisting and retrieving EPOS metadata DTOs. DTOs live in `org.epos.eposdatamodel.*` and are mapped to JPA entities under `model.*` by API classes.

## Core classes and packages

- `abstractapis.AbstractAPI` is the main entry point; use `retrieveAPI(EntityNames.*)`.
- Entity APIs live in `metadataapis.*` and are resolved by `EntityNames`.
- `commonapis.LinkedEntityAPI` resolves and creates linked entities.
- `commonapis.VersioningStatusAPI` handles version records and status transitions.
- `dao.EposDataModelDAO` manages JPA access and identifier lookup.
- `org.epos.handler.dbapi.service.EntityManagerService` configures JPA and connection pooling.

## Entity names

Use `metadataapis.EntityNames` values when requesting APIs or creating `LinkedEntity` references. Some values are not mapped by `AbstractAPI.retrieveAPI(...)` and will return `null`.

Mapped entity types include: `ATTRIBUTION`, `ADDRESS`, `DOCUMENTATION`, `ELEMENT`, `IDENTIFIER`, `SOFTWAREAPPLICATIONINPUTPARAMETER`, `SOFTWAREAPPLICATIONOUTPUTPARAMETER`, `QUANTITATIVEVALUE`, `LOCATION`, `PERIODOFTIME`, `CATEGORY`, `CATEGORYSCHEME`, `CONTACTPOINT`, `DATAPRODUCT`, `DISTRIBUTION`, `EQUIPMENT`, `FACILITY`, `MAPPING`, `OUTPUTMAPPING`, `PAYLOAD`, `OPERATION`, `ORGANIZATION`, `PERSON`, `SOFTWAREAPPLICATION`, `SOFTWARESOURCECODE`, `WEBSERVICE`.

Unmapped or special values include: `LEGALNAME`, `RELATION`.

## Identifier and versioning rules

- Every DTO extends `EPOSDataModelEntity` and carries `uid`, plus versioning fields from `VersioningAndApproval`.
- `instanceId` identifies a specific versioned instance and changes when a new version is created.
- `metaId` identifies the logical entity across versions and statuses.
- `uid` is the business identifier; if set, APIs may perform smart lookup and reuse existing entities.
- `versionId` identifies a specific version record and is generated if not provided.
- Lookup priority is `instanceId` > `metaId` > `uid` > `versionId`.
- `DRAFT` updates the current draft or creates a new draft when a published record exists.
- Non-`DRAFT` statuses update the existing version record in place.

## LinkedEntity rules

- Linked fields are `List<LinkedEntity>` (for example `DataProduct.distribution`).
- Provide `entityType` plus one stable identifier (`instanceId`, `metaId`, or `uid`).
- Relation updates are diff-based: remove missing items, create new items, and update links.
- `relationFromUpdate`/`relationToUpdate` act as a swap pair in `create`.
- `LinkedEntityAPI.retrieveFromLinkedEntity(...)` may create entities if the entity type is unknown; validate `entityType`.

## Connection and pooling configuration

Environment variables used by `EntityManagerService`:

- `PERSISTENCE_NAME` (default `EPOSDataModel`)
- `POSTGRESQL_CONNECTION_STRING` or parts: `POSTGRESQL_HOST`, `POSTGRESQL_DBNAME`, `POSTGRESQL_USERNAME`, `POSTGRESQL_PASSWORD`
- Pool tuning: `CONNECTION_POOL_MAX_SIZE`, `CONNECTION_MAX_LIFETIME`, `CONNECTION_TEST_IDLE_INTERVAL_TIME`, `CONNECTION_LEAK_DETECTION_THRESHOLD`

Defaults:

- `CONNECTION_POOL_MAX_SIZE=10`
- `CONNECTION_MAX_LIFETIME=1800000`
- `CONNECTION_TEST_IDLE_INTERVAL_TIME=600000`
- `CONNECTION_LEAK_DETECTION_THRESHOLD=600000`

## Common pitfalls

- Using `model.*` JPA classes directly instead of DTOs in `org.epos.eposdatamodel.*`.
- Missing `entityType` on `LinkedEntity` references.
- Using `DRAFT` when you intended to update a published record.
- Assuming `uid` and `metaId` are both respected; `metaId` wins during lookup.
- Treating `retrieveAPI` as universal; unmapped entity types return `null`.
