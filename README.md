# Delphi

Delphi is a quantum compiler for High-Level Oracle Synthesis. It takes a high-level specification of a search problem and produces an optimized quantum oracle circuit.

The intended workflow is:

1. A user describes search-space constraints in natural language.
2. An LLM assistant refines these into a formal set of clauses (a SAT-style CNF formula).
3. The compiler maps those clauses onto an efficient quantum oracle using a hierarchy of ancilla-qubit modules called an HRSE tree.

## Why it matters

Quantum search algorithms (Grover's, QAOA variants) require an oracle that marks solutions to a search problem. The obvious construction wastes ancilla qubits and gate depth. Delphi chooses how to group and schedule clauses so that qubits can be reused, which cuts circuit depth. `tests/benchmark_oracle.py` measures the depth this compiler produces against the published figures in the second reference below.

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

### HRSE tree

A **Hierarchical Recursive Synthesis-Evaluation (HRSE) tree** defines the oracle's structure. Each node is a module that uses a fixed number of ancilla qubits. Child nodes are strictly smaller, so a parent reuses the qubits its children release.

The ASDT algorithm builds the tree, optimally for `m` clauses within a budget of `k` ancilla qubits. A budget of `k` holds at most `ceil(3 × 2^(k−4))` clauses, so the budget caps how large a formula you can compile.

### CST

A **Clustered Synthesis Tree (CST)** mirrors the HRSE tree and assigns a *partition* of clause batches to each node. The SeedGrow heuristic builds those batches. It fills the nodes with the most ancilla qubits first and groups the most-conflicted clauses together, so the oracle evaluates clauses that share variables in parallel wherever the budget allows.

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

`CST` is the entry point, and the package root re-exports it. Each constructor
takes an ancilla budget and returns a compiled tree, and `to_oracle` maps that
tree to a Qiskit circuit.

```python
from cstcompiler import CST

# (x ∨ ¬y) ∧ (y ∨ z) ∧ (¬z)
cst = CST.from_named_literals(
    [["x", "~y"], ["y", "z"], ["~z"]],
    ancilla_budget=8,
)

circuit, x_register, output_register = cst.to_oracle()
```

The circuit maps `|x⟩|c⟩|0⟩` to `|x⟩|c ⊕ f(x)⟩|0⟩`. `x_register` holds the input
qubits. The circuit toggles the single qubit in `output_register` when the
assignment satisfies the formula, and restores every other ancilla to `|0⟩`.

A leading `~` or `-` negates a literal, and repeated prefixes cancel in pairs.
`cst.variable_mapping` records the id it assigned to each name.

`from_dimacs` reads a DIMACS document, where `c var` comments name the
variables. `from_Clauses` takes `Clause` objects directly.

```python
from cstcompiler import CST
from cstcompiler.synthesis import Clause

cst = CST.from_dimacs(dimacs_text, ancilla_budget=8)
cst = CST.from_Clauses([Clause.from_literals((1, -2))], ancilla_budget=8)
```

## References

- [Modeling and Resource Optimization for Quantum Oracles (2026)](https://arxiv.org/html/2605.21380v1)
- [From Leaves to Clusters: Depth-Efficient SAT-Oracle Synthesis Based on the HRSE Model (2026)](https://arxiv.org/pdf/2607.11401)
