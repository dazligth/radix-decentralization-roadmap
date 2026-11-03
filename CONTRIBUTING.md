# How to contribute

English · [Español](CONTRIBUTING.es.md)

This repository is open source and intended to be improved by the Radix community. Anyone can propose changes through issues and pull requests (PRs).

## What you can contribute

| Type of contribution | Examples | Where |
|---|---|---|
| **Corrections** | A wrong figure, an outdated `file:line` reference, a broken link, a typo, an inaccurate translation | A PR on the affected document in `docs/en/` and `docs/es/` |
| **Data and measurements** | The real per-column-family ledger size on a mainnet node, measured RAM use, sync time, the index size of a prototype | A PR that updates the figure and cites the measurement (method, date, node version) |
| **New strategies** | An alternative to the light indexer, another way to migrate to Hyperscale, another incentive model | First an issue using the "New strategy" template; then a note in `community/` or a PR to the relevant chapter |
| **Community ideas and notes** | Opinions, proposals, operator experience, reasoned questions | A file in `community/` (see [community/README.md](community/README.md)) |
| **Implementation** | Links to prototypes, repositories or PRs in `babylon-node`, Sargon, the RDT, etc. that advance any of the proposals | A PR that adds the link in the relevant section and in the roadmap |
| **Diagrams** | Improvements or new diagrams in Mermaid | A PR on the document; check that GitHub renders them |

## How to open a pull request

1. Fork the repository and create a branch with a descriptive name (for example, `fix/ledger-size`).
2. Edit the files:
   - **Report content:** `docs/en/` and `docs/es/`. The summaries and the conclusion are in `README.md` and `README.es.md`.
   - **Community notes:** `community/`.
3. Keep **both languages** in sync. If you can only write in one, say so in the PR with the "needs translation" label and someone else will complete it.
4. Open the PR using the template and explain what changes and why.
5. Content changes are reviewed before they are merged. You may be asked for sources or adjustments.

## Quality rules

The value of this report depends on everything in it being verifiable. Therefore:

- **Nothing is made up.** Every figure needs a source: `file:line` and code version, a URL, or a described measurement (method, date, version).
- **Estimates are marked as such** ("estimate", "est.") and their basis is explained.
- **Assessments use the same format** at the start of each section:

  ```markdown
  > [!IMPORTANT]
  > - **Is it possible?** Yes / Yes, with conditions / No / Only partly.
  > - **Difficulty:** Low / Low-medium / Medium / Medium-high / High / Very high.
  > - **Effort and timeline:** ... (estimate).
  ```

- **Register:** clear and neutral, understandable by non-technical readers, without unnecessary jargon or a colloquial tone. Impersonal wording.
- **No emojis** in the text.
- **Diagrams in Mermaid**, so they can be edited as text.
- If a change affects other sections (for example, an effort figure that also appears in the conclusion), update all of them.

## Disagreements and debate

If you disagree with a conclusion or a strategy, open an issue explaining why and bring data. Reasoned alternative positions are welcome. If there is no consensus, the position can be recorded as a community note in `community/` and linked from the relevant section.

## Licence

Contributions are published under the repository's licence (see `LICENSE`).
