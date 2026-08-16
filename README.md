# Ask-your-database

### Building a Text-to-SQL Agent from Scratch

**LangChain + Gemini” a 90-minute, fully explainable build**

This notebook walks through building a Text-to-SQL agent the way a real production system is architected  ” not as a single black-box function call, but as separate, inspectable stages that each do one job well.

## What it does

Given a plain-English question about a sales database (e.g. *"What were total sales by region last quarter?"*), the agent:

1. Reads the real database schema (no guessing table/column names)
2. Checks whether the question is even answerable by this database
3. Writes a safe, schema-grounded SQL `SELECT` query
4. Validates the query before running it
5. Executes it, retrying with the error message if it fails
6. Turns the raw result rows into a plain-English answer

## Architecture

| Stage | Component | What it does |
|---|---|---|
| 1 | **Schema Reader** | Inspects the real SQLite schema and formats it as text the model can read, so it never hallucinates a column or table that doesn't exist |
| 2 | **Scope Guardrail** | Classifies the question as `IN_SCOPE` / `OUT_OF_SCOPE` for this database before any SQL is generated  ” declines off-topic or adversarial questions immediately |
| 3 | **Query Writer** | Prompt-engineered LLM call that outputs a single `SELECT` statement  ” uses role prompting, structured/delimited input, and explicit output-format constraints |
| 4 | **SQL Safety Guardrail** | Code-level validation that rejects anything that isn't a bounded, read-only `SELECT` (blocks `INSERT`, `UPDATE`, `DELETE`, `DROP`, `ALTER`, `ATTACH`, `CREATE`, `REPLACE`, `PRAGMA`)  ” defense in depth on top of the prompt instructions |
| 5 | **Executor with Self-Correction** | Runs the SQL against the real database; on failure, loops back with the actual error message and retries (up to `max_attempts`) |
| 6 | **Response Synthesizer** | Converts the raw query result into a short, plain-English answer for a non-technical business user |

These stages are wired together end-to-end in a single `text_to_sql_agent(question, conn)` function.

## Requirements

- Python 3
- A free Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey)
- Packages (installed in the notebook): `langchain`, `langchain-google-genai`, `langchain-core`, `pandas`, `tabulate`
- `sqlite3` (built into Python  ” no external database or cloud account needed)

## Sample data

The notebook seeds a small, self-contained SQLite database (`workshop_sales.db`) with synthetic sales data across three tables: `regions`, `products`, and `sales`. Data is generated with a fixed random seed so every run is identical.

## How to run

1. Run the cells top to bottom.
2. When prompted, paste your Gemini API key (entered via `getpass`, never printed or saved).
3. The model is initialized as `gemini-2.5-flash` with `temperature=0` for deterministic, repeatable SQL generation.
4. In the "Try your own question" cell, type a sales-related question to see the full pipeline run, or an off-topic question (e.g. "write me a poem") to see the Scope Guardrail decline it.

## Key concepts demonstrated

- **Grounding**  ” the model reads the real schema instead of guessing structure from memory
- **Use-case boundaries**  ” declining questions outside the system's intended scope before spending any SQL-generation calls
- **Prompt engineering**  ” role prompting, structured/delimited input, explicit output format constraints
- **Defense in depth**  ” both prompt instructions and code-level validation guard the database
- **Agent loop**  ” act (run SQL), observe (result or error), correct (retry with the error)  ” not just single-shot chat
- **Sampling parameters**  ” matching `temperature=0` to a task that needs reliability over creativity

## Notes

- Only `SELECT` queries are ever allowed to execute; the safety guardrail rejects anything else outright, regardless of what the model was instructed to do.
- The retry loop caps at `max_attempts` (default 3) to avoid infinite retry cycles on a persistently broken query.
