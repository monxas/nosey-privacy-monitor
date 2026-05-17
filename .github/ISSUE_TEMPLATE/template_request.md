---
name: New device template request
about: Add tracking domains for a device class you've reverse-engineered
labels: template, contribution
---

**Device class**
e.g. Samsung Frame TV / Apple TV 4K / Google Nest Hub

**Domains found**
For each domain, fill the same structure as existing `templates/*.yaml`:

```yaml
- domain: example.tracking.com
  tier: 1                       # 1=safe, 2=tracking, 3=aggressive
  category: ads | telemetry | analytics | account | backbone
  why_block: "What does this endpoint do?"
  breaks_if_blocked: "Honest description of what stops working"
  observed_via: "How did you confirm? (Zeek ssl.log, mitmproxy, etc.)"
```

**Evidence (optional but helpful)**
A few Zeek SSL log lines or equivalent showing the domain in the wild.

**Trade-offs**
What features YOU lose by applying the whole template. The goal is informed consent, not maximal blocking.
