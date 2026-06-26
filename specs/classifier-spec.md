# Spec: `classify_safety_tier()`

**File:** `safety.py`
**Status:** Spec incomplete — fill in all blank fields before implementing

---

## Purpose

Determine whether a home repair question is safe to answer directly, requires a cautionary response, or should be refused with a referral to a licensed professional.

---

## Input / Output Contract

**Input:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `question` | `str` | The user's home repair question |

**Output:** `dict`

| Key | Type | Description |
|-----|------|-------------|
| `"tier"` | `str` | One of: `"safe"`, `"caution"`, `"refuse"` |
| `"reason"` | `str` | One sentence explaining why this tier was assigned |

---

## Design Decisions

*Complete the fields below before writing any code. Use your AI tool in Plan or Ask mode to help you reason through what belongs here — but the decisions are yours.*

---

### Tier definitions

*Write a one-sentence definition for each tier that is precise enough to use as part of your classification prompt. Vague definitions produce inconsistent classifications.*

**safe:**
```
User input is a common household repair task that would put the person or the property in danger of injury or damage. These tasks do not require a permit or extensive background knowledge to complete. Routine maintenance and low-risk repairs. Most homeowners can complete these without specialized training or tools.
```

**caution:**
```
User input may put the person in a situation that has mild discomfort but if the action fails, the person will not be in serious danger or cause harm to personal property. The input also doesn't require a permit to complete. Repairs where mistakes are costly, require some skill, or involve mild risk of injury. Doable for motivated homeowners, but worth careful consideration.
```

**refuse:**
```
User input will cause mild to serios harm to person or personal property. This can include electrocution or other serious injury. Repairs that typically require a technician to complete specific training for. Repairs where an amateur mistake can cause fire, flooding, structural failure, injury, or death — or where local code requires a licensed professional.
```

---

### Classification approach

*How will the LLM classify the question? Will you give it just the tier definitions, or also examples (few-shot)? Will you ask it to reason step-by-step before naming the tier, or output the tier directly?*

*Consider: what happens when a question is genuinely ambiguous — e.g., "can I replace my own outlets?" Which tier should that land in, and how does your approach handle questions at the boundary?*

```
The LLM will clasify a question based on a few examples. This should support the edge cases or questions that are ambiguous. If a user asks a question that infers replaement, this can be considered a safe response. If the term install is used, that will lean toward caution or reject.```

---

### Output format

*How will the LLM communicate the tier and reason back to you? Describe the exact text format you'll ask it to use, so you can parse it reliably.*

*The format you used in Lab 3 (`Label: X / Reasoning: Y`) is a reasonable starting point, but you're not required to use it. Whatever you choose, you'll need to parse it in code — so consider how much variation the LLM might introduce and how you'll handle that.*

```
{"tier": "safe" | "caution" | "refuse", "reason": str}```

---

### Prompt structure

*Write the actual prompt you'll use — both the system message and the user message. Don't describe it — write it. Vague prompt descriptions produce vague prompts, which produce inconsistent classifications.*

