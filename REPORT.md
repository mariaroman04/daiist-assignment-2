# Weld inspection for vehicle chassis brackets: cost-aware defect detection

> **Status:** Section 1 is written. Sections 2 to 8 are placeholders to fill in as each phase is done.
> Every number below has a source or a stated assumption. Assumptions are marked **(assumption)** and are the ones I should be ready to defend and stress-test.

---

## 1. Business problem

### 1.1 Decision supported

A tier-1 automotive supplier welds steel chassis brackets in three robotic cells, running two shifts a day. A camera photographs every weld seam as it leaves the cell. For each weld the system makes one of three calls:

- **Pass**: the bracket continues to packing.
- **Fail**: the bracket goes to the rework station (grind out and re-weld).
- **Human review**: a qualified inspector looks at the image on a screen and makes the call.

**One case is one weld seam.** The **positive class is a defective (non-conforming) weld**. The mapping from dataset classes to positive and negative is fixed in Section 2.

### 1.2 Who acts on it today

Today a qualified inspector or QA technician judges welds by eye. The relevant standard is ISO 17637:2016 (visual testing of fusion-welded joints). It says final acceptance by visual testing is carried out by qualified personnel, and that when a visual result is inconclusive it should be backed up by another surface inspection method. That "inconclusive" case is what the human-review outcome represents in this project.

**Scope limit.** A camera sees surface defects only. Internal flaws (for example buried porosity) need radiography or ultrasound and are out of scope.

### 1.3 Why a false negative is worse than a false positive

- A **false positive** (good weld flagged) is caught inside the factory. It costs a few minutes of technician time and occasionally an unnecessary regrind.
- A **false negative** (defective weld ships) is discovered later, at the customer or in the field, where it is multiplied by sorting, chargebacks and warranty. It also carries a small chance of a recall campaign. A recall can be large: in one well-known case the vehicle maker recovered about 90% of a battery recall bill from its supplier (Warranty Week, 2026), and OEMs commonly split warranty and recall cost with suppliers 50/50 as a simplification (Plante Moran).

### 1.4 Cost per outcome (euros per weld)

| Outcome | What happens | Cost | Because |
|---|---|---|---|
| **True negative** (good weld, passed) | Nothing | **€0** | No action taken. |
| **False positive** (good weld, flagged) | Bracket pulled, re-checked, returned to the line. 10% of the time it is needlessly reground and re-welded. | **€2** | 3 min pull and re-check = €1.25. 10% x 10 min rework = €0.42. Total €1.67, rounded up to €2 for handling. Labour at €25/h (see 1.6). |
| **True positive** (defect caught) | Bracket pulled, ground out, re-welded | **€6** | 3 min handling (€1.25) + 10 min rework (€4.17) + about €0.60 consumables **(assumption)**. |
| **False negative** (defect ships) | Expected cost of an escaped defect, see 1.5 | **€140** | Probability-weighted cost of three downstream outcomes. |

**Cost ratio FN : FP = 70 : 1.**

### 1.5 How the false negative figure is built

Most defective welds that ship never cost anything, because designs carry safety margin. A few cost a lot. So the false-negative cost is an expected value, not a fixed bill.

| Downstream outcome | Probability per escaped defect | Cost if it happens | Expected cost |
|---|---|---|---|
| Customer finds it: sorting, containment, chargeback | 2.5% **(assumption)** | €2,000 **(assumption)** | €50 |
| One vehicle fails in service: tow, repair, goodwill | 0.5% **(assumption)** | €3,000 **(assumption)** | €15 |
| Recall campaign on the affected lot | 0.01% **(assumption)** | €760,000 | €76 |
| **Total** | | | **about €141, used as €140** |

Where the campaign cost comes from: one analysis puts the all-in cost of a recall at about USD 850 per vehicle, which that paper converts to about EUR 760 (USD 10.3 billion over 12.1 million vehicles, Atlas Copco whitepaper; the underlying data is from a 2016 study, so treat it as an order of magnitude). I assume a 2,000-vehicle lot and a 50% supplier share: 2,000 x €760 x 50% = €760,000.

For context on tail risk, Siemens (2024) reports that an idle large automotive production line can cost up to about USD 2.3 million per hour. A line stoppage at the customer is not in my arithmetic, but it is why I do not treat the tail as negligible.

**This is the least certain number in the report.** It is dominated by the campaign term (about half of the total), whose probability I cannot observe. Section 5 repeats the analysis with the false-negative cost between €20 and €1,500.

### 1.6 Operating assumptions

| Assumption | Value | Because |
|---|---|---|
| Output | 1,500 brackets/day | Three robotic cells at about 500/day each. A 2025 study of a tier-1 robotic welding operation reports daily output above 450 units per machine, so this is the right order of magnitude **(assumption)**. |
| Welds per bracket | 6 **(assumption)** | A typical bracket has a handful of seams. |
| Welds per day | **9,000** (about 9.4 per minute over 16 h) | 1,500 x 6. |
| Base rate of defective welds | **1%** (90 per day) | Vendor articles citing American Welding Society data say calibrated robotic cells run below 1% defective, against 5 to 8% for manual welding. I use 1% as the upper end of that range. **Weak source** (vendor blog). Sensitivity range: 0.1% to 5%. |
| Labour cost | **€25/hour** | INE Quarterly Labour Cost Survey, Q1 2026: €24.88 per effective hour (all sectors). A skilled QA technician may cost more; this is conservative. |
| Seconds per human review | **20 s** **(assumption)** | Looking at one stored weld image on a screen and giving a good/bad call. Cost: 20 s x €25/h = **€0.14 per review**. |
| Reviewer productive time | 6 h per 8 h shift (75%) **(assumption)** | Breaks, handovers, interruptions. Used for headcount in Section 6. |
| Human reviewer catch rate | **85%** | A classic experiment (Carroll, 1969, IIE Transactions) found inspectors missed about 15% of defects. Vendor blogs claim 65 to 85%. **Weak sources.** Sensitivity: 70% to 90%. |
| Human reviewer false-alarm rate | **3%** **(assumption)** | The same paper reports a 3% type I error, measured at lot level, so treat it as indicative only. |

