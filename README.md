
# AllMark — IT Admin Deployment & Operations Guide

**Audience:** IT Administrators, Security Engineers, Cloud/Infrastructure Teams

**Product:** AllMark Gate (Browser-only verification gate for external meeting platforms)

**Platforms:** Google Meet, Microsoft Teams, Zoom, Webex (meeting links remain on the customer platform)

**Data posture:** No participant video uploads or storage

---

## 1. Overview

AllMark Gate is a **browser-based verification gateway** designed to reduce the risk of **impersonation and AI-assisted fraud** for high-stakes online meetings. It operates as a **pre-join verification step** and enforces a **mandatory post-join integrity window** while the participant joins the meeting on their chosen meeting platform.

AllMark Gate is intended for use in **government and enterprise** contexts where meeting identity assurance and meeting integrity are critical.

### Key capabilities

* **Pre-meeting verification** in the participant’s browser
* **Enforced join gating:** participants do not receive meeting access unless verification succeeds (or host override is applied)
* **Mandatory post-join integrity window (15 seconds):** detects environment changes after joining
* **Verifier dashboard:** real-time visibility for hosts and security reviewers
* **Host override:** allow or revoke join for exceptional cases
* **Privacy-preserving operation:** no participant video upload or storage

> **Accuracy statement:** AllMark Gate has measured **up to 98% accuracy** in controlled evaluations against defined test scenarios. Real-world performance depends on configuration, user environment, and threat conditions. No security control guarantees absolute prevention.

---

## 2. Privacy, Data Handling, and Trust Boundaries

### What AllMark Gate does **not** do

* Does **not** upload or store participant video
* Does **not** store biometric templates
* Does **not** provide human review of participant media

### What AllMark Gate may store/transmit (customer-controlled)

* Verification outcomes (PASS/FAIL/REVERIFY REQUIRED)
* Correlation codes and audit-friendly event records
* Short-lived integrity indicators required to enforce the verification policy
* Meeting configuration and participant status (for the verifier dashboard)

### Data ownership

AllMark Gate is designed to run in **customer-controlled infrastructure** (AWS), where operational logs and verification records remain within the customer’s security boundary.

---

## 3. System Components

### Participant/Organizer/Verifier

* **Organizer Console** (`/console`): Creates AllMark links and sets meeting policy (e.g., guests allowed).
* **Participant Sidecar** (participant AllMark link): Runs verification and mandatory post-join integrity window.
* **Verifier Dashboard** (host-only link): Live statuses and override controls.

### Backend service

* A FastAPI service that:

  * Creates meetings and issues participant/verifier links
  * Receives verification results
  * Issues short-lived access proofs
  * Enforces join gating and host overrides
  * Receives post-join integrity reports
  * Streams verifier updates (SSE or polling)

### State store

* **Redis** is required for short-lived state and event distribution.

---

## 4. Deployment Model (AWS)

AllMark Gate can be deployed into a customer AWS environment. A typical secure reference architecture is:

* **CloudFront** (public edge) serving:

  * Frontend static assets (S3 origin)
  * Backend API (ALB origin)
* **ALB** → **ECS/Fargate** (or EC2) running the API service
* **Redis** (ElastiCache or MemoryDB)
* **CloudWatch Logs/Metrics**
* Optional: **AWS WAF** in front of CloudFront

### Recommended DNS

* `www.allmark.com` (or customer-owned equivalent) → CloudFront distribution

### Transport security

* HTTPS only
* TLS termination at CloudFront and/or ALB
* HSTS recommended

---

## 5. Configuration (Environment Variables)

Below are typical configuration inputs. Exact variable names may differ by packaging.

**Required**

* `PUBLIC_BASE_URL`
  Public origin for links (e.g., `https://www.allmark.com`)
* `REDIS_URL`
  Redis connection string (ElastiCache/MemoryDB)
* `MEETING_HOST_ALLOWLIST`
  Allowed meeting link domains (Meet/Teams/Zoom/Webex allowlist)
* `SESSION_SECRET` (or equivalent)
  Secret used for session binding / signing

**Recommended**

* `LOG_LEVEL` (`INFO` for production)
* `RATE_LIMIT_*` settings (per-IP / per-meeting thresholds)
* `DEFAULT_TTL_SECONDS` (meeting default expiry)
* `CORS_ORIGIN` (prefer same-origin via CloudFront behaviors; avoid broad CORS)

