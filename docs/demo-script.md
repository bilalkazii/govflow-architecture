# GovFlow — Demo Script & Demo Mode

| | |
|---|---|
| **Status** | Approved (Phase 1) · Implemented in Phase 21 |
| **Goal** | Prove theses A (conversational completion) and B (spike absorption) in one continuous story, < 15 minutes |

## 1. Demo Mode controls (Admin → Demo panel)

| # | Control | Backing API | Expected visible reaction |
|---|---|---|---|
| 1 | Normal traffic | `/admin/demo/scenario {normal}` | Steady GREEN dashboards |
| 2 | 10× / spike traffic (10K→150K) | `/admin/infrastructure/simulate-spike` | Latency/queue/CPU rise; HPA scaling events; routing shifts to AWS; queue depth grows; zero lost applications |
| 3 | Government API failure | `/admin/integrations/{name}/health {failure}` | Circuit breaker OPEN badge; new submissions queue with "government system busy"; no user-facing errors |
| 4 | Queue activation | automatic on spike/failure | Queue dashboard: depth, ages, positions; tracking shows position |
| 5 | Off-peak scheduling | `/admin/demo/scenario {offpeak}` | Deferred queue releases on smoothing curve (5%→25%→5%); backlog drains gradually |
| 6 | Service recovery | `/admin/integrations/{name}/health {recover}` | HALF_OPEN → CLOSED; queued items process; breaker badge clears |
| 7 | Prompt injection | `/admin/security/simulate {prompt_injection}` | Blocked pre-tool; SecurityEvent appears; AI replies safely |
| 8 | API flood | `/admin/security/simulate {api_flood}` | 429s; rate-limit metrics; WAF-style block list entry |
| 9 | Malicious document | `/admin/security/simulate {malicious_document}` | Quarantined; zero tool calls; SecurityEvent `malicious_document` |
| 10 | AI forecast | always-on model on infra metrics | Infra dashboard shows: "Traffic expected to increase 3.2× within 30 minutes" + "+12 workers", "Activate deferred queue" |

## 2. The story (rehearsed order)

**Act I — Citizen journey (Thesis A)**
1. Open Home. "Namaste, Bilal!" — AI assistant, popular services, recent applications.
2. Type or speak: **"Mujhe income certificate banana hai."** → AI identifies Income Certificate, asks jurisdiction → "Maharashtra" → asks purpose → "Education". Suggested-response chips throughout. *(Screens 1–3)*
3. "I found the right service for you." — confirm. *(Screen 3)*
4. Documents required → **Fetch from DigiLocker** → consent screen → mock authorization → select Identity + Income Record → fetched. *(Screens 4–5)*
5. Document analysis: progress → classification → OCR → extracted fields with confidence ("Name: Bilal Kazi · DOB ••/••/•··· · Address: Latur, Maharashtra · Income: ₹2,40,000") → **"Please verify the extracted information."** *(Screen 6)*
6. Pre-filled, editable form; badges show AI-extracted vs user-entered vs verified. *(Screen 7)*
7. Review & confirm: full summary, fee ₹50, standard processing, ETA 15 days → checkbox → **Confirm & Submit**. *(Screen 8)*
8. **Application received — GF-MH-847291**, queued, position shown. *(Screen 9)*
9. Tracking timeline: Received → Documents verified → Prepared → Submitted → Government processing → Completed. *(Screen 10)*

**Act II — Spike absorption (Thesis B)**
10. Switch to Infrastructure Dashboard. Click **SIMULATE TRAFFIC SPIKE** (100K→150K req/s). Narrate: latency rises, error rate stays ~0, queue grows, autoscaling events fire, load balancer shifts flexible traffic to AWS node (93% → 52% → 41% routing), AI recommendation card appears ("+12 workers", "Activate deferred queue").
11. My Applications: new applications keep flowing in — **nothing lost**.

**Act III — Failure & protection**
12. Trigger **government API failure** (mock income endpoint → 500s). Circuit breaker: CLOSED → OPEN. Citizens see "queued — government system busy"; applications persist; queue depth grows safely.
13. Security Dashboard → **SIMULATE PROMPT INJECTION**: chat shows safe refusal, zero tool calls, SecurityEvent (high). **SIMULATE API FLOOD**: 429 storm visible. **SIMULATE MALICIOUS DOCUMENT**: quarantined, event logged.

**Act IV — Recovery & completion**
14. **Recover** the mock API → HALF_OPEN probes → CLOSED → queue drains gradually; off-peak scheduler demo shows the deferred curve (01:00 5% → 02:30 25% → 04:00 5%).
15. Return to Tracking: GF-MH-847291 → **Completed**. Notification appears. Close on: *"One interface. Decentralized trust. Intelligent infrastructure."*

## 3. Demo hygiene checklist

- [ ] Seed script run (demo user Bilal Kazi, synthetic docs, baseline traffic)
- [ ] All mock integrations reset to `healthy`; breaker states CLOSED
- [ ] Simulator reset to normal traffic; queues empty of stale items
- [ ] Security event list cleared (archived), threat level NORMAL
- [ ] LLM provider = demo key or mock provider; token budget set
- [ ] Fallback plan rehearsed: full story runs identically on `docker compose` if staging is unavailable
