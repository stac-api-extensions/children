# STAC API - Children Extension Specification <!-- omit in toc -->

- [Overview](#overview)
- [Link Relations](#link-relations)
- [Endpoints](#endpoints)
- [Pagination](#pagination)
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

This STAC API extension, specifies an endpoint that returns a list of all Catalog and Collection
that are referenced from a Catalog or Collection with the relation type `child`.
For this, it contains using the link with relation type `children` to an endpoint `/children`.
The `/children` endpoint returns *all* the Catalog and Collection objects referenced by these
`child` links.

The purpose is to provide a single resource from which clients can retrieve
the *immediate* children of a Catalog or Collection in an efficient way, similar to STAC API - Collections.
While the `child` link relation already allows for describing these relationships,
this scheme requires a client to retrieve each resource URL to find any information about
the children (e.g., title, description), which can cause significant performance issues in user-facing
applications. Implementers may choose to to return only a subset of fields for each Catalog or Collection,
but the objects must still be valid Catalogs and Collections.

## Link Relations

The following Link relations must exist in a Catalog or Collection with link relation type `child`:

| rel        | Media Type         | From                | Description                                         |
| ---------- | ------------------ | ------------------- | --------------------------------------------------- |
| `children` | `application/json` | STAC API - Children | List of children of this STAC Catalog or Collection |

The following Link relations must exist in the `/children` endpoint response:

| rel      | From                | Description                                                    |
| -------- | ------------------- | -------------------------------------------------------------- | 
| `root`   | STAC Core           | The langding pag (root) URI                                    |
| `parent` | STAC Core           | The (parent) URI of the entity containing the `children` link. |
| `self`   | STAC API - Children | Self reference, i.e. the URI to the `.../children` endpoint.   |

## Endpoints

| Endpoint           | Media Type       | Description                                                         |
| ------------------ | ---------------- | ------------------------------------------------------------------- |
| `GET .../children` | application/json | Object with a list of Catalogs and Collections and a list of Links. |

The response of `GET .../children` must be a JSON object with at least two properties:

- `children`: An array of all child Catalogs and Collections
- `link`: An array of Link Objects

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

### Filtering by Type

Because the `children` array is polymorphic (containing both `Catalog` and `Collection` objects), implementations MAY support a `type` query parameter to allow clients to filter the response to a specific resource type.

* `GET .../children?type=Catalog` - Returns only child Catalogs.
* `GET .../children?type=Collection` - Returns only child Collections.

This is recommended for implementations where backend storage (e.g., Elasticsearch indices) or client logic benefits from strict typing.

## Pagination

The `/children` endpoint supports a pagination mechanism that aligns with
the STAC API - Collections and Features Specification, section
[Collection Pagination](https://github.com/radiantearth/stac-api-spec/blob/v1.0.0/ogcapi-features/README.md#collection-pagination).

To the greatest extent possible, the hierarchy should be structured such that all children can be
retrieved from the endpoint in a single call without pagination.

## Example

Below is a minimal example, but captures the essence.

A STAC API Landing Page (`GET /`) could look like the following.
Please note the `child` and `children` link relations:

```json
{
  "stac_version": "1.1.0",
  "id": "example-stac",
  "title": "A simple STAC API Example, implementing STAC API - Children",
  "description": "This Catalog aims to demonstrate the a simple landing page",
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
          "href": "https://stac-api.example.com"
        },
        {
          "rel": "parent",
          "type": "application/json",
          "href": "https://stac-api.example.com"
        },
        {
          "rel": "self",
          "type": "application/json",
          "href": "https://stac-api.example.com/satellites"
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
          "href": "https://stac-api.example.com"
        },
        {
          "rel": "parent",
          "type": "application/json",
          "href": "https://stac-api.example.com"
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
          "href": "https://stac-api.example/drones/filght-1"
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
          "href": "https://stac-api.example.com/drones"
        }
      ]
    }
  ],
  "links": [
    {
      "rel": "root",
      "type": "application/json",
      "href": "https://stac-api.example.com"
    },
    {
      "rel": "parent",
      "type": "application/json",
      "href": "https://stac-api.example.com"
    },
    {
      "rel": "self",
      "type": "application/json",
      "href": "https://stac-api.example.com/children"
    }
  ]
}
```
