# Student Quick Reference: Document-Driven AI Development

**This guide teaches you how to use AI as a thinking tool, not just a code generator.**

The workflow is the same whether you submit work via GitHub, folder snapshots, or in-person review. Check with your instructor which method your class uses.

## The Workflow (3 Steps)

### Step 1: Clarify (AI helps write docs, not code)
Copy this prompt into your AI chat:
```
I want to build a simulated city based on this outline. 
Please help me clarify it before I code.

[Paste your Project from README]

Please:
1. Rewrite the 4 components using clear technical language
2. Identify MQTT topics each agent will publish/subscribe to
3. List configuration parameters (MQTT broker, locations, thresholds)
4. Identify which notebooks to create (one per agent type)
5. Identify what can be modeled as classes (vs simple functions)
6. Suggest what belongs in library code (src/simulated_city/) vs notebooks
7. Point out any ambiguities

Do NOT write code.
```

### Step 2: Plan (AI proposes phases, you approve)
```
Based on the design we just clarified:
[Paste clarified design]

Please propose phased implementation:
- Phase 1: Single basic agent
- Phase 2: Add config file
- Phase 3: Add MQTT publishing
- Phase 4: Add second agent with MQTT subscription
- Phase 5: Add dashboard

For each phase, specify:
1. **New files:** Which notebooks? Which library modules (src/simulated_city/)?
2. **Classes vs functions:** What can be modeled as a class? What's just a helper function?
3. **Notebook vs library:** What logic goes in notebooks (simulation loop, MQTT subscribe)? What goes in library (reusable utilities, data models)?
4. **Tests/Verification:** What commands verify this phase works?
5. **Investigation:** What should you understand before the next phase?

Do NOT write code.
```

### Step 3: Implement (ONE phase at a time)
```
Implement ONLY Phase 1:
[Paste Phase 1]

Rules (from .github/copilot-instructions.md):
- Use anymap-ts (NOT folium)
- Each notebook = ONE agent (NOT monolithic)
- Load config via simulated_city.config.load_config()
- Use mqtt.publish_json_checked() for publishing
- Add all dependencies to pyproject.toml

Only Phase 1. Do NOT jump to Phase 2.
```

After code: Test it, understand it, then ask for Phase 2.

---

## Understanding Architecture: Notebook vs Library Code

One of the most important decisions in Phase 1-2 is **where code should live**:

### Notebooks (notebooks/*.ipynb)
**Use for:**
- Simulation loops (while True: ... sleep())
- MQTT subscriptions and event handlers  
- Agent-specific behavior (one notebook per agent)
- Dashboard/visualization code

**Why:** Notebooks are for running simulations and visualizing results.

### Library Code (src/simulated_city/*.py)
**Use for:**
- Reusable data models (dataclasses: Vehicle, Sensor, etc.)
- Utility functions used by multiple notebooks
- Complex calculations or algorithms
- Configuration schemas
- Anything that needs automated tests (pytest)

**Why:** Library code is for reusable, testable components.

### Classes vs Functions
- **Classes:** For agents with state, data models (Vehicle, Sensor), services
- **Functions:** For simple helpers, transformations, one-off calculations

### Example Architecture

Good design (distributed):
```
notebooks/
  agent_transport.ipynb    # Transport agent: subscribes to traffic, publishes vehicle positions
  agent_environment.ipynb  # Environment agent: simulates pollution based on vehicle data
  dashboard.ipynb          # Visualizes everything on anymap-ts

src/simulated_city/
  models.py               # Vehicle, Sensor dataclasses
  simulation.py           # Helper functions for movement, calculations
```

Bad design (monolithic):
```
notebooks/
  everything.ipynb        # ❌ All agents + dashboard in one file
```

During planning (Step 2), the AI should identify which notebooks and which library modules to create. This makes your code:
- Testable (library code can be tested with pytest)
- Reusable (multiple notebooks can import the same classes)
- Maintainable (each notebook has a clear purpose)

