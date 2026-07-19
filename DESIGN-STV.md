# Meek STV for OpenSlides — cross-service design contract

Goal: ranked-choice multi-winner elections (Meek STV, NZ reference rules) for assignment
polls in a self-hosted OpenSlides fork. Electronic voting only. This document is the
single source of truth for names, payload formats, and JSON schemas shared between
services. **Do not deviate from the identifiers and schemas below** — other agents are
building against them in parallel.

Repo layout (meta-repo at `OpenSlides/`, submodules initialized):
- `OpenSlides/openslides-backend` — Python. Poll actions, models, counting.
- `OpenSlides/openslides-client` — Angular. Voting UI, results page, exports.
- `OpenSlides/openslides-vote-service` — Go. Ballot validation/collection.
- `OpenSlides/openslides-backend/meta` and `OpenSlides/lib/openslides-go/meta` — the
  shared openslides-meta repo (same commit) defining `collections/*.yml`, from which
  backend `models.py`, Go field accessors, and the SQL schema are generated.

## 1. Identifiers

- New `poll.pollmethod` value: **`"rank"`** (ballot shape: option→rank mapping).
  Counting algorithm is separate from ballot shape so other counters (Scottish STV,
  Borda, Hare quota variants) can be added later without touching ballot format.
- New poll field: **`rank_result`** (JSON, nullable). Written by backend on poll stop,
  read by client for the results page and exports. In `meta/collections/poll.yml`, use
  the same `restriction_mode` as `votesvalid` so no autoupdate-service code changes
  are needed.
- New poll field: **`rank_algorithm`** (string enum, nullable): `"meek-nz"` |
  `"scottish-stv"` | `"borda"`. Selects the counting algorithm for rank polls.
  `null`/absent means `"meek-nz"`. Only settable when `pollmethod == "rank"`.
- New poll field: **`rank_quota`** (string enum, nullable): `"droop"` | `"hare"`.
  Selects the quota rule for STV algorithms. `null`/absent means `"droop"`.
  Only settable when `pollmethod == "rank"`; accepted but ignored for `"borda"`
  (recorded as `null` in the result). Both fields use `restriction_mode: A` in
  `meta/collections/poll.yml` (same as `pollmethod`).
- Default algorithm: **`"meek-nz"`**. Recorded inside `rank_result` (the
  `algorithm` id names the computational rules; the quota rule is configurable
  via `rank_quota` and recorded separately as `quota_rule`).
- `pollmethod: "rank"` is allowed **only for assignment polls** and only for
  electronic types (`named`, `pseudoanonymous`) — never `analog`.
- Seats = the assignment's `open_posts` at poll stop time (no separate poll field).
- `onehundred_percent_base` must be `"disabled"` for rank polls.
- `global_abstain` may be true (explicit abstain ballot); `global_yes`/`global_no`
  must be false. `min_votes_amount`/`max_votes_amount`/`max_votes_per_option` are
  ignored for rank polls (leave defaults). `live_voting_enabled` must be false.

## 2. Ballot payload (client → vote service `POST /system/vote?id={poll_id}`)

```json
{ "value": { "<option_id>": 1, "<option_id>": 2, "<option_id>": 3 } }
```
- Keys: option IDs (strings of ints, as in existing methods), subset of `poll.option_ids`.
- Values: ranks. Must be distinct integers forming the contiguous range `1..k`, `k >= 1`.
- Partial ballots allowed (rank any non-empty subset).
- Explicit abstain (only if `poll.global_abstain`): `{ "value": "A" }` (reuses the
  existing global-vote string path).
- Everything else about voting (auth, presence, delegation, vote weight, named vs
  pseudoanonymous user stripping, double-vote prevention) uses the existing machinery
  unchanged.

Vote-service validation for `pollmethod == "rank"` rejects: empty map, unknown option
IDs, duplicate ranks, non-contiguous ranks, ranks < 1, global "Y"/"N", and "A" when
`global_abstain` is false.

## 3. Backend storage on poll stop

In `StopControl.on_stop()` (openslides_backend/action/actions/poll/mixins.py), for
rank polls:

- Each ranked ballot → **one Vote record on the poll's global option** with
  `value` = compact JSON string of the ranking map sorted by rank, e.g.
  `{"12":1,"15":2,"9":3}`. Weight = ballot weight (existing 6-dp decimal string).
  user_id/delegated_user_id/user_token handled exactly like existing methods.
- Abstain ballots go through the existing string path: Vote on global option,
  `value = "A"`, aggregated into the global option's `abstain` field.
- `votescast` = all ballots incl. abstain; `votesvalid` = same (invalid ballots are
  rejected at vote time, so `votesinvalid` stays 0).
