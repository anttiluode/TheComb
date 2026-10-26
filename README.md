# The Comb

A single-page simulation of a picture that came up on a Monday morning: **a neural network as a spinning comb cutting grooves into paper**, and what Ryan Williams' 2025 time-to-space theorem does to that picture.

Open `index.html` in a browser. No install, no build, no server. Everything runs locally in plain JavaScript.

The page has three parts.

## 1. The lathe: spin and keep every sheet

Concentric rings are sheets of paper. The innermost ring is the input record. Each comb sits between two sheets.

- **Teeth are weights.** Copper teeth push (positive weights), blue teeth pull (negative weights).
- **Pressure is the weighted sum.** At each position the comb rests on the sheet below and computes `s = Σ wₖ · groove[i + offsetₖ] + bias`.
- **The stylus is ReLU.** If `s > 0`, the stylus cuts a groove of depth `s` on the next sheet. If `s ≤ 0`, it lifts, and the cell is crushed to zero.
- **Tooth spacing** spreads the teeth so they skip cells (a dilated convolution: the "jumping over others" in the original picture).

Stacked, the combs are a 1-D circular convolutional network with ReLU, cut ring by ring.

**Blank the paper** wipes every sheet. The combs keep spinning, and the grooves settle outward: sheet 2 can only be right once sheet 1 is. The table tracks, sheet by sheet, the fraction crushed to zero and when the sheet matches a full forward pass.

## 2. Regrow: the Williams schedule

Switch the mode and **nothing is kept except the input record**. Click any groove on any sheet. The combs then regrow, depth first, only the grooves that one groove depends on, one tooth press at a time, and forget each one as soon as it has been used.

The counters show the trade:

| | Paper held at once | Presses for one groove at depth L |
|---|---|---|
| Keep every sheet | (L + 1) · N grooves | L · N · K |
| Regrow depth first | the record + L running sums + L bookmarks | K + K² + … + K^L |

Here N = 96 cells per sheet and K = teeth per comb. Paper shrinks from hundreds of grooves to a handful of numbers, and presses grow exponentially with depth. The regrown value is checked against the kept-sheets value and matches **bit for bit**, because the operations are done in the same order.

This is the unsharing step in Williams' proof: turn the web of reused intermediate results into a tree, and recompute instead of store.

**A caveat that matters:** this comb's rule is a running sum, so depth-first regrowing already needs only one number per level. Williams needs more machinery because a general step (a block of Turing-machine steps) needs *all* of its inputs at once, each b bits long. A naive depth-first stack would then hold d · b bits per level. That is the stack Cook–Mertz folds away.

## 3. Borrowed paper: the Cook–Mertz trick, run exactly

Three registers are already full of someone else's data (the junk x̂). We need P(v) for a polynomial P, but there is no clean space to write v. So, for each of m = 8 phases:

1. rotate the junk by ωⁱ and write v on top: `r = ωⁱ · x̂ + v`
2. evaluate P(r) and add it to a running sum
3. take v back off and undo the rotation, so the register holds x̂ again

The identity that makes this work:

$$\sum_{i=0}^{m-1} P(\omega^i \hat x + v) = m \cdot P(v) \qquad \text{whenever } \deg P < m$$

Every term containing the junk to the power k picks up a factor ω^{ik}, and those factors sum to zero around the wheel whenever m does not divide k. Only the junk-free part, P(v), survives.

Arithmetic is in the finite field of 257 elements (ω = 4, since 4⁸ = 1 mod 257), so nothing is rounded. The page shows:

- one direct evaluation on the occupied registers, P(x̂ + v), which is wrong;
- the 8-phase sum ÷ 8, which equals P(v) exactly;
- the registers handed back unchanged.

**Push the degree slider to 8 or higher and it breaks.** The junk⁸ term gets phase ω⁸ⁱ = 1, which never cancels. That limit is the reason the trick is expensive in Williams' proof: each node's rule must first be rewritten as a polynomial of low enough degree, and for a b-bit rule that rewriting costs about 2ᵇ time.

(Williams' paper works in a field of characteristic 2, where adding v twice removes it. Here, in GF(257), v is subtracted instead. The idea is the same.)

## The ledger

The bottom of the page compares the three schedules live with the lathe settings. The one sentence to keep:

> The combs never change. Williams changes the schedule: which grooves stay on paper and which are cut again when needed. Every saving in paper is paid for in turns of the lathe.

## What was checked

- Regrowing matches the full forward pass bit for bit in 27 cases (depth 2, 4, 6 × teeth 3, 5, 9 × three target grooves).
- The borrowed-paper sum recovers P(v) exactly for degree 1, 3 and 7, and fails for degree 8, 9 and 12, over 5 random draws each. The registers are restored exactly in every case.

## What this is not

- **Not a speedup.** Williams' theorem saves memory and costs enormous time. The regrow mode shows exactly that.
- **Not a model of real neurons.** A positive groove is a number, not a spike, and ReLU has no memory of its own. The grooves are activations, not learning.
- **Not a demonstration that physical matter can do Cook–Mertz.** The cancellation needs exact arithmetic and a polynomial below the phase count. Noise, non-polynomial responses and anything like |z|² break it.

## References

- Ryan Williams, *Simulating Time With Square-Root Space*, 2025. [arXiv:2502.17779](https://arxiv.org/abs/2502.17779). The main theorem is TIME[t] ⊆ SPACE[√(t log t)] for multitape Turing machines. The tree-evaluation procedure and equation (5) are in Appendix A. Section 5 discusses why the construction can't simply be repeated.
- James Cook and Ian Mertz, *Tree Evaluation is in Space O(log n · log log n)*, STOC 2024.
- John Hopcroft, Wolfgang Paul and Leslie Valiant, *On Time Versus Space*, 1977 (conference version 1975). The O(t / log t) bound that stood for 50 years.

## Origin

Built by Claude (Opus 5.5) for Antti Luode from his comb picture: spikes of different strength pressing on paper, leaving grooves like a vinyl record, the comb spinning, and the question of whether the longer it spins, the more it does what Williams describes.

MIT licence (see `LICENSE`).
