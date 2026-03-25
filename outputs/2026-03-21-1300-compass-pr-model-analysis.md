> **Question:** If agent-researcher had to operate via PRs, how would the current interactive model via Claude CLI change?

## Compass via PRs — What Would Actually Change

### Current model

```
CEO types question
  → Compass reasons, writes output file, commits, pushes to main
  → CEO reads summary in terminal or opens file
  → CEO asks follow-up
  → repeat
```

The whole thing is synchronous and conversational. Output files are artifacts of a conversation the CEO is already in.

### PR model

```
CEO types question
  → Compass creates branch, writes output file, opens PR
  → CEO must context-switch to GitHub
  → CEO reviews PR, merges
  → Output is now "official"
  → CEO returns to terminal, asks follow-up
  → repeat
```

Every answer becomes a GitHub round-trip. The conversational flow breaks.

---

### Why PRs don't make sense for Compass

The PR model exists to put the CEO in the loop before a work product takes effect. But with Compass, the CEO IS the loop. They are present in the conversation. They asked the question. The output file is the answer to their question — they have already "reviewed" it by participating in the session that produced it.

Merging a PR for a Compass output would be the CEO formally approving... their own advisor's answer to their own question. The gate adds friction with no governance value.

Compare with operational agents:
- Axle writes code that gets deployed — CEO needs to review before it runs
- Compass writes analysis that the CEO reads — CEO is already there

The risk profile is completely different. Compass produces no side effects. Its outputs only matter if the CEO acts on them.

---

### What branch protection on agent-researcher would actually do

It would not improve governance. It would destroy the tool's utility.

- Research iterations (the back-and-forth of this session) would each require a PR
- Multi-step analyses where each output builds on the last become impossible without merge delays between steps
- The CEO would spend more time merging PRs than thinking about the content

The current unprotected state of agent-researcher is **correct by design**, not an oversight.

---

### The right distinction

| Agent | Produces | CEO review needed before... | Model |
|-------|----------|----------------------------|-------|
| Axle, Pixel, etc. | Code, configs | ...it runs or deploys | PR required |
| Compass | Analysis, research | ...nothing — CEO is the audience | Direct commit |

Compass is not autonomous — it only acts when the CEO is in the session. Every commit in this repo is a direct response to something the CEO asked. The interactive CLI session IS the review mechanism.

---

### One scenario where PRs would add value

If Compass were ever used to write to operational repos — identity files, config changes intended for other agents — then a PR gate would matter. But that's not what Compass does. Compass writes to its own research files and proposes changes for the CEO to apply elsewhere. The boundary is clear.

**Conclusion:** Keep agent-researcher unprotected. The current model is correct. Adding PRs would not improve safety; it would add bureaucratic overhead to a tool whose entire value is immediate, iterative thinking.
