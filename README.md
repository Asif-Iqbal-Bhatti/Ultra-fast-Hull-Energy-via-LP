#  Ultra-Fast Energy-Above-Hull via Linear Programming (HiGHS C++)

> **Optimised for High-Entropy Disordered Rock-Salt (HE-DRX) cathode screening — scales to ≥ 2 million entries on a workstation.**

[![Python](https://img.shields.io/badge/Python-3.10%2B-blue?logo=python)](https://www.python.org/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![HiGHS LP Solver](https://img.shields.io/badge/Solver-HiGHS%20C%2B%2B-orange)](https://highs.dev/)
[![PyMatGen](https://img.shields.io/badge/Uses-PyMatGen-red)](https://pymatgen.org/)

---

## 🌐 Algorithm Schematic

👉 **[View the full interactive pipeline visualization](https://asif-iqbal-bhatti.github.io/Ultra-fast-Hull-Energy-via-LP/)**

---

## 🔬 What is this?

This code computes the **energy above the convex hull (Ehull)** — the key thermodynamic stability metric — for millions of DFT-relaxed high-entropy oxide compositions in the n-element chemical space:

```
Li · Al · Co · Cr · F · Fe · Ga · Mg · Mn · Mo · Nb · Ni · O · Sb · Sn · Ti · V · Zn · Zr
```

Standard PyMatGen tools (`PhaseDiagram`, `PatchedPhaseDiagram`) become prohibitively slow at this scale. This implementation replaces QHull-based convex hull construction with **direct Linear Programming (LP)** via the HiGHS C++ solver — combined with vectorisation, system-level batching, and MPI parallelism.

---

## 🚀 Key Innovations over PyMatGen

| Feature | PyMatGen Standard | This Implementation |
|---|---|---|
| **Ehull method** | QHull convex hull | HiGHS C++ LP solver |
| **Formation energy** | Per-entry Python loop | NumPy BLAS matrix multiply (`epa − X@μ`) |
| **Parallelism granularity** | 1 task = 1 entry | 1 task = 1 chemical system (~100× less overhead) |
| **LP matrix construction** | Rebuilt every entry | Built once per system, reused for all entries |
| **Output format** | CSV | Parquet (10× smaller, 10× faster I/O) |
| **Crash recovery** | None | Incremental resume from existing Parquet |
| **Voltage pass** | Coupled to Ehull | Decoupled optional pass — does not block Ehull |
| **Practical scale** | ~10K entries | ≥ 2 million entries |

### Cumulative Speedup
```
PyMatGen baseline      ████░░░░░░░░░░░░░░░░  1×
+ Vectorised FE        ████████░░░░░░░░░░░░  ~5×
+ System batching      ████████████░░░░░░░░  ~25×
+ HiGHS LP solver      ████████████████░░░░  ~50×
+ 16-CPU MPI           ████████████████████  ~200×
```

---

## 🏗️ Algorithm Pipeline

```
[1] Load & filter reference phases (Materials Project, 19-element space)
        ↓  MaterialsProject2020Compatibility corrections
        ↓  Extract elemental chemical potentials μ[El]

[2] Group target cathodes by chemical system key
        e.g.  Co-Li-Mn-Ni-O-Ti  →  bucket of entries

[3] Pre-compute LP matrices per system  (done ONCE per system)
        A  (k × n_refs)  — composition fraction matrix
        c  (n_refs,)     — formation energies of reference phases
        μ_vec (k,)       — elemental ref energies

[4] Parallel LP solving — 16 CPUs, system-level tasks
        For each target entry i in system:
            b_eq = [x_i ; 1.0]          ← only this changes
            LP:  min  cᵀλ
                 s.t. A·λ = x_i         (composition constraint)
                      1ᵀλ = 1           (convex combination)
                      λ ≥ 0
            Ehull_i = FE_i − LP_optimal

[5] (Optional) Voltage via PatchedPhaseDiagram — separate pass

[6] Write Parquet output + incremental resume support
```

---

## 📦 Installation

```bash
pip install pymatgen scipy numpy pandas tqdm pyarrow monty
```

HiGHS is bundled with `scipy >= 1.9` — no separate install needed.

---

## ⚙️ Configuration

Edit the top of `4_get_ehull_mpi_fast_VER3.py`:

```python
REF_DB_PATH   = 'newAPI_Entries_vs_chemsys_list_CSE_unprocessed.pkl'  # MP reference database
TARGET_JSON   = '../HEDRX_processed.json'   # your DFT entries (ComputedStructureEntry)
OUT_PARQUET   = Path("CATHODE_VOLTAGE_DFT") / "CATHODE_POT_DFT.parquet"
CPUS          = 16          # parallel workers
FE_CUTOFF     = 2.0         # eV/atom — filter unstable refs
COMPUTE_VOLTAGE = False     # set True to also compute cathode voltage
ALLOW_NEGATIVE_EHULL = True # allow negative Ehull (useful for MLIP screening)
```

---

## ▶️ Usage

```bash
python 4_get_ehull_mpi_fast_VER3.py
```

**Output columns in Parquet/CSV:**

| Column | Description |
|---|---|
| `composition` | Pretty formula string |
| `rel_dir` | Source directory (trajectory file parent) |
| `reduced_formula` | Reduced formula |
| `energy` | Total DFT energy (eV) |
| `Ehull_DFT_eV_per_atom` | Energy above hull (eV/atom) |
| `Min_V`, `Max_V`, `Avg_V` | Cathode voltage range (optional) |
| `Ref_uLi0` | Li reference chemical potential (optional) |

---

## 📊 Output Summary (example)

```
============================================================
 Output: CATHODE_VOLTAGE_DFT/CATHODE_POT_DFT.parquet
 Total rows:        2,143,870
 New this run:        215,400
 Ehull range:       -0.0312 – 1.9874 eV/atom
 Stable (<0.05):    128,441  (6.0%)
 Near-stable (<0.10):  241,803  (11.3%)
```

---

## 🧪 Application: HE-DRX Cathode Design

This tool is part of a high-throughput DFT + MLIP workflow to identify thermodynamically stable **High-Entropy Disordered Rock-Salt (HE-DRX)** cathode materials containing 5+ transition metals. The 19-element design space yields a combinatorial explosion of compositions — making fast Ehull screening the critical bottleneck that this code solves.

---

## 📄 Citation

If you use this code in your research, please cite:

```bibtex
@software{bhatti2025ultrafast,
  author    = {Asif Iqbal Bhatti},
  title     = {Ultra-Fast Energy-Above-Hull via LP (HiGHS C++) for HE-DRX Cathode Screening},
  year      = {2025},
  url       = {https://github.com/Asif-Iqbal-Bhatti/Ultra-fast-Hull-Energy-via-LP},
}
```

---

High-Entropy DRX Cathode Design Space Calculation

---

## 📜 License

MIT License — see [LICENSE](LICENSE) for details.