---

## 6. CloudFront Behaviors (Recommended)

To keep cookies and security posture simple, run frontend and backend on the **same domain** and route API paths to the backend.

### Suggested behaviors

* Default `/*` → S3 (frontend)
* API paths → ALB (backend), for example:

  * `/meetings`
  * `/join`
  * `/snapshot/*`
  * `/events/*`
  * `/override/*`
  * `/health`
  * Participant submit/report endpoints as implemented in your backend

### Headers policy (recommended)

* `Cache-Control: no-store` for sensitive pages/endpoints
* `Referrer-Policy: no-referrer`
* Strict CSP (Content Security Policy) appropriate for a camera-using web app
* `X-Content-Type-Options: nosniff`

---

## 7. Operational Workflow (What to Expect)

### Organizer workflow

1. Organizer opens `/console`
2. Pastes meeting link and chooses:

   * **Allow guests**: ON/OFF
   * **Assurance**: Medium/High
3. Organizer shares:

   * **Participant link** (one link for all participants)
   * **Verifier link** (host-only)

### Participant workflow

1. Opens participant link
2. Enters name + email
3. Completes short verification
4. Clicks **Join meeting**
5. Keeps the AllMark tab open for **mandatory 15-second integrity window**

   * Closing early will trigger **Reverification Required**

### Verifier workflow

1. Host opens verifier link
2. Monitors real-time participant outcomes
3. Uses **override allow/revoke** when appropriate

---

## 8. Mandatory 15-Second Integrity Window

After a participant clicks **Join meeting**, the AllMark sidecar enforces a **15-second integrity period**.

### Why it exists

This short window is used to detect rapid changes in the participant’s environment immediately after meeting entry.

### Policy enforcement behavior

* If the participant **closes the AllMark tab early**, the system marks the participant as **Reverification Required** (or equivalent).
* Hosts may request re-verification or apply override per meeting policy.

---

## 9. Overrides and Review Policy

AllMark Gate supports host-driven exceptions.

### Typical situations for override

* Legitimate participants in constrained environments
* Known device issues
* False positives requiring human judgment

### Recommended governance

* Restrict verifier link access to meeting organizers and approved security staff
* Log override actions for audit purposes
* Use overrides sparingly for sensitive meetings

---

## 10. Monitoring, Logging, and Audit

### What to log

* Meeting creation events
* Participant verification outcomes
* Post-join integrity outcomes
* Override actions
* Correlation codes for support triage

### Recommended integrations

* CloudWatch dashboards and alarms for:

  * elevated failure rates
  * OTP disabled (if applicable in future)
  * unusual spikes in verification requests
  * backend error rate / latency

---

## 11. Security Hardening Checklist (Recommended)

**Network**

* Restrict backend security group ingress to ALB only
* Lock down Redis in private subnets
* Enable WAF rules for common bot/abuse patterns

**Application**

* Use short-lived, one-time proof tokens for join gating
* Rate limit verification attempts
* Enforce meeting domain allowlist
* Use `no-store` headers for sensitive responses
* Ensure secrets are stored in AWS Secrets Manager / Parameter Store

**Access**

* Verifier link treated as privileged; protect distribution
* Use least-privilege IAM roles for compute

---

## 12. Limitations and Responsible Use

AllMark Gate is designed to **reduce risk** and help organizations detect and deter fraud. It does not guarantee prevention of every possible attack and should be part of a layered security approach.

### Recommended complementary controls

* Use meeting platform controls:

  * waiting room / lobby
  * host admit controls
  * authenticated user restrictions
  * meeting lock after start
* Use corporate identity governance:

  * managed endpoints
  * security training and phishing defenses

---

## 13. Support and Troubleshooting

### Common user-facing issues

* Camera permission denied → user must grant permission and retry
* Closed sidecar → user marked **Reverification Required**
* Network interruptions → user may need to retry verification

### What to collect for support

* Correlation code from participant screen
* Meeting ID and timestamp
* Verifier dashboard status screenshot (if allowed by policy)

---

## 14. Appendix: Organizer Quick Guide (Shareable)

**For meeting organizers**

1. Create an AllMark link at `/console`
2. Share the participant link in the invite
3. Open the verifier dashboard before the meeting starts
4. Remind participants: **keep AllMark open for 15 seconds after joining**
5. If someone fails, they may retry once; you may override if appropriate


