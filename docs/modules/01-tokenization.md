# Module 1: Tokenization and Vocabulary

## The big idea, simply put

Your AI model doesn't read sentences. It reads **chunks**.

Take this service desk ticket:

> *"VPN still disconnects every 5 minutes after latest patch. Please help ASAP."*

Before the model processes a single word, it chops that sentence into **tokens** — the raw units it thinks in. Here's what that actually looks like:

```
["VP", "N", " still", " disconnect", "s", " every", " 5", " minutes",
 " after", " latest", " patch", ".", " Please", " help", " AS", "AP", "."]
```

That's **17 tokens** for one short sentence. Notice a few things:
- "VPN" splits into `VP` + `N` — the model has never "seen" VPN as one word
- "disconnects" splits into `disconnect` + `s`
- "ASAP" splits into `AS` + `AP`

This isn't a bug. It's how models handle words they weren't trained on, technical jargon, and unusual capitalization. Roughly speaking: **1 token ≈ ¾ of an English word**, or about 4 characters.

Now compare that to a longer ticket:

> *"Hi team, this is a follow-up to ticket #4821. Since the patch deployed Tuesday, our VPN client disconnects every 5 minutes for all remote staff in the APAC region. We've tried reinstalling, cleared DNS cache, tested on 3 different machines. Issue persists. Senior engineer Sarah has escalated this internally. We need resolution before the board meeting Thursday 9am SGT."*

That's roughly **90 tokens** — five times more — and it still hasn't included any system instructions, conversation history, or space for the model to write a response.

**This is why tokens matter**: you have a fixed budget per request. Blow the budget, and something gets cut.

---

## The math that matters

Think of your token budget like packing a suitcase with a strict weight limit:

$$
T_{total} = T_{system} + T_{history} + T_{user} + T_{output}
$$

| Part          | What it is                                                           | Example size                          |
|:--------------|:---------------------------------------------------------------------|:--------------------------------------|
| $T_{system}$  | Your instructions to the model ("You are a helpful IT assistant...") | 200–800 tokens                        |
| $T_{history}$ | Every prior message in the conversation                              | Grows fast — 50 turns ≈ 5,000+ tokens |
| $T_{user}$    | The current ticket or message                                        | 20–300 tokens                         |
| $T_{output}$  | Space reserved for the model's reply                                 | 500–2,000 tokens                      |

**The hard rule: $T_{total}$ cannot exceed your context window.** Most models today allow 8,000–200,000 tokens, but cheaper/faster tiers cap at 4,000–8,000. If you hit the wall, the model silently drops the oldest messages — and with them, critical context.

---

## Watch tokenization happen on a real ticket

Here's the same escalation ticket, processed three different ways depending on budget:

**Scenario A — Plenty of budget (32k context)**
```
System prompt:          600 tokens
Conversation (8 turns): 2,400 tokens
Current ticket:         90 tokens
Reserved output:        1,000 tokens
─────────────────────────────────────
Total:                  4,090 / 32,000  ✅ Fine
```
Full history kept. Model can see that this ticket is a follow-up to #4821. Response is accurate.

**Scenario B — Tight budget (4k context)**
```
System prompt:          600 tokens
Conversation (8 turns): 2,400 tokens
Current ticket:         90 tokens
Reserved output:        1,000 tokens
─────────────────────────────────────
Total:                  4,090 / 4,000  ❌ Over by 90 tokens
```
System silently drops the oldest 2 conversation turns to fit. The model loses the detail that Sarah already escalated this. It suggests escalation — wasting everyone's time.

**Scenario C — Smart summarization**
```
System prompt:          600 tokens
Summarized history:     300 tokens  ← compressed 8 turns into a summary
Current ticket:         90 tokens
Reserved output:        1,000 tokens
─────────────────────────────────────
Total:                  1,990 / 4,000  ✅ Comfortable, nothing lost
```
Summarize old turns before they eat your budget. The model still knows the full story.

---

## The trap most people fall into

> **"I'll just count the user's message length."**

Wrong. A user's message is usually the *smallest* part of the budget. Here's what a real service desk copilot actually spends tokens on:

```
Your system prompt alone:  ~600 tokens
("You are an IT helpdesk assistant. Always ask for the ticket number.
Escalate P1 issues within 15 minutes. Never suggest rebooting without
first checking patch history. Use the following knowledge base...")
```

That's 600 tokens *before the user types a single word*. Add 10 turns of conversation history and you've spent 3,000 tokens before the current question arrives.

**The smart move:** Reserve your output budget first (worst-case response length), then see what's left for input. Work backwards, not forwards.

---

## How to practice this

Open `notebooks/math-foundations/01_tokenization.ipynb` and you'll:

1. Paste real service desk tickets and see the exact token count
2. Watch what gets dropped when you exceed your budget
3. Build a summarizer that compresses old turns before they fill your window

---

## Checklist before you ship

- [ ] Do you know your typical token spend per request (system + history + user + output)?
- [ ] Are you reserving output tokens *before* filling input?
- [ ] Do long conversations get summarized rather than just truncated?
- [ ] Do you log when truncation starts happening often? (It means your prompts are growing)

--8<-- "_abbreviations.md"