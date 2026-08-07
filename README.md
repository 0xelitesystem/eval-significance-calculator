# Eval Significance Calculator

Paste two eval runs, or just the counts, and find out whether that 3-point score move is a real improvement or sampling noise.

## Live demo

https://0xelitesystem.github.io/eval-significance-calculator/

**The thesis: eval runs share the same cases, so the correct test is a paired one.** The standard move is to put a confidence interval on each run's pass rate and check whether they overlap. That is wrong twice over. Overlapping intervals do not imply a non-significant difference, and the unpaired framing throws away the pairing, which is where most of the available power lives. This tool runs the exact paired test on the cases that actually changed, and shows the unpaired comparison only as a labelled counterexample.

## Features

- **Three input modes.** Counts (the 15-second path: total cases plus the two discordant counts), run files (JSONL, JSON array, or CSV, joined on case ID), and scored files (a numeric score per case, giving a paired difference).
- **Exact by default.** Under the null, `b` is Binomial(`b + c`, 0.5), so the two-sided exact p is `min(1, 2 * P(X <= min(b, c)))`. The chi-square approximation with continuity correction is shown beside it as a labelled comparison, with a note that the approximation is least reliable exactly where eval sets live.
- **Effect size at least as prominent as the p-value.** The difference in pass rate with a confidence interval, the discordant split itself, and the discordant odds ratio where it is defined, all above the significance numbers on the page.
- **The A/A noise floor.** Run the same configuration twice, paste those discordant counts, and the tool refuses to call an A/B delta real when it sits inside the variation your harness produces with no change at all. The floor is always your measurement; no default is shipped.
- **Multiple comparisons.** Tell it how many variants you tried before picking the winner and the uncorrected p-value is labelled optimistically biased, with a Holm-corrected value alongside it.
- **Power and required sample size.** Given the discordance rate you measured and a target effect, it computes how many cases you would need, and prints the formula rather than only the number.
- **It refuses.** Below a stated discordant-pair threshold no verdict is printed at all. You get the discordant count, a plain statement that the data cannot support a conclusion either way, and how many more cases would be needed.
- **Seeded bootstrap in scored mode.** A percentile interval on the mean paired difference, with the seed shown and editable, so a reload reproduces the exact same numbers. The paired t-statistic is reported as a secondary line with its normality assumption stated.
- **Copyable markdown summary** for pasting into a PR description, plus a small inline SVG of the discordant split with the null expectation marked, and the bootstrap distribution with its interval shaded.
- Dark by default, light and dark themes persisted across visits, responsive to 360px, keyboard accessible, no external dependencies.

## How it works

Everything is one HTML file. The statistics are implemented from scratch in plain JavaScript, in pure functions that take input and return a result object.

- **Paired test.** McNemar's test on the discordant pairs `b` (baseline failed, candidate passed) and `c` (baseline passed, candidate failed). Cases where both runs agree are discarded on purpose: they carry no information about which configuration is better, and including them buries the signal in their variance.
- **Exact p-value.** The binomial CDF is summed in log space using a Lanczos log-gamma, so it does not overflow at large discordant counts. Above two million discordant pairs it falls back to a continuity-corrected normal approximation and says so on the page.
- **Interval.** When all four cells are known, a hybrid score interval for the paired difference: Wilson score limits on each rate combined square-and-add with the observed pairing correlation (a construction commonly attributed to Newcombe). When only `b`, `c` and `n` are known, a Wald interval, labelled as such.
- **Tail functions from scratch.** `erf` via the Abramowitz and Stegun 7.1.26 rational approximation (maximum absolute error 1.5e-7), the inverse normal via Acklam (relative error below 1.15e-9), and the Student t tail via a regularized incomplete beta using a modified Lentz continued fraction (relative tolerance 3e-14). The page names all of these and their accuracy limits in a visible note, because the exact test's credibility rests on them.
- **Bootstrap.** Matched case indices resampled with replacement using an inline seeded xorshift32, default 10,000 iterations with the count exposed as an input. Total work is capped so a very large file cannot freeze the tab; when the cap engages the page says how many iterations it actually ran.
- **Joining runs.** Files are joined on case ID. Cases present in only one run are reported in both directions rather than silently inner-joined, duplicate IDs are flagged and counted once rather than twice, and non-numeric values in a numeric column are reported rather than coerced. Lookup maps are null-prototype, so a case ID of `constructor` or `__proto__` behaves like any other string.

