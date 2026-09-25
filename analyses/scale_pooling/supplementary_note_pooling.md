# Supplementary Note: Pooling item embeddings into scale vectors, and when a match is shown

SynthNet represents each scale by one vector, the mean of its item
embeddings, and ranks database scales by the cosine between the query
scale's vector and theirs. Two decisions affect what a user sees: how
reverse-keyed items enter the mean, and above which cosine a database scale
is displayed as related. We evaluated both against empirical scale
correlations from the two SurveyBot3000 validation datasets, in which every
scale-scale correlation was estimated from respondent data: the pilot
holdout (Bainbridge et al., 2022 data; 493 respondents, 418 items, 113
scales, 6,245 scale pairs) and the pre-registered Prolific validation study
(387 respondents, 246 items, 80 scales, 3,119 pairs). Scale pairs nested
within one another (a scale and its own facet) were excluded, as in the
SurveyBot3000 paper. Item embeddings came from SurveyBot3000, whose encoder
returns unit-length vectors. Scripts and all tables are in
`analyses/scale_pooling/` of the SynthNet repository.

## S1. How reverse-keyed items enter the scale vector

We compared five pooling rules. All first normalise item vectors to unit
length.

- **A, plain**: mean of item vectors, keying ignored.
- **B, documented**: reverse-keyed items multiplied by -1, then the mean.
- **C, aligned**: as B, then iteratively flip any item whose dot product
  with the current centroid is negative until signs stop changing.
- **D, hyperplane** (the rule SynthNet used previously): compute the
  centroids of positively and reverse-keyed items, reflect reverse-keyed
  items through the hyperplane midway between the two centroids, but only
  those whose cosine with the positive centroid is negative, then the mean.
- **E, positive only**: mean of positively keyed items.

As references we used the composite correlation implied by the predicted
item correlation matrix under documented keying, and the SurveyBot3000
paper's own scale-level prediction (a Pearson correlation across embedding
dimensions between keyed mean embedding vectors, which is numerically
identical to B).

Table S1. Accuracy of the pooled cosine against the empirical scale
correlation. Sign errors are counted among pairs with |empirical r| > .10.

| Pooling rule | Pilot: r | Pilot: MAE | Pilot: sign errors | Validation: r | Validation: MAE | Validation: sign errors |
|---|---|---|---|---|---|---|
| A plain | .23 | .21 | 36 % | .33 | .19 | 41 % |
| B documented | .87 | .11 | 7 % | .75 | .14 | 20 % |
| C aligned | .87 | .11 | 7 % | .75 | .14 | 20 % |
| D hyperplane | .55 | .17 | 23 % | .46 | .18 | 37 % |
| E positive only | .64 | .15 | 20 % | .57 | .17 | 31 % |
| Composite-correlation reference | .87 | .11 | 7 % | .75 | .14 | 20 % |

Documented-sign pooling (B) was the most accurate rule on both datasets and
matched the reference; SynthNet now uses it. The previous hyperplane rule
(D) fell well short. The reason is visible at the item level: the model
usually places a reverse-keyed item on the correct side of its scale, but
with much smaller magnitude than for positively keyed items (median cosine
with the rest of the scale -.09 for reverse-keyed versus +.31 for
positively keyed items in the validation study). D therefore left 48 of 78
reverse-keyed item memberships in the validation study unreflected, and for
scales consisting only of reverse-keyed items it pointed the vector against
the construct. Sign alignment (C) changed the sign of only 4 of 374 item
memberships in the validation study and 13 of 739 in the pilot; judged by
the item's empirical correlation with the rest of its scale, 14 of these 17
were model errors rather than documentation errors, so we use the
alignment step only as a curation report, not as an automatic override.

The same ordering held for retrieval. With each scale as query and all
scales from other instruments as the database, ranked by absolute cosine,
B reached precision at 5 of .73 and NDCG at 10 of .87 in the pilot, against
.66 and .81 for D. In the smaller validation study, top-5 precision was
within noise for all rules (.42 to .44), partly because 28 of its 80
scales have no cross-instrument neighbour with |r| >= .40 at all.

Note that the pilot's "documented" keying was itself derived from the
empirical data by the SurveyBot3000 paper (one hand-coded item per scale,
the rest keyed by the sign of their correlation with it), which is why its
sign-error rate is so low; the Prolific validation study has documented
per-item keying and is the stricter test.

## S2. Robustness to wrong keying

Keying in the SynthNet corpus comes from extraction and is imperfect. We
corrupted the documented keying of both datasets in three ways, 50 random
draws per condition: unmarking a fraction u of reverse-keyed item
memberships (u = 0, .10, .25, .50, .75, 1), marking 3 % of positively keyed
memberships as reverse, and inverting the whole keying of 3 % of scales.

