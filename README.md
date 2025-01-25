# Rest in Peace - API Design

A modern, pragmatic approach to designing REST APIs that prioritize **functionality**, **consistency**, and **ease of
use** while eliminating unnecessary complexity.
This eliminates 99% of REST API design headaches.
You may not agree with every guideline and might have a few “WTF” moments, but the goal is to avoid wasting time on
designing or reinventing APIs and just focus on building them.
While Guidelines like [Zalando guidelines](https://opensource.zalando.com/restful-api-guidelines/#api-design-principles)
are comprehensive, they are overly detailed, edge-case-driven, and often add unnecessary complexity, making API creation
cumbersome rather than productive.

---

Table of Contents
=================

* [Overview](#overview)
* [1. General Principles](#1-general-principles)
* [2 Versioning](#2-versioning)
* [3 Methods](#3-methods)
* [4 Path Structure](#4-path-structure)
* [5 Request Structure](#5-request-structure)
    * [Example Filter / Query](#example-filter--query)
    * [Example Update](#example-update)
* [6 Response Structure](#6-response-structure)
    * [Example Response](#example-response)
    * [Example Error Response](#example-error-response)
* [7 Status Codes](#7-status-codes)
* [8 Redirects](#8-redirects)

---

## Overview

1. **POST only** - with operations
2. **Simple Paths** - operations-based paths `/list` or `/save`, `/modify`, `/delete`.
3. **Flat Body** - Only `filter`, `data`, and `meta`.
4. **No HTTP Status Codes** - Use meta for application errors.
5. **UTC Milliseconds** - Simple, universal time format.
6. **No Redirects** - Always return final responses.

## 1. General Principles

1. Consistency: APIs must follow a uniform structure to ensure predictable usage.
2. Simplicity: The API structure must be flat and easy to parse.
3. Security: Sensitive information must not be exposed in query or path parameters. Use the request body instead.
4. Usability: APIs should reduce ambiguity by removing legacy complexity.
5. **No resource hypertext/links in responses** - Use ID's only as urls can change in while processing in the async
   world. It also can create issues when it comes to internal and external URLS.
6. **Authorization** - Use `Bearer` tokens in the `Authorization` header. (oAuth, JWK, etc.)
    * keywords like `Bearer`, `jwt`, or `token` in front of the token are optional and not recommended.

---

## 2 Versioning

* Version must be a simple number, reflected in the path: `/v1`.
* Version increases by any breaking change in behaviour or payload structure.
    * Adding or removing fields in the payload **is not** considered **a breaking change**.

## 3 Methods

* `POST` for all actions

### Optional

* `HEAD` is supported for checking the existence and size of a resource like files.
* `OPTION` is supported for security mechanisms like CORS / preflight requests.

### WHY

* Easier to use and keep consistency
* There are too many exceptions like GET with bodies is unsupported in many tools, proxies, and gateways. Its unclear
  how server or consumers infrastructure will react
    * **RFC2616 GET with body** not
      allowed [HTTP/1.1 spec, section 4.3](https://www.rfc-editor.org/rfc/rfc2616#section-4.3), [HTTP/1.1 spec, section 9.3](https://www.rfc-editor.org/rfc/rfc2616#section-9.3)
    * **RFCs 7230-7237 GET with body** allowed [HTTP 1.1 2014 Spec](https://www.rfc-editor.org/rfc/rfc7231#page-24)
* PATCH makes it hard to track which fields are modified or unset. Many frameworks handle it inconsistently.
* [...] list goes endless on. There is no reason to follow exceptions depending on methods and moon phases.

## 4 Path Structure

* Flat structure: `/<version>/<resource>/<operation>` (`POST /v1/user/list`)
* Operations:
    * **GET**: `/list`
    * **POST**: `/save`
    * **PATCH**: `/modify`
    * **DELETE**: `/delete`
* **No path query or header parameter** - filtering is done by the request body.

### Optional:

* Use an **ID** in the path only when necessary: `/v1/document/1234/pages/list`.
* It's allowed to use additional custom Operations like `/v1/user/activate` as long as they are self-explanatory.

### Why:

* Short, easy to read / understand, **consistent**, and **predictable**. Reduces complexity and ambiguity.
* There is **no big documentation needed** to understand the API.
* No encoding / decoding needed for the path.
* Reduces a lot of different URLs e.g. `/v1/user/1234?name="John"` && `/v1/user/1234/eyes?name="Silver"` ==
  `/v1/user/list`. Filtering is done by the request body. [5 Request Structure](#5-request-structure)
* **Security** - sensitive information is not exposed in the URL while the body is encrypted by HTTPs.
* **Interoperability** - Ensures that the API is compatible with a wide range of tools and platforms.
* **Maintainability** - Simplifies the process of updating and maintaining and designing APIs.

## 5 Request Structure

* **Body** Format: **JSON** or **multipart/form-data** for legacy systems
* **Body** must be **flat** - as possible - no nested objects
* **Body** is structured in following:
    * `filter` Optional when filtering is needed.
    * `data` Optional e.g. for updating or saving data.
    * `binary64` Optional binary data as base64 encoded string.
    * `binary64gz` Optional e.g.binary data as base64 gzip encoded string for download.
* Support for both `binary` types is mandatory

### Optional:

* filtering complex objects can be done by aggregating keys like `{ "person.child.name" : "Yuna" }`

#### Example Filter / Query

```json
{
  "filter": {
    "name": "John",
    "age": 30,
    "sort_by": "created_at",
    "sort_order": "desc"
  }
}
```

#### Example Update

```json
{
  "data": {
    "set": [
      {
        "name": "Yuna"
      }
    ],
    "unset": [
      "age",
      "address"
    ]
  }
}
```

#### Example File Upload

* **Multipart/form-data** is used for file uploads.
* **Data** is sent as a JSON string in the `data` field.

```shell
curl -X POST https://api.example.com/v1/file/upload \
-H "Authorization: Bearer <token>" \
-F "data={\"description\":\"Profile picture\", \"user_id\":\"1234\"}" \
-F "file=@/path/to/profile_picture.png"

```

### Why:

* **Simplicity** - Easy to read / understand, **consistent**, and **predictable**. Reduces complexity and ambiguity.
* **Consistency** - All requests have the same structure. Use one API and know them all.
* **Security** - Sensitive information is not exposed in the URL.

## 6 Response Structure

* **Body** Format: **JSON**.
* **Body** must be **flat** as possible - no nested objects
* **Body** is structured in following:
    * `meta`
        * `code`: HTTP status code (`200`)
        * `message`: **English** explanation of the code
        * `time`: **UTC timestamp** in milliseconds (`1734432019759`)
        * `page`: pagination, current page (`1`)
        * `page_total`: pagination, total pages (`10`)
        * `page_size`: pagination, page size (`100`)
        * `total`: optional pagination total records (`1000`)
    * `data` Optional e.g. for returning data.
    * `error` Optional e.g. for returning error.
    * `binary64` Optional binary data as base64 encoded string.
    * `binary64gz` Optional e.g.binary data as base64 gzip encoded string for download.
* Support for both `binary` types is mandatory

### Optional:

* `meta` can have additional custom fields
* `meta.code` can have custom codes

### Why:

* **Flat design** cause every sub object adds another complexity layer and probably a foreign resource, makes it hard to
  reuse.
* **Pagination** is really important for performance to keep the responses small and fast.
* The timestamp in **UTC** to avoid confusion/ The milliseconds avoids heavy date format guessing and parsing. The APIs
  are made for machines not for humans, there is already something wrong when debugging sensitive body information.
* **Simplicity** - Easy to read / understand, **consistent**, and **predictable**. Reduces complexity and ambiguity.
* **Consistency** - All responses have the same structure. Use one API and know them all.

#### Example Response

```json
{
  "meta": {
    "code": 200,
    "message": "OK",
    "time": 1734432019759,
    "page": 1,
    "page_total": 10,
    "page_size": 100,
    "total": 1000
  },
  "data": [
    {
      "name": "Yuna",
      "age": 30,
      "created_at": 1734430019759,
      "updated_at": 1734432064941,
      "created_by": "Steve",
      "updated_by": "Jobs"
    }
  ]
}
```

#### Example Error Response

```json
{
  "meta": {
    "code": 401,
    "message": "Unauthorized",
    "time": 1734430019759,
    "details": [
      {
        "field": "name",
        "message": "Name is required."
      }
    ]
  },
  "data": []
}
```

## 7 Status Codes

* **No HTTP status codes** - Use `meta` for application errors.
* Only allowed `200` or `201` (error is in the `meta` field.) and `5xx` for any **unexpected** server errors

### WHY

* It's important to differentiate between infrastructure errors and application errors. It makes debugging a lot easier!
* Sending status codes is another unnecessary layer of complexity which the server and consumer would need to agree on.
  Many frameworks are lousing the context on anything else than `2xx` which makes error handling more complex.
* **Consistency** - All responses have the same structure. Use one API and know them all.

## 8 Redirects

* **No redirects** - Always return final responses.

### WHY

Too many unknowns & risks:

* **Security** - redirects can be used for phishing attacks and can contain internal tokens which the consumer can see.
* **Compatability** - Browsers and other Consumers might not follow the redirect, or follow but without cookie handler,
  or block redirects cause of CORS / preflight policies, or does not even understand what to do when a `307` instead of
  `302` is returned.
* **Consistency** - All responses have the same structure. Use one API and know them all.