`b` and `c` are exactly the counts that [eval-regression-viewer](https://github.com/0xelitesystem/eval-regression-viewer) already reports as new failures and new passes. That repo answers what changed. This one answers whether it means anything.

## The three samples

All sample data is synthetic and is labelled synthetic in the interface itself, not only here. No model, vendor, or benchmark is named, and no claim is made that any model or prompt beat any other.

1. **The headline that is noise.** 200 cases, baseline 142 pass, candidate 148 pass. A plus-6-case, plus-3-point improvement that reads like a win in a changelog. The discordant split is 14 one way and 8 the other, and the exact two-sided p is 0.286. Not significant.
2. **The modest result that is real.** 200 cases, baseline 142 pass, candidate 156 pass, split 18 and 4. Exact p is 0.00434, clearly significant on the paired test, while a naive unpaired comparison of 142/200 versus 156/200 gives p = 0.108 and does not reach significance. The page shows both side by side. This is the power argument made concrete rather than asserted: the same data reads as no evidence unpaired and as a clear result paired.
3. **The A/A floor.** The same configuration run twice on the same 200 cases, split 9 and 7. It observes a 1.00-point difference where nothing changed at all, and its interval still allows 4.92 points in either direction. That 4.92 is the floor. Loaded alongside sample 1, it flips that verdict to "within your measured noise floor": a harness with that much slack cannot be credited with having detected a 3-point move.

The **Load sample** button loads sample 1 with sample 3 as its floor, which is the whole argument in one click.

## Assumptions

Reproduced verbatim from the page, where they sit next to the result rather than in a footer.

- Cases are treated as **independent**. If your eval set contains near-duplicates, paraphrases of one prompt, or several turns of one conversation, they are not, and every p-value here is too small.
- Cases are treated as **drawn from the distribution you actually care about**. This is the assumption that is usually false. A hand-picked set of 100 prompts you wrote while debugging is not a sample of production traffic, and no test on this page can fix that.
- A non-significant result means **insufficient evidence of a difference**. It is not proof that the two configurations are the same.
- **Statistically significant and worth shipping are different questions.** Read the effect size and its interval, then decide whether the smallest effect the interval still allows would be worth the change.
- This is a calculator, not a statistician. If the decision is expensive, get a real one to look at your design.

## What this tool refuses to do

- **It will not print a verdict below your discordant-pair threshold.** It shows the count, says the data cannot support a conclusion either way, and tells you how many more cases you would need. Refusing is a feature, and it is designed as one rather than as an error state.
- **It ships no default flakiness rate.** The A/A noise floor is always your own measurement of your own harness. If you paste nothing, there is no floor, and the tool never implies it knows one.
- **It will not tell you a result is worth shipping.** It answers whether a difference is distinguishable from noise. Whether the size of that difference justifies the change is your call.
- **It will not fabricate a sample-size answer.** If your target effect is too large relative to the discordance rate you measured, it returns the reason instead of a reassuring number.
- **It will not treat an unmatched case as a failure.** Cases missing from one run are reported, not silently folded into the table.
- **It does not run your evals, call any API, or upload anything.** There is no key field because there is nothing to authenticate to.

## Definitions

Every test this tool runs, with somewhere to read about it.

- [McNemar's test](https://en.wikipedia.org/wiki/McNemar%27s_test) - the paired test on the discordant pairs.
- [Exact binomial test](https://en.wikipedia.org/wiki/Binomial_test) - how the exact p-value is obtained.
- [Continuity correction](https://en.wikipedia.org/wiki/Yates%27s_correction_for_continuity) - the correction applied to the chi-square comparison.
- [Wilson score interval](https://en.wikipedia.org/wiki/Binomial_proportion_confidence_interval#Wilson_score_interval) - the per-rate limits the paired interval is built from.
- [Odds ratio](https://en.wikipedia.org/wiki/Odds_ratio) - the discordant odds ratio `b/c`.
- [Two-proportion z test](https://en.wikipedia.org/wiki/Z-test) - the unpaired comparison, shown only as a counterexample.
- [Holm-Bonferroni method](https://en.wikipedia.org/wiki/Holm%E2%80%93Bonferroni_method) - the multiple-comparison correction.
- [Bootstrapping](https://en.wikipedia.org/wiki/Bootstrapping_(statistics)) - the percentile interval in scored mode.
- [Paired t-test](https://en.wikipedia.org/wiki/Student%27s_t-test#Dependent_t-test_for_paired_samples) - the secondary line in scored mode.
- [Statistical power](https://en.wikipedia.org/wiki/Power_of_a_test) - what the required sample size is solving for.
- [Lanczos approximation](https://en.wikipedia.org/wiki/Lanczos_approximation), [error function](https://en.wikipedia.org/wiki/Error_function), [xorshift](https://en.wikipedia.org/wiki/Xorshift) - the numerical machinery underneath.

## Privacy

Everything runs in your browser. There is no backend, no analytics, no telemetry, and no network request of any kind: the page loads no fonts, scripts, styles, or images from anywhere else. Run counts, case IDs, scores, and every file you paste stay in the tab and are never transmitted. The only thing written to storage is your light or dark theme preference. Save the file and it works offline, unchanged.

## License

MIT. See [LICENSE](LICENSE).

## More

- More tools: https://0xelitesystem.github.io/
- https://elitesystem.ai
