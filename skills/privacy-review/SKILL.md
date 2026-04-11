
---
name: privacy-engineer
description: >
  Review code and content for privacy concerns using a privacy-by-design lens.
  Use when asked to review, audit, or check code for privacy issues, PII exposure,
  data collection practices, consent handling, or data retention. Also use when asked
  to review privacy policies, privacy notices, terms of service, cookie banners, or
  any user-facing privacy content for completeness, accuracy, and regulatory alignment.
  Frameworks: CCPA/CPRA, OWASP Top 10 Privacy Risks, NIST Privacy Framework.
metadata:
  author: migmaqer
  version: '1.0'
---

# Privacy Engineer

A privacy-by-design review skill for code and content. Identifies privacy risks, regulatory gaps, and data-handling issues, then reports findings by severity with actionable remediation guidance.

## When to Use This Skill

Use this skill when the user asks you to:

- Review code for privacy concerns, PII leakage, or data-handling issues
- Audit data collection, consent flows, or data retention logic
- Review privacy policies, privacy notices, or terms of service for completeness
- Check cookie banners or consent UIs for regulatory compliance
- Assess alignment with CCPA/CPRA, OWASP Privacy Risks, or NIST Privacy Framework
- Perform a privacy-by-design review of a feature, service, or architecture

## Frameworks

This skill references three frameworks. Apply whichever are relevant to the review context:

### CCPA / CPRA

Key requirements to check:

- Right to know, delete, correct, and opt-out of sale/sharing
- Sensitive personal information (SPI) handling and opt-out of use
- Service provider / contractor / third-party data sharing distinctions
- Required disclosures in privacy notices (categories collected, purposes, retention periods)
- Do Not Sell or Share My Personal Information link and Global Privacy Control (GPC) support
- Financial incentive / loyalty program notice requirements

### OWASP Top 10 Privacy Risks

Check for these risks in code and architecture:

1. Web application vulnerabilities exposing personal data
2. Operator-sided data leakage (logs, error messages, backups)
3. Insufficient data breach response capabilities
4. Insufficient deletion of personal data (retention beyond purpose)
5. Non-transparent policies, notices, or data practices
6. Collection of data not required for the stated purpose (data minimization failures)
7. Sharing of data with third parties without adequate controls
8. Outdated personal data retained without correction mechanisms
9. Missing or insufficient session expiry
10. Insecure data transfer (unencrypted PII in transit)

### NIST Privacy Framework (Core Functions)

- **Identify-P:** Inventory personal data, map data flows, understand privacy risks
- **Govern-P:** Policies, roles, risk management strategy for privacy
- **Control-P:** Data processing management, disassociability, access controls
- **Communicate-P:** Transparency, consent mechanisms, data processing awareness
- **Protect-P:** Security safeguards for personal data at rest and in transit

## Instructions

### Step 1: Determine Review Type

Ask the user (if not obvious) whether this is:

- **Code review** — source code, configuration, infrastructure-as-code, API definitions
- **Content review** — privacy policy, notice, terms, cookie banner, consent UI copy
- **Both** — a combined review (e.g., a PR that includes code and updated policy text)

### Step 2: Gather Context

Before reviewing, establish:

- What personal data is involved (PII categories)?
- Who are the data subjects (customers, employees, minors)?
- What jurisdictions apply (California / US federal / other)?
- What is the purpose of data processing?

If the user provides a file or code block, infer as much context as possible and state your assumptions.

### Step 3: Conduct the Review

#### For Code Reviews

Examine the code for:

1. **PII exposure** — Personal data logged, hardcoded, returned in error messages, or stored in plaintext
2. **Data minimization** — Collecting or processing more data than necessary for the stated purpose
3. **Consent and opt-out** — Missing or insufficient consent checks; ignoring GPC signals or opt-out flags
4. **Data retention** — No TTL, expiration, or deletion logic for personal data stores
5. **Third-party sharing** — Data sent to external services, SDKs, or analytics without documented purpose
6. **Encryption** — PII transmitted without TLS or stored without encryption at rest
7. **Access controls** — Missing authentication or authorization on endpoints that return personal data
8. **Session management** — Sessions that never expire or lack proper invalidation
9. **De-identification** — Opportunities to pseudonymize or anonymize data that are missed
10. **Breach surface** — Error handling that could expose PII; missing audit logging

#### For Content Reviews

Examine the document for:

1. **Completeness** — All required disclosures present per applicable regulation (CCPA notice requirements)
2. **Accuracy** — Statements match actual data practices described by the user or inferred from code
3. **Clarity** — Language is understandable by the average consumer (plain language, not legalese overload)
4. **Data categories** — All collected categories of personal/sensitive information are listed
5. **Purpose specification** — Each category has a clear, specific purpose for collection
6. **Retention periods** — Retention timelines or criteria are disclosed
7. **Consumer rights** — Rights to know, delete, correct, opt-out are explained with instructions
8. **Third-party disclosures** — All sharing and sale/sharing of data is disclosed with categories of recipients
9. **Contact information** — Methods for consumers to submit requests are provided
10. **Update mechanism** — How and when the policy is updated, and how users are notified

### Step 4: Report Findings

Organize findings into a severity-tiered report:

```
## Privacy Review Findings

### Critical
Issues that create immediate regulatory risk or active PII exposure.
- [Finding title]: Description, location (file:line or section), and why it matters.
  - **Remediation:** Specific fix or approach.

### High
Issues likely to cause compliance gaps or significant privacy risk.
- ...

### Medium
Issues that weaken privacy posture but are not immediately exploitable.
- ...

### Low
Improvements and best-practice recommendations.
- ...

### Summary
- Total findings: X (Critical: N, High: N, Medium: N, Low: N)
- Frameworks referenced: [list]
- Key recommendation: [one-sentence priority action]
```

### Step 5: Provide a Remediation Roadmap

After the findings table, provide a short prioritized list of next steps:

1. Fix all Critical items immediately
2. Address High items before next release
3. Schedule Medium items into the backlog
4. Track Low items as improvement opportunities

## Severity Definitions

| Severity | Definition | Example |
|----------|-----------|---------|
| Critical | Active PII exposure or clear regulatory violation | API endpoint returns full SSN without auth |
| High | Likely compliance gap or significant risk | Privacy policy omits required CCPA disclosures |
| Medium | Weakened privacy posture, not immediately exploitable | Logs contain user email but access is restricted |
| Low | Best-practice improvement | Could pseudonymize user IDs in analytics pipeline |

## Tips

- When in doubt about severity, err on the side of higher severity and note your reasoning.
- If the user provides both code and a privacy policy, cross-reference them — flag any mismatches between what the policy says and what the code does.
- Always state which framework(s) inform each finding.
- Keep remediation suggestions concrete and actionable, not vague ("consider improving...").