- Each option's `yes` field is set to its **weighted first-preference total** (6-dp
  decimal string) so existing generic tables/charts show something sensible.
- Then run the counter (§4) and write its output to `poll.rank_result` (§5).
- Seats: fetch the assignment via `content_object_id`, use `open_posts` (default 1 if
  unset/invalid). Tie-break seed: random 31-bit int generated at stop time.

## 4. Counting module (backend, pure Python, pluggable)

Package: `openslides_backend/rank_voting/` — no imports from action/model code, so it
stays independently testable and reusable.

```python
# rank_voting/base.py
@dataclass
class RankBallot:
    weight: Decimal          # > 0
    ranking: list[int]       # candidate (option) ids, most preferred first, no dups

@dataclass
class RankResult:
    ...                      # fields mirror the rank_result JSON schema in §5
    def to_json(self) -> dict: ...

class RankCounter(ABC):
    name: ClassVar[str]      # e.g. "meek-nz"
    def count(
        self,
        ballots: list[RankBallot],
        candidate_ids: list[int],   # all option ids in ballot-paper order
        seats: int,
        seed: int,                  # for reproducible tie-breaking
        quota_rule: str = "droop",  # "droop" | "hare"; ignored by borda
    ) -> RankResult: ...

# rank_voting/registry.py
def register_counter(cls) / get_counter(name: str) -> RankCounter
```

`rank_voting/meek.py` implements NZ Meek per the NZ Local Electoral Regulations 2001
Schedule 1A (Meek's method as specified by Hill/Wichmann/Woodall, "Algorithm 123 —
Single Transferable Vote by Meek's method", Computer Journal 30(3), 1987):

- All arithmetic with `Decimal`, 9 decimal places, **truncation** (ROUND_DOWN) where
  the regulations specify truncation.
- Keep factors start at 1; iterate: distribute each ballot down its ranking, each
  candidate keeps `weight_so_far * keep_factor`, passes on the rest; quota =
  (total votes − exhausted) / (seats + 1) at 9 dp (Droop; with `quota_rule ==
  "hare"`: (total votes − exhausted) / seats), recomputed each iteration; elected
  candidates' keep factors converge via `kf = kf * quota / votes` iteration until
  surpluses are below 1e-9.
- Exclude the lowest candidate when no one new reaches quota; ties broken
  pseudo-randomly with `random.Random(seed)` and recorded in the round data.
- Elect by quota; when remaining candidates == remaining seats, elect them all.
- Every round appended to the result (§5 `rounds`).

`rank_voting/scottish.py` implements **Scottish STV** (name `"scottish-stv"`) per
the Scottish Local Government Elections Order 2007 (SSI 2007/42), Weighted
Inclusive Gregory Method:

- All arithmetic with `Decimal`; transfer values **truncated to 5 decimal places**.
- Quota computed **once** at stage 1 and never recomputed:
  - `droop` (the official rule): `floor(total_valid / (seats + 1)) + 1` (an integer).
  - `hare`: `floor(total_valid / seats)`, minimum 1 (an integer).
- Stage 1: each ballot counts for its first preference at full ballot weight.
- A candidate whose total ≥ quota is elected (marked in that stage). Surpluses are
  transferred one per stage, largest surplus first: **every** ballot paper held by
  the elected candidate transfers to its next *hopeful* preference at
  `new_value = paper_value × (surplus / candidate_total)` truncated to 5 dp; the
  elected candidate retains exactly the quota. Surplus value on papers with no
  hopeful next preference becomes non-transferable (added to `exhausted`).
- When no surplus remains untransferred and seats are unfilled: exclude the
  candidate with the fewest votes; all their papers transfer at **current value**
  to the next hopeful preference (no next preference → exhausted).
- Elect all remaining hopefuls when hopefuls == remaining seats; stop when all
  seats are filled.
- Ties (fewest votes for exclusion, or equal surpluses): decided by the
  candidates' totals at the **earliest stage where they differ** (stage 1
  forward); if equal at every stage, pseudo-randomly via `random.Random(seed)`.
  Recorded in the stage's `tie_break`.
- One `rounds` entry per stage: `votes` = totals at the end of the stage,
  `keep_factors` = `{}` (empty — not a Meek concept), `quota` = the static quota,
  `exhausted` = cumulative non-transferable value, elected/excluded/tie_break
  as they occur.

`rank_voting/borda.py` implements the **Borda count** (name `"borda"`):

