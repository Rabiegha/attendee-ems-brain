# Pre-call — GCP / Gotenberg review — 23 July 2026

## Short opening statement

The private OIDC flow from our real OVH staging container to Cloud Run is now working.

Today at 08:47:11 UTC:

- the authenticated health check succeeded;
- anonymous requests were rejected with HTTP 403;
- Gotenberg generated a valid 8,117-byte PDF in 271 ms;
- the NestJS integration is deployed on staging;
- Puppeteer remains available as an automatic fallback;
- production is still using Puppeteer and Gotenberg is not enabled there.

We now need to validate the production settings, monitoring, capacity and security before running
the real LFD business smoke test and the progressive load test.

## Questions for the GCP team

### Cloud Run deployment

1. Which exact Gotenberg image and immutable digest are currently deployed?
2. What are the current values for minimum instances and maximum instances?
3. Can we keep one minimum instance running during the LFD peak period to avoid cold starts?
4. What concurrency value is configured per instance?
5. How much CPU and memory are allocated to each instance?
6. Is CPU always allocated, or only while a request is being processed?
7. Is the current 60-second application timeout aligned with the Cloud Run request timeout?

### Logs and observability

8. Can you find our smoke-test request from 23 July at 08:47:11 UTC in Cloud Run Logs?
9. Do the logs show any cold-start delay or abnormal latency for that 271 ms request?
10. Which Cloud Run metrics should we monitor during the load test?
11. Which alerts do you recommend for HTTP 429/5xx errors, latency, instance saturation and
    container restarts?
12. Can we create a dashboard showing request count, error rate, p95 latency, active instances,
    CPU and memory?
13. How long are the Cloud Run logs retained, and is that sufficient for post-event analysis?

### Capacity and load testing

14. Are there any project or regional quotas that could affect a workload of approximately
    3,000 PDF generations in one day?
15. Do you agree with a progressive load-test ramp of 10, 100, 500, 1,000 and then 3,000 PDFs?
16. What stop thresholds would you recommend for error rate, HTTP 429/5xx responses, p95 latency,
    CPU, memory and cost?
17. Should we limit application-side concurrency to protect both Cloud Run and the OVH worker?
18. Is `europe-west9` the best region for latency from our OVH server and for data residency?
19. Can Cloud Run scale fast enough for our expected peak, or should we temporarily increase the
    minimum number of instances?

### OIDC and IAM security

20. Can you confirm that the service account has only `roles/run.invoker` on this specific Cloud
    Run service?
21. Are there any broader project-level roles attached to this service account that we should
    remove?
22. Do you recommend rotating the current service-account key before the event?
23. What key rotation and revocation procedure should we follow?
24. After the event, would you recommend Workload Identity Federation instead of keeping a
    long-lived service-account key on OVH?
25. Should the credential be stored in GCP Secret Manager, or is a protected read-only file on OVH
    acceptable for the event period?

### Chromium assets and networking

26. Are outbound requests from the Gotenberg container allowed for Google Fonts and external
    images?
27. Are there any egress restrictions, DNS limitations or additional costs for those requests?
28. Would you recommend embedding fonts and images directly in the request to make PDF rendering
    deterministic?
29. Is the announced 32 MB request limit correct for our Cloud Run configuration?

### Worker integration and production readiness

30. The staging API can authenticate successfully, but the future BullMQ worker does not yet have
    the credential mounted. Do you agree that the same read-only credential can be mounted in the
    worker?
31. Do you recommend using the same service account for the API and worker, or separate identities?
32. What evidence would you require before enabling Gotenberg in production?
33. Should we perform a controlled Cloud Run failure test to validate the local Puppeteer fallback?
34. Is there anything else you would change before we run the real LFD badge and email test?

## Decisions required before ending the call

- Confirm the immutable Gotenberg image digest.
- Confirm minimum/maximum instances, concurrency, CPU, memory and timeout.
- Agree on the progressive load-test plan and stop thresholds.
- Agree on monitoring, alerts and ownership during the event.
- Confirm the IAM scope and credential-rotation plan.
- Assign an owner and date for mounting the credential in the worker.
- Define the conditions required for production approval.

## Proposed closing question

Based on today’s successful authenticated smoke test, do you see any blocker on the GCP side before
we run the real LFD badge test, the fallback test and the progressive load test?
