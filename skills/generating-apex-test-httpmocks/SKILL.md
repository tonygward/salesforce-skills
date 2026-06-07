---
name: generating-apex-callout-test
description: "Generate Apex test classes for HTTP callout code using the self-shunt pattern. The test class itself implements HttpCalloutMock, capturing every outbound request in a private list and injecting status code and response body via constructor. Use this skill when writing tests for any Apex class that makes HTTP callouts (REST, SOAP, named credentials), when you need to verify the exact requests that were sent (endpoint, method, headers, body), or when you need to simulate success, failure, and error HTTP responses. Triggers on classes that call Http.send(), HttpRequest, or HttpCalloutMock. Do NOT use for non-callout tests – use the generating-apex-test skill instead."
metadata:
  version: "1.1"
---

# Generating Apex Tests for HTTP Callouts (Self-Shunt Pattern)

Generate test classes for Apex HTTP callout code where the **test class itself is the mock** — the self-shunt pattern. This eliminates a separate mock class file, keeps request-capture logic co-located with assertions, and makes constructor-injected response configuration explicit.

---

## Core Rules

1. **Self-shunt only** — the `@isTest` class implements `HttpCalloutMock` directly. Never create a separate mock class file for callout tests.
2. **Capture all requests** — every call to `respond()` appends the inbound `HttpRequest` to a `private List<HttpRequest> requests`. Assertions verify what was sent.
3. **Constructor injection for response** — status code and response body are set via a constructor, not hardcoded. One constructor covers all scenarios; instantiate with different values per test method.
4. **Honour all base skill rules** — `Assert` class only, `Test.startTest()`/`Test.stopTest()` wrapping, Given/When/Then layout, meaningful assertion messages.

---

## Self-Shunt Class Structure

```apex
@isTest
private class {ClassUnderTest}Test implements HttpCalloutMock {

    // ─── Mock State ───────────────────────────────────────────────────────

    /** Every HttpRequest that was passed to respond() is captured here. */
    private List<HttpRequest> requests = new List<HttpRequest>();

    /** HTTP status code to return from respond(). */
    private Integer mockStatusCode;

    /** Response body to return from respond(). */
    private String mockBody;

    /**
     * Inject the desired status code and response body.
     * Instantiate once per test method with the scenario-specific values.
     *
     * @param statusCode  e.g. 200, 400, 401, 500
     * @param body        JSON (or other) response body string
     */
    public {ClassUnderTest}Test(Integer statusCode, String body) {
        this.mockStatusCode = statusCode;
        this.mockBody       = body;
    }

    // ─── HttpCalloutMock ──────────────────────────────────────────────────

    /**
     * Called by the Apex runtime for every Http.send() made during the test.
     * Appends the inbound request to `requests`, then returns the configured response.
     */
    public HTTPResponse respond(HTTPRequest request) {
        requests.add(request);                   // capture for later assertion

        HttpResponse response = new HttpResponse();
        response.setStatusCode(mockStatusCode);
        response.setBody(mockBody);
        response.setHeader('Content-Type', 'application/json');
        return response;
    }

    // ─── Test Methods ─────────────────────────────────────────────────────

    @isTest
    static void shouldReturnParsedResult_WhenCalloutSucceeds() {
        // Given
        String successBody = '{"id":"abc123","status":"ok"}';
        {ClassUnderTest}Test mock = new {ClassUnderTest}Test(200, successBody);
        Test.setMock(HttpCalloutMock.class, mock);

        // When
        Test.startTest();
        {ResultType} result = {ClassUnderTest}.{methodUnderTest}();
        Test.stopTest();

        // Then — verify the response was processed correctly
        Assert.areEqual('abc123', result.id, 'Should parse id from response body');

        // Then — verify the outbound request
        Assert.areEqual(1, mock.requests.size(), 'Exactly one callout should be made');
        Assert.areEqual('GET', mock.requests[0].getMethod(), 'Should use GET method');
        Assert.isTrue(
            mock.requests[0].getEndpoint().contains('/expected/path'),
            'Endpoint should target the correct path'
        );
    }

    @isTest
    static void shouldThrowCalloutException_WhenServerReturns500() {
        // Given
        {ClassUnderTest}Test mock = new {ClassUnderTest}Test(500, '{"error":"Internal Server Error"}');
        Test.setMock(HttpCalloutMock.class, mock);

        // When / Then
        Test.startTest();
        try {
            {ClassUnderTest}.{methodUnderTest}();
            Assert.fail('Expected {YourException} to be thrown on 500 response');
        } catch ({YourException} e) {
            Assert.isTrue(
                e.getMessage().contains('500'),
                'Exception message should include the HTTP status code'
            );
        }
        Test.stopTest();

        // Verify callout was still attempted
        Assert.areEqual(1, mock.requests.size(), 'One callout should have been attempted');
    }

    @isTest
    static void shouldThrowAuthException_WhenServerReturns401() {
        // Given
        {ClassUnderTest}Test mock = new {ClassUnderTest}Test(401, '{"error":"Unauthorized"}');
        Test.setMock(HttpCalloutMock.class, mock);

        // When / Then
        Test.startTest();
        try {
            {ClassUnderTest}.{methodUnderTest}();
            Assert.fail('Expected {YourException} to be thrown on 401 response');
        } catch ({YourException} e) {
            Assert.isTrue(
                e.getMessage().containsIgnoreCase('unauthorized'),
                'Exception should mention unauthorized access'
            );
        }
        Test.stopTest();
    }

    @isTest
    static void shouldSendCorrectHeaders_WhenCalloutIsMade() {
        // Given
        {ClassUnderTest}Test mock = new {ClassUnderTest}Test(200, '{"id":"xyz"}');
        Test.setMock(HttpCalloutMock.class, mock);

        // When
        Test.startTest();
        {ClassUnderTest}.{methodUnderTest}();
        Test.stopTest();

        // Then — inspect headers on the captured request
        Assert.areEqual(1, mock.requests.size(), 'One callout should be made');
        Assert.areEqual(
            'application/json',
            mock.requests[0].getHeader('Content-Type'),
            'Content-Type header should be application/json'
        );
        Assert.isTrue(
            String.isNotBlank(mock.requests[0].getHeader('Authorization')),
            'Authorization header must be present'
        );
    }

    @isTest
    static void shouldSendCorrectBody_WhenPostCalloutIsMade() {
        // Given — for POST/PUT methods
        {ClassUnderTest}Test mock = new {ClassUnderTest}Test(201, '{"id":"new123"}');
        Test.setMock(HttpCalloutMock.class, mock);

        // When
        Test.startTest();
        {ClassUnderTest}.{methodUnderTest}('inputData');
        Test.stopTest();

        // Then — verify the request body content
        Assert.areEqual(1, mock.requests.size(), 'One callout should be made');
        Assert.areEqual('POST', mock.requests[0].getMethod(), 'Should use POST');

        String sentBody = mock.requests[0].getBody();
        Assert.isTrue(
            sentBody.contains('inputData'),
            'Request body should contain the serialised input'
        );
    }
}
```

