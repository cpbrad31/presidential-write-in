# Open Write-In Presidential Poll: Product & Technical Spec

Oct 19, 2026 · @Connor Bradley

## Overview

This is a one-question web poll, "Who do you want to be president?", with no preset ballot. It measures who people gravitate toward when nothing constrains the answer, and how that differs by who is answering.

**What it is:** an experiment that collects free-text votes, resolves them to real or fictional entities, and reports the distribution. Votes are final, results appear only after voting, and an optional verified subset sits beside the open sample.

**What it is not:** an election, a forecast, or a representative survey. The sample is self-selected, so every public result is labeled "open poll" and shown with its verified counterpart.

**Proposed success criteria** (adjust to your goals):

- A vote takes under 30 seconds and at least 85% of people who start finish.
- At least 70% of votes resolve automatically to an entity; the rest are reviewed or bucketed.
- Zero private individuals' names appear in any public view.
- Open and verified tallies are both published, with the gap between them reported.



## User flow

A person types a name, chooses a suggestion or category, confirms once, and only then sees results. Typed names are the only path through the moderation filter, and verification and demographics are optional extras after the vote.

[embedded content: voter flow · 4 choices, 1 filter, 1 confirmation

The three non-typed choices go straight to confirmation. A typed name passes the moderation filter first, and a blocked entry returns the person to rewrite it.

## Vote entry and suggestions

The landing page is one text field and nothing else: "Type the name of the person you want to be president." Suggestions are an aid for spelling, not a ballot, so they must not nudge anyone toward famous names.

**Suggestion rules**

- Nothing is shown until the person has typed 3 characters, and there is no default or "popular" list.
- Suggestions come only from the verified entity table: public figures matched to Wikidata, plus fictional and joke entries that have cleared the promotion threshold below.
- Each suggestion shows a name, photo and one-line role descriptor so "John Smith" can be told apart. It never shows vote counts.
- Order is by match quality, with ties shuffled per session, so no entry gets a permanent top slot.

**Fixed options, always one tap away**

- **Someone I know:** a friend, a family member, or a coworker. The vote is recorded by category only, so a private person's name never enters the database.
- **No one / none of the above:** counted and reported as its own result, since protest votes are a finding.

**Typing a name that is not suggested**

1. The raw text is checked by the moderation filter (slurs, threats, obvious harassment) and rejected with a neutral message if it fails.
2. If it passes, the vote is recorded with the raw text and sent to the resolution pipeline.
3. The person sees a confirmation screen ("You're voting for: name") and submits. Votes are final, with no edits after submission.

**Promotion threshold:** a new typed entry appears in other people's suggestions only after it has 5 or more independent votes, resolves to a public figure or a clearly fictional entity, and passes moderation review. This stops one person or one brigade from seeding a name into everyone else's dropdown.

## Seed data

Before launch, a one-time import fills the entity table so suggestions work from the first vote. Seeding supplies names for the dropdown only. It does not put anyone on a ballot, and a seeded entity still needs 5 or more votes and review before it is named in public results.

**Inclusion rules**


| Group                | Who is included                                                              | Candidate source                             |
| -------------------- | ---------------------------------------------------------------------------- | -------------------------------------------- |
| Federal executive    | President, Vice President, Cabinet secretaries                               | Wikidata                                     |
| Federal judicial     | Supreme Court justices and Article III judges                                | Federal Judicial Center directory            |
| Congress             | All sitting House and Senate members, including delegates                    | congress-legislators dataset, Congress.gov   |
| State executive      | Governors                                                                    | Wikidata, Ballotpedia                        |
| State legislative    | All state senators and representatives (roughly 7,400, approximate)          | Open States                                  |
| Business and culture | Up to 50 people in each of 10 business and culture categories (listed below) | AI candidate pipeline, confirmed in Wikidata |


The political groups total on the order of 9,000 people (approximate). Check each source's license before relying on it.

**Rules for every seeded person**

- **Individuals only.** Organizations, brands and fan accounts are excluded. Each person must match a human entity so aliases and a descriptor can be attached.
- **Adults only.** Anyone under 18 is excluded, with no exceptions. This matters mostly for content creators and athletes.
- **Current as of the import date.** The table is refreshed monthly and after election results are certified, so officeholders stay current.
- **One entity per person.** Duplicates across sources are merged, and a person in two groups (a senator with a large following) carries both tags.
- **Photos only if freely licensed** (for example, from Wikimedia Commons). Otherwise the suggestion shows a neutral placeholder.

**Seed step**

1. Pull records from each source.
2. Apply the inclusion rules and drop anything that fails them.
3. Normalize names and merge duplicates on Wikidata ID where one exists.
4. Generate aliases (nicknames, "Sen." and "Gov." forms, common misspellings).
5. Attach a role descriptor, and a photo if one is licensed.
6. Spot-check a random sample from each group by hand (proposed: 100 per group).
7. Load into the entities table with the source and seed group recorded and display status set to eligible.

**Business and culture candidates come from an AI pipeline, not a follower count.** Instagram offers no public bulk list of accounts by follower count, so the 1M-follower rule is dropped as a gate. Candidates are generated by category and confirmed against Wikidata instead. Follower count is kept as an optional tag when Wikidata has it.

**Categories.** The same 13 categories are run, with up to 50 names generated per category (at most 650 leads before filtering):

- **Political:** federal legislative politicians, federal executive politicians, state politicians (governors and state legislators)
- **Business:** banking and finance executives, technology executives, business executives (insurance, telecom, logistics and similar)
- **Culture:** actors, musicians, professional athletes, content creators, authors and writers, comedians, media personalities

For the three political categories, the official lists in the table above stay authoritative. Pipeline output for them is cross-checked against those lists and duplicates are merged.

**Candidate pipeline**

1. **Fix the categories.** The 13 above are the full set. The model does not add categories without your approval.
2. **Generate.** For each category, an AI proposes up to 50 names. Each category is run with more than one prompt or model and the results are merged, so one model's blind spots do not set the list.
3. **Reconcile.** Match each name to a Wikidata ID. A name with no match is dropped, which removes invented names. The match must be a human whose occupation fits the category, to guard against namesakes.
4. **Enrich.** Pull aliases, a description, birth date, a freely licensed photo, and the Instagram handle and follower count where present.
5. **Filter.** Drop anyone under 18. Send anyone with no birth date on file to review. Apply a notability cutoff (proposed: a minimum number of Wikipedia language editions).

**What to watch for.** The categories set the recall: a group missed here never appears, and models skew toward famous, English-language, Western names. Newer creators and rising stars may be missing, which matters most for content creators. A missing name is not a missing vote, because the typed path still catches it.

## Entity resolution

Every vote is stored as raw text first, then resolved to one entity asynchronously. The raw text is never overwritten, and each resolution is versioned, so the pipeline can be re-run as it improves.

**Pipeline**

1. **Normalize:** lowercase, strip punctuation and honorifics, collapse whitespace.
2. **Exact match:** look up the normalized string in the alias table ("jd vance", "j.d. vance", "vice president vance").
3. **Candidate retrieval:** for misses, fuzzy and embedding search against public figures in Wikidata, returning the top 5.
4. **Adjudication:** an LLM call gets the raw text and candidates, and returns one entity type, a chosen entity (if any), and a confidence score.
5. **Routing:** at confidence 0.9 or above, accept automatically. Between 0.6 and 0.9, send to the review queue. Below 0.6, leave unresolved. These are starting thresholds to tune on the first few hundred votes.
6. **Cluster the unresolved:** group near-duplicate strings so reviewers see "1 name, 40 spellings" instead of 40 rows.
7. **Human review:** a weekly pass over the largest clusters and the whole review queue. Approved clusters add new aliases, which makes step 2 catch them next time.

**Entity types**


| Type               | Example                                   | Shown publicly                              |
| ------------------ | ----------------------------------------- | ------------------------------------------- |
| Public figure      | A sitting or former official, a celebrity | Yes, once at or above the display threshold |
| Fictional or joke  | A cartoon character, a meme entry         | Yes, in a separate "fictional" group        |
| Category           | Friend, family member, coworker           | Yes, as category totals                     |
| None of the above  | "No one"                                  | Yes                                         |
| Private individual | A named coworker or relative typed in     | Aggregate count only, never the name        |
| Unresolved         | Illegible or ambiguous text               | Aggregate count only                        |


Two cases need a rule decided up front. Ambiguous names (several people called "John Smith") stay unresolved unless context picks one, and the system never guesses. Religious and historical figures ("Jesus", "Abraham Lincoln") are tagged by a reviewer and reported in their own group, separate from living public figures.

## Moderation and privacy

The rule that governs everything here: a private individual's name is never displayed, exported or published, whatever someone types.

**Moderation layers**

1. **At entry:** a slur and threat filter (word list plus a classifier) blocks the submission with a neutral message. Blocked text is logged without a vote attached.
2. **After entry:** the resolution pipeline labels anything that is not a public figure, fictional entity or category as private or unresolved, and routes it to the aggregate bucket.
3. **At display:** only entities at or above the display threshold are named, and only if reviewed. Everything else is counted, not named.
4. **On report:** a takedown address and form let anyone ask for a name to be removed from the data and results, with a stated turnaround.

**Privacy rules**

- No name, email, phone number or login is stored with a vote.
- The device token is a salted hash, and raw IP addresses are not stored. Hashes are deleted when the poll closes.
- Email, if given for verification, is stored only as a keyed hash in a table with no link back to the vote (see verification below).
- Demographics are coarse (age bracket, state, party lean) so that a row cannot identify someone in a small town.
- Entry requires a self-attested 18+ and US-resident checkbox. Anyone who unchecks it is not counted.
- Raw free text is kept internally for analysis. Any published dataset contains only resolved public entities and category labels.

**Legal framing:** the site says plainly that it is an informal poll, not an election, and carries a short privacy notice and terms before launch. A lawyer should review that text, and any claim about the results being "anonymous", before it goes live.

## Verification, anti-abuse and data collected

Voting needs no account. Trust comes in tiers, so the open sample stays large and a cleaner verified subset sits beside it.

**Data collected**


| Data                             | When                     | Stored as                                |
| -------------------------------- | ------------------------ | ---------------------------------------- |
| Raw vote text or category choice | Required                 | As typed, immutable                      |
| Timestamp                        | Required                 | UTC                                      |
| Device token                     | Required                 | Salted hash, deleted at poll close       |
| 18+ and US-resident self-check   | Required                 | Boolean                                  |
| Verified flag                    | Optional, after the vote | Boolean plus verification date (no time) |
| Age bracket                      | Optional, after the vote | 18-24, 25-34, 35-44, 45-54, 55-64, 65+   |
| State                            | Optional, after the vote | Two-letter code                          |
| Party lean                       | Optional, after the vote | Dropdown                                 |
| 2024 presidential vote           | Optional, after the vote | Dropdown                                 |


**Tier 1: open vote.** Anyone can vote. A device-token hash and per-IP rate limits stop repeat voting, and an invisible bot challenge runs only at submit.

**Tier 2: verified vote.** After voting, the person can verify by magic-link email. The link carries a one-time token tied to their vote. When it is clicked:

1. The vote is marked verified.
2. A keyed hash of the email is inserted into a table with a unique constraint, so one email can verify only one vote.
3. No vote ID is stored beside the hash, and the vote keeps only a verified flag and a date.

Phone verification is not built at launch. It is added only if bot patterns appear in the open sample.

**Abuse signals** (flag, do not delete, so analysis can include or exclude them):

- Bursts of identical typed entries from one IP range or device cluster
- Votes submitted faster than a person could read the page
- A sudden spike for a new name, which holds that entry out of public results until reviewed

Flagged votes stay in the raw data with a suspicious marker, so results can always be reported with and without them.

## Results display

Results unlock only after a vote is submitted, so no one sees a leaderboard before choosing. The page shows a cached snapshot, not live counts, so people cannot watch a brigade in real time.

- **Refresh:** the snapshot is rebuilt on a fixed delay (proposed: hourly).
- **Headline view:** the top 20 named entities, plus separate totals for categories (friend, family, coworker), none of the above, fictional or joke entries, and the unresolved or private bucket.
- **Display threshold:** an entity is named only at 5 or more votes and after review. Below that, it is counted in an aggregate bucket.
- **Open vs verified:** two columns side by side, each with its vote count. The gap between them is shown, not hidden.
- **Demographic splits:** shown only where a group has at least 50 votes, so small cells cannot identify anyone.
- **Your vote in context:** "You voted for entity; x% of votes went to the same choice."
- **Labeling:** every results view says "open poll, not representative of the US population."

Weighting to census margins (age, state) is a possible later layer. If added, it is shown as a separate, clearly labeled "experimental weighted" view, never replacing the raw numbers.

## Data model

Seven tables in Postgres. The vote row is small and immutable, and everything interpretive (resolution, review, display status) lives in separate tables, so it can change without touching the original record.


| Table              | Key fields                                                                                                                                | Notes                                             |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| votes              | id, createdat, rawtext, choicetype (typed, suggested, category, none), category, devicehash, verified, verifieddate, suspicious, eligible | Append-only. No edits after insert.               |
| entities           | id, name, type, wikidataid, descriptor, photourl, displaystatus, source, seedgroup                                                        | Public figures, fictional entries and categories. |
| entityaliases      | id, entityid, aliasnormalized                                                                                                             | Grows from reviewer approvals.                    |
| resolutions        | id, voteid, entityid, entitytype, confidence, method (exact, llm, human), version, createdat                                              | One current row per vote, older versions kept.    |
| demographics       | voteid, agebracket, state, partylean, vote2024                                                                                            | Optional. Coarse values only.                     |
| emailverifications | emailhmac (unique), createddate                                                                                                           | Deliberately has no voteid.                       |
| moderationevents   | id, kind (blockedentry, takedown, reviewdecision), entityid, createdat, note                                                              | Blocked text is logged without a vote attached.   |


**Derived and cached:** a results snapshot table (one row per refresh, holding the JSON served to the page) and a promotion-candidates view (typed entries with 5 or more independent votes awaiting review).

**Retention:** device hashes are deleted at poll close. Votes, resolutions and demographics are kept for analysis. Any takedown removes the entity from public views immediately and from exports at the next rebuild.

## Architecture

A small, conventional stack is enough, and every piece is replaceable:

- **Front end:** a server-rendered web app (one vote page, one results page), mobile first.
- **API:** stateless serverless functions that run rate limits, the bot check and the moderation filter, then write the vote.
- **Database:** Postgres for all seven tables, with an append-only votes table.
- **Resolution worker:** a background job that reads unresolved votes, matches them against Wikidata, calls an LLM to adjudicate, and writes resolutions.
- **Review queue:** a small admin page for the weekly human pass.
- **Results:** a scheduled job rebuilds the cached snapshot, which a CDN serves to the web app.
- **Email:** a transactional email provider sends the magic links for verification.

[embedded content: system architecture · 7 components around one database

The request path (web app, API, database) stays fast and simple. Resolution, review and results rebuilds run in the background, so slow work never blocks a vote.

## Analysis plan

The questions the data should answer, each computed on open, verified, and non-suspicious subsets:

- **Concentration:** how much of the vote goes to the top 5 and top 20 entities, and how many distinct entities appear per 1,000 votes.
- **Closeness to home:** the share choosing someone they know versus a public figure.
- **Protest and play:** the combined share of none of the above and fictional or joke entries.
- **Who gravitates where:** top entities by party lean, age bracket and 2024 vote, where cells are large enough.
- **Data quality:** the open-versus-verified gap, the automatic resolution rate, and the share of votes flagged suspicious.



## Risks


| Risk                                        | Mitigation                                                                                      |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Brigading or bot flood skews results        | Device hash, rate limits, flagged votes reported separately, verified tally beside the open one |
| A private person's name is exposed          | Never displayed; aggregate bucket only; takedown path                                           |
| Suggestions anchor the vote                 | 3-character minimum, no popularity ranking, no counts shown                                     |
| Low resolution rate floods the review queue | Alias table grows from reviews; thresholds tuned during soft launch                             |
| Results read as a forecast                  | Persistent "open poll, not representative" label and published method                           |
| Moderation misses abusive text              | Two-stage filter, display threshold, report and takedown                                        |




## Launch plan

1. **Build** the vote flow, resolution pipeline and moderation filter with a hidden results page.
2. **Soft launch** to a few hundred people you know. Check completion rate, resolution rate and the unresolved clusters, then tune thresholds.
3. **Public launch** with results visible. Watch the first thousand votes for bot patterns, and add phone verification only if they appear.
4. **Publish** a short write-up with open and verified results and the method.

Timing matters: the 2026 midterm elections fall on November 3, so a launch in the next two weeks will mix with real-election attention. Decide whether that is a feature or noise.

## Open questions

- Should "myself" be a fixed option beside friend, family member and coworker? - YES
- Do you want to allow under-18 voters, or keep the 18+ gate? (The spec assumes 18+.) - NO
- Should the cleaned dataset be published for researchers, and if so, in what form? 
- Is hourly the right results delay, or should results hold until the poll closes?
- How should fictional, joke and religious entries be grouped in the headline view?

- Should the judicial group cover only the Supreme Court, or every Article III judge? - every judge
- Should former officeholders be seeded, or current ones only?
- What notability cutoff should apply to the AI-generated groups (for example, a minimum number of Wikipedia language editions)?

