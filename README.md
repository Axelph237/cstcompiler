# Delphi

Delphi is a quantum compiler for **High-Level Oracle Synthesis** — a synthesis pipeline that takes a high-level specification of a search problem and produces an optimized quantum oracle circuit.

The intended workflow is:

1. A user describes search-space constraints in natural language.
2. An LLM assistant refines these into a formal set of clauses (a SAT-style CNF formula).
3. The compiler maps those clauses onto an efficient quantum oracle using a hierarchy of ancilla-qubit modules called an **HRSE tree**.

## Why it matters

Quantum search algorithms (Grover's, QAOA variants) require an oracle that marks solutions to a search problem. Building that oracle naively wastes ancilla qubits and gate depth. Delphi automates the structural optimization — choosing how to group and schedule clauses so that qubits can be reused — making otherwise impractical oracle sizes feasible.

## Architecture

```
natural language
       │  LLM
       ▼
   CNF clauses
       │  compiler
       ├── HRSE tree synthesis   (backbone.py)
       ├── CST construction      (synthesis.py)
       └── oracle mapping        (clause_pack.py, Qiskit)
```

### HRSE Tree

A **Hierarchical Recursive Synthesis-Evaluation (HRSE) tree** is the structural blueprint for the oracle. Each node represents a module that uses a fixed number of ancilla qubits, and child nodes are strictly smaller, which lets qubits be reused across levels.

Trees are synthesized by the **ASDT algorithm**, which produces an optimal tree for `m` clauses within a budget of `k` ancilla qubits. The maximum clause capacity for a given `k` is `ceil(3 × 2^(k−4))`, so the ancilla budget bounds how large a formula can be compiled.

### CST

A **Clustered Synthesis Tree (CST)** mirrors the HRSE tree and assigns a *partition* of clause batches to each node. Batches are built greedily by the **SeedGrow heuristic**, which fills the nodes with the most ancilla qubits first and groups the most-conflicted clauses together, so clauses that share variables are evaluated in parallel wherever the budget allows.

## Getting started

`cstcompiler` is a standalone [uv](https://docs.astral.sh/uv/) project. The
companion package `delphi-interface` turns natural language into the CNF clauses
this compiler consumes, and depends on this package.

```bash
# Create .venv and install the package in editable mode
uv sync

# Run the test suite
uv run pytest

# Run performance benchmarks
uv run python -m tests.benchmark_cst

# Compare oracle Clifford+T depth with the LTC paper's Table III
uv run python -m tests.benchmark_oracle
```

### Building an oracle

`CST` is the entry point, re-exported from the package root. Each constructor
takes an ancilla budget and returns a compiled tree, and `to_oracle` maps it to
a Qiskit circuit.

```python
from cstcompiler import CST

# (x ∨ ¬y) ∧ (y ∨ z) ∧ (¬z)
cst = CST.from_named_literals(
    [["x", "~y"], ["y", "z"], ["~z"]],
    ancilla_budget=8,
)

circuit, x_register, output_register = cst.to_oracle()
```

The circuit maps `|x⟩|c⟩|0⟩ → |x⟩|c ⊕ f(x)⟩|0⟩`: `x_register` holds the input
qubits, the single qubit in `output_register` is toggled when the formula is
satisfied, and every other ancilla is restored to `|0⟩`.

A leading `~` or `-` negates a literal, and repeated prefixes cancel in pairs.
`cst.variable_mapping` records the name each variable was assigned.

The same formula can come from a DIMACS document, where `c var` comments name
the variables, or from `Clause` objects directly.

```python
from cstcompiler import CST
from cstcompiler.synthesis import Clause

cst = CST.from_dimacs(dimacs_text, ancilla_budget=8)
cst = CST.from_Clauses([Clause.from_literals((1, -2))], ancilla_budget=8)
```

## References

- [Modeling and Resource Optimization for Quantum Oracles (2026)](https://arxiv.org/html/2605.21380v1)
- [From Leaves to Clusters: Depth-Efficient SAT-Oracle Synthesis Based on the HRSE Model (2026)](https://arxiv.org/pdf/2607.11401)