### 1.7 What these numbers imply before any model is trained

These are simple expected-value calculations per weld. Section 5 does the real analysis with the model's actual errors.

1. **The ideal flagging rule.** For a perfectly calibrated model, flag a weld when the probability it is defective is above FP / (FN - TP + FP) = 2 / (140 - 6 + 2) = **about 1.5%**. This is close to the 1% base rate, so only welds that look worse than average should be flagged.
2. **Flagging everything is worse than doing nothing.** Ignoring all defects costs 1% x €140 = **€1.40 per weld**. Flagging everything costs 1% x €6 + 99% x €2 = **€2.04 per weld**. Flag-everything only wins if the false-negative cost is above about €204. This is why the model has to be selective.
3. **Checking everything by hand beats both.** Reviewing every weld costs about **€0.46 per weld** (€0.14 review + rework + false alarms + the 15% of defects the reviewer misses). It needs 9,000 x 20 s = 50 reviewer-hours a day, which is about **8 people** at 6 productive hours each. A model that safely passes the clearly good welds could cut that workload sharply.
4. **A base-rate threshold.** Checking everything by hand costs more than ignoring defects altogether if the true defect rate falls below about **0.17%** (1,700 ppm), at a €140 false-negative cost.

---

## 2. Dataset

*To do (Phase 2).*

- Roboflow workspace / project / version: `TODO`
- Not forked: public dataset used as is.
- Instances per class, train/val/test sizes, object size relative to image: `TODO`
- Mapping of dataset classes to positive/negative: `TODO` (Bad Weld and/or Defect = positive, Good Weld = negative)
- What the stats imply (small classes, duplicates or leakage between splits, test images vs production camera images): `TODO`

## 3. Models and baselines

*To do (Phase 3).*

- Where I trained, hardware, settings (model size, epochs, image size, seed): `TODO`
- Fine-tuned model: `TODO`
- Baseline 1, COCO-pretrained model zero-shot: `TODO`
- Baseline 2, trivial (majority class or predict nothing): `TODO`
- All compared on the same test split.
- Best weights committed to `weights/`.

## 4. Per-class results

*To do (Phase 3).*

- Positive class: defective weld.
- Precision and recall per class, confusion matrix, AP50, mAP50-95: `TODO`
- Which class carries the headline number, and where the model fails: `TODO`

## 5. Cost and threshold

*To do (Phase 4).*

- Costs and base rate from Section 1 (FP €2, TP €6, FN €140, TN €0, base rate 1%).
- Dataset base rate vs real base rate, and how results are reweighted: `TODO`
- Cost vs confidence threshold plot and chosen minimum: `TODO`
- Sensitivity: FN cost €20 to €1,500, base rate 0.1% to 5%: `TODO`

## 6. Human review

*To do (Phase 4).*

- Pass / fail / review thresholds: `TODO`
- Cases per day to review, 20 s each, headcount at 6 productive hours: `TODO`
- Realistic? `TODO`
- Four options priced per day and per 1,000 cases: do nothing, human checks everything, model alone, model plus human review: `TODO`

## 7. Dashboard

*To do (Phase 5).* How to run: `uv run python main.py train` and `uv run python main.py app`.

## 8. Limitations

*To do.* Surface defects only. Public-dataset images versus real line-camera images. Cost figures are assumptions. Human reviewer accuracy is not measured.

---

## Sources

- INE, Encuesta Trimestral de Coste Laboral, Q1 2026 (cost per effective hour €24.88): https://ine.es/dyngs/Prensa/ETCL1T26.pdf
- ISO 17637:2016, Non-destructive testing of welds, visual testing of fusion-welded joints: https://scc-ccn.ca/standardsdb/standards/8159925 (text of the standard: https://weldcalc.ssab.com/sisStandards/ISO%2017637.pdf)
- Frontiers in Robotics and AI (2025), reliability-centred maintenance for tier-1 robotic welding (daily output above 450 units): https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12454091/
- EVS International, robotic welding defect rates (vendor blog, weak source): https://www.evsint.com/robotic-welding-quality-control-ai-vision-ndt-2026/ and https://www.evsint.com/robotic-welding-cell-roi-capex-payback-2026/
- Atlas Copco, Motor vehicle industry: rework costs analysis (recall cost about USD 850 / EUR 760 per vehicle): https://www.atlascopco.com/content/dam/atlas-copco/industrial-technique/products/error-proofing/documents/Whitepaper%20MVI%20rework%20costs%20analysis.pdf
- Plante Moran, warranty claims as a hidden financial risk for auto suppliers (50/50 cost sharing): https://www.plantemoran.com/explore-our-thinking/insight/2014/06/warranty-claims-a-hidden-financial-risk-for-auto-suppliers
- Warranty Week, 4 June 2026 (GM recovered about 90% of Bolt recall cost from LG): https://www.warrantyweek.com/archive/ww20260604.html
- Siemens, The True Cost of Downtime 2024 (automotive about USD 2.3M per hour): https://assets.new.siemens.com/siemens/assets/api/uuid:1b43afb5-2d07-47f7-9eb7-893fe7d0bc59/TCOD-2024_original.pdf
- Carroll, J. M. (1969), Estimating Errors in the Inspection of Complex Products, IIE Transactions 1(3), 229-235: https://ibl.kb.nl/article/0000000020254/1/3/7
