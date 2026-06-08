# INTEGRATION.md — hodge-consensus-rs × sheaf-coherence-rs × entropy-conservation-rs

**Hodge consensus** decomposes agent disagreements into gradient
(hierarchical), curl (cyclic), and harmonic (irreconcilable) components.
It connects to sheaf coherence for local-to-global consistency and to
entropy conservation for information-loss analysis during consensus.

## Synergy Map

```
sheaf-coherence-rs            hodge-consensus-rs             entropy-conservation-rs
┌──────────────────┐          ┌──────────────────────┐       ┌─────────────────────┐
│ Sheaf             │◄────────►│ HodgeConsensus       │◄─────►│ HodgeComponents     │
│ RestrictionMap    │          │ decompose_disagree   │       │ decompose           │
│ coherence_score   │          │ consensus_gap        │       │ gradient_energy     │
│ diffusion_sync    │          │ consensus_vote       │       │ harmonic_energy     │
└──────────────────┘          │ disagreement_matrix  │       └─────────────────────┘
                              └──────────────────────┘
```

## Key Insight

When agents vote or estimate parameters, their pairwise disagreements form
a matrix. Hodge decomposition reveals whether disagreement is purely
hierarchical (one leader is right, followers are wrong), cyclic (agents
form cliques with no global winner), or harmonic (genuinely irreconcilable
differences). Sheaf coherence fixes cyclic disagreements via restriction
maps; entropy conservation quantifies information lost in the process.

## Example 1: Diagnose Why Consensus Failed

Decompose a disagreement matrix to find the source of dissent.

```rust
use hodge_consensus::{disagreement_matrix, decompose_disagreement, consensus_vote};

fn diagnose_consensus_failure(opinions: &[f64]) {
    let d = disagreement_matrix(opinions);
    let hodge = decompose_disagreement(&d);

    println!("Consensus gap: {:.4}", hodge.consensus_gap());
    println!("Gradient (hierarchical): present");
    println!("Curl (cyclic): present");
    println!("Harmonic (irreconcilable): present");

    if hodge.is_consensus(1e-6) {
        println!("Agents already agree.");
    } else if hodge.consensus_gap() > 5.0 {
        println!("Genuine disagreement — consensus impossible without new info.");
    } else {
        let vote = consensus_vote(opinions, &vec![1.0; opinions.len()]);
        println!("Simple majority vote: {:.2}", vote);
    }
}
```

## Example 2: Sheaf-Based Consensus Repair

Use sheaf coherence to synchronize local estimates, then verify consensus.

```rust
use sheaf_coherence::sheaf::{Sheaf, Cell, RestrictionMap, Assignment};
use sheaf_coherence::synchronization::diffusion_synchronize;
use hodge_consensus::{disagreement_matrix, decompose_disagreement};

fn repair_with_sheaf(local_estimates: &[f64]) {
    let mut sheaf = Sheaf::new();
    for (i, &est) in local_estimates.iter().enumerate() {
        sheaf.add_cell(Cell::new(i, 0), 1);
        sheaf.assign(Assignment::new(i, vec![est]));
    }
    // Identity restriction maps between all pairs
    for i in 0..local_estimates.len() {
        for j in (i+1)..local_estimates.len() {
            sheaf.add_restriction_map(RestrictionMap::new(i, j, vec![vec![1.0]]));
        }
    }

    diffusion_synchronize(&mut sheaf, 0.1, 100);

    let global = sheaf.global_assignment();
    let d = disagreement_matrix(&global);
    let hodge = decompose_disagreement(&d);
    println!("Post-sheaf consensus gap: {:.6}", hodge.consensus_gap());
}
```

## Example 3: Entropy Analysis of Opinion Dynamics

Track how information content changes during consensus formation.

```rust
use entropy_conservation::hodge_decomposition::decompose;
use hodge_consensus::disagreement_matrix;

fn entropy_of_opinions(opinions: &[f64]) {
    let d = disagreement_matrix(opinions);
    let hodge = decompose(&d);

    println!("Gradient energy: {:.4}", hodge.gradient_energy());
    println!("Curl energy: {:.4}", hodge.curl_energy());
    println!("Harmonic energy: {:.4}", hodge.harmonic_energy());

    let total = hodge.reconstruct().len() as f64;
    let info_lost = hodge.harmonic_energy() / total;
    println!("Information lost to irreconcilable disagreement: {:.2}%", info_lost * 100.0);
}
```

## Cargo.toml Wiring

```toml
[dependencies]
hodge-consensus = { git = "https://github.com/SuperInstance/hodge-consensus-rs" }
sheaf-coherence = { git = "https://github.com/SuperInstance/sheaf-coherence-rs" }
entropy-conservation = { git = "https://github.com/SuperInstance/entropy-conservation-rs" }
```

## Design Patterns

### Pattern: Iterative Consensus with Convergence Check

Run rounds of negotiation, decomposing disagreement after each round
until consensus gap falls below threshold:

```rust
use hodge_consensus::{disagreement_matrix, decompose_disagreement, consensus_vote};

fn iterative_consensus(opinions: &mut Vec<f64>, max_rounds: usize) -> bool {
    for round in 0..max_rounds {
        let d = disagreement_matrix(opinions);
        let hodge = decompose_disagreement(&d);
        if hodge.is_consensus(1e-4) {
            println!("Consensus reached in {} rounds", round);
            return true;
        }
        let vote = consensus_vote(opinions, &vec![1.0; opinions.len()]);
        for o in opinions.iter_mut() { *o = (*o + vote) / 2.0; }
    }
    false
}
```

### Pattern: Weighted Hodge Decomposition

Apply per-agent credibility weights before decomposing:

```rust
use hodge_consensus::disagreement_matrix;

fn weighted_decompose(opinions: &[f64], weights: &[f64]) -> (Vec<f64>, Vec<f64>, Vec<f64>) {
    let weighted: Vec<f64> = opinions.iter().zip(weights)
        .map(|(o, w)| o * w).collect();
    let d = disagreement_matrix(&weighted);
    decompose_disagreement(&d)
}
```
