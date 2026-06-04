# Math Tutor — Ukrainian Math Problem Generator (RAG + Multi-Agent)

A small research project that **generates math problems in Ukrainian**
from a topic, together with a correct answer, and then **automatically verifies**
that answer with a computational tool.

The notebook follows a classic *baseline → improvement* structure with two
experiments.

| | Experiment 1 (Baseline) | Experiment 2 (Multi-agent) |
|---|---|---|
| Context | static corpus + TF-IDF search | dynamic Wikipedia (RAG) |
| Agents | Teacher + Judge (2) | Teacher (A) + Solver (B) + Judge (3) |
| Solving | Teacher answers itself | Solver agent + SymPy code |
| Verification | real Wolfram Alpha API | SymPy symbolic equivalence |

## Model & tool choices

* **Generation / solving — Google Gemini (`gemini-3.5-flash`).** Chosen for strong
  Ukrainian-language generation, low latency. The same
  model powers both the Teacher and the Solver.
* **Verification — tools, not a model.** Because LLMs make arithmetic mistakes,
  the answer key is produced by a deterministic engine:
  * **Experiment 1 — Wolfram Alpha (Short Answers API).** Used as the baseline: the problem is translated into a Wolfram query and the result
    is compared to the Teacher's answer.
  * **Experiment 2 — SymPy.** The Solver emits a SymPy expression that is evaluated
    in a sandboxed namespace; the result is compared symbolically.

## Experiment 1 — Baseline

**Architecture.** A single Teacher (Gemini) retrieves the most relevant chunk from
a static corpus via TF-IDF, then **creates and solves** a problem in one pass
(`## ЗАДАЧА / ## РОЗВ'ЯЗОК / ## ВІДПОВІДЬ`). The static corpus was artificially
generated using Claude and GPT. The Judge verifies the answer with the
**Wolfram Alpha Short Answers API**: it translates the problem into a Wolfram
query, sends it, and compares the returned answer to the Teacher's.

**Limitations observed.**
* One unspecialized agent — generation and solving are not separated, so there is
  no independent check on the Teacher.
* Static knowledge base — the TF-IDF corpus only covers the local files.
* Verification bottleneck — the Wolfram API is sensitive to phrasing, often fails
  to parse multi-step queries (HTTP 501), and the text comparison rejects correct
  but differently-formatted answers.

**Results:** The baseline achieved an accuracy of **56%**. Testing was performed on
a test set of **18 randomly selected topics**, where a new math problem was
generated for each topic and then verified through the Wolfram-based pipeline.
## Experiment 2 — Multi-agent + RAG + SymPy

**Architecture (3 agents).**
1. **Dynamic context (Wikipedia) — true RAG.** For each topic the system fetches a
   Wikipedia summary on the fly.
2. **Agent A (Teacher)** generates a *new* problem and its answer from that context.
3. **Agent B (Solver)** receives only the statement, solves it, and translates it
   into a single **SymPy** expression.
4. **Judge** evaluates the SymPy expression in a safe namespace and compares it
   symbolically to the Solver's answer; non-matching items are discarded.

**Results:** The improved multi-agent pipeline achieved an accuracy of **94%** on
the same evaluation setup: **18 randomly selected topics**, with generated
problems verified through the Solver + SymPy checking mechanism.

## Why does it improve on the baseline?**

Experiment 2 improves the baseline by replacing fragile Wolfram-based verification with symbolic verification through SymPy. In the baseline, the system compared the Teacher’s answer with the Wolfram result in a more text-oriented way, so even mathematically correct answers could be rejected because of formatting differences, such as `1/2` vs `0.5`, different root order, spacing, or equivalent algebraic forms. With SymPy, the Solver converts the problem into a symbolic expression and computes the result exactly, so the Judge checks mathematical correctness instead of surface-level similarity.

Another improvement is that the context is no longer limited to a static local corpus. Instead, the system retrieves topic-specific information dynamically from Wikipedia, which makes the pipeline more flexible and scalable because new topics can be processed without manually editing the local files.

Finally, an important improvement is the use of a stronger multi-agent architecture with two independent LLM-based roles. Instead of relying on a single model output, the pipeline now separates generation and solving: the Teacher generates the problem and its answer, while the Solver independently solves the same problem and converts it into a SymPy expression. This creates an additional verification layer, because the Teacher’s answer is no longer trusted directly.

## Setup

Before running the project, configure the required API keys for Gemini and Wolfram Alpha by setting the `GEMINI_API_KEY` and `WOLFRAM_APP_ID` environment variables. Then run the `math_tutor.ipynb` Jupyter Notebook to execute the full pipeline.

Keep these next to the notebook:
* `context/*.txt` — the static corpus for Experiment 1's TF-IDF search
* `safe_sympy_ns.pkl` — the sandboxed SymPy namespace used by the Judge in Experiment 2

## Files

```
math_tutor.ipynb        # main deliverable: both experiments + evaluation
eval_dataset.jsonl      # generated dataset (fields: input, output, type, expected_answer)
context/                # Ukrainian "textbook" corpus for TF-IDF (Experiment 1)
  algebra_context.txt
  calculus_context.txt
  geometry_context.txt
  probability_context.txt
safe_sympy_ns.pkl       # safe namespace for evaluating Solver's SymPy expressions
```

## Dataset format

Each line of `eval_dataset.jsonl` is one example:

```json
{
  "input": "Квадратні рівняння",
  "output": "Розв'яжіть рівняння x^2 - 5*x + 6 = 0 ...\nВідповідь: x in {2, 3}",
  "type": "task",
  "expected_answer": "x in {2, 3}"
}
```
