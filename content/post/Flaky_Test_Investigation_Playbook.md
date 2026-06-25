---
title: "Flaky Test  Strategy for  Applications"
description: "A comprehensive strategy for identifying, classifying, detecting, governing, and reducing flaky tests in  applications"
date: 2026-06-24
image: "/images/flaky.png"
tags: [ "Software Testing", "Faky Test Case Handling", "Automation", "Quality Engineering", "Flaky Tests"]
categories: ["Testing"]
---

# **Flaky Test Investigation Playbook**

## **Standard Operating Procedure (SOP) for Identifying and Mitigating Flaky Tests**

This document establishes a rigorous, standardized methodology for diagnosing, documenting, and resolving flaky tests within our test suites. Flaky tests erode trust in CI/CD, slow down deployment pipelines, and mask legitimate product regressions.

## **Step 1: Defining a "Flaky Test"**

A test is officially classified as **flaky** if it exhibits non-deterministic behavior under identical code and environmental configurations.

                               ┌─────────────────────────┐  
                               │   Test Run Executed     │  
                               └────────────┬────────────┘  
                                            │  
                      ┌─────────────────────┴─────────────────────┐  
                      ▼                                           ▼  
             ┌─────────────────┐                         ┌─────────────────┐  
             │     Passes      │                         │     Fails       │  
             └────────┬────────┘                         └────────┬────────┘  
                      │                                           │  
                      └─────────────────────┬─────────────────────┘  
                                            ▼  
                           Is code or environment different?  
                                    /              \\  
                                  No                Yes  
                                  /                  \\  
                        ┌─────────────────┐    ┌──────────────────┐  
                        │   FLAKY TEST    │    │ Deterministic    │  
                        │   Confirmed     │    │ Behavior         │  
                        └─────────────────┘    └──────────────────┘

### **Core Characteristics of Flakiness**

* **Non-Deterministic Results:** The test passes and fails across multiple executions on the same commit hash without any changes to the code under test or the test code itself.  
* **Non-Reproducible with Fixed Input:** Rerunning the test locally with the exact same inputs and database state does not consistently replicate the failure.  
* **Intermittent Failures:** The test exhibits an active pass/fail toggle rate (![][image1]) under normal execution cycles where:  
  ![][image2]

## **Step 2: Establish a Reproducibility Matrix**

When flakiness is suspected, run the test under controlled variations to pinpoint the vector of instability. Use this matrix to guide your investigation:

| Test Run Environment | Variation Config | Diagnostic Focus | What a Failure Suggests |
| :---- | :---- | :---- | :---- |
| **Local Machine** | Single run vs. 50x iterations | Baseline Stability | If it fails locally in loops, it's likely a test logic, timing, or local state issue. |
| **CI Pipeline** | Sequential vs. Parallel execution | Concurrency & Race Conditions | If it fails *only* during parallel execution, look for shared data pollution or resource starvation. |
| **Execution Mode** | Headless vs. Headed (UI only) | Rendering & Event Loops | If it fails *only* in headless mode, there may be rendering/timing optimizations or viewport issues. |
| **Browser Engines** | Chromium vs. WebKit vs. Firefox | Browser Implementation | If it fails on a single browser, it is a browser-specific rendering engine quirk or driver bug. |
| **Target Environment** | Staging replica vs. Local Mock | Infrastructure Dependency | If it fails *only* on shared environments, suspect latency, 3rd-party APIs, or dirty shared databases. |

## **Step 3: Collect Execution Evidence**

Do not attempt to debug or fix a flaky test without cold, hard artifacts. Before concluding why a test is flaky, collect and attach the following evidence to your investigation ticket:

### **A. Test Execution Artifacts**

* \[ \] **Playwright Traces (trace.zip):** Essential for step-by-step DOM state, network timeline, and console logs.  
* \[ \] **Visual Proof:** Screenshots captured precisely at the point of failure and continuous video recordings.  
* \[ \] **Console Out:** Combined stdout/stderr from both the browser console and the Node.js/test runner execution terminal.

### **B. Network Layer Data**

* \[ \] **HAR Files:** HTTP Archive to analyze response times, payloads, and protocol headers.  
* \[ \] **Status Code Analysis:** Check for unexpected HTTP 4xx/5xx responses, or gateway timeouts (504).  
* \[ \] **API Payload Mismatch:** Verify if the schema or payload returned under stress differs from the mock or expected database state.

### **C. CI/CD Pipeline Signals**

