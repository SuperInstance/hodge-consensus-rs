# hodge-consensus-rs

Hodge decomposition of agent disagreements. Decomposes pairwise disagreement matrices into gradient (hierarchical), curl (cyclic), and harmonic (irreconcilable) components to predict which disputes will resolve and which won't.

## What It Does

When agents in a fleet disagree — about priorities, resource allocation, or any scalar parameter — their pairwise disagreements form a matrix. Hodge decomposition splits that matrix into three components:

- **Gradient**: Hierarchical disagreement. One agent is "more right" than another. This resolves naturally by following the gradient.
- **Curl**: Cyclic disagreement. Agent A > B > C > A. No global ranking exists, but local fixes can break the cycle.
- **Harmonic**: Irreconcilable disagreement. No amount of pairwise adjustment resolves it. Requires new information or authority intervention.

The key insight: **if most disagreement energy is in the gradient component, consensus is reachable. If it's in the harmonic component, consensus requires external input.**

## Quick Start

```toml
[dependencies]
hodge-consensus = { git = "https://github.com/SuperInstance/hodge-consensus-rs" }
```

```rust
use hodge_consensus::{disagreement_matrix, decompose_disagreement, consensus_vote};

fn main() {
    // 4 agents rate a parameter. Near-consensus with one outlier.
    let opinions = vec![10.0, 10.5, 9.8, 50.0];

    let d = disagreement_matrix(&opinions);
    let hodge = decompose_disagreement(&d);

    println!("Consensus gap: {:.4}", hodge.consensus_gap());
    println!("Gradient energy: {:.4}", hodge.gradient_energy());
    println!("Curl energy: {:.4}", hodge.curl_energy());
    println!("Harmonic energy: {:.4}", hodge.harmonic_energy());

    if hodge.is_consensus(1e-4) {
        println!("Agents agree — no action needed.");
    } else {
        let vote = consensus_vote(&opinions, &vec![1.0; opinions.len()]);
        println!("Weighted vote: {:.2}", vote);
    }
}
```

## Disagreement Matrix

Given N agents with scalar opinions `x_1, ..., x_N`, the disagreement matrix `D` is:

```
D[i][j] = x_i - x_j
```

```rust
use hodge_consensus::disagreement_matrix;

let opinions = vec![5.0, 3.0, 8.0];
let d = disagreement_matrix(&opinions);
// d[0][1] = 5 - 3 = 2
// d[1][0] = 3 - 5 = -2
// d[0][2] = 5 - 8 = -3
```

The matrix is skew-symmetric: `D[i][j] = -D[j][i]`, and `D[i][i] = 0`.

## Hodge Decomposition

The `decompose_disagreement` function takes the disagreement matrix and returns a `HodgeComponents` struct with three orthogonal components.

```rust
use hodge_consensus::{disagreement_matrix, decompose_disagreement};

let opinions = vec![1.0, 2.0, 3.0, 4.0];
let d = disagreement_matrix(&opinions);
let hodge = decompose_disagreement(&d);

// Access components
println!("Gradient energy: {:.4}", hodge.gradient_energy());
println!("Curl energy: {:.4}", hodge.curl_energy());
println!("Harmonic energy: {:.4}", hodge.harmonic_energy());
```

### Reading the Components

| Component | Meaning | Resolution Strategy |
|-----------|---------|-------------------|
| Gradient high | Agents disagree hierarchically | Follow the gradient — rank agents |
| Curl high | Agents disagree cyclically | Break cycles with tie-breaking rules |
| Harmonic high | Agents disagree irreconcilably | Need external authority or new info |

### Consensus Gap

```rust
let gap = hodge.consensus_gap();
// gap close to 0 → agents agree
// gap large → significant disagreement

if hodge.is_consensus(0.01) {
    println!("Close enough to consensus");
}
```

## Consensus Vote

Compute a weighted majority vote across agent opinions.

```rust
use hodge_consensus::consensus_vote;

let opinions = vec![10.0, 11.0, 9.5, 50.0];
let weights = vec![1.0, 1.0, 1.0, 0.1]; // downweight the outlier

let vote = consensus_vote(&opinions, &weights);
println!("Weighted consensus: {:.2}", vote);
```

With equal weights, this gives a simple average. With credibility weights (from past accuracy, agent rank, etc.), it gives a trust-weighted estimate.

## Predicting Which Disputes Resolve

The ratio of gradient to total energy predicts resolvability:

