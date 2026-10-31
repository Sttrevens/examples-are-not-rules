---
name: examples-are-not-rules
description: Stops an agent from turning the user's example into a rule's subject, which leads to code that hardcodes by kind (this is about special-casing, not secrets in source). Use when turning user requests into specs, acceptance criteria, or task contracts; when implementing a rule the user explained with an example ("click a person and follow them"); when handing implementation to another agent; and when reviewing code or diffs for kind/type/role/species/action-name checks that should be capability checks. Always use it when the user says "don't hardcode", "not just for X", "this should work for other things later", "simulate everything", "make it generic", or complains that a feature only works for the one case they mentioned.
---

# Examples are not rules

When the user says "click a person and my character should follow them", "person" is the rule's first example, not its subject. The subject is whatever has the properties the rule actually reads: anything that has a position, whose position can change, and that can be seen.

Once an example becomes the subject, every later step amplifies it faithfully. The spec says "person", the contract says "follow the person", the tests only cover people, and the code reads a person's body. Each step is reasonable on its own; together they produce a feature that only works for people. A line like "don't hardcode" doesn't prevent this, because nobody can check whether it was followed. This skill replaces it with things you can check: the rule's subject, a second example, the line between the two kinds of concreteness, and a review pass for kind-based branches.

## 1. Writing a rule: subject first

Write these lines for every rule before any details. If the rule has no example noun in it (say, "log every playtest session"), one line, "Subject: no example", is enough.

```
Rule: <one sentence>
Subject: <a class of things>; decided by: <which capability or property it needs>
First example: <the one the user gave>
Second example: <a different kind with the same capabilities>
Doesn't apply to: <things lacking a capability, stated as the capability> (optional)
```

Finding the subject:
- The user's words often already contain the class, right before "for example" or "like". In "if I click something interactable, like another person", the subject is "something interactable". The class and the example may be in different messages, so read the whole conversation.
- If you proposed the design and the user just said "ok", the nouns in it are still yours. They are not a subject the user chose, so run this step on them too.
- If the user gave only an example, ask: which properties of the example does the rule actually read? Following reads "position" and "can be seen", so the subject is everything that has a position and can be seen. The properties the rule uses define the subject, not the example's kind.
- A class name can be too narrow as well. "Creature" means "has a body and a mind", but following only needs position and visibility. Whatever the name demands beyond what the rule reads is the range of things you'll wrongly exclude.
- When you turn an open-ended phrase from the user ("if something gets in the way") into a list, write the list items as capabilities ("out of reach", "can't see it", "no path"), not as states of one kind of thing ("target died" also excludes a corpse being dragged away).
- Check: swap the first example for the second in the rule's sentence. If it still reads true, the subject is right.
- If you're not sure the user wants the whole class, ask once: give your proposed subject and 2-3 candidate second examples to pick from (does "follow" include a dog? a car?), not an open question. Don't silently narrow it to the example, and don't silently widen it to "everything".

A capability is a fact the rule reads directly (can hear, has a position, has export permission). A kind is a name (person, admin, Safari, `run`). Checking the kind assumes the name will always come bundled with those capabilities. Web developers learned this the hard way: browser sniffing by User-Agent breaks; feature detection doesn't.

## 2. The second example

- Who picks it: whoever writes the spec, never whoever implements it. An implementer that picks its own second example picks one it already passes. Whoever writes the test must not be whoever takes it. Propose it yourself and write it next to the subject; only bring the user in when the subject is a real design decision (section 1), not on every rule.
- How to pick it: it has every capability the rule reads, but differs from the first example in kind, data shape, or where its data lives. The more shortcuts it breaks, the better. If the target is "a person", pick "an item carried by a person": it has no position of its own, its position comes from whoever is holding it, and a person-only implementation fails immediately.
- Acceptance: every rule has at least one test that exercises the behavior with the second example. An implementation that only passes with the first example is not done. Tests that regex-match the source code don't count.
- Same code path: the second example must go through the same code as the first. Adding a separate branch for it just hardcodes two examples instead of one.
- No second example exists in the system yet: use the cheapest stand-in available, such as a test fixture that carries only the capability fields the rule needs. Don't build the future feature just to have something to test.
- Optionally add a negative example: something missing a capability should be rejected, and the failure reason should name the capability ("can't hear"), not the kind ("not a person").

## 3. Two kinds of concreteness: where "don't over-engineer" fits

Hardcoding and over-engineering look like opposite failure modes, but they govern different things:

| | Rules the user stated | Places nobody mentioned |
|---|---|---|
| Subject | Implement for the class; accept with the second example | Write for today's need; concrete is fine |
| Mechanism | Least code possible | Least code possible; YAGNI |

