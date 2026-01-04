# Enhanced Progenitor Glycosylation Scanner

[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Validation: FDA Therapeutics](https://img.shields.io/badge/Validated-18%20FDA%20Antibodies-green.svg)](https://github.com)
[![Version](https://img.shields.io/badge/version-3.0-blue.svg)](https://github.com)

**A computational tool for predicting Fab glycosylation risks in therapeutic antibodies, designed to complement ML design tools like RFdiffusion that don't screen for post-translational modification liabilities.**

---

## The Problem

Tools like **RFdiffusion** and other ML-based antibody design platforms are revolutionizing *de novo* antibody generation. They produce novel sequences with remarkable speed and structural accuracy. However, these tools optimize for structure and predicted binding—they are blind to **post-translational modification liabilities**, including N-linked glycosylation in the Fab region.

(Note: While such tools can place glycosylation sites by design, they won't warn you when N-X-S/T motifs arise as unintended byproducts of sequence generation.)

**Why Fab glycosylation matters:**
- Alters antigen binding affinity and specificity
- Introduces batch-to-batch heterogeneity in manufacturing
- Can trigger immunogenic responses (e.g., Cetuximab's α-Gal epitope)
- Complicates analytical characterization and regulatory approval

By the time these issues surface in development, significant resources have already been invested.

---

## The Insight: Progenitor Sites

van de Bovenkamp *et al.* (2018, 2023) demonstrated that **79-86% of Fab N-glycosylation sites in natural antibody repertoires originate from "progenitor" positions**—sequences where a single somatic hypermutation event converts a latent motif into an actual N-X-S/T sequon.

```
Example:
  Current sequence:  D - G - S  (not glycosylated)
                     ↓ (one mutation: GAT→AAT)
  After mutation:    N - G - S  (glycosylated!)
```

This means scanning only for existing N-X-S/T motifs misses the majority of glycosylation risk. In the context of therapeutic development—or ML-generated sequences that haven't been through evolutionary selection—these progenitor sites represent **latent liabilities**.

---

## Our Approach

The **Enhanced Progenitor Glycosylation Scanner** operationalizes this insight:

| Component | Description |
|-----------|-------------|
| **Actual site detection** | N-X-S/T motifs — immediate, high-weight risk (25-45 points) |
| **Progenitor site detection** | D-X-S/T (D→N) and N-X-[A/V/I/L] (→S/T) — weighted by mutation probability (2-10 points) |
| **X-position efficiency** | Shakin-Eshleman scoring — not all sequons are equal |
| **NXT vs NXS differentiation** | Threonine-containing motifs ~2.5× more efficient |
| **Structural context** | CDR sites affect binding; Vernier zone (IMGT 75-88) affects VH/VL packing |

The result is a **risk score (0-100)** that prioritizes sequences for review before they enter the development pipeline.

### Vernier Zone Flagging

The **Vernier zone** (IMGT positions 75-88) deserves special attention. These framework residues sit at the VH/VL interface and support CDR loop conformations. Glycosylation here may not block antigen binding directly, but can alter domain stability, aggregation propensity, and expression yields—liabilities that surface late in development when they're expensive to fix.

---

## v3.0 Scoring Calibration

The scoring is deliberately **asymmetric** to ensure antibodies with actual sites score distinctly higher than those with only progenitor risks:

| Site Type | Score Range | Rationale |
|-----------|-------------|-----------|
| Actual CDR | 30-45 pts | Worst case — glycan in binding site |
| Actual Vernier | 27-40 pts | Affects VH/VL packing |
| Actual Framework | 25-37 pts | Still affects stability/immunogenicity |
| Progenitor D→N (CDR) | 6-10 pts | ~16% SHM conversion probability, bad location |
| Progenitor D→N (FR) | 3-5 pts | SHM hotspot, tolerable location |
| Progenitor →S/T | 2-7 pts | Lower conversion probability |

**Risk thresholds:**
- **HIGH (≥50):** Actual N-X-S/T sites present → Immediate review
- **MEDIUM (25-50):** Elevated progenitor risk → Monitor during development  
- **LOW (<25):** Baseline progenitor load → Standard QC

---

## Validation Results

### Dataset: 18 FDA-Approved Therapeutic Antibodies

| Antibody | Target | Risk Level | Score | Actual Sites | Progenitors |
|----------|--------|------------|-------|--------------|-------------|
| **Cetuximab** | EGFR | **HIGH** | **67.7** | **2** | 3 |
| Tocilizumab | IL-6R | MEDIUM | 32.5 | 0 | 8 |
| Pertuzumab | HER2 | MEDIUM | 29.2 | 0 | 8 |
| Bevacizumab | VEGF-A | LOW | 24.8 | 0 | 6 |
| Panitumumab | EGFR | LOW | 21.1 | 0 | 5 |
| Alemtuzumab | CD52 | LOW | 20.8 | 0 | 6 |
| Atezolizumab | PD-L1 | LOW | 19.6 | 0 | 5 |
| Nivolumab | PD-1 | LOW | 19.4 | 0 | 6 |
| Trastuzumab | HER2 | LOW | 18.0 | 0 | 5 |
| Pembrolizumab | PD-1 | LOW | 16.8 | 0 | 6 |
| Infliximab | TNF-α | LOW | 16.7 | 0 | 5 |
| Adalimumab | TNF-α | LOW | 16.6 | 0 | 5 |
| Ipilimumab | CTLA-4 | LOW | 16.3 | 0 | 5 |
| Palivizumab | RSV | LOW | 14.1 | 0 | 4 |
| Ustekinumab | IL-12/23 | LOW | 13.9 | 0 | 4 |
| Durvalumab | PD-L1 | LOW | 13.2 | 0 | 4 |
| Denosumab | RANKL | LOW | 13.1 | 0 | 4 |
| Rituximab | CD20 | LOW | 12.6 | 0 | 3 |

### Performance Metrics

| Metric | Value |
|--------|-------|
| **ROC AUC** | 1.000 |
| **Sensitivity** | 1.000 |
| **Specificity** | 1.000 |
| **Accuracy** | 1.000 |

**Ground truth:** Only **Cetuximab** has documented Fab glycosylation (N88 in VH FR3). The remaining 17 antibodies lack N-X-S/T sequons in their Fab regions.

The scanner correctly:
- ✅ Identifies Cetuximab as the sole HIGH-risk antibody (score 67.7)
- ✅ Clears all 17 non-glycosylated antibodies (scores 12.6-32.5)
- ✅ Provides clear separation between classes (35+ point gap)

### Why Perfect Specificity?

FDA-approved therapeutics are **curated survivors**. They've been through:

1. **Humanization** — problematic sequences removed
2. **Developability screening** — glycosylation risks flagged early
3. **Lead optimization** — liabilities engineered out
4. **Manufacturing optimization** — clones with clean glycoprofiles selected

The 17 "non-glycosylated" antibodies aren't random sequences—they're winners where progenitor sites were mutated out during development. **Cetuximab** is the exception where the glycosylation site was functionally constrained or clinically acceptable.

This is why the scanner's real value is in **screening unselected sequences**: RFdiffusion outputs, early-stage candidates, and natural repertoire antibodies.

---

## Limitations

- **Small positive class (n=1):** Only Cetuximab has documented Fab glycosylation in this panel. Broader validation against natural antibody repertoire data would strengthen confidence.
- **Survivor bias:** FDA-approved therapeutics have been pre-screened for developability. This inflates specificity—the scanner's true value is catching liabilities in *unscreened* sequences.
- **Occupancy ≠ certainty:** The scanner predicts *risk*, not guaranteed glycosylation. Actual occupancy depends on cell line, culture conditions, and stochastic factors.

---

## Quick Start

### Installation

```bash
# Clone repository
git clone https://github.com/christophergaughan/progenitor-glycosylation-scanner.git
cd progenitor-glycosylation-scanner

# Install dependencies
pip install -r requirements.txt
```

### Requirements

```
numpy>=1.21.0
pandas>=1.3.0
matplotlib>=3.4.0
seaborn>=0.11.0
scikit-learn>=0.24.0
antpack==0.3.8.6
```

### Basic Usage

```python
from scanner import EnhancedGlycosylationScannerV3

# Initialize scanner
scanner = EnhancedGlycosylationScannerV3()

# Analyze an antibody
result = scanner.scan_antibody(
    heavy_seq='EVQLVESGGGLVQPGGSLRLSCAASGFNIKDTYIH...',
    light_seq='DIQMTQSPSSLSASVGDRVTITCRASQDVNTAVA...'
)

# View results
print(f"Risk Level: {result['overall_risk_level']}")
print(f"Risk Score: {result['overall_risk_score']:.1f}")
print(f"Actual Sites: {result['total_actual']}")
print(f"Progenitor Sites: {result['total_progenitor']}")
```

### Google Colab

The validation notebook runs directly in Google Colab with automatic dependency installation.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/christophergaughan/progenitor-glycosylation-scanner/blob/main/fda_therapeutics_validation.ipynb)

---

## Use Cases

### 1. ML Design Pipeline Integration

```python
# Screen RFdiffusion designs before synthesis
scanner = EnhancedGlycosylationScannerV3()

approved_designs = []
for design in rfdiffusion_outputs:
    result = scanner.scan_antibody(design['heavy'], design['light'])
    if result['overall_risk_score'] < 25:  # Low risk threshold
        approved_designs.append(design)
    elif result['total_actual'] > 0:
        print(f"⚠️ {design['name']}: Actual N-X-S/T site detected!")
```

### 2. Lead Optimization

```python
# Compare variants during affinity maturation
variants = load_variants()
results = {v['name']: scanner.scan_antibody(v['heavy'], v['light']) 
           for v in variants}

# Flag any variants that acquired new sites
for name, result in results.items():
    if result['total_actual'] > parent_result['total_actual']:
        print(f"⚠️ {name}: New glycosylation site introduced!")
```

### 3. Repertoire Screening

```python
# Screen natural antibody sequences for progenitor load
for antibody in repertoire:
    result = scanner.scan_antibody(antibody['vh'], antibody['vl'])
    antibody['glyc_risk_score'] = result['overall_risk_score']
    antibody['progenitor_count'] = result['total_progenitor']

# Analyze distribution
high_risk = [ab for ab in repertoire if ab['glyc_risk_score'] >= 50]
print(f"High-risk antibodies: {len(high_risk)}/{len(repertoire)}")
```

---

## Scientific Basis

### Primary References

| Reference | Contribution |
|-----------|--------------|
| van de Bovenkamp *et al.* (2018) PNAS 115:1901-1906 | 79-86% of Fab glycosylation from progenitor sites |
| van de Bovenkamp *et al.* (2023) | Extended progenitor analysis |
| Shakin-Eshleman *et al.* (1996) JBC 271:6363-6366 | X-position efficiency scoring |
| Kasturi *et al.* (1995) JBC 270:14756-14761 | NXT vs NXS efficiency (~2.5×) |

### X-Position Efficiency (Shakin-Eshleman)

| Efficiency | X-Residues |
|------------|------------|
| **Blocking (0.00)** | Pro |
| **Low (0.25-0.30)** | Trp, Asp, Glu |
| **Medium (0.40-0.65)** | Arg, His, Lys, Tyr, Asn, Gln, Cys, Phe, Leu, Ile |
| **High (0.72-0.90)** | Met, Val, Gly, Ala, Thr, Ser |

---

## Cell Line Independence

The scanner predicts **sequon presence** and **occupancy probability** based on oligosaccharyltransferase (OST) recognition, which is conserved across mammalian systems.

A flagged site will likely be glycosylated regardless of cell line. However, the **glycan structure** (and thus immunogenicity) varies:

| Cell Line | α-Gal | NGNA | Immunogenicity Risk |
|-----------|-------|------|---------------------|
| CHO | No | No | Lower (industry standard) |
| HEK293 | No | No | Lowest |
| SP2/0 | Yes | Yes | Higher |
| NS0 | Yes | Yes | Higher |

**Example:** Cetuximab's Fab glycan is occupied in all systems, but only SP2/0-produced cetuximab carries the α-Gal epitope responsible for anaphylaxis in tick-bite sensitized patients.

---

## Roadmap

### Current (v3.0)
- [x] Progenitor site detection (D→N, →S/T)
- [x] X-position efficiency scoring
- [x] NXT vs NXS differentiation
- [x] Vernier zone flagging
- [x] AntPack IMGT numbering
- [x] FDA validation notebook

### Planned (v3.1)
- [ ] REST API for pipeline integration
- [ ] Expanded validation (natural repertoire data)
- [ ] Batch processing utilities
- [ ] Integration examples for RFdiffusion

### Future (v4.0)
- [ ] Structural accessibility scoring (DSSP/SASA)
- [ ] AlphaFold3 structure integration
- [ ] Web interface

---

## License

MIT License — see [LICENSE](LICENSE) for details.

**Commercial use permitted.** For consulting or custom development:
- **Company:** AntibodyML Consulting LLC
- **Contact:** clgaughan@proton.me

---

## Citation

```bibtex
@software{progenitor_glycosylation_scanner,
  author = {Gaughan, Christopher},
  title = {Enhanced Progenitor Glycosylation Scanner},
  year = {2025},
  version = {3.0},
  url = {https://github.com/christophergaughan/progenitor-glycosylation-scanner}
}
```

**Please also cite the foundational work:**
- van de Bovenkamp FS, *et al.* (2018) PNAS 115:1901-1906
- Shakin-Eshleman SH, *et al.* (1996) JBC 271:6363-6366

---

## Contact

- **GitHub Issues:** [Report bugs or request features](https://github.com/christophergaughan/progenitor-glycosylation-scanner/issues)
- **Email:** clgaughan@proton.me
- **LinkedIn:** [Christopher Gaughan](https://www.linkedin.com/in/gaughanchristopher/)
- **Company:** AntibodyML Consulting LLC

---

**Built with:** Python, AntPack, NumPy, Pandas, scikit-learn, Matplotlib, Seaborn

**Designed for:** Computational biologists, antibody engineers, ML antibody design pipelines

**Fills the gap** that ML design tools leave open — because structure prediction doesn't catch post-translational modification liabilities.