---

## Checklist

| # | Rule | Detail |
|---|------|--------|
| 1 | Class declaration | `@isTest private class {Name}Test implements HttpCalloutMock` |
| 2 | Request capture | `private List<HttpRequest> requests = new List<HttpRequest>();` — initialised at declaration |
| 3 | Mock fields | `private Integer mockStatusCode` and `private String mockBody` |
| 4 | Constructor | `public {Name}Test(Integer statusCode, String body)` assigns both fields |
| 5 | `respond()` signature | `public HTTPResponse respond(HTTPRequest request)` — always appends to `requests` first |
| 6 | `Test.setMock` placement | Called **before** `Test.startTest()` in every test method |
| 7 | Request count assertion | Always assert `mock.requests.size()` equals the expected number of callouts |
| 8 | Assert class | `Assert.areEqual`, `Assert.isTrue`, `Assert.fail` only — no legacy `System.assert*` |
| 9 | Test wrapping | `Test.startTest()` / `Test.stopTest()` around the code under test |
| 10 | Metadata file | Deliver `{ClassName}Test.cls-meta.xml` alongside the `.cls` file |

---

## Naming Conventions

Follow the base skill naming pattern applied to callout scenarios:

| Scenario | Method Name Example |
|----------|---------------------|
| Happy path | `shouldReturnParsedResult_WhenCalloutSucceeds` |
| Server error | `shouldThrowException_WhenServerReturns500` |
| Auth failure | `shouldThrowAuthException_WhenServerReturns401` |
| Request shape | `shouldSendCorrectHeaders_WhenCalloutIsMade` |
| POST body | `shouldSendCorrectBody_WhenPostCalloutIsMade` |
| Timeout / null body | `shouldHandleEmptyBody_WhenCalloutReturns204` |

---

## Mandatory Deliverables

For every callout test class produce **both** files:

```
{ClassUnderTest}Test.cls
{ClassUnderTest}Test.cls-meta.xml
```

Standard metadata XML:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ApexClass xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>66.0</apiVersion>
    <status>Active</status>
</ApexClass>
```

Match the API version of the class under test. Default to `66.0`.

---

## What to Assert on Captured Requests

| Aspect | Method on `HttpRequest` |
|--------|------------------------|
| HTTP method | `request.getMethod()` — `'GET'`, `'POST'`, `'PUT'`, `'DELETE'` |
| Endpoint URL | `request.getEndpoint()` — check `contains()` or `startsWith()` |
| Header value | `request.getHeader('Header-Name')` |
| Request body | `request.getBody()` — deserialise or use `contains()` |
| Callout count | `mock.requests.size()` — exact integer |

---

## Relationship to Base Skill

This skill **extends** `generating-apex-test`. All base skill rules apply:

- `Assert` class only
- Given/When/Then structure
- `Test.startTest()` / `Test.stopTest()` wrapping
- `TestDataFactory` for any SObject setup
- `{ClassName}Test.cls-meta.xml` always delivered

The **only divergence** is the mock strategy: self-shunt replaces a separate `HttpCalloutMock` implementation class.