---

## Common AI Mistakes (And How to Fix Them)

### ❌ AI doesn't identify notebooks and library structure in planning
You: "Before coding, tell me: which notebooks to create? Which classes go in src/simulated_city/? Which code is notebook-specific vs reusable?"

### ❌ AI tries to code without clarifying design
You: "No code yet. Use Phase 1 prompt from README.md to clarify the design first."

### ❌ AI proposes all 5 phases at once
You: "We'll implement one at a time. Give me only Phase 1 implementation."

### ❌ AI uses folium instead of anymap-ts
You: "No, use anymap-ts. Check .github/copilot-instructions.md for the rules."

### ❌ AI creates one big notebook instead of agent notebooks
You: "Split into separate notebooks. Each agent publishes/subscribes via MQTT. See docs/exercises.md."

### ❌ AI uses !pip install in notebook
You: "Don't install in notebooks. Add to pyproject.toml and run: pip install -e '.[notebooks]'"

---

## Validation Commands

```bash
# Check dependencies are correct
python scripts/verify_setup.py

# Check code structure (no monolithic notebooks, no folium)
python scripts/validate_structure.py

# Run tests
python -m pytest

# Open notebooks and test manually
python -m jupyterlab
```

---

## If Your Model Switches (Auto Mode)

The workflow is **model-agnostic**. It works with any AI because:
1. You write artifacts (README clarification, approved design)
2. Each prompt is self-contained with full rules embedded
3. You validate output locally before moving forward

If a new model doesn't follow rules, respond with the "Common AI Mistakes" section above.

---

## Submitting Your Work (Using GitHub Desktop & VS Code)

### What is a Pull Request?

A **Pull Request (PR)** is how you submit code for review on GitHub:

1. You work in VS Code (in a branch: separate copy of the project)
2. You commit and sync using VS Code's Source Control panel (or GitHub Desktop)
3. You create a PR on GitHub.com (ask instructor: "Ready for review?")
4. Instructor reviews your code in the PR
5. If approved, your changes merge into the main project

**Why?** This enforces phase-gating. You can't start Phase 2 until Phase 1 is approved.

### The Branch Flow

Here's how phases progress:

```
main branch
  │
  ├── Create phase-1 branch
  │   ├── Code Phase 1
  │   ├── Commit + Sync
  │   └── Create PR → Instructor reviews → Merge into main ✅
  │
  ├── Pull updated main
  │
  ├── Create phase-2 branch
  │   ├── Code Phase 2
  │   ├── Commit + Sync
  │   └── Create PR → Instructor reviews → Merge into main ✅
  │
  └── Repeat for Phase 3, 4, 5...
```

Each phase builds on the previous approved phase.

### Checklist Before You Commit

Before you commit your Phase 1 work:

- [ ] README filled in with 4-component template (your project idea)
- [ ] AI clarified design (saved in your PR description or a doc)
- [ ] You approved the implementation plan
- [ ] **Only ONE phase implemented** ← Most important!
- [ ] Tests passing: `python scripts/verify_setup.py && python -m pytest`
- [ ] Structure valid: `python scripts/validate_structure.py`
- [ ] PR description will say which phase(s) included (e.g., "Phase 1: Basic agent")

### Workflow: VS Code (with GitHub Desktop as alternative)

All Git operations can be done in VS Code. GitHub Desktop is an alternative if you prefer a visual Git interface.

#### Step 1: Create a branch (in VS Code)
```
1. Open VS Code
2. Look at the bottom left corner — you'll see the current branch name (e.g., "main")
3. Click on it
4. Select "+ Create new branch" from the dropdown
5. Type the name: phase-1
6. Press Enter
```

**Alternatively:** You can create the branch in GitHub Desktop:
```
1. Open GitHub Desktop
2. Click "Current Branch" at the top
3. Click "New Branch"
4. Name it: phase-1
5. Click "Create Branch"
6. Then switch back to VS Code
```

