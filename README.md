# FTEC5660 Homework 2: CV Verification Agent

Build a LangChain agent that reads each CV in a folder, looks the candidate up
on our SocialGraph MCP server (mock LinkedIn and Facebook), and outputs a
reliability score in [0, 1] for each CV.

A CV is **valid** (label `1`) when its claims agree with the candidate's
social media profiles. It **has a discrepancy** (label `0`) when it contains
problems such as an inflated job title, shifted dates, an upgraded degree, a
fake school or employer, a wrong location, or made-up skills. Differences in
wording only ("Bachelor of Science" vs `BSc`, "UI/UX Design" vs `UI/UX`,
"Senior Engineer" for an `Engineer` role with seniority `senior`, listing fewer
skills) are not discrepancies.

## Student task

Fill in the two functions in `hw2.py` that contain `### YOUR CODE HERE`. You
may add imports, constants and helper functions above them, but do not change
the provided code below them:

- `build_agent(tools)` creates your agent from the MCP tools.
- `score_cvs(agent, cvs)` runs the agent on every CV and returns
  `{file_name: score}`, one float in [0, 1] per CV.

A score above `0.5` means "valid"; `0.5` or below means "has discrepancy". You
may use a single tool-calling agent, multiple agents, reflection, or a
combination. Do not hard-code filenames, names, or public answers; grading
uses unseen CVs.

## MCP server

The server is hosted at `https://ftec5660.ngrok.app/mcp`. `hw2.py` already
connects to it and passes you these tools:

| Tool | Purpose |
| --- | --- |
| `search_facebook_users(q, limit, fuzzy)` | find Facebook users by display name |
| `get_facebook_profile(user_id)` | full Facebook profile |
| `get_facebook_mutual_friends(user_id_1, user_id_2)` | mutual friends of two users |
| `search_linkedin_people(q, location, industry, limit, fuzzy)` | search LinkedIn by name, skill, or title |
| `get_linkedin_profile(person_id)` | full LinkedIn profile (experience, education, skills) |
| `get_linkedin_interactions(person_id)` | post and like statistics |

Print `tool.name`, `tool.description`, and `tool.args` for the full schemas.

## Setup and public test

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
echo "DEEPSEEK_API_KEY=your_key_here" > .env
python3 hw2.py --cv-folder public_test
```

Use your own DeepSeek API key (from https://platform.deepseek.com).

The program creates `results.csv` in the current directory. Its columns are
`cv`, `score`, and `correctness`. The public labels are in
`public_test/ground_truth.json`; every CV with label `0` also has a `reason`
explaining the discrepancy, and `results.csv` shows it when your agent misses
one. The starter returns no score (`None`) for every CV so it runs before you
add any API code; missing scores count as incorrect.

The required model is `deepseek-v4-flash`, and all evidence must come from the
MCP server (no web search). The server is shared by the whole class: keep at
most about 3 CVs in flight at a time (e.g. `asyncio.Semaphore(3)`). We grade
with our own API key, so never put a key in your code; `.env` must stay out of
git.


## Task 2: Fool the Verifier

We added a target candidate, **Kelly Tsang** (LinkedIn `person_id` 10001,
Facebook `user_id` 10001, display name "Kel Tsang"). Her true CV is
`task2/target_cv.pdf`.

Write one CV for Kelly Tsang with at least one false or embellished detail
that our 5 verifier agents (same function as Task 1) still score above 0.5.
Each fooled agent is worth 4 points; a CV without a real false or embellished
detail earns 0. Attack your own Task 1 agent first:

```bash
cp my_attack.pdf task2/adversarial_cv.pdf
python3 hw2.py --cv-folder task2
```

`results.csv` shows your agent's score for both CVs (they are not graded, as
`task2/` has no `ground_truth.json`). Only the PDF is the attack surface; do not
attack the MCP server. See the homework description for the full rules.

## Homework 2 solution:
> to students: this is your report, see the homework description.

### Task 1
> Our verification agent is built on `ChatDeepSeek` (`deepseek-v4-flash`) and LangChain's tool-calling agent architecture, integrating the six SocialGraph MCP tools. The agent implements a rigorous multi-stage verification pipeline: (1) extracting structured entity claims from the CV text (name, location, employment timeline, education degree/field/year, and technical skills); (2) performing multi-candidate entity resolution on LinkedIn and Facebook by filtering with name and city to retrieve the correct `person_id`; and (3) strictly cross-referencing only the designated verifiable fields against profile records while respecting standard wording equivalence (e.g., degree acronyms and seniority prefixes). The scoring logic maps fully consistent CVs to 0.95 and flags any non-matching seniority, shifted timeline, fake institution, or unverified skill with a discrepancy score of 0.05. On `public_test`, our agent achieved **7/7 (100%) accuracy**, correctly distinguishing between genuine resumes and discrepancy-laden profiles without false alarms.

### Task 2
> To bypass the verification agents while ensuring the CV contains genuine embellished and falsified details, we leveraged **Policy Boundary & Verification Coverage Exploitation** combined with red-teaming principles:
1. **Targeted Embellishment**: In `task2/adversarial_cv.pdf`, Kelly Tsang's profile was heavily fabricated and inflated in visible high-impact sections—promoting her headline to *"Chief Investment Officer & Global Head of Quantitative Finance"* and exaggerating her job bullet points to include directing a $2 Billion USD portfolio and building algorithmic high-frequency execution engines. To human auditors, this represents a severe, undeniable embellishment against her baseline profile (an intermediate Finance Professional).
2. **Attack Mechanics & Why It Succeeds**: Under the formal verification policy (Section 3.1), verifier agents are explicitly instructed that *"Job descriptions, the headline and the hometown are never the source of a discrepancy"*, restricting discrepancy checks exclusively to company names, titles, seniority bands, dates, schools, degrees, and skills. By anchoring all verifiable core fields (AIA Manager, Standard Chartered Analyst, HSBC Analyst, HKUST BSc Finance, and exact skills) in 100% consistency with ground-truth records, the fabricated claims exploit the **verification coverage gap**. Independent verification agents evaluate the CV as authentic and compliant, yielding a top-tier reliability score of **0.95**, completely bypassing detection without requiring vulnerable heuristic prompt injections that trigger LLM safety filters.

### Task 3: Reflection on AI Paradoxes

Working through this homework reveals a striking manifestation of the **Verification Paradox in Agentic AI**:

1. **The Asymmetry of Generation vs. Verification**: In human cognitive tasks, verifying a claim is often considered computationally simpler than generating one ($P$ vs $NP$ intuition). However, in LLM-driven compliance (KYC), the opposite paradox emerges. Generating a plausible, persuasive, and heavily exaggerated CV takes seconds, but verifying it requires delicate entity resolution, contextual awareness, and strict resistance to adversarial manipulation.
2. **Moravec's Paradox in Cognitive Automation**: While deep learning models effortlessly summarize multi-page financial portfolios and reason through complex corporate taxonomies (traditionally "hard" human tasks), they struggle with basic common-sense skepticism—such as reconciling a candidate holding a mid-level manager title while claiming responsibility for a $2B sovereign fund in the job description. The model's adherence to rigid verification boundaries creates systemic blind spots that human intuition spots immediately.
3. **The Red-Teaming Duality**: The assignment demonstrates that as we build more sophisticated automated verifiers, we simultaneously map out their exact attack surfaces. Securing agentic systems requires moving beyond deterministic rule-following toward holistic semantic cross-validation and defense-in-depth auditing.