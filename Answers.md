# Technical Exercise: Support Data Scenario

Thanks for taking the time to work through this technical exercise. It's meant to reflect the kind of data work that comes up as a team lead. Note: nothing here requires advanced SQL, just clear thinking through a realistic situation.

**Time estimate:** 30–45 minutes.

## How to Submit

1. Fork this repo (or clone the gist).
2. Add your answers into a copy of this file (`ANSWERS.md`), with your SQL and short answers inline.
3. Submit the link to your fork (or your branch/PR) using the form link provided.

You're welcome to use whatever tools you'd normally reach for on the job. However, please be ready to walk through your reasoning afterward.

---

## Sample Schema

Assume three tables from a support ticketing system:

**tickets**
| Column | Type | Description |
|---|---|---|
| ticket_id | integer | Primary key |
| agent_id | integer | Foreign key to agents.agent_id |
| category | text | e.g. Billing, Technical, Account, Shipping |
| priority | text | Low, Medium, High, Urgent |
| status | text | open, closed, reopened |
| opened_at | timestamp | When the ticket was created |
| closed_at | timestamp | When the ticket was closed (null if still open) |
| first_response_minutes | integer | Minutes until the first agent reply |
| reopened_count | integer | Number of times the ticket was reopened after closing |

**agents**
| Column | Type | Description |
|---|---|---|
| agent_id | integer | Primary key |
| name | text | Agent name |
| team | text | Team the agent sits on |
| hire_date | date | Date the agent started |

**csat_responses**
| Column | Type | Description |
|---|---|---|
| response_id | integer | Primary key |
| ticket_id | integer | Foreign key to tickets.ticket_id |
| score | integer | 1–5 satisfaction score |
| submitted_at | timestamp | When the customer submitted the score |

---

## The Scenario

You're leading a small support team. Leadership has noticed that CSAT scores for the **Billing** category dropped 12% last month. Ticket volume in Billing stayed roughly flat over the same period, and nothing changed in staffing. You've been asked to look into it and report back.

Work through the following as you would if this landed in your inbox tomorrow.

**1. First response time by team**
Write a query returning the average `first_response_minutes` for tickets closed in the last 30 days, grouped by agent team.

SELECT
  a.team
  AVG (t.first_response_minutes) AS avg_first_response_minutes
FROM tickets t
JOIN agents a ON t.agent_id = a.agent_id
WHERE t.closed_at >= NOW() - Interval '30 days'
GROUP BY a.team;

**2. Agents with above-average reopen rates**
Write a query returning each agent's reopen rate (reopened tickets ÷ total tickets they handled) for agents whose rate is higher than their own team's average.

WITH agent_rates AS (
  SELECT
    a.agent_id, a.name, a.team
    SUM (CASE WHEN t.reopened_count> 0 THEN 1 ELSE 0 END):: float
        /COUNT (t.ticket_id) AS reopen_rate
  FROM tickets t
    JOIN agents a ON t.agent_id = a.agent_id
    GROUP BY a.agent_id, a.name, a.team
),
team_avg AS(
  SELECT team. AVG (reopen_rate) AS team_avg_rate
  FROM agent_rates
  GROUP BY team
)
SELECT ar.name, ar.team, ar.reopen_rate. ta.team_avg_rate
FROM agent_rates ar
JOIN team_avg ta ON ar.team=ta.team
WHERE ar.reopen_rate > ta.team_avg_rate;


**3. CSAT trend by category**
Write a query returning the average CSAT score per category, per month, for the last 3 months.

SELECT
  t.category
  DATE_TRUNC ('month', c.submitted_at) AS month
  AVG (c.score) AS avg_csat
FROM csat_responses c
JOIN tickets t ON c.ticket_id = t.ticket_id
WHERE c.submitted_at >= NOW() - INTERVAL '3 months'
GROUP BY t.category, DATE_TRUNC ('month', c.submitted_at)
ORDER BY t.category, month;

**4. Digging in**
Beyond the three tables above, what additional data would you want to pull to understand the Billing drop — and which of the existing tables would you start with, and why?

// I would like to pull additional reports such as over the avg SLA tickets date, ticket subcategory, and CSat comments, so I can look for more reasons and details on each ticket with an outlier SLA, and get answer on why is the CSat decreasing.

**5. Testing a theory**
Pick one specific theory for what might be driving the drop. State your theory, then write a query using the tables above that would help confirm or rule it out.

// My theory: After a recent update on our CRM system the automatized billing process is failing, the system is pulling the billing address from the main contact rather than the account specific contact, in some cases this creates a mismatch, if this true the reopened cases must bee showing a rise from the rollout date, while the rest categories stay flat.

SELECT
  category,
  DATE_TRUNC ('week' opened at) AS week,
  AVG (reopened_count) AS avg_reopens,
  COUNT (*) AS ticket_volume
FROM tickets
WHERE opened_at>= NOW() - INTERVAL '3 months'
GROUP BY category, DATE_TRUNC ('week', opened_at)
ORDER BY category, week;

Probably this is not showing directly the address-mismatch, but show a spike on the volume coinciding with the system update.


**6. What you'd actually do**
Say your query in Question 5 showed that reopened tickets in Billing spiked, and most of the reopens trace back to one specific issue type (e.g., refund timing questions). What would you actually do with that in the next week? Be concrete. What would you say to the team, what (if anything) would you change in a process or macro, and how would you know if it worked?

// I would have a sync with the team, focusing that we have a gap on the process and not on the team is underperforming, as we find an error happening in the system, so talk about the increase on the billing reopened tickets, take 5-8 tickets in this specific situation and read them all together, to listen together to the customers complain, brainstorm solutions we can provide them with our communication or giving them clarity on how we are gonna fixing the situation.

For a process/macro change:
- Meet with an Master Data team and find a solution so the sync to the correct address is updated.
- Work with our finance team and work on a fast track solution if there are any refunds o rebill to the customer
- Set a new subcategory that can be set and we can track this issue in the next weeks.



**7. Reporting up and coaching down**
You need to update your own manager on this in two sentences, and separately coach one agent on it in a 1:1. How would those two conversations differ?

// To my manager, simple and direct, share the root cause analysis, share the rollout from the solutions I´m working with the third party teams and add this report weekly to find the decrease of the reopened tickets. After a month share the trend after the implementation rollout.
Coaching an agent (1:1): find an example and ask the agent to guide me over the process it follows and understand on what can be done better, call for overcomunication and couriosity on the team.

 *
