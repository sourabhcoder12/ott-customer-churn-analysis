# Customer Churn & Revenue Impact Analysis (OTT Platform)

**Tech stack:** SQL (SQLite), Python, Pandas, NumPy, SciPy, Matplotlib, Seaborn

## 1. The Problem

Subscription businesses lose revenue quietly — customers rarely leave without warning signs. This project asks three business questions using a sample of OTT subscriber data:

1. **Who is churning?** — which customer segments (contract type, plan tier, acquisition channel) show elevated churn
2. **What does it cost?** — how much monthly revenue and customer lifetime value (CLTV) is at risk
3. **What predicts it early?** — do support tickets, tenure, or other signals show up before a customer cancels

The goal is to move from "we lost X% of customers" to a specific, testable set of retention actions.

## 2. The Data

Three relational SQLite tables, joined on `customerid`:

| Table | Contents |
|---|---|
| `db_customer` | demographics — gender, state, country |
| `db_subscription` | contract type, plan tier, acquisition channel, monthly charges, CLTV, churn score, start/cancellation dates |
| `db_support` | support tickets — complaint date, escalation flag, CSAT score |

**Sample size: 21 customers, 9 support tickets.** This is a small, relational sample — not a large flat file. It was chosen deliberately over a bigger single-table dataset (e.g. the common Telco churn CSV) because it forces real analyst skills a flat file doesn't: joining across tables, handling a one-to-many relationship (one customer can have multiple support tickets), and deciding how to aggregate before joining rather than working with data that's already pre-flattened.

**Honest limitation:** 21 customers is too small to generalize from with confidence. Every finding below is reported as *directional* and paired with a statistical test so the strength of the evidence is explicit, not implied. A natural next step (in progress) is re-running the same logic on a larger public churn dataset (~7,000 rows) to check whether these patterns hold at scale.

## 3. The Analysis

**Data preparation:** cleaned inconsistent categorical values, standardized data types, and fixed a duplicate-row bug — customers with multiple support tickets were initially being collapsed by keeping only their most recent ticket, silently discarding escalation history. This was corrected by aggregating all tickets per customer (count, whether *any* ticket was escalated, minimum CSAT) before joining.

**KPIs built (SQL — JOINs, CTE, window function, GROUP BY/HAVING, and Pandas):**
- Overall and segment-level churn rate (by contract type, plan tier, tenure bucket, acquisition channel)
- Churn rate by support-ticket presence
- Revenue at risk and CLTV at risk by plan tier
- Customer ranking by CLTV within each plan tier (window function)
- Plan tiers with above-average churn (HAVING clause)

**Statistical testing (SciPy):** every categorical relationship was tested with a chi-square test, and every numeric comparison with a Mann-Whitney U test (chosen over a t-test since normality can't be assumed at this sample size). This distinguishes "the numbers look different" from "the numbers are statistically unlikely to be chance."

## 4. Key Insights

| Finding | Evidence | Statistical significance |
|---|---|---|
| Every customer who escalated a support complaint churned (5/5); 0 of 15 non-escalated retained customers did the same | churn rate 85.7% (ticket present) vs 0% (no ticket) | chi-square p = 0.0005 — strongly significant |
| Churned customers have significantly shorter tenure than retained customers | — | Mann-Whitney p = 0.0004 — strongly significant |
| Churned customers have significantly lower CLTV than retained customers | — | Mann-Whitney p = 0.011 — significant |
| Monthly-contract customers churn far more than annual-contract customers | 55.6% (5/9) vs 8.3% (1/12) | chi-square p = 0.060 — **directional, not conclusive at this sample size** |
| Churn is concentrated in low-value accounts: churned customers are 28.6% of the base but only ~19% of monthly revenue and ~12% of CLTV | — | descriptive |
| Referral-acquired customers churn far more than organically-acquired customers (5/6 vs 0/9) | — | descriptive, small cell counts |

**Recommendations:**
- Prioritize retention outreach for any customer who raises a support ticket, not only high-tenure or high-CLTV customers — this is the strongest, most statistically robust signal in the data
- Treat "push customers to annual contracts" as a hypothesis, not a proven fix, since plan tier and contract type are confounded in this sample (Premium has no monthly customers) — test with an A/B rollout before committing
- Investigate the referral acquisition channel's onboarding/expectations, given its high churn rate, before scaling that channel further

**Limitations:** small sample size (n=21) limits statistical power — several directional findings (contract-type gap, referral churn) would need a larger sample to confirm. Support-ticket timing relative to cancellation wasn't controlled for, so causation can't be claimed. Plan tier and contract type are structurally confounded in this dataset.
