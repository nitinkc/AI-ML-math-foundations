# Module 10: Sampling Controls

## What's the big idea?

After a model generates logits and softmax turns them into probabilities, you still have a choice: pick the winner deterministically, or let randomness decide?

And even if you do add randomness, which options are even *allowed* to compete?

That's what **sampling controls** do. They're the knobs you twist to control reliability vs creativity.

- **Low [temperature]**: Ignore weak options, play it safe. Boring but consistent.
- **High [temperature]**: All options on the table. Spicy and unpredictable.
- **[top_p] and [top_k]**: "Only consider the top-K options" or "only options that sum to top P%."

## The math in plain terms

After you apply top-p or top-k filtering, the probabilities need to be rescaled so they still sum to 1:

$$
P'(i)=\frac{P(i)}{\sum_{j \in S} P(j)} \quad \text{for } i \in S
$$

Where $S$ is the set of options you kept after filtering.

**Translation:** If you only keep the top 5 options (top-k=5), you scale them up so their sum is 100%.

## Real-world scenario: Summary profile presets

For your IT copilot, you might define two profiles:

**Safe profile (compliance notes):**
- Low temperature (0.3): Very focused
- Tight top-p (0.9): Only strong options

**Creative profile (draft suggestions):**
- Moderate temperature (1.2): More diverse
- Broader top-p (0.95): More options on the table

Route by use case: compliance-critical notes use safe profile. Optional draft suggestions use creative profile.

Then your system can say: "This is a compliance note, using safe settings—expect repetability."

## How to try this

Run `notebooks/math-foundations/10_sampling_controls.ipynb` and:

- Apply top-k and top-p filtering to probability distributions
- See how probabilities get rescaled
- Compare output diversity under different settings

## The traps to avoid

❌ **The trap:** One global sampling profile for every task  
✅ **The smart move:** Define task-specific profiles, document them, version control them

## Checklist for you

- [ ] Are your decoding profiles documented by business use case?
- [ ] Do you track temperature and top-p changes in release notes?
- [ ] Do compliance-critical outputs use conservative settings?
- [ ] Do you monitor output drift after making sampling changes?

--8<-- "_abbreviations.md"
