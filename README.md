# Don't Let the Model Write the Query

**Software architecture for repeatable chat responses over SQL data.**

A talk by John Ulett · SageTechTN Vibe Coder

- 📄 [Slides (PDF)](dont-let-the-model-write-the-query.pdf) · 🖼 [Slide images](slides/)
- 🗒 [Slide-by-slide speaker notes](SPEAKER-NOTES.md)

The short version: **the model chooses, the code composes.** One model call classifies
what *shape* of answer the question wants. Everything after that — resolving entities,
building the SQL, validating it, running it, rendering it — is deterministic code.

---

## The problem: repeatability is not accuracy

Every text-to-SQL benchmark measures the same thing: *is the generated SQL correct?*
That is a benchmark property. The property a production system needs is different:
*is it the same SQL every time?*

A system that is right 90% of the time and differently right on each run is unshippable
for four separate reasons, and only one of them is correctness. You can't cache a sampled
query. You can't unit-test one. You can't audit one. And when a user says *"this number
changed and I don't know why,"* you have nothing to show them but a token log.

> I'd rather be 85% right the same way every time than 95% right in a way I can't reproduce.

## Three ways to build this

|  | The model returns | Failure mode |
|---|---|---|
| **Text-to-SQL** | Any string | Confidently wrong SQL, differently each time |
| **Semantic match over a query library** | A retrieved snippet | Nearest neighbour ≠ right answer shape |
| **Classify the row unit, then compose** | A typed struct | Confident mis-routing |

This is the third one, and it gets confused with the second constantly. Semantic search
over a template library matches on *phrasing*. This architecture asks a different question
entirely.

## The pipeline

```
question → ROUTER → entity resolution → SQL build → guard → execute → format → answer
            ▲
        the model            └──────────── deterministic code ─────────────┘
```

The router is not choosing a query. It answers one question first: **what is one row of
the answer?** A search? A person? A candidate in a pipeline? An aggregate bucket?

Once you know the row unit, the template is implied. That is why the library is two dozen
templates and not two hundred — they are organized by result *shape*, not by question
phrasing.

## The anchor: a typed output surface

This is the entire surface through which the model influences the database:

```python
class RouterDecision(BaseModel):
    intent: Literal["query", "expansion", "clarify_response", "chitchat"]
    template_id: Optional[str] = None
    question_type: Optional[Literal["count", "list", "table", "summary", "scalar"]] = None
    parameters: list[RouterParam] = Field(default_factory=list)
    expansions: list[str] = Field(default_factory=list)
    confidence: float = 0.0
    clarifying_question: Optional[str] = None
    limit: Optional[int] = None
    sort: Optional[str] = None
```

Nine fields. Four closed enumerations. Zero ways to name a table.

This isn't a prompt asking politely for JSON — it's the response format, constrained by the
API. The model cannot emit SQL, cannot name a table, cannot add a column or reach a row it
shouldn't, because **there is no field in which to express any of that.**

Compare to text-to-SQL, where the output surface is *any string* and the only thing between
it and your database is a paragraph asking it to behave.

The transferable idea, and it isn't about SQL: **constrain the model's output surface, not
its input.**

## Templates are data, not code

A template is a parameterized SQL frame plus a description of what can be filtered and what
can be shown. It lives as a config row in the database, so adding a capability is an
`INSERT`, not a deploy.

```yaml
base_sql: |
  SELECT {columns} FROM thrive_searches s
  WHERE 1=1 {filters}
  ORDER BY {order_by} LIMIT {limit}
filters:
  client:  {sql: "AND s.company_name = %(client)s"}
  status:  {sql: "AND s.status = %(status)s"}
  title:   {sql: "AND s.job_title LIKE %(title)s"}
```

Two load-bearing invariants:

1. **User values are never interpolated, only bound.** That `%(client)s` stays a
   placeholder all the way into the driver.
2. **Structural SQL is never model-authored.** Even sorting: the model picks a sort *key*,
   a short string, and a human wrote the `ORDER BY` clause that key maps to.

## "But templates don't scale"

This is the objection everyone has. The instinct is *one template equals one answer*, and
then you imagine maintaining four hundred.

One real template has 14 freely-combining filters — 16,384 combinations on their own —
times 5 result shapes, times 2¹³ subsets of its optional columns, times sort key, row limit
and follow-up state. From one hand-written SQL frame.

One template isn't one answer. It's one **result shape**. And business domains have a
useful property: the question space is infinite, but the shape space is small — and it
stops growing while the questions never do.

## Where domain knowledge goes to live

My favourite bug in the system, because everything about it was working:

A user asked for `"VP of Operations"`. The query became `%VP of Operations%`. It matched
zero live searches. The real titles were *Market VP Operations*, *SVP Operations (IDD)*,
*COO/VP Operations*. Question clear, routing correct, SQL valid, answer empty — because
nobody stores a job title the way anybody asks for it.