* \[ \] **Orchestrator Logs:** Detailed stage execution logs from Jenkins, GitHub Actions, or GitLab CI.  
* \[ \] **Retry Counts:** Tracking metrics showing how many retries are typically needed before a pass occurs.  
* \[ \] **Resource Telemetry:** Check for CPU, memory, or network IOPS spikes on the runner container at the time of execution.

### **D. Application/Server-Side Logs**

* \[ \] **Request Correlation:** Correlate frontend execution with backend logs using X-Request-ID or trace headers.  
* \[ \] **Auth Token Lifespans:** Confirm that test timeouts aren't caused by silent token expiration during long-running suites.  
* \[ \] **Rate Limiting:** Check if the CI runner’s IP was throttled/rate-limited by application security firewalls (WAF).

## **Step 4: Failure Pattern Analysis**

To understand *how* the test is breaking, analyze the historical data and log outputs across multiple failures using the following diagnostic questions:

                                  Failure Analysis  
                                         │  
       ┌─────────────────────────────────┼─────────────────────────────────┐  
       ▼                                 ▼                                 ▼  
Step Localization               Timing Dependencies               Execution Order  
\- Same step every time?         \- Failures during peak hours?     \- Dependent on previous test?  
\- Random failure points?        \- Latency-dependent steps?        \- State/Database pollution?

* **Step Localization:** Does the test fail at the exact same assertion or step every time, or does it fail randomly across different execution lines?  
* **Timing Dependencies:** Is the failure rate correlated with specific times of day (e.g., scheduled database backups, heavy CI queue hours)?  
* **Order Dependencies (Test Pollution):** Does the test pass when run in isolation, but fail when run in a suite or in a randomized order?  
* **Instant Recovery:** Does a simple automatic retry immediately fix the test? If so, the issue is highly likely a microsecond race condition or dynamic rendering lag.

## **Step 5: Rule Out Common Non-Flaky Causes**

Before marking a test as flaky, you must rule out deterministic issues that are actually valid failures caused by design constraints:

* **Data Dependency & Shared State:**  
  * Multiple tests are writing to or modifying the same user profile or database entity concurrently.  
  * *Verification:* Run the test suite with single concurrency. If failures disappear, restructure your database seeding to use unique, dynamically generated IDs per test.  