- With `n = len(candidate_ids)`: a ballot gives its rank-`r` candidate
  `(n − r)` points × ballot weight (1st preference gets `n − 1`); unranked
  candidates get 0 from that ballot. Partial ballots thus simply award fewer
  points. Not an STV method — `quota_rule` is ignored and recorded as `null`.
- The `seats` highest-scoring candidates are elected (in descending score
  order). A tie at the seat boundary is broken pseudo-randomly via
  `random.Random(seed)` and recorded in `tie_break`.
- Result shape: a **single** `rounds` entry — `number: 1`, `quota: null`,
  `votes` = the weighted point totals, `exhausted: "0"`, `keep_factors: {}`,
  `elected` = elected list, `excluded: null`. Top-level `quota_final: null`,
  `exhausted_final: "0"`.

Unit tests in `tests/unit/rank_voting/`: single-seat reduces to IRV behavior;
published Meek worked example; exhausted ballots; weighted ballots; seeded tie-break
reproducibility (same seed ⇒ same result, different seed can differ); seats ≥
candidates edge case; all-abstain/empty edge cases.

## 5. `poll.rank_result` JSON schema

```json
{
  "version": 1,
  "algorithm": "meek-nz",
  "quota_rule": "droop",
  "seats": 2,
  "seed": 123456789,
  "candidates": [12, 15, 9],
  "elected": [15, 12],
  "quota_final": "25.000000000",
  "ballot_count": 100,
  "abstain_weight": "3.000000",
  "exhausted_final": "4.123456789",
  "rounds": [
    {
      "number": 1,
      "quota": "33.333333333",
      "votes": { "12": "30.000000000", "15": "45.5", "9": "10.0" },
      "exhausted": "0.000000000",
      "keep_factors": { "12": "1", "15": "0.732600732", "9": "1" },
      "elected": [15],
      "excluded": null,
      "tie_break": null
    },
    {
      "number": 2,
      "quota": "…", "votes": {}, "exhausted": "…", "keep_factors": {},
      "elected": [], "excluded": 9,
      "tie_break": { "among": [9, 12], "chosen": 9 }
    }
  ]
}
```
All decimals serialized as strings. `candidates` is ballot-paper order (used for BLT
candidate indexing 1..n). `elected` is in order of election.

`algorithm` is `"meek-nz"`, `"scottish-stv"`, or `"borda"`. `quota_rule` is a
required field: `"droop"`, `"hare"`, or `null` (borda, which has no quota —
`quota`/`quota_final` are also `null` there). No backward compatibility with
pre-`quota_rule` results is needed; existing test polls can be discarded.

## 6. Client

- `PollMethod.Rank = "rank"` in poll-constants.ts; verbose name "Ranked choice
  (Meek STV)"; offered only in the **assignment** poll form.
- Form (assignment-poll-form + base-poll-form): when rank is selected — hide
  min/max votes amount, max votes per option, global yes/no (keep global abstain
  toggle), force `onehundred_percent_base: "disabled"`, disable live voting and
  analog type. Additionally show two selects: **Counting algorithm**
  (`rank_algorithm`: "Meek STV (New Zealand rules)" / "Scottish STV" /
  "Borda count", default meek-nz) and **Quota** (`rank_quota`: "Droop quota" /
  "Hare quota", default droop, hidden when borda is selected).
- Voting UI (assignment-poll-vote): two connected CDK drop lists — "Candidates" pool
  and numbered "Your ranking" — drag or [+]/[–] tap to move, drag to reorder; abstain
  via existing global-abstain button; submit builds the §2 payload. Must work on
  mobile (tap fallback) and support delegated voting like existing methods.
- Results page (assignment-poll-detail): when `pollmethod === "rank"` and
  `rank_result` present, show an STV results block instead of the generic Y/N/A
  tables: elected candidates (in order, with quota), summary line (ballots, abstains,
  seats, algorithm, quota rule, seed), and a round-by-round table (per-candidate
  votes and keep factors per round, exhausted column, elected/excluded markers,
  tie-break notes). The block renders per algorithm: **meek-nz** as described;
  **scottish-stv** the same but the table is labeled "Stage" and the keep-factor
  rows/columns are omitted (`keep_factors` is empty); **borda** shows a single
  points table (candidate, points, elected marker) with no quota/exhausted/round
  concepts.
  Votes table (named polls): render a ballot's JSON value as "1. Carol · 2. Dave · …".