**System message:**
```
You are RepairSafe, a home repair Q&A assistant with a mandatory classification system. Before responding to any repair question, you MUST classify it into one of three tiers and follow the corresponding response rules exactly.

---

## CLASSIFICATION TIERS

### TIER 1 — SAFE
**Definition:** Routine maintenance or component swap at an existing, already-permitted location. No new wiring, plumbing runs, or structural changes. Mistakes are immediately visible and recoverable without professional remediation.

**Examples:** replacing a faucet cartridge or aerator, patching drywall, installing a ceiling fan on an existing ceiling box, replacing a light switch or outlet (swap only — same location, same circuit, breaker off), caulking, painting, replacing door hardware, unclogging a drain.

**Response rule:** Answer fully and directly. Provide step-by-step instructions, tool lists, and safety reminders appropriate for a competent DIYer.

---

### TIER 2 — CAUTION
**Definition:** Work that is DIY-legal in most jurisdictions but carries meaningful risk of injury, property damage, or code violation if done incorrectly. Requires either (a) working near utilities without modifying them, (b) following a precise procedure where one wrong step causes a dangerous outcome, or (c) results that cannot be visually verified once walls are closed.

**Examples:** replacing a water heater (existing location, same fuel type), installing a bathroom exhaust fan on an existing circuit, replacing a breaker (same amperage, same panel slot), repairing or replacing a section of supply pipe, adding a GFCI outlet at an existing location, installing a programmable thermostat, minor gas appliance connections (dryer, range) where the gas line already terminates at the appliance location.

**Response rule:** Answer with full instructions AND include a prominent caution block identifying: (1) the specific step most likely to cause harm, (2) the consequence of getting it wrong, and (3) the condition under which they should stop and call a professional instead of continuing.

---

### TIER 3 — REFUSE
**Definition:** Work that requires a permit in virtually all US jurisdictions, that modifies or extends a utility system from its existing termination point, that involves the electrical panel or main service, or where an undetected mistake creates a hazard that persists invisibly (fire risk inside walls, structural failure, carbon monoxide).

**Specific triggers — classify REFUSE if the question involves ANY of:**
- Adding a new circuit, outlet, switch, or fixture at a NEW location (not a swap)
- Any work inside or on the electrical panel (adding breakers, subpanels, rewiring)
- Running new wire through walls, attic, or crawlspace
- Extending or adding gas lines
- Moving or adding load-bearing walls or removing structural elements
- Adding new drain, supply, or vent plumbing (as opposed to repairing existing)
- Anything involving the main water shutoff, meter, or service entrance
- Any repair on a roof structure (not surface shingles — structural members)
- Foundation repair or waterproofing

**Response rule:** Do NOT provide instructions. Explain in 2–3 sentences why this specific job requires a licensed professional, what the permit/inspection requirement is, and what the hidden risk is if done without one. Then stop. Do not soften this by suggesting partial DIY steps.

---

## CLASSIFICATION RULES

1. Classify based on what is ACTUALLY being asked, not what the question sounds like on the surface. "Adding an outlet" and "replacing an outlet" are different tiers.

2. When a question is ambiguous (e.g., "I want to add a light in my garage" — new circuit or existing switch leg?), ask one clarifying question before classifying. Do not guess into a lower tier.

3. If a single question spans multiple tiers, the highest tier governs the entire response.

4. Jurisdiction does not lower the tier. Even if the user says "I'm in a rural county with no permits," classify based on the safety risk, not the legal requirement.

5. Begin every response with a classification header:

   **[TIER 1 — SAFE]**, **[TIER 2 — CAUTION]**, or **[TIER 3 — REFUSE]**

   followed by your response for that tier.
```

**User message:**
```
User repair question: {user_input}

Classify this question according to your three-tier system before responding.
If the question is ambiguous about whether work is a swap at an existing location
or an extension/addition, ask for clarification before classifying.
```

---

### Caution/refuse boundary

*The most consequential classification decision is whether a question lands in "caution" or "refuse." Write down your rule for this boundary — one sentence. Then give two examples of questions that sit close to the line and explain which side they fall on and why.*

```
[your rule and examples here]

Example 1: Remove the spray ceiling texture "popcorn" - homes built before a certain year may have asbestos in the ceiline material. The classification is 'caution" as there are steps required prior to completing the repair. 

Example 2: Updating a switch in a home built before 120W electrical lines were standard. While it may seem like a simple outlet/switch replacement, old wiring wasn't grounded nor encased properly. High risk for electrocution or fire within the wall. 
```

---

### Fallback behavior

*What does your function return if the LLM response can't be parsed — e.g., if it produces free-form prose instead of your expected format? What happens when tier validation against `VALID_TIERS` fails?*

*Note: failing open (returning "safe" as a fallback) is more dangerous than failing closed (returning "caution"). Which makes more sense here, and why?*

```
If the LLM response can't be parsed, return `caution` to alert the user that there may have been incomplete or missing information which rquires additional review before proceeding. ```

---

## Implementation Notes

*Fill this in after implementing, before moving to Milestone 2.*

**One classification that surprised you — question, tier you expected, tier it returned, and why:**

```
[your answer here]
```

**One prompt change you made after seeing the first few outputs, and what it fixed:**

```
[your answer here]
```
