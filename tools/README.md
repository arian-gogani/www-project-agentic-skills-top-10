# Tools

## `build_pptx.py` — Top 10 slide deck generator

Builds a professional **12-slide** PowerPoint deck for the OWASP Agentic Skills
Top 10:

1. **Overview** — all ten risks on one slide, colour-coded by severity.
2. **AST01–AST10** — one slide per risk, with an *issues → mitigations* diagram.
3. **Summary** — severity distribution and key takeaways.

The OWASP logo (`assets/images/owasp-logo.png`) appears on every slide.

Content is parsed live from the `astNN.md` source files (severity, description,
attack scenarios, preventive mitigations), so the deck stays in sync with the
documentation.

### Build locally

```bash
pip install "python-pptx==1.0.2" "Pillow>=10,<14"
python tools/build_pptx.py --out dist/OWASP-Agentic-Skills-Top10.pptx
```

### Build in CI

The [`Build Top 10 PPTX`](../.github/workflows/build-pptx.yml) workflow runs on:

- any push to `main` that touches `ast*.md`, the generator, the logo, or the
  workflow itself;
- a manual **Run workflow** (`workflow_dispatch`);
- a published **Release** (the deck is attached to the release assets).

Every run uploads the deck as the **`OWASP-Agentic-Skills-Top10-pptx`** artifact,
downloadable from the workflow run's *Artifacts* section.
