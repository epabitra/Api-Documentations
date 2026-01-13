# AI Connection Message Rewrite – Functional Specification

## 1. Purpose

This document explains the **AI-powered connection message rewrite feature** that will be used in our application to automatically generate human-like, relationship-first messages.

The goal is to replicate and compete with the experience users get from tools like **LinkedIn AI Rewrite** or **SuperGrow AI**, while maintaining strict control over tone, intent, and structure.

This feature rewrites a base message into multiple natural alternatives that feel personal, trustworthy, and non-spammy.

---

## 2. Core Objective

- Generate **relationship-first messages** that feel human, warm, and natural
- Preserve the **original intent** of the base message
- Improve **clarity, flow, and readability**
- Avoid robotic, salesy, or automated tone
- Ensure all outputs are **safe for automated sending**

---

## 3. AI Role Definition

The AI is instructed to behave as:

> **SSF AI – a relationship-based copywriter**

Key expectations:
- Writes like a real person
- Builds trust before action
- Uses soft, observational language
- Avoids pressure or urgency

---

## 4. Tone & Voice Rules

### Mandatory Constraints

- The AI **must strictly follow** the provided:
  - `tone`
  - `sampleVoice`

- Every generated message:
  - Must begin with a greeting inspired by `sampleVoice`
  - Must feel like it was written by the *same person* as the sample voice

### Example

**Sample Voice:**

"Hey! Hope everything’s going well today"

**Valid Openings:**
- "Hey! Hope everything’s going well today."
- "Hey! Hope everything’s going well today—just wanted to reach out."

**Invalid Openings:**
- "Hello Sir"
- "Greetings"
- "Dear User"

---

## 5. Message Structure (Strict & Ordered)

Each message must follow this exact order:

1. **Friendly greeting** (aligned with sampleVoice)
2. **Clear reason for connecting**
3. **Polite invitation to continue the connection**

> ❗ No reordering is allowed
> ❗ No section may be omitted

---

## 6. Base Message Rule

- `message_1` from the input is the **base message**
- The AI must:
  - Rewrite `message_1` for better clarity and flow
  - Generate `message_2` and `message_3` as stylistic alternatives

All three messages must:
- Convey the **same intent**
- Differ only in **wording, rhythm, and warmth**
- Be fully standalone

---

## 7. Language & Naturalness Rules

To ensure human-like output:

- Prefer soft attribution:
  - "I noticed"
  - "It looks like"
  - "Seems like"

- Avoid:
  - Hard claims
  - Assumptions
  - Sales or promotional language

- Messages should:
  - Sound natural when read aloud
  - Avoid stacked or mechanical clauses

---

## 8. Semantic Consistency Rules

- All messages must communicate the **same overall meaning**
- Differences should be **stylistic**, not semantic
- No message may:
  - Introduce new facts
  - Add external context
  - Change intent

---

## 9. Output Constraints

- Each message must:
  - Be grammatically complete
  - End cleanly (no trailing phrases)
  - Stay under **700 characters**

- The AI must:
  - Return **ONLY valid JSON**
  - Preserve the **exact input structure**
  - Not add, remove, or rename keys

---

## 10. Input JSON Example

```json
{
  "messageResponse": {
    "message_1": "Good to meet another <LOCATION> local. Facebook shows we both live here and know some of the same people. Friend request sent! Let’s stay in touch."
  },
  "sampleVoice": "Hey! Hope everything’s going well today",
  "tone": "Friendly"
}
```

---

## 11. Output JSON Example

```json
{
  "messageResponse": {
    "message_1": "Hey! Hope everything’s going well today. It’s nice to meet another <LOCATION> local—Facebook showed we’re both in the area and share a few connections. I’ve sent a friend request and would be glad to stay in touch.",
    "message_2": "Hey! Hope everything’s going well today. I noticed we’re both based in <LOCATION> and have a few mutual connections, which felt worth reaching out about. I’ve sent a friend request and would be happy to stay connected.",
    "message_3": "Hey! Hope everything’s going well today. Always good to come across someone else from <LOCATION>, especially when we already share some connections. I’ve sent a friend request and would love to keep in touch."
  },
  "sampleVoice": "Hey! Hope everything’s going well today",
  "tone": "Friendly"
}
```

---

## 12. User Experience Outcome

With these rules in place, the feature ensures:

- Messages feel **personal, not automated**
- Users gain **confidence** sending connection requests
- Consistent brand voice across all messages
- Safe, scalable automation without sounding spammy

---

## 13. Summary

This AI rewrite system balances:

- **Strict structure** (for safety and consistency)
- **Flexible language** (for human warmth)
- **Trust-first communication** (for better connection acceptance)

This makes it suitable for production use in automated networking, relationship-building, and social connection workflows.
