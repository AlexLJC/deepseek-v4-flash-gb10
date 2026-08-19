# Detailed experiment reports

This directory contains focused, evidence-backed experiments that extend the
aggregate tuning notes in the repository root. Each experiment keeps its report
and presentation assets together; large native logs are published only after a
separate redaction review.

| Experiment | Scope | Status |
|---|---|---|
| [`SpinCondition` bounded sleep](spincondition-bounded-sleep/README.md) | Matched C16 full A/B of a 250-microsecond bounded sleep in vLLM's shared-memory busy-wait path; process CPU, thermals, capacity, latency, and PR guidance | First matched pair complete; CPU and thermal claims supported; performance replication pending |
