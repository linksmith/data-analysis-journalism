# data-analysis-journalism

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

An AI agent skill for journalistic exploratory data analysis. Find story leads in Dutch open data through a structured 8-step EDA sequence. Compatible with Claude Code, Open Code, Kilo Code, Cursor, Windsurf, Cline, Aider, and 40+ other AI coding tools.

## Overview

This skill transforms an AI agent into a data journalism editor that surfaces newsworthy findings from a clean pandas DataFrame. It follows a structured 8-step sequence:

1. **Shape & structure** — understand what you have before diving in
2. **Distributions** — flag columns where mean and median diverge (sign of outliers or skew)
3. **Rankings** — top 5 and bottom 5 with national average context
4. **Year-over-year trends** — absolute and percentage change, inflection point detection
5. **Group comparisons** — absolute values and per-capita rates by category or region
6. **IQR outlier detection** — framed as story leads, not just statistics
7. **Correlations** — with explicit ecological fallacy warnings
8. **Missing value patterns** — treated as potential stories, not just data quality issues

Every finding is contextualised: **"X heeft 3× het landelijk gemiddelde"**, not just "de waarde is 45,2". Output follows journalistic structure: Hoofdbevinding (the lede), Kerngetallen, Uitschieters, Kanttekeningen, Vervolgvragen. Responds in Dutch if the user writes in Dutch.

