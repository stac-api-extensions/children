# STAC API - Children Extension Specification <!-- omit in toc -->

- [Overview](#overview)
- [Link Relations](#link-relations)
- [Endpoints](#endpoints)
- [Pagination](#pagination)
- [Filtering by Type](#filtering-by-type)
- [Example](#example)

## Overview

- **Title:** Children
- **OpenAPI specification:** [openapi.yaml](openapi.yaml)
  ([rendered version](https://stac-api-extensions.github.io/children/v1.0.0-rc.2))
- **Conformance Classes:**
  - <https://api.stacspec.org/v1.0.0-rc.2/children>
- **Scope:** STAC API - Core
- **[Extension Maturity Classification](https://github.com/radiantearth/stac-api-spec/tree/v1.0.0/README.md#maturity-classification):** Proposal
- **Dependencies**:
  - [STAC API - Core](https://github.com/radiantearth/stac-api-spec/tree/v1.0.0/core) or any later 1.x.x version
- **Owner**: @m-mohr

A STAC API, including its Landing Page, can have links with relation type `child` that point to
Catalogs and Collections. This can be used to create arbitrarily complex hierarchies similar to
[static STAC catalogs](https://github.com/radiantearth/stac-spec). If APIs implement such behavior,
it often also implements the [Browsable Extension](https://github.com/stac-api-extensions/browseable).
The drawback of static catalogs is, that catalogs have to be traversed and a lot of requests for the 
children have to be executed.

This STAC API extension specifies an endpoint that returns a list of all Catalogs and Collections
that are referenced from a Catalog or Collection with the relation type `child`.
For this, it contains a link with relation type `children` which points to an endpoint `/children`.
The `/children` endpoint returns *all* the Catalog and Collection objects referenced by these
`child` links.

The `/children` endpoint is scoped to the `child` link relations only.
The Collections listed at the `/collections` endpoint (referenced from the Landing Page via the `data`
link relation, as defined by STAC API - Collections) are **not** implicitly part of the `/children`
response. A Collection is only included in `/children` if it is explicitly referenced through a `child`
link. Conversely, a Collection may be exposed via both endpoints if it is referenced by both a `data`
(indirectly) and a `child` link.

The purpose is to provide a single resource from which clients can retrieve
the *immediate* children of a Catalog or Collection in an efficient way, similar to STAC API - Collections.
While the `child` link relation already allows for describing these relationships,
this scheme requires a client to retrieve each resource URL to find any information about
the children (e.g., title, description), which can cause significant performance issues in user-facing
applications. Each Catalog and Collection returned in the `children` array must be a complete and valid
Catalog or Collection. Unlike the STAC API - Collections endpoint, implementations must not return reduced
entities (i.e., a subset of the fields); clients can rely on the returned objects being complete and do not
need to request the full entity from its `self` location.

## Link Relations

The following Link relations must exist in a Catalog or Collection with link relation type `child`:

| rel        | Media Type         | From                | Description                                         |
| ---------- | ------------------ | ------------------- | --------------------------------------------------- |
| `children` | `application/json` | STAC API - Children | List of children of this STAC Catalog or Collection |

The following Link relations must exist in the `/children` endpoint response:

| rel      | From                | Description                                                    |
| -------- | ------------------- | -------------------------------------------------------------- | 
| `root`   | STAC Core           | The landing page (root) URI                                    |
| `parent` | STAC Core           | The (parent) URI of the entity containing the `children` link. |
| `self`   | STAC API - Children | Self reference, i.e. the URI to the `.../children` endpoint.   |

The following Link relations must exist in each Catalog and Collection listed in the `children` array:

| rel    | From      | Description                                                                                     |
| ------ | --------- | ----------------------------------------------------------------------------------------------- |
| `self` | STAC Core | Self reference, i.e. the absolute URI at which the individual Catalog or Collection is located. |

The `self` link is required so that clients can unambiguously determine the location of each entity and
correlate the entities returned by the `/children` endpoint with the corresponding STAC entities (e.g., the
resources referenced by the `child` link relations of the parent).

## Endpoints

| Endpoint           | Media Type       | Description                                                         |
| ------------------ | ---------------- | ------------------------------------------------------------------- |
| `GET .../children` | application/json | Object with a list of Catalogs and Collections and a list of Links. |

The response of `GET .../children` must be a JSON object with at least two properties:

- `children`: An array of all child Catalogs and Collections
- `links`: An array of Link Objects

The children endpoint can occur at any depth, for example:
- for a landing page (`GET /`),
  the children endpoint would be available at `GET /children`
- for a collection available at `GET /missions/sentinel-2`,
  the children endpoint would be available at `GET /missions/sentinel-2/children`
- for a catalog available at `GET /catalogs/{id}`,
  the children endpoint would be available at `GET /catalogs/{id}/children`

Note that although the endpoint in general allows to return both Catalogs and Collections,
implementations may only return a single type if the children only consist of a single type.

It is considered a best practice to structure the hierarchy in a way that the children for each 
individual request only consist of a single type.

## Pagination

The `/children` endpoint supports a pagination mechanism that aligns with
the STAC API - Collections and Features Specification, section
[Collection Pagination](https://github.com/radiantearth/stac-api-spec/blob/v1.0.0/ogcapi-features/README.md#collection-pagination).

To the greatest extent possible, the hierarchy should be structured such that all children can be
retrieved from the endpoint in a single call without pagination.

## Filtering by Type

This section describes an OPTIONAL conformance class that adds a `type` query parameter for filtering:

- **Conformance Class:** <https://api.stacspec.org/v1.0.0-rc.2/children#type-filter>

Because the `children` array is polymorphic (containing both `Catalog` and `Collection` objects),
implementations MAY support a `type` query parameter to allow clients to filter the response to a specific resource type.
Results SHALL be *filtered*, a conversion between Catalog and Collection is not foreseen.

- `GET .../children?type=Catalog` - Returns only child Catalogs.
- `GET .../children?type=Collection` - Returns only child Collections.

This is recommended for implementations where backend storage (e.g., Elasticsearch indices) or client logic benefits from strict typing.

## Example

Below is a minimal example, but captures the essence.

A STAC API Landing Page (`GET /`) could look like the following.
Please note the `child` and `children` link relations:

```json
{
  "stac_version": "1.1.0",
  "id": "example-stac",
  "title": "A simple STAC API Example, implementing STAC API - Children",
  "description": "This Catalog aims to demonstrate a simple landing page",
  "type": "Catalog",
  "conformsTo": [
    "https://api.stacspec.org/v1.0.0/core",
    "https://api.stacspec.org/v1.0.0-rc.2/children",
    "https://api.stacspec.org/v1.0.0-rc.3/browseable"
  ],
  "links": [
    {
      "rel": "self",
      "type": "application/json",
      "href": "https://stac-api.example"
    },
    {
      "rel": "root",
      "type": "application/json",
      "href": "https://stac-api.example"
    },
    {
      "rel": "service-desc",
      "type": "application/vnd.oai.openapi+json;version=3.0",
      "href": "https://stac-api.example/api"
    },
    {
      "rel": "service-doc",
      "type": "text/html",
      "href": "https://stac-api.example/api.html"
    },
    {
      "rel": "children",
      "type": "application/json",
      "href": "https://stac-api.example/children"
    },
    {
      "rel": "child",
      "type": "application/json",
      "title": "Satellite Imagery",
      "href": "https://stac-api.example/satellites"
    },
    {
      "rel": "child",
      "type": "application/json",
      "title": "Drone Imagery",
      "href": "https://stac-api.example/drones"
    }
  ]
}
```

The `GET /children` endpoint response object could look as follows:

```json
{
  "children": [
    {
      "stac_version": "1.1.0",
      "id": "satellites",
      "title": "Satellite Imagery",
      "description": "This category contains data captured by satellites (currently empty).",
      "type": "Catalog",
      "links": [
        {
          "rel": "root",
          "type": "application/json",
          "href": "https://stac-api.example"
        },
        {
          "rel": "parent",
          "type": "application/json",
          "href": "https://stac-api.example"
        },
        {
          "rel": "self",
          "type": "application/json",
          "href": "https://stac-api.example/satellites"
        }
      ]
    },
    {
      "stac_version": "1.1.0",
      "id": "drones",
      "title": "Drone Imagery",
      "description": "This category contains data captured by drones and other airbone vehicles (from two flights).",
      "type": "Catalog",
      "links": [
        {
          "rel": "root",
          "type": "application/json",
          "href": "https://stac-api.example"
        },
        {
          "rel": "parent",
          "type": "application/json",
          "href": "https://stac-api.example"
        },
        {
          "rel": "children",
          "type": "application/json",
          "href": "https://stac-api.example/drones/children"
        },
        {
          "rel": "child",
          "type": "application/json",
          "title": "Flight 1",
          "href": "https://stac-api.example/drones/flight-1"
        },
        {
          "rel": "child",
          "type": "application/json",
          "title": "Flight 2",
          "href": "https://stac-api.example/drones/flight-2"
        },
        {
          "rel": "self",
          "type": "application/json",
          "href": "https://stac-api.example/drones"
        }
      ]
    }
  ],
  "links": [
    {
      "rel": "root",
      "type": "application/json",
      "href": "https://stac-api.example"
    },
    {
      "rel": "parent",
      "type": "application/json",
      "href": "https://stac-api.example"
    },
    {
      "rel": "self",
      "type": "application/json",
      "href": "https://stac-api.example/children"
    }
  ]
}
```
