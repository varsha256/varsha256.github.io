
---
title: "End-to-End Automation Architecture for Complex Microservices"
description: "A comprehensive guide to scalable, maintainable, and efficient testing strategies."
date: 2026-04-06
image: "cover.jpg"
categories:
    - Engineering
    - QA Automation
tags:
    - Microservices
    - CI/CD
    - Test Architecture
---

## 8. Assertion & Validation Layer

The assertion layer ensures that the system state matches expectations across multiple data planes.

### Test Method Flow
* **Response Assertion**
    * `Assert.assertEquals(bookingState, BookingState.OTW_PICKUP)`
* **Custom Assertion**
    * `E2ECustomerPlatformAssertions.assertBookingCreated(booking)`
* **Database Validation**
    * `AliceDBValidations.validateDriverAssignment(driverId)`
* **Kafka Event Validation**
    * `BidLogKafkaManager.consumeMatchingEvents()`
* **State Machine Validation**
    * `bookingState.hasReached(BookingState.COMPLETED)`

---

## 9. Reporting Pipeline

Our pipeline ensures full visibility of test execution across multiple platforms.

### Test Results Distribution
* **HTML Report (ReportNG):** `build/test-output/index.html`
* **Extent Report:** `ExtentReports.html`
* **ReportPortal Integration:**
    * Publish results & Create launch
    * Upload screenshots/logs
    * Update real-time dashboard
* **TestRail:** Update test case status and link to ReportPortal.
* **Slack Notifications:** * Pass/Fail counts and Test summary
    * Direct links to ReportPortal
    * Detailed error logs for failures
* **Dashboard Listener:** Metrics collection, trend analysis, and performance tracking.
* **CSV Export:** Raw test execution summary.

---

## 10. Configuration Loading Priority

We use a layered configuration approach to balance flexibility and stability.

1.  **System Properties (Highest Priority)**
    * `tag`, `e2e_rp`, `e2e_nego_rp`
2.  **Environment Variables**
    * `CI_JOB_NAME`, `SERVICE_AREA`, `REPORT_PORTAL_API_TOKEN`
3.  **Gradle Properties**
    * `version`, `org.gradle.jvmargs`, `sourceCompatibility`
4.  **Resource Files (`src/main/resources/`)**
    * `testConfig.json`, `merchantConfig.json`, `transportConfig.json`
5.  **Code Defaults (Lowest Priority)**
    * Fallback values defined within the source.

---

## 11. Module Dependencies Graph

The framework follows a modular "bottom-up" dependency structure to maximize code reuse.

* **Entry Point:** `gojek-e2e` (E2E Tests)
* **Domain Modules:** `marketplace`, `pricing`, `gojek-commons` (Shared Utils)
* **Service Modules:** `transport`, `gofood`
* **Platform Layer:** `customer-platforms`
* **Foundational Layers:** `cartography` → `core` (Assertions)

---

## 12. Test Execution Timeline

### T0: Task Start
* Gradle loads configurations and initializes environmental settings.

### T1: Suite Setup
* `@BeforeSuite` executes.
* `PrerequisiteHandler` checks system health.
* ReportPortal initialized.

### T2: Data Provider Execution
* Test data is dynamically generated.
* Thread pool (e.g., 5 threads) is allocated.

### T3: Parallel Test Execution
* **Thread 1:** `@BeforeMethod` → `@Test (300ms)` → `@AfterMethod`
* **Thread 2:** `@BeforeMethod` → `@Test (400ms)` → `@AfterMethod`
* **Thread 3:** `@BeforeMethod` → `@Test (500ms)` → `@AfterMethod`
* *Performance Note:* Total time ~500ms for 5 tests (vs 2.1s sequentially).

### T4: Suite Teardown
* `@AfterSuite` executes.
* Generate HTML reports and upload to ReportPortal/TestRail.
* Dispatch Slack notifications and write final summary.

### T5: Task Complete
* Final test results are made available for the CI/CD pipeline.

> This comprehensive architecture ensures scalable, maintainable, and efficient end-to-end testing of complex microservices.