* **Async Timing / Eventual Consistency:**  
  * The test code checks the database or DOM before the background worker has completed processing.  
  * *Verification:* Ensure you are awaiting dynamic network responses or polling with assertions that have built-in retry logic (e.g., Playwright's auto-retrying assertions) instead of relying on static timeouts (sleep(1000)).  
* **Parallel Execution Collisions:**  
  * Port collisions, shared memory leaks, or local file system writes in the runner.  
* **Unstable External Integrations:**  
  * Flakiness caused by sandbox environments of third-party vendors (e.g., Stripe sandbox, Auth0, SendGrid).  
  * *Verification:* Intercept and mock third-party network requests during integration testing.

## **Step 6: Decision Criteria**

Use the following flowchart to officially determine if a test qualifies for the **Flaky** status:

                  ┌──────────────────────────────────────────┐  
                  │    Suspected Test Failure Identified     │  
                  └────────────────────┬─────────────────────┘  
                                       │  
                                       ▼  
                   /───────────────────────────────────────\\  
                  \<  Does it fail consistently across runs? \>  
                   \\───────────────────────────────────────/  
                                   /       \\  
                             Yes  /         \\  No  
                                 /           \\  
                                ▼             ▼  
                     ┌──────────────────┐   /─────────────────────────\\  
                     │ Genuine Bug or   │  \<  Are there environment   \>  
                     │ Env Outage       │  \<  outages or infra issues?\>  
                     └──────────────────┘   \\─────────────────────────/  
                                                  /             \\  
                                            Yes  /               \\  No  
                                                /                 \\  
                                               ▼                   ▼  
                                     ┌──────────────────┐    /──────────────────────────\\  
                                     │ Infrastructure   │   \< Is failure non-reproducible\>  
                                     │ Issue            │   \<  deterministically?      \>  
                                     └──────────────────┘    \\──────────────────────────/  
                                                                   /             \\  
                                                             Yes  /               \\  No  
                                                                 /                 \\  
                                                                ▼                   ▼  
                                                      ┌──────────────────┐    ┌──────────────────┐  
                                                      │  Label As FLAKY  │    │  Verify Dynamic  │  
                                                      │  (Quarantine)    │    │  Data Elements   │  
                                                      └──────────────────┘    └──────────────────┘

A test is labeled **Flaky** ONLY if:

1. It fails intermittently across ![][image3] runs (e.g., ![][image4] failure rate in a sample of ![][image5] runs).  
2. No product defect or service outage correlates with the exact timestamps of the failures.  
3. The failure cannot be reproduced reliably on-demand by setting a static input, configuration, or environment parameter.

## **Step 7: Outcome Handling**

Once a test has met the decision criteria for flakiness, execute the following triage steps immediately:

### **1\. Classification**

Categorize the flakiness into one of the following buckets to assign the right engineering resource:

* flaky:timing \- Animation, API latency, or asset load speeds.  
* flaky:data \- State pollution, shared DB IDs, or hardcoded seed data conflicts.  
* flaky:infrastructure \- Runner resource exhaustion, network drops, rate limits.  
* flaky:test-design \- Missing await statements, unhandled promises, or improper assertions.

### **2\. Isolation & Quarantine**

Never leave a flaky test active in the mainline block of a merge queue or release branch.

* **Skip/Quarantine:** Decorate the test with a quarantine skip flag (e.g., test.skip(), @flaky tag, or .skip in Playwright/Cypress).  
* **Create Tracking Ticket:** Open a Jira/GitHub Issue linking directly to the test name, file path, collected artifacts (from Step 3), and classification.

### **3\. Resolution**

When fixing the test, follow these mandates:

* **No Blind Retries/Sleeps:** Inserting page.waitForTimeout(3000) or raising global runner retries is forbidden. Fix the root race condition or assertion.  
* **Verify Fix via Stress Testing:** Run the fixed test **50 times sequentially** in CI before pulling it out of quarantine.  
* **De-quarantine:** Merge the fix, remove the skip/quarantine flags, and monitor the test’s pass-rate metrics over the next 48 hours.

[image1]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAKAAAAAaCAYAAAAwnlc+AAAFnElEQVR4Xu1ZWWicVRSepu7Lg0sMZJl/MhOMxgfBiIKCC+bFVgRBpfjiQoqIuCIWsWhRW4sKVkurqChWQRANtlpsXZ6EusQNXAoFEewm0WoXRWus8Tsz54xnvtx/MhknM4HcDw733u+ce865d87c//4zmUxERERERETEXEE2m80zFzFHkSTJRK3Cc6cC5vwM+daN99TraybQ29t7DnLZS+v8BXLIjZfzvLkA2pNq8i7PrQvmkHkBTqxb0nTVEPKZz+ezzLUaWN9VmutDnu/s7DxG+X2eny5m23qngfm6/s9ZIQD/O/bufubrQqhYPKrp0pDL5Y5C0+a5QqFwSj2+ZhKuAJeyDtyo6GBzJetqxWxbb61A3nfpvixgnQD8Y5CLma8LGqhio7q7u090+o1eVy/EJ8dpNVwB3hvQbVbd26yrBT09PZfPtvXWCuS9j3Pv7+8/3vrYm8cHBwcP9/q6ESpABHghoycY+megOcx0sL0GJ9wGtN9LIsYboFsE/j7oX/Y8uBN8HNidJo94tE+gXSzcwMDAEbAZxvhhtCPO9lqMl0New3C+6OUDNr2zWyl5QR5lXQjVChDcr6JDnEJAdzPkPchW+FjCenDnq1+RBSq93ka+kMh3PfhR2N/mda2G5U5c+TqCfAe97n+Bg2FjjtZxxSNUIIWpumJBor+WbbGpy9inIFCAF2K8UTgpQuEkNvqreD76DxgHGYOvW6VvxWH3S7xcnKn2d/r5aUgrQPCblL/C84KkVOA+t085lsb/UX1IX+Qs0suceTLGmrdhfLDsoLUo3v+Q0ydGoH89r7Fh0E2aJJlwAU56KVH7LZ5zheq5igI0CGcF6Dm2xfgz4+T4T9yjUe0/+M+6zJVP0RBcAe6QDUf7ZSi2B3Svs17G8LWCuG/YTtGm9pd4Ujj7AqUBNi+lyDrk/6Lu+/OQ5yDP8vxaAB93Sy4s8P8q2zYEFoC47ZlAATLc2+Jez2O8hn02oACLLwWeE/T19bWrjxvxAXaYgDsAOcT2HmknoKwnFCsEWxfaNzyfpBRgonvjc9V8JY8H2b7Z0H3jvR/Jlq5ijYcuvCIgPsyVmZQChO0zOmdLrnQ3k37FzxUYP8k+qxUgZHWA400IFiC4YeHlmwsZIqk4ZRhVCvB05d/3vAHrfkf0aNfDx9XW9zZJegHu0Hw516GE7omtgK6b936nHzcUoYBpSAIng87f7zk50diuWgFC9xRzbJukFCBOj361H2bdVHAFWPEzTEdHx7GhHATK/8Mc1ryBuK/9fPS3avtKyG8twLxHpiM8fyro1WYCMsq6GYMGrGlD1HZdgDuADyCH9gblVrNPe8xk9OJt0PkV95VQTkngsm9Q+/I/LwaktI05DytA2C1jXSiHQqHQo3xxnQbl3oSf67L6d2Oi90lvI639HCUnp+kU8zD/HuKaCuS1VNdyKes8kPti2HyBl8Czk9LdfBfb1AwNOCE/gbCOoba7bayP4HHh0b/IEk8C3/JEH2u4N55MvPj8yMbwc7tyPP875gwo7gtEh425zDj0Vwjv7RiIdZPGWsM6l4O98Rdja5xNZpfVFy7IV1LI7e3txym/xOXbhvHTNgf8W6LD/fVIx41Zv1VADn+6nFMBmxGs9Q6s6QcdTzlnEpJS4chPBXIn2Q7ZmZT+w019e5LNTfSSqrJQj+3xnD6C0P8Nskv9jslPJWgPKidxJOYe86mPuz/MpxZysS+iPuXR7+dP+rD0hJX/c23uQrYxQHdeUvp/2tYu7U9ygplNPp8/1fn6MKv/iOAE68P4b9OBH5BTT8dry0EyxSL8WPmKlzQBYi1y/v/q6uo6iW2ahaT0w/P+pLR/IrLf43Lis60B+t1Yw7nMR0Q0BfLFYS4iommIBRjRMuBqMYTH7yrmIyIiIiIiIiIiIiIiIiIiQvgX8/CDURA3toEAAAAASUVORK5CYII=>

[image2]: <data:image/png;base64,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>

[image3]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACoAAAAZCAYAAABHLbxYAAABmElEQVR4Xu2VP0sDQRDFI6bwA4jFeX85RezERkHQRivBwi4fQcGvIFY2VoKWIjapLCxFrKy0tBE7CxE7IxoFD0HfhFU2486yF6MQ2B8M4d6b2Xu3m1wqFY/HU02SpODiF2EYjsRxfIGeD9Qp99tA44JqXOVep2Cta7Vmq7hPpGk6p3u4npB620DgGbXwFvc6BWs9STcnHeFWmFagznVNBMcxiuY3VJ17ZZGC5nk+RDp96jqCn5j6rQRBMIihBobPuOeKFBSnty7o+ybdCQQdwPAN6gqX/dy3YQl6ZNKh7Zp0Z9RRPeIGx9yzIQWlUxL0bdLx9RvmnpUsy8YwWGCBA+65IAWFVhf0HaVXuWcEAWdpADu4yb0ySEGl7yi0PZP+A+xcjRqTLr1TLUFbr8LSv3oMrlFDFEVL3PsNUlBCbcgy05qoB137Bk+xAXOS690A675KQdXuvWtSH/VCTzXtb8ENn1H3qFtVd7RTOLGc9V2iXlCHFBInO6/7vQm9s/A0iy6FU5nm8/8G/VUiwJRLIew4n/d4PD3EJ0VikhcGWpB1AAAAAElFTkSuQmCC>

[image4]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAF4AAAAZCAYAAAC4j5m6AAAEeElEQVR4Xu2Ya4hVVRTHFSex7CHGNDmvfecBYxNFNYSIREaFvdCwb4mBH+uDlCA9KUj9mPjqQ0REYpYQjhDRg0CUEDUoIrXEsA8iBqmBaZL56Le8e91Zrrv3fYyKDJw/LM5e//Xfe6+9ztn7nHvHjStQoEAN9PT0tHV3d9/peYuhoaHrPDeWwXoHOzo6Oj1vUSqVbvfcFQOD7w4hfEIii7ke93FBZ2fn9cQueH6sgrUcxZZjK7G9Pi6A7xnVminoS9gmubPRn85AG7i+qBr8aXZw2t9iJ7B92JPY49hB0XR1dT2gumuMFvI540kFD0k/a94lOct6fBzuGeyc8f+U8ajLdmwO9jT+X9KfHXGr7dsQQvmOyuTWfrQaEvxUePUp7v1wT0W3RZ70+LT/pppGgP6k5y4XjPmrXYuPCyjagzaGf4/X4v+BfW/8pQMDAzdJu729/QYuLXLswm+sdGoGdH6LzmuY/COub0BN8Br4PS6xCfjLjC+aytPRKPxiryRCeUcmxxee9T7vuDPYTqvBtqqPfja1elh91Vi/KUixsYc8b8GEL9tJ8B+V7aY+sTXNHjHo2y8r8TrIFb6vr+824eVqedbzjdXT/jKYc532CjSTjP/DqI4YBQO8Xq/wvb29t7ikKi/X1tbWG/EPqN8I+DoakPGiPSHGzRzyOha6iNhh2Y2+UPUQMoVnnjcz/IeWZ84Zbs3+aPpY/VGBQV5lkBUysE4O957Xwc/H/sN+R7fQ8OetrhHQZwn2ncwV20uYc57GBwcHJ8Y8HjN9TmNr1a+HkC/8lhQP967nmX91KL9AT/Cw3G20Vf2bhiyaZL52nBTkkjM8BblB9J2lfog3EHvW6lJILVQRyjf4F0fLV4qMPc3xSYRM4cl5e4aXIl/gI6HDxyzQ/IxmqvrxYZX3w71WNyrEBVYlZ9Hf338zmv3q0/6CJL6K7c0cDV0j6mqETOH1t4DdAYqYV2XOWgiZwsNtzPDrIt/iYwr5mkOzXn3aR8jzldg+xGV8RdwAqsQMci6VnIWPJ/zkjyxFyBSem/ec8PIC9zHhU31SCJnC5854uA9SvIWLj/d6/GHr10RczLEEl01Cthd3eoahUklk+wvCyBOm/sWjpRS/qbkuGlFXNDXzsgj5ws8S3r+sS+6rxkPyk48M48/0evxT1q+JuJilCS6ZBAlOIbbH817vfQ/GWWU1vo19rr7jX/N8CiFTeEEcZ77jTobMLg3lIr/vuDv8+KHJwv/Ded2qPgWZLQPKJ5/VKfxkCs/nFqGQHSN95F0hfjA3k7N0rsT0l2KM78BOq18Psi6fkyI+3WcNdXHHwpcMV0HIfLn58UMzR42ADsdkEDW2Y6/XCIit5+V3l+cFxN4h702xPVzv60CAbkOcs+pXr/wjGMp/Umleb3tNCuj+xo5gh6Idxo5zM/uc7ifsFPaZjM+aH7FxBbH9bW1tkz0vCOWH4YXYbvrl2jBIbpvnLOLxcR5b4GNjFaxluecsiA9j/2L3+ViBAgUKFChQoECBAlcB/wPScop05a2HJwAAAABJRU5ErkJggg==>

[image5]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACUAAAAZCAYAAAC2JufVAAABdElEQVR4Xu1UPUsDQRQ8sbIRG7v7lAPFVu21ECzU0tbCxlIjFkHFIlrZ2QmCCDb+BLG2sVFB9FekSSDYiM6DjbxMdoNZElDYgWF5s+/mze19RFFAwBCQ5/kueJum6aypZ7Isu8G6w73Qj8AG2AK3eH9ggPkJ+EV8svS9gfeqfgUfdM/AgBM6hvk5TuYa6yGkUe4py3JcwrIuGq6bYF0D/stJkiyw3hMSBFxiXQP7z65Q4CXrGgi94hPq4BehZLgrVJeugZNa7TsU7qQK41Mxh8GVrNAudI9ruEvXgOda36FgWsGFd6TJsBrVXcNduoZXKBt4GNcuHcPnmdiv4OQ3WRdqL8YICzD69Ay1zsR+DdxjXai9OmCM6xZNh2r2CPXOuobX4zPG+xbtJwSMN1yhsDfHuoZvqBZ+jpPtGs9/UYYVRTFNffJVbqv6zBaU4RVKAPN6+3TM3U9xTxzHYybYI9YX8COyvI8M71DDxJ8MFRAQ8F/wDRFPkc/2l04rAAAAAElFTkSuQmCC>