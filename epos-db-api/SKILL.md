---
name: epos-db-api
description: Integrate and operate the EPOS DB-API Java library for metadata persistence and retrieval. Use when working with db-api entities, AbstractAPI/entity-specific APIs, linked-entity resolution, versioning/status behavior, or DB-API environment configuration and troubleshooting.
---

# EPOS DB-API

## Overview

Use this skill to integrate or troubleshoot the EPOS DB-API Java library: create/update/retrieve metadata entities, resolve linked entities, and reason about versioning/status behavior and environment configuration.

## Quick start

1. Confirm connection configuration and pooling vars for `EntityManagerService`.
2. Use DTOs from `org.epos.eposdatamodel.*` and obtain APIs via `AbstractAPI.retrieveAPI(EntityNames.*)`.
3. Create or update entities with the correct `StatusType`, and provide stable identifiers (`uid`, `metaId`, or `instanceId`).
4. Manage relations via `List<LinkedEntity>` and the `LinkedEntityAPI` helper.

## Common tasks

### Create or update an entity

```java
AbstractAPI api = AbstractAPI.retrieveAPI(EntityNames.DATAPRODUCT.name());
DataProduct dp = new DataProduct();
dp.setUid("dp-123");
dp.setStatus(StatusType.PUBLISHED);
dp.setEditorId("user-1");
dp.setChangeComment("Initial publish");
api.create(dp, StatusType.PUBLISHED, null, null);
```

### Resolve a linked entity

```java
DataProduct dp = (DataProduct) AbstractAPI.retrieveAPI(EntityNames.DATAPRODUCT.name()).retrieveByUID("dp-123");
LinkedEntity linked = dp.getDistribution().get(0);
Object resolved = LinkedEntityAPI.retrieveFromLinkedEntity(linked);
```

### Replace a single relation

```java
AbstractAPI api = AbstractAPI.retrieveAPI(EntityNames.DISTRIBUTION.name());
Distribution dist = (Distribution) api.retrieveByUID("dist-1");
LinkedEntity oldLink = dist.getAccessService().get(0);
LinkedEntity newLink = new LinkedEntity().entityType(EntityNames.WEBSERVICE.name()).uid("ws-new");
api.create(dist, null, oldLink, newLink);
```

## What to load next

- Read `references/epos-db-api.md` for entity names, identifier/versioning rules, connection variables, and common pitfalls.