- Exports (poll detail, for finished/published polls): client-side downloads —
  **BLT** (`<n_candidates> <seats>` header; one line per ballot `weight pref… 0`
  using 1-based candidate indexes in `rank_result.candidates` order; terminating `0`;
  quoted candidate names; quoted title), **CSV** (one row per ballot:
  ballot number, weight, choice_1..choice_k as candidate names), **JSON** (poll meta +
  candidates + raw ballots + full `rank_result`). Ballots reconstructed from the vote
  records on the global option (skip "A" values for BLT/CSV ballot lines; report
  abstain count in JSON). For non-anonymous polls (`type == "named"` and not
  `is_pseudoanonymized`) the ballots carry the voter identity: CSV gets
  `voter` and `structure_level` columns after `ballot`, JSON ballots get a
  `voter: { name, structure_level }` object, and ballots are sorted by voter
  name; BLT stays voter-free (the format has no field for it). Anonymized or
  pseudoanonymous polls export without any voter data. Non-integer weights are written as-is in BLT (note: some
  third-party tools only accept integer weights).
- Ballot papers PDF (assignment-poll-pdf.service, "Ballot papers" menu entry):
  for rank polls each ballot prints an instruction line ("Rank the candidates in
  order of preference: 1 for your first choice, …") followed by one empty
  14×14 pt box per candidate (to write the rank number into) and — if
  `global_abstain` — an "Abstain" circle. Y/YN/YNA ballots are unchanged.

## 6a. Projector service (openslides-projector-service)

- `PollSlideHandler` routes polls with `pollmethod: "rank"` to a dedicated
  `poll_rank` slide (`pkg/projector/slide/poll_rank.go`,
  `templates/slides/poll_rank.html`, `web/src/slide/poll_rank.css`) for every
  projection variant (incl. requested single-votes projections — ranked ballots
  are never shown as a Y/N/A grid). Non-published states keep the generic
  state texts ("Voting in progress", …).
- The slide parses `poll.rank_result` (§5) and shows: subtitle
  "<algorithm> · <quota rule>", a table of all candidates — elected first in
  order of election (bold, "✓ Elected"), then the rest by final vote total
  (Borda: points) with "Excluded (Round/Stage n)" markers — and the key
  figures (Seats, Quota, Ballots, Abstentions, Non-transferable votes,
  Invalid votes, Counting rounds/stages) as one compact `rank-sums` line
  between the title block and the table (scrolls with the content, not
  pinned), NOT as 45px table rows (those made even a 3-candidate slide
  overflow a 16:9 projector). Decimals are shortened to 3 places. The
  error variant of `rank_result` renders "Counting of the votes failed:
  <error>".
- The service reads `Poll_RankResult` through the shared generated models in
  `lib/openslides-go` (regeneration chain of §7 covers it; the dev compose
  mounts `lib/openslides-go` into the container).

## 7. Meta/models regeneration chain (backend integration agent)

1. Edit `openslides-backend/meta/collections/poll.yml`: add `"rank"` to the
   `pollmethod` enum; add `rank_result` (type `JSON`, restriction_mode same as
   `votesvalid`); add `rank_algorithm` (string enum meek-nz/scottish-stv/borda)
   and `rank_quota` (string enum droop/hare), both nullable, restriction_mode A.
2. `make join-models-yml` + `make generate-models` in openslides-backend (inspect
   Makefile/cli for exact flow); check how the relational schema/migrations pick up
   new fields (`migrations/`, `0100_init_reldb.py` era conventions) and add whatever
   migration the repo's conventions require.
3. Mirror the same meta edit in `lib/openslides-go/meta` (init that nested submodule
   if needed), regenerate Go artifacts (`metagen`, dsfetch/dskey generated files,
   `schema_relational.sql`) so autoupdate/vote services see the field. The
   autoupdate service needs no code change if restriction_mode reuses an existing
   mode for poll.
4. Note for e2e: forked Go services may need `go.mod replace` pointing at
   `lib/openslides-go` to pick up regenerated code.

## 8. Validation summary (backend poll actions)

- create/update: `pollmethod == "rank"` ⇒ content object is an assignment; type in
  {named, pseudoanonymous}; `onehundred_percent_base == "disabled"`;
  `global_yes`/`global_no` false; `live_voting_enabled` false/absent.
  `rank_algorithm`/`rank_quota` may only be set when the poll's method is
  `"rank"` (enum values enforced by the model); both nullable with backend
  defaults `"meek-nz"`/`"droop"` applied at count time.
- stop: `get_counter(poll.rank_algorithm or "meek-nz").count(..., quota_rule=
  poll.rank_quota or "droop")`, write `rank_result`, votes/options per §3.
  Counter failures must
  not lose ballots — ballots are persisted as Vote records before counting; if the
  counter raises, store the error (`rank_result = {"version":1, "error": "..."}`).
- reset: clears `rank_result` along with existing vote-clearing behavior.