#### Step 2: Make your changes (in VS Code)
```
1. Open VS Code
2. Edit/create your notebooks, code, docs
3. Run validation commands in the terminal:
   python scripts/verify_setup.py
   python scripts/validate_structure.py
   python -m pytest
```

#### Step 3: Commit (in VS Code)
```
1. In VS Code, click the Source Control icon (Git icon on the left sidebar)
2. You'll see your changed files listed
3. Click the "+" next to each file to stage it (or click "+" at the top to stage all)
4. Type a commit message in the box: "Phase 1: Basic agent with MQTT"
5. Click the "✓ Commit" button
```

#### Step 4: Sync (in VS Code)
```
1. After committing, click the "Sync Changes" button that appears
   (Or click the ↻ icon at the bottom left)
2. This uploads your changes to GitHub.com
3. First time: VS Code may ask "Publish Branch?" → Click "OK"
```

#### Step 5: Create a Pull Request (on GitHub.com)
```
1. Go to GitHub.com and open your repository
2. You should see a notification: "Compare & pull request"
3. Click it
4. Fill in the PR description (see template below)
5. Click "Create pull request"
```

### What to Put in Your PR Description

When you create the PR, use this template:

```
## What Phase Is in This PR?

Phase 1: Basic agent with MQTT

## Design (from clarification phase)

[Paste the design AI clarified for you]

## What I Investigated

[Briefly: what did you learn/test from this phase?]

## Verification

Ran these commands successfully:
- [x] python scripts/verify_setup.py
- [x] python scripts/validate_structure.py
- [x] python -m pytest
- [x] Manually tested notebooks (they run without errors)
```

### After Your Instructor Reviews

Your instructor will:
1. Look at your Phase 1 code
2. Run validation scripts
3. Either approve or ask for changes

**If approved:**
1. Instructor clicks "Merge pull request" on GitHub.com
2. Your Phase 1 code is now in the main branch ✅
3. **You** can now start Phase 2:
   ```
   1. In VS Code, switch to main branch (click branch name bottom left)
   2. Click the ↻ sync icon to pull the updated main
   3. Create a new branch: phase-2 (click branch name → "+ Create new branch")
   4. Start implementing Phase 2
   ```

**If changes needed:**
1. Fix them in VS Code (stay on the phase-1 branch)
2. Commit again
3. Sync
4. The PR updates automatically
5. Instructor re-reviews

---

### Quick Reference: VS Code Source Control

| Action | How to Do It |
|--------|--------------|
| **See changes** | Click Source Control icon (left sidebar) |
| **Stage files** | Click "+" next to file (or "+" at top for all) |
| **Commit** | Type message, click "✓ Commit" button |
| **Sync** | Click "Sync Changes" or ↻ icon (bottom left) |
| **Switch branch** | Click branch name (bottom left) → select branch |
| **Create branch** | Click branch name → "+ Create new branch" |

**Alternatively:** You can use GitHub Desktop for all Git operations if you prefer a visual interface. The workflow is the same (create branch → commit → sync → PR).

---

### Troubleshooting

**"I don't see the 'Sync Changes' button"**  
Look at the bottom left of VS Code for the ↻ sync icon. Click it to sync.

**"VS Code asks me to publish the branch"**  
Click "OK" or "Publish Branch". This is normal the first time you sync a new branch.

**"My changes don't show in Source Control"**  
Make sure you saved your files (Ctrl+S or Cmd+S). Then click the Source Control icon to refresh.

**"I'm on the wrong branch"**  
Click the branch name at the bottom left → select the branch you want (e.g., `phase-1`).

**"How do I see what I changed?"**  
Click Source Control icon. Each file shows what changed (red = removed, green = added). Click a file to see the diff.

**"I prefer a visual Git tool"**  
Use GitHub Desktop instead. The workflow is the same, just with buttons instead of VS Code's Source Control panel.
