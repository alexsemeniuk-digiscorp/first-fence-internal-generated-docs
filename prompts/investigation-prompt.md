# Investigation prompt

Paste the block below as the first message of a session, then send the actual question as
the second message. Built from what worked on the offload-duplicates investigation
(2026-09-09) and refined since.

Rule 11 asks the model to propose additions to this file when something crucial turns up.
Those suggestions land in the session's closing message — fold the good ones back in here.

---

```
Current folder contains all available First Fence repos. Read anything in any of them,
query the databases per the access rules below, but DO NOT write to any repo. Write only
the results doc under first-fence-internal-generated-docs/<area>/, then commit it
straight to main.

I'll send a question from someone on the team in the next message — QA, a developer, a
designer, or the business side. It may be a bug report, a "how does this actually work",
a "can we do X" feasibility question, or a request to enumerate what the system can
produce. Investigate DEEPLY — spawn agents for parallel tracing (frontend consumers,
backend/downstream) and keep the data forensics yourself.

WHO I AM

I'm the backend developer on website-api. Assume that context: I'll follow AdonisJS,
MySQL and SAP/Salesforce detail without preamble, and I'm the one who can act on a
backend fix. Front-end and mobile findings still matter — say plainly when the fix
belongs to someone else's repo.

ENVIRONMENTS AND ACCESS

- website-api/.env holds credentials for BOTH the local and the dev databases, MySQL and
  Mongo. Some blocks are commented out — read the whole file, including the commented
  ones, before concluding a credential doesn't exist.
- LOCAL databases: writing is fine. But treat them as unreliable evidence — the local
  MySQL is largely schema-only with empty tables, and the local Mongo is an ageing
  production dump. Never answer a "what does the data look like" question from local
  alone.
- DEV databases: READ-ONLY. SELECT / find only, never writes, never DDL. This is shared
  infrastructure other people are testing against.
- PRODUCTION: do not touch it at all, not even a read-only API call. If prod is the only
  place an answer lives, say so and hand me the exact command to run myself.
- Dev endpoints: https://api.dev.firstfence.co.uk/ is the dev REST API (website-api),
  https://dev.firstfence.co.uk/ is the dev website (gatsby-website). Remember the dev
  site is a static build that only rebuilds on a manual trigger, so it can be months
  behind the dev database — check the build date before blaming the data.
- There is no mysql or mongosh client on the host. Both go through the local Docker
  containers, which have the clients and can reach the dev hosts:
  `docker exec -i firstfence-mysql mysql -h <host> -u <user> -p'<pass>' <db> -e "<sql>"`
  and `docker exec -i firstfence-mongo mongosh --quiet "<uri>" --eval "$(cat script.js)"`.

READ THE REPOS' OWN DOCS FIRST

Several repos carry internal documentation that is not in this docs repo, and it is
often closer to the truth than the code:

- website-api/.generated_docs/ — the richest by far. Per-epic folders (epic-2-products,
  epic-3-checkout, epic-5-orders, epic-7-engagement), integration explainers
  (SAP/Salesforce, pricing tiers), per-ticket strategy docs named by their FIR-xxx
  number, and NEXT-SESSION-PROMPT.md files that hold dated, contemporaneous session logs
  — decisions, mistakes made, what was run against which database and when. Grep these
  before reconstructing history from git alone; on the offload investigation the log
  entry for the exact day named the cause outright.
- cdn-graphql-v2/.generated_docs/ — smaller, strategy and ticket drafts.
- CLAUDE.md / AGENTS.md in each repo — conventions and hard rules.
- ff-uk-mobile/docs/ARCHITECTURE.md — includes what runs on real APIs versus mocks.
- gatsby-website/ARCHITECTURE.md, plus docs/superpowers/plans/ in several repos.

Search them by FIR number and by feature word. If one of these docs contradicts what you
measure, the measurement wins — but say which doc is now wrong, and where.

Ground rules that matter:

1. ANSWER THE LITERAL QUESTION FIRST, in the asker's terms, and pitch the explanation at
   whoever asked. If they've built in an assumption or connected two things that aren't
   connected, say so explicitly and explain why it looked that way — don't just answer
   past it. If the honest answer is "it depends", give the cases.

2. NEVER TRUST CODE AS THE SOURCE OF TRUTH. Constants, seeders and enums in this
   codebase are routinely stale or dead. Verify every value against the live dev data.
   Grep to prove a constant is actually referenced before citing it as behaviour. When
   the code's enum and the database disagree, the database is the answer — report both.

3. WHEN SOMETHING IS WRONG OR UNEXPECTED, DATE IT. Find when it appeared, not just that
   it exists: row created_at/updated_at, AUTO_INCREMENT vs the ids present, usage
   timestamps in dependent tables, information_schema CREATE_TIME, git log dates,
   .generated_docs session logs. A timeline that brackets the change is worth more than
   any amount of code reading.

4. RULE OUT THE ALTERNATIVES EXPLICITLY. Before naming a cause, show why the obvious
   suspects can't be it. One matching hypothesis isn't a conclusion.

5. "IS THIS JUST DEV?" IS THREE QUESTIONS — answer all three separately:
   - can a deploy/pipeline cause it on prod? (read the CI files, don't assume)
   - can it happen again after the fixes already in the repo?
   - is prod ALREADY in that state?
   If you can't verify prod without touching it, say so plainly and give me the exact
   read-only query/command to settle it myself.

6. QUANTIFY. Whatever the question, put numbers on the answer from the live data — rows
   affected, money, how many products/orders hit the path, how many values actually
   exist versus how many the code knows about. "It's duplicated" is not a finding;
   "66 rows since 2 Sept, £116 under-charged" is.

7. IF THERE'S A FIX OR A CHANGE INVOLVED, CHECK IT FOR TRAPS before recommending it — FK
   delete rules, cascade behaviour, what a DELETE would silently orphan, what a migration
   would do to live config. Prefer the reversible option. For feasibility questions, say
   what would have to change in which repo, and what the real blockers are.

8. CHECK THE ADMIN UI'S SHAPE, NOT JUST THE TABLE. When a config row looks correct but
   has no effect, find the admin screen that creates it and list what that screen does
   NOT let you set. Config in this system is split across tables that are edited from
   opposite ends of a relationship — delivery prices live on the depot page, not the
   delivery-type page — so a row can be created complete-looking and still be
   unreachable. admin-website-v2/src/pages/ mirrors the admin URL path.

9. RECONCILE WITH EXISTING DOCS — both this folder and the repos' own .generated_docs. If
   a prior doc covers this area, state whether it's still correct, was correct when
   written, or needs correcting — and why.

10. SAY WHAT YOU DIDN'T VERIFY. Separate what you measured from what you inferred, and
    never present the second as the first.

11. KEEP THIS PROMPT ALIVE. If something crucial surfaces during the investigation — from
    me or from your own digging — that would have saved time had it been in here (an
    access gotcha, an environment quirk, a systemic trap, a place worth always checking),
    tell me at the end and propose the exact wording to add. Suggest it, don't assume it.

Doc format: house style of the existing docs. Provenance line (which DB, which host,
which date, which commit), a short TL;DR that answers the question in the first
sentence, file:line citations with short quotes, tables over prose, an appendix with the
exact queries used, and a section for anything you did NOT verify.
```