```rust
use hodge_consensus::{disagreement_matrix, decompose_disagreement};

fn predict_resolvability(opinions: &[f64]) -> &'static str {
    let d = disagreement_matrix(opinions);
    let hodge = decompose_disagreement(&d);

    let total = hodge.gradient_energy() + hodge.curl_energy() + hodge.harmonic_energy();
    if total < 1e-10 {
        return "Already in consensus";
    }

    let gradient_ratio = hodge.gradient_energy() / total;
    let harmonic_ratio = hodge.harmonic_energy() / total;

    if harmonic_ratio > 0.5 {
        "Unresolvable — need external authority"
    } else if gradient_ratio > 0.7 {
        "Resolvable — gradient dominance means ranking exists"
    } else {
        "Partially resolvable — break cycles first"
    }
}

fn main() {
    // Near-consensus: mostly gradient
    let a = predict_resolvability(&[10.0, 10.5, 9.8, 10.2]);
    println!("Case A: {}", a); // Resolvable

    // Cyclic: A > B > C > A
    let b = predict_resolvability(&[1.0, 5.0, 3.0]);
    println!("Case B: {}", b);

    // Divergent: no pattern
    let c = predict_resolvability(&[1.0, 100.0, -50.0, 0.001]);
    println!("Case C: {}", c);
}
```

## Iterative Consensus with Convergence Check

Run rounds of opinion adjustment, checking Hodge decomposition after each round:

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

        println!("Round {}: gap={:.4}, gradient={:.4}, harmonic={:.4}",
            round,
            hodge.consensus_gap(),
            hodge.gradient_energy(),
            hodge.harmonic_energy(),
        );

        // Move each opinion toward the weighted vote
        let vote = consensus_vote(opinions, &vec![1.0; opinions.len()]);
        for o in opinions.iter_mut() {
            *o = (*o + vote) / 2.0;
        }
    }

    println!("No consensus after {} rounds", max_rounds);
    false
}

fn main() {
    let mut opinions = vec![1.0, 2.0, 3.0, 10.0];
    let reached = iterative_consensus(&mut opinions, 50);
    println!("Final opinions: {:?}", opinions);
}
```

## Weighted Hodge Decomposition

Apply per-agent credibility weights before decomposition:

```rust
use hodge_consensus::{disagreement_matrix, decompose_disagreement};

fn weighted_decompose(opinions: &[f64], weights: &[f64]) {
    let weighted: Vec<f64> = opinions.iter().zip(weights)
        .map(|(o, w)| o * w).collect();
    let d = disagreement_matrix(&weighted);
    let hodge = decompose_disagreement(&d);

    println!("Weighted gradient: {:.4}", hodge.gradient_energy());
    println!("Weighted harmonic: {:.4}", hodge.harmonic_energy());
}
```

## Integration with sheaf-coherence-rs

When cyclic disagreement (curl) is high, sheaf coherence can repair local estimates via diffusion synchronization, then verify consensus with Hodge decomposition.

```rust
use hodge_consensus::{disagreement_matrix, decompose_disagreement};

fn repair_cyclic_disagreement(local_estimates: &[f64]) {
    let d = disagreement_matrix(local_estimates);
    let hodge = decompose_disagreement(&d);

    if hodge.curl_energy() > hodge.gradient_energy() {
        println!("Cyclic disagreement detected — applying sheaf diffusion");

        // sheaf_coherence::synchronization::diffusion_synchronize would go here
        // After repair:
        let repaired = vec![5.0, 5.1, 4.9]; // example post-diffusion estimates
        let d2 = disagreement_matrix(&repaired);
        let hodge2 = decompose_disagreement(&d2);
        println!("Post-repair gap: {:.6}", hodge2.consensus_gap());
    }
}
```

## Integration with entropy-conservation-rs

Track information loss during consensus formation. Harmonic energy represents information that cannot be recovered through pairwise negotiation.

```rust
use hodge_consensus::{disagreement_matrix, decompose_disagreement};

fn entropy_analysis(opinions: &[f64]) {
    let d = disagreement_matrix(opinions);
    let hodge = decompose_disagreement(&d);

    let total_energy = hodge.gradient_energy()
        + hodge.curl_energy()
        + hodge.harmonic_energy();

    if total_energy > 0.0 {
        let info_lost_pct = hodge.harmonic_energy() / total_energy * 100.0;
        println!("Information lost to irreconcilable disagreement: {:.1}%", info_lost_pct);

        let recoverable_pct = hodge.gradient_energy() / total_energy * 100.0;
        println!("Recoverable through ranking: {:.1}%", recoverable_pct);
    }
}
```

## Fleet-Wide Disagreement Diagnosis

Analyze disagreements across an entire fleet:

```rust
use hodge_consensus::{disagreement_matrix, decompose_disagreement, consensus_vote};