**This skill is designed to work AFTER data has been fetched** (e.g. by [cbs-statline-skill](https://github.com/linksmith/cbs-statline-skill)) **and cleaned** (e.g. by [data-cleaning-dutch](https://github.com/linksmith/data-cleaning-dutch)). It does not retrieve or clean data — it finds stories in already-prepared DataFrames.

## Installation

### Dependencies

```bash
pip install pandas
```

### Quick Install (Any Agent)

If you use the [Vercel Skills CLI](https://github.com/vercel-labs/skills), this works across 40+ agents:

```bash
npx skills add linksmith/data-analysis-journalism
```

See below for tool-specific instructions.

### Cursor

**Option 1: Clone to Cursor rules directory**

```bash
git clone https://github.com/linksmith/data-analysis-journalism.git ~/.cursor/rules/data-analysis-journalism
```

**Option 2: Add to project `.cursorrules`**

```bash
curl -L https://raw.githubusercontent.com/linksmith/data-analysis-journalism/main/SKILL.md -o .cursorrules
```

**Option 3: Project-level installation**

```bash
git clone https://github.com/linksmith/data-analysis-journalism.git .cursor/data-analysis-journalism
```

Then reference in `.cursorrules`:
```
Use the data-analysis-journalism skill in .cursor/data-analysis-journalism/ for journalistic EDA.
```

### Windsurf (Codeium)

**Option 1: Global rules**

```bash
git clone https://github.com/linksmith/data-analysis-journalism.git ~/.windsurf/rules/data-analysis-journalism
```

**Option 2: Project-level**

Create `.windsurf/rules/data-analysis-journalism.md` in your project:
```bash
curl -L https://raw.githubusercontent.com/linksmith/data-analysis-journalism/main/SKILL.md -o .windsurf/rules/data-analysis-journalism.md
```

### Claude Code / Open Code / Kilo Code

All three tools support the same plugin format:

**Option 1: Install as a plugin** (recommended, no npm/node required)

```bash
claude plugin install --from https://github.com/linksmith/data-analysis-journalism
```

Replace `claude` with `open` or `kilo` depending on your tool.

**Option 2: Add as a project skill**

```bash
mkdir -p .claude/skills
git clone https://github.com/linksmith/data-analysis-journalism.git .claude/skills/data-analysis-journalism
```

**Option 3: Add as a slash command**

```bash
mkdir -p .claude/commands
curl -L https://raw.githubusercontent.com/linksmith/data-analysis-journalism/main/SKILL.md \
  -o .claude/commands/data-analysis-journalism.md
```

### Cline (VS Code Extension)

**Option 1: Add to .clinerules**

```bash
curl -L https://raw.githubusercontent.com/linksmith/data-analysis-journalism/main/SKILL.md -o .clinerules
```

**Option 2: Workspace settings**

1. Clone the skill to your workspace:
```bash
git clone https://github.com/linksmith/data-analysis-journalism.git .cline/data-analysis-journalism
```

2. In VS Code settings, add to `cline.customInstructions`:
```
Use data-analysis-journalism skill for journalistic EDA. See .cline/data-analysis-journalism/SKILL.md
```

### Roo Code (VS Code Extension)

**Option 1: Add to .roorules**

```bash
curl -L https://raw.githubusercontent.com/linksmith/data-analysis-journalism/main/SKILL.md -o .roorules
```

**Option 2: Custom instructions**

In VS Code, open Roo Code settings and add to Custom Instructions:
```
For journalistic EDA on Dutch open data, reference the skill at:
https://github.com/linksmith/data-analysis-journalism
```

### Aider

**Option 1: Add as read-only context**

```bash
git clone https://github.com/linksmith/data-analysis-journalism.git ~/skills/data-analysis-journalism

aider --read ~/skills/data-analysis-journalism/SKILL.md \
      --read ~/skills/data-analysis-journalism/references/eda-checklist.md
```

**Option 2: Add to .aider.conf.yml**

```yaml
read:
  - ~/skills/data-analysis-journalism/SKILL.md
  - ~/skills/data-analysis-journalism/references/eda-checklist.md
```

### OpenHands

**Option 1: Add to workspace**

```bash
git clone https://github.com/linksmith/data-analysis-journalism.git .openhands/data-analysis-journalism
```

**Option 2: Custom instructions**

Add to `.openhands/instructions.md`:
```
For journalistic EDA on Dutch open data:
1. Read .openhands/data-analysis-journalism/SKILL.md for the 8-step workflow
2. Reference references/eda-checklist.md for statistical tests and ecological fallacy guidance
```

### Goose (Block)

**Option 1: Add to Goose extensions**

```bash
git clone https://github.com/linksmith/data-analysis-journalism.git ~/.goose/extensions/data-analysis-journalism
```

**Option 2: Add instruction file**

```bash
curl -L https://raw.githubusercontent.com/linksmith/data-analysis-journalism/main/SKILL.md -o ~/.goose/instructions/data-analysis-journalism.md
```

### GitHub Copilot

**Option 1: Add to .github/copilot-instructions.md**

```bash
mkdir -p .github
curl -L https://raw.githubusercontent.com/linksmith/data-analysis-journalism/main/SKILL.md -o .github/copilot-instructions.md
```

**Option 2: Reference in VS Code settings**

In `.vscode/settings.json`:
```json
{
  "github.copilot.chat.codeGeneration.instructions": [
    {
      "file": ".github/copilot-instructions.md"
    }
  ]
}
```

### Generic AI Assistants

For any AI assistant that supports custom instructions or context files:

1. Download the SKILL.md file:
```bash
curl -L https://raw.githubusercontent.com/linksmith/data-analysis-journalism/main/SKILL.md -o data-analysis-journalism-instructions.md
```

2. Paste the contents into your AI assistant's custom instructions or system prompt.

## Usage

The skill is designed for journalists and data reporters working with Dutch open data. Once activated, prompt your agent naturally in Dutch or English.

### Example Prompts

```
Analyseer deze dataset voor verhaalleads
```

```
Rangschik de gemeenten en vergelijk met het landelijk gemiddelde
```

```
Vind de uitschieters in deze kolom
```

```
Is er een verband tussen inkomen en zonnepanelen?
```

### Recommended Workflow

This skill fits in the middle of a three-skill pipeline:

1. **Fetch data** — use [cbs-statline-skill](https://github.com/linksmith/cbs-statline-skill) to retrieve CBS open data
2. **Clean data** — use [data-cleaning-dutch](https://github.com/linksmith/data-cleaning-dutch) to normalise Dutch column names, region codes, and types
3. **Find stories** — use this skill to run the 8-step EDA and surface newsworthy findings

### Output Format

Every analysis closes with a structured journalistic summary:

```
## Bevindingen

**Hoofdbevinding (de lede):**
[One sharp sentence with the most newsworthy finding and its context]

**Kerngetallen:**
1. [Fact + context, e.g. "Utrecht: 34%, landelijk gemiddelde: 18%"]
2. [Fact + context]
3. [Fact + context]

**Uitschieters en verrassingen:**
- [Outlier + possible explanation]
- [Surprising correlation or pattern]

**Kanttekeningen:**
- [Data limitations, measurement period, methodology notes]
- [What the data CANNOT tell you]

**Vervolgvragen:**
1. [Question the data raises but cannot answer]
2. [Question for further investigation]
3. [Question for an expert or additional source]
```

## Skill Structure

```
data-analysis-journalism/
├── SKILL.md                        # Main skill definition (8-step EDA workflow)
├── references/
│   └── eda-checklist.md            # Statistical tests, ecological fallacy explainer, denominator guide
└── evals.json                      # Evaluation prompts for skill testing
```

**Note:** This skill is pure prompt — no Python helper module is included. All analysis code is generated on-the-fly by the agent following the SKILL.md workflow.

## Features

### 8-Step EDA Sequence

The skill enforces a consistent, journalism-first analysis order:

- **Step 1** — Shape and column types before any analysis
- **Step 2** — Distribution flags: warns when mean/median diverge >20%
- **Step 3** — Rankings always include national average deviation
- **Step 4** — Year-over-year: both percentage and absolute change, plus inflection detection
- **Step 5** — Group summaries include `afwijking_pct` from the national average
- **Step 6** — IQR outliers framed as story leads with deviation from national average
- **Step 7** — Correlations always include an ecological fallacy warning
- **Step 8** — Missing data clustered by region/time and posed as journalistic questions

### Story Framing Rules

- Every finding is contextualised: "X heeft 3× het landelijk gemiddelde", never just raw numbers
- Counter-intuitive findings are flagged: "Je zou verwachten dat... maar de data laat zien..."
- Always 2–3 follow-up questions per analysis
- Per-capita / per-household rates computed when group sizes differ

### Visualization Handoff

This skill surfaces the story. For charts, the agent redirects to [data-viz-journalism](https://github.com/linksmith/data-viz-journalism). For choropleth maps, it redirects to [dutch-choropleth-maps](https://github.com/linksmith/dutch-choropleth-maps).

## License

MIT License — see [LICENSE](LICENSE) for details.

## Contributing

Contributions welcome! Please feel free to submit issues or pull requests.

- **Issue Tracker:** https://github.com/linksmith/data-analysis-journalism/issues
- **Pull Requests:** https://github.com/linksmith/data-analysis-journalism/pulls

## Resources

- [CBS StatLine open data](https://opendata.cbs.nl/)
- [cbs-statline-skill](https://github.com/linksmith/cbs-statline-skill) — fetch CBS data
- [data-cleaning-dutch](https://github.com/linksmith/data-cleaning-dutch) — clean Dutch datasets