- Implementing a stated rule for its whole class is the requirement, not scope creep. "Minimal change" constrains the mechanism, not the subject.
- Implementing for the class usually needs no new abstraction, just a different field to read: read the `senses` component instead of `species`; call `can(user, 'export')` instead of checking `role === 'admin'`. If going class-wide seems to require a registry, plugin system, or interface hierarchy, stop. Either the data is missing the fact the rule needs to read (add the data, not a framework), or this rule doesn't actually need to generalize (ask).
- The second example is also the upper bound on scope. Once two examples work through the same path, stop. Don't write code for an imagined third.
- World-level statements like "simulate everything" or "support multi-tenancy" tell you which direction rules generalize in. They don't license generalizing everything. They only take effect when they reach each rule's subject: one sentence of vision at the top of a doc loses to one concrete noun inside a rule.

## 4. Code review: find branches on kind

"Kind" here means more than entity type. It includes the names of actions, items, events, roles, and gaits.

**Step 1: trace the data, starting from the rule.** For each rule the user stated, find the code that implements it and list every fact it reads:
- Where is it read from? If it reads data only one kind of thing has, that's a finding. Example: visibility reads `body.heightCells`, when shape should be something any entity can have.
- Who writes it? A capability field that no production code writes, and only tests set, means everything runs on the default value. That's a finding, not a footnote.
- Is one capability decided by two different facts? Example: "can it be eaten" checks `category === 'food'`, while "how much does eating restore" reads `nutrition`. Anything with nutrition but a different category will fall through the gap between them.

Grep can't catch this class of problem, which is why this step comes first. If the spec itself lists kinds ("click a person, an item, or furniture"), code that follows it is being faithful. The problem is in the spec; go back to section 1.

**Step 2: grep for candidates** (use `grep -rnE` if `rg` isn't available):

```bash
# Kind checks
rg -n "\b(kind|type|species|category|role)\b\s*(===|!==|==|!=)\s*['\"]" <dirs>
rg -n "instanceof |isinstance\(|case ['\"]" <dirs>
# Hardcoded lists of names; type decided by string prefix
rg -n "\[\s*['\"][a-z_]+['\"]\s*,\s*['\"]" <dirs>
rg -n "startsWith\(['\"]|split\(['\"]:['\"]\)" <dirs>
# The spec's example nouns, as literals or inside function, parameter, or field names
rg -n -i "['\"](<example1>|<example2>)['\"]|<example1>|<example2>" <dirs>
```

**Step 3: ask one question of every candidate: is this branching on what it is, or on what it can do?**

- Branching on what it can do: fine.
- Dispatch by kind at the definition site: fine. Each kind's own behavior lives in a table or switch keyed by kind, such as message handlers, a closed protocol enum, or a species table that yields capabilities. Display names, sprites, and exceptions the user explicitly stated also count. The right shape is: kind → data table → capability → rule.
- Kind check at a query site: this is the bug. Logic somewhere else holds a list of names and asks "is it one of these?" to decide a shared property: can it be resumed, does it count as work, is it moving, should it run. When a new kind shows up, nobody remembers to update that list. The property belongs in the kind's own data, such as `resumable: true` in the action table or a speed in the gait table. Generating options by capability and then, at execution time, only handling one of the names also falls in this bucket.
- Telling definition sites from query sites: the definition site is the kind's own definition, meaning its data row and its handler. A list of names outside the kind's definition is a query site, even if it's a named constant (`WORK_ACTIONS`). If the same list appears twice with different contents, someone has already forgotten to update one.
- Names the data format requires every member to have (say, every body must have `walk`) are safe to rely on. Names only some members have (`run`) must not be assumed present.

Report each real problem like this:

```
Location: file:line
Rule: which rule (quote the spec)
Second example: what would fail here, and how
Should read: which capability field or query, and which layer owns it
Confidence: confirmed / needs checking (say what evidence is missing)
```

Check the tests too: if a rule's tests only use the first example, the missing second-example test is itself a finding.

## 5. Handing off to another agent

The receiving agent only sees the contract, and each handoff loses a little more of the "why" from the layer above. Every rule in the contract needs the items below, even if the contract links to a design doc, because the receiver will build from the contract:

- The subject and what decides it;
- The first example, the second example, and the test for the second example. The second example is fixed: the receiver may not swap it or pick its own;
- A failure condition: "An implementation that only works for <first example> is rejected";
- Names that follow the subject, not the example (`perceivedEntity`, not `perceivedPerson`);
- A requirement that the report back lists every kind-based branch in the new code and why each one is fine.

The items most often missed are the failure condition and the test list. The subject gets written correctly, but the failure condition says "when following a person, walks to the old tile" and the browser test says "follow a walking person", and the receiver still builds a person-only feature.

If you're the receiver and a rule in the contract has an example but no subject: point it out before writing code, or fill it in using this section and state in your report which subject you added. If it has no second example, send it back and ask for one before writing code. Don't invent one: you'd pick one you already pass. (Contract says "only admins can delete" with no second example? Ask whether an editor should; don't decide that `role === 'admin'` is the rule.)

## Self-check

- Does every rule have a subject, what decides it, and a second example chosen by the spec writer, not the implementer?
- Does the second example have a test, and does it run through the same code as the first?
- Is there any new abstraction that wasn't needed to make the second example pass? If so, delete it.
- Has every kind branch in the new code been asked "what it is, or what it can do?"
