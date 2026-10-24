# API Security Testing: Methodology and Lab Writeups

Notes and practical walkthroughs from PortSwigger Web Security Academy's API Testing module, covering API recon, hidden endpoint discovery, HTTP method manipulation, and mass assignment vulnerabilities.

## Table of Contents

- [What is an API](#what-is-an-api)
- [API Reconnaissance](#api-reconnaissance)
- [API Documentation](#api-documentation)
- [HTTP Methods in REST APIs](#http-methods-in-rest-apis)
- [Identifying Supported Content Types](#identifying-supported-content-types)
- [Lab Walkthrough 1: Exploiting an API Endpoint Using Documentation](#lab-walkthrough-1-exploiting-an-api-endpoint-using-documentation)
- [Lab Walkthrough 2: Finding and Exploiting an Unused API Endpoint](#lab-walkthrough-2-finding-and-exploiting-an-unused-api-endpoint)
- [Using Intruder to Find Hidden Endpoints](#using-intruder-to-find-hidden-endpoints)
- [Finding Hidden Parameters](#finding-hidden-parameters)
- [Mass Assignment Vulnerabilities](#mass-assignment-vulnerabilities)
- [Prevention](#prevention)
- [Key Takeaways](#key-takeaways)

## What is an API

An API (Application Programming Interface) is the layer that lets two systems exchange data without the client needing to know how the server's internal logic works. A client sends a request to a defined endpoint, the server processes it, and returns a structured response, typically JSON or XML.

In a security testing context, APIs matter because they frequently expose more functionality than the associated front-end UI reveals. Endpoints referenced only in JavaScript, mobile apps, or internal documentation often lack the same input validation, authentication checks, or rate limiting as user-facing flows. This makes API recon and endpoint discovery a core part of any web application assessment.

## API Reconnaissance

The first step in API testing is mapping the attack surface. This involves:

1. **Identifying endpoints** – locations where the API receives requests about a specific resource (e.g. `/api/books`, `/api/products/1/price`).
2. **Understanding how to interact with each endpoint** – required and optional parameters, accepted HTTP methods, accepted content types, rate limits, and authentication requirements.

A useful recon pattern: if you find a deep endpoint such as `/api/swagger/v1/users/123`, walk back up the path to check for documentation or additional functionality at each level:

```
/api/swagger/v1/users/123
/api/swagger/v1
/api/swagger
/api
```

## API Documentation

APIs are commonly documented for integration purposes, in both human-readable form (explanations, examples) and machine-readable form (OpenAPI/Swagger JSON or YAML, designed for tooling).

**Where to look for documentation:**
- `/api`
- `/swagger/index.html`
- `/openapi.json`

If documentation isn't linked anywhere discoverable, it may still exist. Burp Scanner can crawl the application to surface references to it, and Burp's built-in browser allows manual exploration. A wordlist run through Intruder against common documentation paths is also effective when nothing turns up organically.

**Working with machine-readable documentation:**
- Burp Scanner can crawl and audit OpenAPI specs directly.
- The OpenAPI Parser BApp parses specs into a navigable format within Burp.
- Dedicated tools like Postman or SoapUI can also consume and test documented endpoints.

Note that documentation can be inaccurate or stale. It's worth cross-referencing documented behavior against what the application actually does by browsing it directly, since JavaScript files frequently reference endpoints never triggered through the standard UI. The JS Link Finder BApp automates extraction of these references at scale.

## HTTP Methods in REST APIs

| Method | Purpose |
|---|---|
| `GET` | Retrieve a resource |
| `POST` | Create a new resource |
| `PUT` | Replace an entire resource with the submitted data |
| `PATCH` | Apply a partial update to specific fields of a resource |
| `DELETE` | Remove a resource |
| `OPTIONS` | Retrieve the list of methods permitted on a resource |

An endpoint's documented or front-end-exposed method isn't necessarily the only one it accepts. Testing alternate methods against the same endpoint is a standard technique for uncovering hidden functionality, since authorization logic is sometimes applied inconsistently across methods (for example, a `PUT` request might be properly access-controlled while a `PATCH` to the same resource isn't).

Burp Intruder's built-in HTTP verbs payload list can automate cycling through methods against a target endpoint.

> **Caution:** When testing different HTTP methods, target low-priority objects first. Method switching can have real side effects, altering or deleting live data, or creating excessive records, if run against production-critical resources.

## Identifying Supported Content Types

API behavior can change based on the `Content-Type` of the request body. Testing different content types against the same endpoint can:

- Trigger verbose error messages that leak implementation details.
- Bypass input validation that was only implemented for one format.
- Expose inconsistent processing logic between formats (e.g. an endpoint hardened against injection in JSON but not in XML).

The Content Type Converter BApp automates converting request bodies between JSON and XML for this kind of testing.

## Lab Walkthrough 1: Exploiting an API Endpoint Using Documentation

**Objective:** Use exposed API documentation to find and abuse a user deletion endpoint.

**Steps:**

1. Capture a `PATCH` request from the application in **Proxy > HTTP History** and send it to Repeater.
2. Use the base path of the captured endpoint to probe for exposed documentation, progressively shortening the path:
   ```
   /api/user/wiener
   /api/user
   /api
   ```
3. Locate the documentation endpoint and identify an endpoint not used by the front end, such as a user deletion route.
4. Construct and send a request to that endpoint directly in Repeater.
5. Right-click the response and select **Show response in browser** to render the result and confirm the action (e.g. user deletion) succeeded.

**Key lesson:** exposed or discoverable API documentation can reveal functionality that was never wired into the UI, but is still live and reachable on the backend. Documentation discovery should be a standard first step in any API assessment.

## Lab Walkthrough 2: Finding and Exploiting an Unused API Endpoint

**Objective:** Manipulate a product price via an undocumented `PATCH` endpoint to purchase an item for free.

**Steps:**

1. Browse the application and click into a product. In **Proxy > HTTP History**, identify the API request tied to that product, e.g. `/api/products/3/price`.
2. Send the request to Repeater.
3. Change the method from `GET` to `OPTIONS` and resend. The response's `Allow` header reveals which methods the endpoint accepts, in this case, `GET` and `PATCH`.
4. Change the method to `PATCH` and resend. An `Unauthorized` response indicates the endpoint requires authentication.
5. Log in via Burp's browser using valid low-privilege credentials (`wiener:peter`).
6. Navigate to the target product (the "Lightweight l33t Leather Jacket"), capture its price endpoint request, and send it to Repeater.
7. Resend as `PATCH`. The response errors on `Content-Type`, indicating the server expects `application/json`.
8. Add the header `Content-Type: application/json` and send an empty JSON body `{}`. The server responds that the `price` field is required, this error message effectively defines the expected schema.
9. Send the corrected body:
   ```json
   {"price": 0}
   ```
10. Reload the product page to confirm the price now reads `$0.00`.
11. Add the item to the basket and place the order to solve the lab.

**Key lesson:** server error messages are a reconnaissance tool, not just noise. Each error in this chain (`Content-Type` mismatch, missing field) incrementally revealed the exact request shape the endpoint expected.

### Practical Debugging Notes from This Lab

A few non-obvious gotchas came up reproducing this in Repeater that are worth documenting for future reference:

- **`Content-Length` must match the actual byte length of the body.** A stale or incorrect value (left over from editing the request) causes the server to read a truncated or malformed body, producing misleading errors (either a 500 crash or a generic validation failure, depending on the backend) rather than a clean parse error. In Burp Repeater, right-click → **Update Content-Length header** keeps this correct automatically after editing the body.
- **The blank line separating headers from the body is load-bearing.** If a header like `Content-Type` ends up below that line, it's interpreted as part of the body instead of as a header, breaking JSON parsing and typically returning a `500 Internal Server Error` rather than a usable validation message.
- **Malformed JSON produces a generic validation error, not a parse error, on some backends.** A body like `{"price": 0}` with stray characters or incorrect structure can surface as `'price' parameter must be a valid non-negative integer` even when a valid value was intended, because the parser never successfully extracted the field. When a value that should clearly pass still fails validation, check the raw body for syntax issues before assuming the server-side logic is at fault.

## Using Intruder to Find Hidden Endpoints

Once initial endpoints are mapped, Intruder can brute-force sibling endpoints that follow the same naming convention. For example, given:

```
PUT /api/user/update
```

a payload could be placed at the `/update` position with a wordlist of common action names (`delete`, `add`, `remove`, `disable`, etc.) to surface endpoints that exist but aren't referenced anywhere in the visible application.

Effective wordlists combine common REST/API naming conventions with terms specific to the target application, drawn from earlier recon (e.g. naming patterns seen in JS files or documentation).

## Finding Hidden Parameters

Beyond endpoints, individual parameters can be hidden, supported by the backend but never exposed in any request the front end actually sends. Tools for this:

- **Burp Intruder** – brute-force parameter names using a wordlist, either replacing existing parameters or appending new ones.
- **Param Miner BApp** – automatically guesses up to 65,536 parameter names per request, weighted toward names relevant to the target based on scope data.
- **Content Discovery** – surfaces content and parameters not linked from anything reachable by normal browsing.

## Mass Assignment Vulnerabilities

Mass assignment (also called auto-binding) occurs when a framework automatically binds incoming request parameters directly to fields on an internal object, without an explicit allowlist. This can expose parameters that were never intended to be user-controllable.

### Identifying the Vulnerability

Compare what a `PATCH` request sends against what a corresponding `GET` request returns for the same object.

Request sent by the application:
```json
PATCH /api/users/
{
    "username": "wiener",
    "email": "wiener@example.com"
}
```

Object returned by the API:
```json
GET /api/users/123
{
    "id": 123,
    "name": "John Doe",
    "email": "john@example.com",
    "isAdmin": "false"
}
```

The `GET` response exposes `id` and `isAdmin`, fields the `PATCH` request never explicitly sends. If the backend binds request bodies to this object without restricting which fields can be set, both become candidate attack parameters.

### Testing the Vulnerability

1. Add the suspected hidden field with a valid-looking value:
   ```json
   {"username": "wiener", "email": "wiener@example.com", "isAdmin": false}
   ```
2. Send a second request with an intentionally invalid value for the same field:
   ```json
   {"username": "wiener", "email": "wiener@example.com", "isAdmin": "foo"}
   ```
3. Compare the two responses. If the valid value is accepted silently but the invalid value triggers an error, that's a strong signal the field is being processed by the backend, not ignored.
4. If confirmed, attempt the exploit:
   ```json
   {"username": "wiener", "email": "wiener@example.com", "isAdmin": true}
   ```
5. Verify impact by browsing the application as the affected user and checking for elevated (e.g. admin) functionality.

**Key lesson:** mass assignment turns an informational disclosure (seeing extra fields in a `GET` response) directly into a privilege escalation vector, if the server doesn't enforce which fields a client is allowed to set.

## Prevention

From a secure design standpoint, the following practices mitigate the issues covered above:

- Don't expose API documentation publicly unless the API is intended for external consumption.
- Keep documentation current, stale documentation misleads legitimate testers and developers alike, while attackers rely on actual observed behavior regardless.
- Enforce an explicit allowlist of permitted HTTP methods per endpoint, rather than trusting that undocumented methods won't be tried.
- Validate `Content-Type` strictly server-side, don't assume the client will only ever send the expected format.
- Return generic error messages in production. Detailed validation errors are useful for debugging but also hand an attacker the exact expected schema.
- Apply the same protective controls across all API versions in production, not just the current default version.
- To prevent mass assignment specifically: allowlist which properties a client can set, and explicitly blocklist sensitive fields (roles, permissions, internal IDs) from ever being bound to user input.

## Key Takeaways

- API endpoints frequently expose more functionality than the UI surfaces. Method switching (`GET` → `PATCH`, `OPTIONS` to enumerate allowed methods) is a fast way to find it.
- Server error messages are reconnaissance data. Iterating on malformed requests and reading the resulting errors is often enough to reverse-engineer an undocumented schema.
- Malformed requests and malformed exploits look identical from the outside (both return errors), the fix is almost always in request hygiene: correct `Content-Length`, correct header placement, valid JSON syntax, before assuming the target logic itself is the obstacle.
- Comparing what an API accepts (`PATCH`/`POST` bodies) against what it returns (`GET` responses) is the fastest way to surface mass assignment candidates.

---

*Based on PortSwigger Web Security Academy's API Testing learning path. Labs completed in a sandboxed environment for educational purposes.*
