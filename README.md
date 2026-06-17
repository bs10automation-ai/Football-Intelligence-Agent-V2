# Football-Intelligence-Agent-V2
An AI-powered match analysis tool for the FIFA World Cup 2026 that estimates outcome probabilities for any two national teams based on live, current-season data rather than historical reputation. Built entirely in n8n, combining a real-time sports data API with Google Gemini for structured statistical reasoning. 
# Football Intelligence Agent V2


**This is not a betting tool.** It is an analytical exercise in building a reliable, explainable AI pipeline that openly flags missing data and never invents facts.

## What it does

A user submits two national team names through a simple web form. The workflow then:

1. Resolves each team name to a unique team ID on Sofascore
2. Pulls five categories of live data for each team in parallel: recent form, current-tournament statistics, FIFA world rankings, squad composition, and head-to-head history
3. Normalizes everything into a single clean JSON object, explicitly marking any missing or low-sample data rather than silently omitting it
4. Builds a constrained prompt and sends it to Gemini for analysis
5. Parses and validates the response (probabilities are checked to sum to exactly 100)
6. Displays a formatted, readable analysis directly on the form's completion screen — no database, no spreadsheet, no email step

The output includes a plain-language prediction, a home/draw/away probability split, key factors driving the estimate, explicitly stated risks and data limitations, a confidence score, and a short summary — always closing with a reminder that this is an analytical estimate, not a betting recommendation.

## Why this approach

Most "AI football prediction" demos either hardcode stats or let the model freely guess. Two design decisions here were deliberate:

**Recent form over reputation.** The system prompt explicitly instructs the model to weigh the last 5–10 matches more heavily than historical prestige or current ranking. A team's name and history shouldn't outweigh the fact that they've lost three of their last five.

**Honesty about data gaps over confident guessing.** Every data-fetching branch can independently fail or come back empty (a team simply may not have World Cup 2026 statistics yet, or no recent head-to-head meeting exists). Rather than letting the model paper over these gaps, every field carries an explicit `dataAvailable` flag, and the prompt instructs Gemini to name the gap and explain how it limits confidence — visible directly in the final "Risks & Uncertainties" section of the output.

## Architecture

```
Form Trigger (Home Team, Away Team)
        │
        ├─ Home Team Search → Extract Team ID ─┬─ Last Matches → Process Form
        │                                        ├─ Statistics Seasons → Select WC2026 Season → Statistics → Process Statistics
        │                                        ├─ Rankings → Process Rankings
        │                                        └─ Squad → Process Squad
        │
        ├─ Away Team Search → Extract Team ID ─┬─ (same five branches)
        │
        └─ Find H2H Match (from already-fetched Last Matches) → H2H Data → Process H2H
                        │
                Merge All Branches (5 inputs)
                        │
                Normalize All Data
                        │
                Build Gemini Prompt
                        │
                Message a Model (Google Gemini, native node)
                        │
                Parse Gemini Response → validate probabilities sum to 100
                        │
                Format Final Output
                        │
                Form Ending (Show Text) → result rendered on completion page
```

## Tech stack

- **n8n** — workflow orchestration (self-hosted on n8n Cloud)
- **Sofascore API via RapidAPI** — live team search, match history, statistics, rankings, squads, head-to-head data
- **Google Gemini 2.0 Flash** — structured natural-language analysis, called through n8n's native Gemini node
- **JavaScript (n8n Code nodes)** — all data transformation, filtering, and validation logic

No database. No external storage. The result lives only for the duration of the request and is rendered straight back to the user.

## Key engineering decisions

**Team identification without ambiguity.** A name-based search can return club teams, youth squads, or teams from the wrong sport. The team-resolution logic filters explicitly for `sport.slug === 'football'`, `national === true`, and adult gender, then ranks by relevance score — preventing, for example, a search for "France" from accidentally matching a youth team.

