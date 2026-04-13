**Credit to**

- S. Lee, W. Lee, S. Cho, S. Lee, and J. Lee, “CARTEL: Consensus adapting real-time and efficient logging,” in Proceedings of the 2025 IEEE Real-Time Systems Symposium (RTSS), Dec. 2025, pp. 461–473. IEEE.

- D. Ongaro and J. Ousterhout, “In search of an understandable consensus algorithm,” in Proc. USENIX Annual Technical Conference (USENIX ATC), 2014, pp. 305–319.

- Github User psjung, “Raft-Uppaal,” GitHub repository. [Online]. Available: [Raft-Uppaal repository](https://github.com/psjung/Raft-Uppaal). Accessed: Apr. 12, 2026.

## Excerpt from my paper, translated to Markdown:

### Model Overview

Three UPPAAL models were created for this study:

1. **`raft`**
   An idealized model of the standard Raft election state machine that assumes:

   * Zero network delay
   * Instant synchronization across nodes

   To minimize the state space, this model uses a small election timeout range of **5–10 time units**.

2. **`raft_time_delay`**
   A more realistic model featuring:

   * Asynchronous message passing
   * Randomly sampled message transmission delays
   * Wider timeout ranges (**150–300 time units**)

3. **`cartel_time_delay`**
   A modified version of `raft_time_delay` that replaces standard Raft voting with the **CARTEL voting strategy**.

---

### `raft` Model
<img width="975" height="400" alt="image" src="https://github.com/user-attachments/assets/590ca281-edfe-43af-a945-3a67536b2096" />

**Figure 1.** *Timed automaton for the standard Raft election protocol.*

The `raft` model was built using an existing example model and enhanced to improve correctness and testability. Key improvements include:

* Refined logic for candidates and leaders stepping down
* Fixed bug in election winner determination
* Addition of terminal states to verify convergence to a stable state

Using UPPAAL’s symbolic model checker, this model was used to validate correctness via:

* **Liveness:** No deadlocks occur
* **Reachability:** A leader is eventually elected
* **Safety:** If multiple leaders exist, the system eventually converges to one

Due to its simplified assumptions, exhaustive verification was feasible.

---

### `raft_time_delay` Model

<img width="967" height="350" alt="image" src="https://github.com/user-attachments/assets/1cc4bee1-21c4-4492-aa37-8d084f9827f9" />

**Figure 2.** *Timed automaton for Raft with asynchronous message delays.*

The `raft` model assumes zero-delay communication, making it unrealistic for distributed systems. The `raft_time_delay` model introduces more realistic timing behavior.

#### Message Representation

Each vote request message is modeled as a tuple:

```
{src, dest, netDelay, netTimer, delivered}
```

* `src`: sending node
* `dest`: receiving node
* `netDelay`: random delay (uniformly sampled from **20–40 ms**)
* `netTimer`: clock tracking elapsed time since sending
* `delivered`: flag (0 → in transit, 1 → delivered when `netTimer ≥ netDelay`)

The delay range (20–40 ms) gives:

* Expected one-way latency: **30 ms**
* Approximate round-trip latency: **~60 ms**, matching empirical results from CARTEL

A **pendingTimeout of 40 ms** is used, satisfying 

<img width="682" height="49" alt="image" src="https://github.com/user-attachments/assets/03e6c10d-94ad-4740-b5aa-473f1045daf4" />

Where δ is the maximum network delay and $$ET^{\text{min}}$$ is the minimum election timeout.

---

### Message Broadcasting

<img width="761" height="331" alt="image" src="https://github.com/user-attachments/assets/f78eb9a3-56d3-4782-91bc-0041b7a7b116" />

**Figure 3.** *`sendVoteRequest()` function.*

Vote requests are tracked in an **N × N matrix `msg`**, where:

* `msg[i][j] = 1` represents node *i* sending a vote request to node *j*

Each message is assigned an independent random delay.

---

### Network Automaton

<img width="831" height="251" alt="image" src="https://github.com/user-attachments/assets/937663f9-3a1d-44ff-a1ca-73a43ea6acf4" />

**Figure 4.** *Network automaton and supporting logic.*

An auxiliary **Network automaton** manages message delivery:

* Continuously checks for deliverable messages using `hasDeliverable()`
* Marks messages as delivered via the `run()` function

Delivery tracking uses an **N × N matrix `delivered`**, where:

* `delivered[i][j] = 1` indicates message arrival from node *i* to node *j*

#### Important Constraint

UPPAAL cannot model an automaton that:

* Continuously fires transitions
* While also allowing time to pass

Therefore, the Network alternates between:

* **Sleeping** (allowing time to pass)
* **Waking** (processing deliveries)

This introduces minor artificial latency but does not significantly impact comparative results.

---

### Split Vote Tracking

<img width="818" height="270" alt="image" src="https://github.com/user-attachments/assets/68c0353c-a403-4d71-9b67-fb21327cec92" />

**Figure 5.** *SplitVoteCounter automaton and logic.*

The **SplitVoteCounter** automaton:

* Detects split votes during simulations
* Avoids double-counting within the same election term
* Transitions to a terminal **Done** state once the system stabilizes

This enables:

* Measurement of split vote frequency
* Verification that no split votes occur in `cartel_time_delay`

---

### `cartel_time_delay` Model

<img width="975" height="321" alt="image" src="https://github.com/user-attachments/assets/c4ce1bf2-3ba5-479a-8525-52bea2bd89d8" />

**Figure 6.** *Timed automaton for the CARTEL election protocol.*

This model is identical to `raft_time_delay` except for the voting mechanism.

#### Key Difference: CARTEL Voting

Upon receiving a vote request:

* The node **does not vote immediately**
* Instead, it transitions to a **`VotePending`** state
* Waits for `pendingTimeout`
* Then votes for the candidate with the **smallest election timeout**

This behavior follows the CARTEL specification.

---

### Verification vs. Simulation

The realistic timing behavior in `raft_time_delay` and `cartel_time_delay` results in **state spaces too large for exhaustive verification** using UPPAAL’s symbolic model checker.

Instead, these models are used for **statistical simulation**, enabling:

* Estimation of event probabilities
* Measurement of expected leader election time
* Analysis of split vote frequency

These simulations form the basis for comparing Raft and CARTEL under realistic conditions.
