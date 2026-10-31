# examples-are-not-rules

A skill for coding agents (Claude Code, Codex, or anything that reads markdown) that stops them from turning the user's **example** into the rule's **subject**, the most common way "don't hardcode" gets ignored.

You say "click a person and follow them." The spec says "the target is the person." The test follows a walking person. The code reads the person's body. Every step is faithful, and the result only works for people.

A reminder like "don't hardcode" can't be checked. This skill replaces it with things that can:

1. **Name the subject.** Every rule states what class it applies to and which capability it reads, not just the example noun.
2. **A second example.** Every rule gets a test with a different kind of thing that has the same capability, running through the same code path. The spec writer picks it, never the implementer: whoever writes the test must not be whoever takes it.
3. **Generalize stated rules, YAGNI the rest.** That line resolves the tension with "don't over-engineer."
4. **Review for kind checks.** Does this branch ask what the thing *is*, or what it *can do*?
5. **Handoffs carry the subject, the second example, and a failure condition.**

## Install

```bash
git clone https://github.com/Sttrevens/examples-are-not-rules ~/.claude/skills/examples-are-not-rules
```

For other agents, point your `AGENTS.md` at `SKILL.md`, or paste it in.

## License

MIT