Table S2. Pearson r of the pooled cosine with the empirical scale
correlation (mean over draws), when only reverse keys are missing.

| u unmarked | Pilot: B | Pilot: D | Validation: B | Validation: D |
|---|---|---|---|---|
| 0 | .87 | .55 | .75 | .46 |
| .25 | .78 | .41 | .68 | .39 |
| .50 | .61 | .32 | .57 | .35 |
| 1.0 | .23 | .23 | .33 | .33 |

Accuracy of B declines roughly linearly with the share of missing reverse
keys and only reaches the plain-pooling floor when every reverse key is
lost; with half the reverse keys missing it still exceeds the hyperplane
rule on perfect keying. Adding the 3 % false marks and inverted scales
lowers every curve by about .10 and widens the spread across draws; an
inverted scale is the costly error because it flips the sign of every pair
it enters. Sign alignment (C) recovered only a minority of unmarked reverse
items (25 % at u = .10 in the validation study) and none of the inverted
scales, so it cannot substitute for correct keying.

## S3. Which matches to display

The interface shows database scales whose absolute cosine with the query
scale exceeds a threshold, together with the sign. We evaluated this rule
on cross-instrument pairs only (5,448 in the pilot, 2,949 in the validation
study), treating a pair as truly related when |empirical r| >= t.

Table S3. Precision (share of displayed pairs that are truly related) and
recall (share of truly related pairs that are displayed) with bootstrap 95 %
intervals over scale pairs, and the base rate of related pairs.

| Dataset | t | Base rate | Displayed | Precision | Recall | Sign agreement among displayed |
|---|---|---|---|---|---|---|
| Pilot | .30 | .30 | 1,295 | .74 [.71, .76] | .59 [.56, .61] | .998 |
| Pilot | .40 | .15 | 657 | .69 [.65, .72] | .55 [.51, .58] | 1.000 |
| Pilot | .50 | .07 | 269 | .70 [.64, .75] | .50 [.45, .55] | 1.000 |
| Validation | .30 | .23 | 288 | .81 [.77, .86] | .35 [.31, .39] | 1.000 |
| Validation | .40 | .11 | 124 | .85 [.79, .91] | .32 [.27, .37] | 1.000 |
| Validation | .50 | .06 | 47 | .83 [.71, .93] | .24 [.18, .31] | 1.000 |

Table S4. Calibration: empirical correlation by bin of predicted cosine.

| Predicted cosine | Pilot: n | Pilot: mean |r| | Pilot: share |r| >= .40 | Validation: n | Validation: mean |r| | Validation: share |r| >= .40 |
|---|---|---|---|---|---|---|
| 0 to .1 | 1,561 | .13 | .01 | 1,472 | .13 | .01 |
| .1 to .2 | 1,536 | .18 | .04 | 850 | .18 | .06 |
| .2 to .3 | 1,056 | .25 | .11 | 339 | .28 | .21 |
| .3 to .4 | 638 | .33 | .29 | 164 | .40 | .53 |
| .4 to .5 | 388 | .41 | .55 | 77 | .49 | .77 |
| .5 to .6 | 186 | .53 | .84 | 35 | .59 | 1.00 |
| .6 to .7 | 63 | .65 | 1.00 | 12 | .69 | 1.00 |
| .7 and above | 20 | .72 | 1.00 | 0 | | |

At a display threshold of .40, between two thirds and five sixths of the
scales shown are related at |r| >= .40 in the empirical data, four to eight
times the base rate, and the displayed sign was correct for every such
pair in both datasets. The sign errors reported in Table S1 are confined
to pairs the model predicts near zero, which are not displayed. Recall is
the weaker side: about half of the truly related cross-instrument pairs in
the pilot and two thirds in the validation study fall below .40, so the
threshold should be read as a precision-oriented default that users can
lower. The two datasets differ in calibration rather than in ranking
quality: in the pilot the predicted cosine tracks the empirical correlation
almost one to one, whereas in the validation study predictions are shrunk
toward zero (pairs predicted between .30 and .40 average .40 empirically
and no pair exceeds .70), which is why precision is higher and recall lower
there, and why a threshold of .30 would already give .81 precision in that
dataset. For reference, ranking cross-instrument scales by absolute cosine
gives precision at 5 of .75 in the pilot (110 query scales with at least
one related scale) and .67 in the validation study (52 query scales).

Figures S1 and S2 (`results/calibration_pilot_holdout.png`,
`results/calibration_validation_prolific.png`) show the pair-level scatter
of empirical against predicted absolute correlations with bin means and the
identity line.