struct FleetDiagnosis {
    consensus_gap: f64,
    gradient_ratio: f64,
    curl_ratio: f64,
    harmonic_ratio: f64,
    resolvable: bool,
    recommended_action: String,
}

fn diagnose_fleet(agent_opinions: &[f64]) -> FleetDiagnosis {
    let d = disagreement_matrix(agent_opinions);
    let hodge = decompose_disagreement(&d);

    let total = hodge.gradient_energy()
        + hodge.curl_energy()
        + hodge.harmonic_energy();

    let (g, c, h) = if total > 1e-10 {
        (hodge.gradient_energy() / total,
         hodge.curl_energy() / total,
         hodge.harmonic_energy() / total)
    } else {
        (0.0, 0.0, 0.0)
    };

    let action = if h > 0.5 {
        "External authority needed".to_string()
    } else if c > 0.5 {
        "Break cycles with tie-breaking rules".to_string()
    } else if g > 0.5 {
        "Rank agents and follow gradient".to_string()
    } else {
        "Consensus reached or near-reached".to_string()
    };

    FleetDiagnosis {
        consensus_gap: hodge.consensus_gap(),
        gradient_ratio: g,
        curl_ratio: c,
        harmonic_ratio: h,
        resolvable: g > 0.5,
        recommended_action: action,
    }
}

fn main() {
    let opinions = vec![10.0, 10.5, 9.8, 50.0, 10.2];
    let diag = diagnose_fleet(&opinions);
    println!("Gap: {:.4}", diag.consensus_gap);
    println!("Gradient: {:.1}%, Curl: {:.1}%, Harmonic: {:.1}%",
        diag.gradient_ratio * 100.0,
        diag.curl_ratio * 100.0,
        diag.harmonic_ratio * 100.0);
    println!("Action: {}", diag.recommended_action);
}
```

## si-cli Integration

```bash
# Decompose agent disagreements
si fleet disagree --agents 0,1,2,3 --opinions 10.0,10.5,9.8,50.0

# Run iterative consensus
si fleet consensus --max-rounds 50 --threshold 0.01

# Diagnose why consensus failed
si fleet diagnose --show-components
```

## si-fleet-api Integration

```
POST /v1/fleet/hodge/decompose
{
    "opinions": [10.0, 10.5, 9.8, 50.0],
    "weights": [1.0, 1.0, 1.0, 0.1]
}

→ {
    "consensus_gap": 0.0234,
    "gradient_energy": 0.8,
    "curl_energy": 0.05,
    "harmonic_energy": 0.15,
    "is_consensus": false,
    "vote": 10.125,
    "recommended_action": "Rank agents and follow gradient"
}
```

## Supabase Integration

```sql
CREATE TABLE fleet_disagreements (
    fleet_id UUID REFERENCES fleets(id),
    round INT NOT NULL,
    disagreement_matrix FLOAT[][] NOT NULL,
    gradient_energy FLOAT NOT NULL,
    curl_energy FLOAT NOT NULL,
    harmonic_energy FLOAT NOT NULL,
    consensus_gap FLOAT NOT NULL,
    vote FLOAT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (fleet_id, round)
);
```

## Cargo.toml Wiring

```toml
[dependencies]
hodge-consensus = { git = "https://github.com/SuperInstance/hodge-consensus-rs" }
sheaf-coherence = { git = "https://github.com/SuperInstance/sheaf-coherence-rs" }
entropy-conservation = { git = "https://github.com/SuperInstance/entropy-conservation-rs" }
```

## Architecture

```
src/
└── lib.rs    — disagreement_matrix, decompose_disagreement, consensus_vote, HodgeComponents
```

## API Reference

### `disagreement_matrix`

```rust
pub fn disagreement_matrix(opinions: &[f64]) -> Vec<Vec<f64>>
```

Compute the pairwise skew-symmetric disagreement matrix `D[i][j] = x_i - x_j`.

### `decompose_disagreement`

```rust
pub fn decompose_disagreement(matrix: &[Vec<f64>]) -> HodgeComponents
```

Decompose a disagreement matrix into gradient, curl, and harmonic components.

### `consensus_vote`

```rust
pub fn consensus_vote(opinions: &[f64], weights: &[f64]) -> f64
```

Compute a weighted majority vote.

### `HodgeComponents`

```rust
impl HodgeComponents {
    pub fn gradient_energy(&self) -> f64;
    pub fn curl_energy(&self) -> f64;
    pub fn harmonic_energy(&self) -> f64;
    pub fn consensus_gap(&self) -> f64;
    pub fn is_consensus(&self, threshold: f64) -> bool;
}
```

## Testing

```bash
cargo test
```

## License

MIT
