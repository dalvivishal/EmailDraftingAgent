# Email Drafting Agent

A minimal **CrewAI** two-agent system that turns a one-line description of a situation into a
complete, send-ready professional email. An analyst agent first converts the raw context into a
structured brief; a writer agent then drafts the email from that brief.

Splitting the work in two is the point: a single-prompt LLM call tends to start writing before it
has decided what the email is *for*. Forcing an explicit planning step — purpose, key points,
call to action, subject line — produces drafts with a clearer structure and a real CTA.

**Framework:** CrewAI (sequential process) • **LLM:** `gpt-4o-mini` • **Interface:** CLI

---

## Table of Contents

- [How it works](#how-it-works)
- [The agents](#the-agents)
- [Installation](#installation)
- [Usage](#usage)
- [Example output](#example-output)
- [Using it as a library](#using-it-as-a-library)
- [Customizing](#customizing)
- [Project layout](#project-layout)
- [Cost and latency](#cost-and-latency)
- [Troubleshooting](#troubleshooting)

---

## How it works

```
  --context / --tone / --recipient
              |
              v
  +-------------------------------+
  |  Email Context Analyst        |   analyze_task
  |  extracts: purpose, key       |
  |  points, CTA, subject line    |
  +---------------+---------------+
                  |  Task(context=[analyze_task])
                  |  the brief is passed forward automatically
                  v
  +-------------------------------+
  |  Professional Email Writer    |   write_task
  |  subject, greeting, body,     |
  |  closing, signature (<200 w)  |
  +---------------+---------------+
                  v
           printed to stdout
```

`Process.sequential` runs the tasks in order. The wiring that carries the analyst's output into
the writer's prompt is the `context=[analyze_task]` argument on `write_task` — CrewAI injects the
completed task's output rather than the two agents sharing a chat history.

`temperature=0.3` keeps the tone stable across runs while allowing some variation in phrasing.
Both agents run with `verbose=False`, so the only output is the finished email.

---

## The agents

| | **Email Context Analyst** | **Professional Email Writer** |
|---|---|---|
| **Goal** | Understand the context, extract key points, define the structure | Draft clear, concise, effective professional emails |
| **Backstory** | An expert business communication analyst who distills complex situations into clear email requirements | A professional copywriter specializing in business emails that get responses |
| **Produces** | A structured brief: purpose, key points, CTA, suggested subject line | A complete formatted email ready to send |

The writer is explicitly constrained to keep the body **under 200 words** and to include a subject
line, greeting, body paragraphs, closing, and a signature placeholder.

---

## Installation

Requires **Python 3.9+**.

```bash
cd EmailDraftingAgent

python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

pip install -r requirements.txt
```

Pinned dependencies:

```
crewai==0.80.0
langchain-openai==0.2.0
python-dotenv==1.0.1
```

### API key

Create a `.env` file in this directory:

```
OPENAI_API_KEY=sk-...
```

`agent.py` calls `load_dotenv()` at import, so the key is picked up automatically. An exported
shell variable works equally well.

---

## Usage

### Defaults

```bash
python agent.py
```

Runs a built-in example — a follow-up to a product demo from last Tuesday, addressed to a
potential client, in a professional and friendly tone.

### Custom email

```bash
python agent.py \
  --context "Apologize for the delayed delivery of the software project" \
  --tone "apologetic but confident" \
  --recipient "the client project manager"
```

### Arguments

| Flag | Default | Description |
|---|---|---|
| `--context` | *"Follow up on our product demo from last Tuesday. They seemed interested but haven't responded."* | What the email is about. The more situational detail you give, the better the brief. |
| `--tone` | `"professional and friendly"` | Free text. Try `"formal"`, `"warm but direct"`, `"apologetic but confident"`, `"urgent"`. |
| `--recipient` | `"a potential client"` | Who receives it. Drives salutation and register — `"my manager"` and `"a vendor we are about to drop"` produce very different emails. |

All three are plain strings interpolated into the task descriptions, so there is no fixed
vocabulary to learn.

### More examples

```bash
# Negotiating a deadline extension
python agent.py \
  --context "We need two more weeks on the API migration because of an unexpected auth rewrite. We are still on track for the overall Q4 date." \
  --tone "transparent and solution-oriented" \
  --recipient "the engineering director"

# Cold outreach
python agent.py \
  --context "Introduce our document-intelligence product to a law firm that handles high-volume discovery" \
  --tone "concise and value-focused" \
  --recipient "a managing partner at a mid-size law firm"

# Declining politely
python agent.py \
  --context "Decline an invitation to speak at a conference next month due to a release deadline, but offer to do it next year" \
  --tone "warm and appreciative" \
  --recipient "the conference organizer"
```

---

## Example output

```
✉️  Drafting email...

============================================================
📧 DRAFTED EMAIL
============================================================
Subject: Quick follow-up on last week's demo

Hi [Name],

Thank you for taking the time to walk through the platform with us last Tuesday.
You raised a good question about how the reporting module handles multi-team
rollups, and I wanted to make sure it did not go unanswered.

...

Would Thursday or Friday afternoon work for a 20-minute call?

Best regards,
[Your Name]
```

Exact wording varies between runs — `temperature=0.3` is low but not deterministic.

---

## Using it as a library

`build_email_crew()` is a plain function and can be imported directly:

```python
from agent import build_email_crew

email = build_email_crew(
    context="Remind the client that invoice #4021 is 15 days overdue",
    tone="firm but polite",
    recipient="the accounts payable contact",
)
print(email)
```

It returns the writer's output as a string. Note that the crew is constructed fresh on every
call, so there is no state carried between invocations.

---

## Customizing

**Change the model** — edit the `ChatOpenAI` line in `agent.py`:

```python
llm = ChatOpenAI(model="gpt-4o", temperature=0.3)
```

**Change the length limit** — the "under 200 words" constraint lives in `write_task`'s
`description`. Edit that string.

**Add a third agent** — a reviewer that checks tone and grammar before output:

```python
reviewer = Agent(
    role="Email Quality Reviewer",
    goal="Verify tone, grammar, and that the call to action is unambiguous",
    backstory="A meticulous editor who has caught every embarrassing typo before it shipped.",
    llm=llm,
    verbose=False,
)

review_task = Task(
    description="Review the drafted email. Fix tone drift, grammar, and weak calls to action.",
    agent=reviewer,
    expected_output="The final polished email",
    context=[write_task],
)

crew = Crew(agents=[analyst, writer, reviewer],
            tasks=[analyze_task, write_task, review_task],
            process=Process.sequential, verbose=False)
```

**See the agents reason** — flip `verbose=True` on the agents and the crew to print each step's
thinking. Useful when a draft comes out wrong and you need to know whether the analyst or the
writer caused it.

---

## Project layout

```
EmailDraftingAgent/
├── agent.py           # build_email_crew() + argparse CLI
├── metadata.yaml      # catalog metadata (framework, tags, difficulty, entrypoint)
├── requirements.txt   # pinned dependencies
├── README.md
└── .gitignore
```

### `metadata.yaml`

Descriptive metadata for agent catalogs and registries — framework, LLM, tags, difficulty,
entrypoint. It is not read by `agent.py` at runtime.

---

## Cost and latency

One run makes two `gpt-4o-mini` calls (one per task), plus CrewAI's own framework overhead.
At `gpt-4o-mini` pricing a single draft costs a fraction of a cent, and typically completes in
5–15 seconds depending on API latency.

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `AuthenticationError` / `api_key must be set` | No `OPENAI_API_KEY`. Create `.env` in this directory, or export the variable in your shell. |
| `ModuleNotFoundError: crewai` | The virtualenv is not active, or dependencies were not installed. Re-run `pip install -r requirements.txt`. |
| Draft is generic and vague | `--context` is too thin. Feed it real specifics: names, dates, what was said, what you want back. |
| Email is far longer than 200 words | The word limit is a prompt instruction, not a hard cap. Tighten the wording in `write_task`, or add a reviewer agent. |
| Dependency resolution conflicts | The pins are exact and `crewai==0.80.0` is opinionated about its `langchain` versions. Install into a clean virtualenv. |
| Output is wrapped in stray markdown fences | The model occasionally formats its answer as a code block. Strip fences from the returned string, or add "return plain text, no markdown" to `write_task`. |