The fix was ten lines: drop stopwords, match significant words in order. It went into one
filter specification, once, and every job-title query in the system got permanently better.

Now ask what happens to that fix when the model writes SQL fresh on every call. There is
nowhere to put it, except an ever-growing prompt that re-teaches your data's quirks on
every request. Templates are where institutional knowledge accumulates — and that, more
than safety, is the compounding argument.

## One question, all the way down

> "open searches for Northwind with recruiter and candidate counts"

**Everything the model said:**

```json
{ "intent": "query",  "template_id": "template_search_01",
  "question_type": "table",  "confidence": 0.94,
  "parameters": [{"name":"client", "value":"Northwind"},
                 {"name":"status", "value":"Open"}],
  "expansions": ["recruiter", "candidate_count"] }
```

It translated "open" through a business glossary, and turned "with recruiter and candidate
counts" into two *column keys* — not into SQL, into two strings that must match keys the
template declares.

**The SQL the code built** — a deterministic function of the struct and the template record,
no model involved:

```sql
SELECT s.name AS `Search`, s.company_name AS `Client`,
       s.assigned_to_name AS `Recruiter`,
       s.candidacies_count AS `Candidates`
FROM thrive_searches s
WHERE 1=1 AND s.company_name = %(client)s
           AND s.status = %(status)s
ORDER BY s.created_at DESC LIMIT 100
```

The model said `"Northwind"`; code resolved that to `"Northwind Health"` by fuzzy match —
not embeddings, not a second model call. The user typed a company name and that text is
still not in the SQL string. There is no injection surface here, not because we sanitized
well, but because **user text never becomes SQL text.**

## The guard

Every statement is parsed into an AST before it executes. Not regex — parsed.

1. Exactly one statement, and it must be a `SELECT`
2. Every table on the allow-list
3. No restricted column in the projection
4. `LIMIT` injected when missing, clamped when too high

Rule three has a subtlety worth your time: the restriction is on **what reaches the user**,
not what the query *touches*. A foreign key in a `JOIN` or a `WHERE` is fine. The same
column in the `SELECT` list is rejected — so the guard resolves table aliases and walks
into scalar subqueries in the projection, because that is a real path to the user's screen.

The same guard runs on template SQL and on model-written SQL. **The trust boundary is in
code, after the model — never in the prompt.**

## The escape hatch

This is not an argument against model-generated SQL. It is an argument against it being the
*default*. Some questions are genuinely one-offs, and answering them imperfectly beats
refusing.

- **Below the confidence threshold**, SQL is written from a schema description with
  forbidden tables and columns *removed* — absent, not merely prohibited.
- **Guarded identically** — same AST parse, same allow-list, same projection rules.
- **Logged and flagged** — every fallback lands in the query log with its confidence and
  its reason.

Every flagged fallback is a template you haven't written yet. The fallback rate is a KPI,
and it should trend toward zero. Your escape hatch should also be your backlog.

## Honest costs

- **Coverage grows linearly with human effort.** A new result shape means someone writes
  SQL. That is a real ceiling.
- **The template library is a second schema**, kept in sync with the real one by hand.
- **Confident mis-routing is the scary failure.** Bad generated SQL errors loudly. A wrong
  template returns a beautiful, correctly formatted table answering a question nobody
  asked. That is why every turn logs its template id and confidence — it's the only way to
  find these after the fact.
- **Config-as-data has no type checker.** A malformed template row degrades quietly rather
  than failing the build.

Two mitigations worth stealing: confidence thresholds shouldn't be cliffs — if the model
picks a valid template *and* extracts a real filter but lands just under the bar, run it
and log it as its own route class, because a vetted query beats freeform SQL. And when the
shape is genuinely ambiguous, "ask a clarifying question" is a valid output.

## Takeaways

1. **Constrain the model's output surface, not its input.** A nine-field struct beats any
   amount of prompt engineering.
2. **Put the trust boundary in code, after the model.** Write it as though the prompt
   already failed. Sometimes it has.
3. **Make the escape hatch measurable.** The thing you fall back to should tell you what to
   build next.

The model is doing something genuinely hard here — resolving pronouns against conversation
state, translating business vocabulary, deciding when to ask instead of guess. That's real
intelligence. It just isn't the intelligence that should be writing your `WHERE` clause.

**The model chooses. The code composes.**

---

## License

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](LICENSE)

Slides, speaker notes and prose © John Ulett, licensed under
[CC BY 4.0](LICENSE) — share and adapt freely, including commercially, with attribution.

The code snippets are short illustrative excerpts; consider them MIT if you want to lift
them into something.