**Explicit result encoding to prevent model misreading.** An early version passed raw scorelines like `"1-3"` with a separate `venue: "away"` field into the prompt. This is exactly the kind of format a language model can misread — is `1-3` a win or a loss depends on which side is "ours," and combining that with a venue flag adds another layer of indirection. The fix: each recent match is pre-computed into both a human-readable scoreline (`"France 3 - 1 Colombia"`) and an explicit `result: "win" | "draw" | "loss"` field, computed in code rather than left for the model to infer.

**Race-condition-safe data merging.** With five independent data-fetching branches per team (form, statistics, rankings, squad, head-to-head) running in parallel, the normalization step initially read from branches before they had necessarily finished executing, causing intermittent missing data even when the underlying API calls succeeded. This was solved with a dedicated **Merge node** with five inputs, placed immediately before normalization, which n8n guarantees won't fire until all five inputs have arrived — eliminating the non-determinism entirely.

**Head-to-head lookup without a dedicated search endpoint.** The Sofascore API's head-to-head endpoint requires a specific `matchId`, not a pair of team IDs — but there's no endpoint to search for a head-to-head matchId directly. The workaround: scan the already-fetched "last matches" array for the home team and find any past fixture against the away team's ID, reusing data that had already been retrieved instead of making an extra API call. Because World Cup matches are played at neutral venues, the API's raw `homeWins`/`awayWins` fields (which refer to whichever team was historically "home" in that specific past fixture) are explicitly remapped onto the form's actual Home/Away teams by comparing team IDs — avoiding a subtle but real mislabeling bug.

**JSON body construction moved out of the HTTP layer.** An early implementation built the Gemini request body manually inside the HTTP Request node's JSON field, interpolating a multi-line prompt string directly into hand-written JSON. Because the prompt itself contains nested quotes and newlines (it includes a full JSON dump of the match data), this broke n8n's JSON parser. The fix was to construct the entire request body as a JavaScript object inside the Code node and pass it through a single `JSON.stringify()` expression — letting the language handle escaping instead of doing it by hand.

**Free-tier API quota troubleshooting.** Google tightened Gemini API free-tier quotas significantly in December 2025, and a freshly created API key without billing enabled returned a hard `limit: 0` on every request when called through a direct HTTP Request node. Switching to n8n's native **Gemini node** (LangChain-backed "Message a Model" operation), using the exact same API key and project, resolved the issue — the native integration path was not subject to the same quota wall as a raw HTTP call to the same endpoint.

**Output validation, not blind trust.** The model is asked to return three probabilities that sum to exactly 100. The parsing step explicitly checks this (`Math.abs(sum - 100) < 1`) and exposes a `probabilitiesValid` flag, so a future iteration could automatically retry or flag a malformed response rather than silently displaying broken math to the user.

## Sample output

> 🏆 England vs Croatia
>
> Based on recent form and squad strength, England is estimated to be more likely to win this match against Croatia.
>
> **Probabilities** — England: 67% · Draw: 20% · Croatia: 13%
>
> **Key factors** include England's 8-1-1 record across their last 10 matches with a +22 goal difference, a defense that has conceded only twice in that span, and a significantly higher FIFA ranking and squad market value than Croatia.
>
> **Risks & uncertainties** are stated plainly: no head-to-head history was available between the two teams, detailed possession/shot statistics weren't available for either side, and several recent matches were friendlies rather than competitive fixtures — all noted as genuine limitations on the estimate's confidence, not glossed over.

## What I'd improve next

- Add lightweight caching for team ID resolution, since the same national teams get searched repeatedly across different match analyses
- Retry logic for the rare malformed Gemini response, using the existing `probabilitiesValid` flag as the trigger
- A small evaluation set of past, already-decided matches to sanity-check how the model's probability estimates compare to actual outcomes

## About this project

Built as a hands-on project while developing practical skills in AI workflow automation, prompt engineering, and API integration — with a deliberate focus on the parts that are easy to skip in a demo but matter in anything resembling a real tool: handling partial data honestly, validating model output instead of trusting it blindly, and designing prompts that constrain a language model to reason from evidence rather than from vibes.
