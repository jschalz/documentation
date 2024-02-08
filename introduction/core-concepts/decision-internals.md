---
description: How do decisions work, exactly?
---

# Decision Internals

## Dempster-Shafer Theory

Bulwark uses a powerful technique called [Dempster-Shafer theory](https://en.wikipedia.org/wiki/Dempster%E2%80%93Shafer\_theory), which enables the system to automatically handle uncertainty and make informed choices even when there is incomplete or conflicting information.

Bulwark's decision structure is tailored specifically for web application security, and it simplifies the concepts of Dempster-Shafer theory to be more easily understood by users who may not have prior knowledge of it.

## Structure

When Bulwark makes decisions, it considers three main values: `allow`, `restrict`, and `unknown`. These values represent different possibilities or outcomes based on the evidence available. The `allow` value indicates the degree to which evidence supports accepting a request, while the `restrict` value represents the degree to which evidence supports restricting or blocking a request. The `unknown` value reflects uncertainty or the lack of evidence for either outcome.

It's important to note that uncertainty is not an actionable outcome, even though it is quantitatively represented. Also, while these component values share notable similarities with probabilities, they are not probabilities themselves. They may be treated as rough estimates and plugin authors may usefully assign them values that represent a well-informed guess.

### Example

$$
\big\{a\big\} = 0.0, \big\{r\big\} = 0.4, \big\{u\big\} = 0.6
$$

In the example above, the `accept` value is 0.0, the `restrict` value is 0.4, and the `unknown` or power set value is 0.6. In this example, the decision's components indicate a higher confidence for a `restrict` or blocking outcome than an `accept` outcome, but also indicate substantial uncertainty.

## Combination

To combine decisions effectively, Bulwark uses the [Murphy combination rule](https://doi.org/10.1016/S0167-9236\(99\)00084-6). This rule was chosen for its ability to handle scenarios where plugin decisions may have significant conflict or disagreement. It also has good performance characteristics and is straightforward to implement. Importantly, it doesn't assume that its plugin decision inputs are independent, as independence cannot be guaranteed. It also ensures that not-a-number (NaN) values are never produced. The specific rule used for combination is opaque to the plugin API.

## Scoring

Bulwark provides a "score" value to report the decisions made by individual plugins and the combined ensemble. This score is calculated using a technique called a [pignistic transformation](https://en.wikipedia.org/wiki/Pignistic\_probability). The algorithm evenly distributes the `unknown` component value by dividing it in half and redistributing to the `allow` and `restrict` values. Then, for the purposes of reporting a single number, the `allow` value is disregarded, and the `restrict` value becomes the reported score.

Scores in Bulwark are reported in terms of the `restrict` value, following the common practice in security and fraud tools where higher scores indicate a greater level of risk.

Score values offer a better way to apply a decision against a threshold because they provide a single value with a clear meaning associated with the midpoint of the range. The range midpoint, 0.5, represents maximum uncertainty, where there is equal evidence on both sides of the decision. Values above the midpoint indicate more evidence in favor of a `restrict` outcome, while values below it indicate more evidence in favor of an `accept` outcome.

### Example

Considering the previous example decision values, to convert to a score we would divide the `unknown` value of 0.6 by 2 and then add the result, 0.3, to both the `accept` and `restrict` values. The score would then be the new `restrict` value of 0.7.

$$
\big\{a\big\} = \big\{a\big\} + \big\{u\big\} / 2,\\\big\{r\big\} = \big\{r\big\} + \big\{u\big\} / 2,\\\big\{u\big\} = 0.0
$$

$$
\big\{a\big\} = 0.0, \big\{r\big\} = 0.4, \big\{u\big\} = 0.6 \\ \downarrow \\ \big\{a\big\} = 0.3, \big\{r\big\} = 0.7
$$

## Thresholds

End-users of Bulwark have the flexibility to choose appropriate thresholds that align with their risk-tolerance for false positives and false negatives. These thresholds can be set independently of the scores reported by the ensemble.

## Weighting

Individual decision scores can be re-weighted to tune the results from the ensemble. The weighting algorithm involves discarding the `unknown` value, multiplying the `accept` and `restrict` values by the weighting factor, and then scaling the result to ensure it falls within the valid range. Any remainder between 1.0 and the sum of the other two components is assigned back to the `unknown` value. This approach maintains the relative relationship between `accept` and `restrict` values and guarantees a valid result. Weighting by a number less than 1.0 will increase that decision's uncertainty, and may be used to discount a decision value. Weighting by a number greater than 1.0 will reduce a decision's uncertainty, and may be used to boost a decision value, up to the cap, where it will generally be clamped.

### Example

With an `accept` value of 0.3, a `restrict` value of 0.2, and an `unknown` or power set value of 0.5, weighting this by a multiple of 0.5 will result in a new `accept` value of 0.15, a new `restrict` value of 0.1, and a new `unknown` value of 0.75.&#x20;

$$
(\big\{a\big\} = 0.3, \big\{r\big\} = 0.2, \big\{u\big\} = 0.5) * 0.5 = \\
\big\{a\big\} = 0.15, \big\{r\big\} = 0.1, \big\{u\big\} = 0.75
$$
