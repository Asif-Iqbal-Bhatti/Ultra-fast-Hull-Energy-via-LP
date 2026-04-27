# Ultra-fast-Hull-Energy-via-LP

PyMatGen rebuilds a full phase diagram per entry (Python loops + QHull). This implementation builds the LP matrix once per chemical system, solves with the HiGHS C++ LP solver (not QHull), uses NumPy BLAS matrix multiply for formation energies, dispatches whole systems to 16 parallel workers, and writes Parquet output — making 2 million Ehull calculations practical in minutes instead of days.
