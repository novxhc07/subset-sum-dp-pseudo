# subset sum problem: the DP table, why O(n·W) is only pseudo-polynomial, and how to answer it in an interview

Two different questions get filed under the same name, and most of the confusion around the subset sum problem starts there.

The first version: given a list of positive integers and a target `W`, does some subset add up to exactly `W`? Yes or no. The second version, which shows up in a lot of university lecture notes under the same heading, asks for the subset with the largest total that stays at or below `W`. That one is 0/1 knapsack with value equal to weight, and the recurrence differs by one word (`OR` versus `max`). Everything else — the table, the complexity, the pseudo-polynomial caveat — carries over.

If you're here because a problem set, an online assessment, or an interview loop put this in front of you, the useful things to walk away with are: the state definition, the recurrence, the loop order that keeps each item usable only once, and the reason `O(n·W)` is not the polynomial time you might assume it is.

## What the problem actually says

Given `n` items with integer weights `w₁ … wₙ` and a bound `W`, decide whether there is a subset `S ⊆ {1 … n}` with


Σ wᵢ (for i in S) = W


That's the exact-sum decision version. Weights are positive integers. If you're allowed arbitrary rational numbers, you can multiply everything by a common denominator without changing the answer, which is why most write-ups just assume integers.

A few things are true immediately and are worth saying out loud in an interview:

- The empty set sums to 0, so target 0 always succeeds.
- If `W` is bigger than the total of all weights, the answer is no.
- Any single weight greater than `W` can never be part of an exact-sum solution in the positive-integer version. Skipping those items shrinks the table before you build it.

## The brute force baseline

Every subset of `n` items is a bit vector of length `n`, and there are `2ⁿ` of them. Checking each one costs up to `n` additions, so the naive approach runs in `O(n · 2ⁿ)`. Written recursively, the "take item `i` or skip it" split gives `T(n) = 2T(n-1) + O(1)`, which resolves to `Θ(2ⁿ)`.

For `n = 20` that's about a million subsets and you're fine. For `n = 40` you're past a trillion and the interviewer is watching to see whether you notice.

## The state that makes the recursion collapse

The recursion branches because the same sub-question gets asked repeatedly: "can the first `i` items hit a remaining sum of `v`?" Different branches arrive at the same `(i, v)` pair, and once you know the answer for that pair, nothing else matters — it doesn't matter *which* earlier items produced that remaining sum.

That's the whole trick. Define:


m(i, v) = true if some subset of the first i items sums to exactly v


The recurrence:


m(i, v) = m(i-1, v)  OR  m(i-1, v - aᵢ)      if aᵢ ≤ v
m(i, v) = m(i-1, v)                          if aᵢ > v
m(i, 0) = true                                for all i


Read it as: either item `i` isn't in the solution and you inherit the answer for `i-1`, or item `i` is in it and the first `i-1` items need to cover `v - aᵢ`.

There are `(n+1) × (W+1)` entries, each filled in constant time, so the table costs `O(n·W)` time and `O(n·W)` space. The answer lives at `m(n, W)`.

Most courses fill the table row by row, index increasing:

python
def subset_sum_table(nums, target):
    n = len(nums)
    m = [[False] * (target + 1) for _ in range(n + 1)]
    for i in range(n + 1):
        m[i][0] = True
    for i in range(1, n + 1):
        for v in range(1, target + 1):
            m[i][v] = m[i-1][v]
            if nums[i-1] <= v:
                m[i][v] = m[i][v] or m[i-1][v - nums[i-1]]
    return m[n][target]


Copy that into an interview and the follow-up is almost always "can you cut the memory?" Yes, and that's where a lot of people lose the thread.

## Rolling the table down to one array

You only ever read row `i-1` while writing row `i`, so one array of size `W+1` is enough — provided you iterate the sums backwards:

python
def subset_sum(nums, target):
    dp = [False] * (target + 1)
    dp[0] = True
    for x in nums:
        for v in range(target, x - 1, -1):
            dp[v] = dp[v] or dp[v - x]
    return dp[target]


The descending loop is the part that trips people up. When you write `dp[v]` from `dp[v - x]`, the ascending direction means `dp[v - x]` may already have been updated using the *current* item, which lets you reuse it more than once. That's the unbounded variant — coin change, where each coin can be taken repeatedly. Backwards keeps `dp[v - x]` at the previous item's state, which is what "each item at most once" requires.

Space drops to `O(W)`. What you lose is the reconstruction: a flattened boolean array no longer remembers which items produced the sum. If the question asks for the actual subset, either keep the 2D table and walk backwards, or store a per-item snapshot / parent pointer.

The backward walk on the 2D table is short. Start at `(n, W)`; if `m[i][v] == m[i-1][v]`, item `i` wasn't used, move to `(i-1, v)`; otherwise item `i` was used, move to `(i-1, v - aᵢ)`. That's `O(n)` after the table is built.

## The complexity line that gets marked wrong

`O(n·W)` time, `O(W)` space. Say it that way and you're fine. Say "the subset sum problem is solvable in polynomial time" and you've invited a follow-up you probably don't want.

The input to this problem is not `W`; it's `n` weights plus the number `W`. Storing `W` takes about `log W` bits. The running time of the DP is `O(n · W)`, which is exponential in the *length* of the input encoding, since `W` sits in an exponent there. That property has a name: pseudo-polynomial.

Subset sum is NP-complete — it's one of the original Karp problems, and it's the standard companion to the partition and knapsack problems in complexity chapters. The precise classification is *weakly* NP-complete, which matters practically: weakly NP-hard problems admit pseudo-polynomial algorithms and fully polynomial-time approximation schemes, and subset sum has both. Strongly NP-complete problems have no pseudo-polynomial algorithm unless P = NP.

The engineering translation is short and it changes what you should code:

- `W` up to roughly `10⁴`–`10⁶` and `n` in the hundreds or thousands: build the DP table. It's fast and the code is 10 lines.
- `W` up to `10⁹` or the weights are 64-bit: the table is hopeless, and no amount of careful indexing saves it. Change approach.

## Variants you'll actually be asked

The same table shape reappears under different names. Knowing which knob to turn is most of the skill.

| Variant | Target / state | What changes |
| --- | --- | --- |
| Exact subset sum | `m(i, v)` boolean | The base case above |
| Max sum ≤ W (0/1 knapsack with value = weight) | `OPT(i, w)` = best total | `max(OPT(i-1,w), wᵢ + OPT(i-1, w-wᵢ))` instead of `OR` |
| Count the subsets | `dp[v]` = number of ways | `dp[v] += dp[v-x]`, no short-circuit |
| Partition equal subset sum | target = total / 2 | Total must be even; one item > half means false |
| Unbounded / coin change | same array | Ascending inner loop |
| Bitset acceleration | one big integer | `bits \|= bits << x`, checks bit `target` at the end |

The bitset version is worth knowing for Python and for C++ `std::bitset`, because the `O(n·W)` work gets packed into machine words and runs roughly 64× fewer operations. It's still `O(n·W)` in the theory sense, just with a much better constant.

Negative numbers break the array indexing. The usual fixes are an offset shift (allocating a range from the smallest achievable sum to the largest) or a hash set / dictionary of reachable sums instead of an array. Both change the complexity in ways you should state rather than hand-wave.

## When dynamic programming is the wrong answer

Huge `W` is the case that separates candidates who memorized the DP from candidates who understand it. If `W` is `10⁹`, the `O(n·W)` table never gets built, so the useful alternative is meet-in-the-middle:

1. Split the items into two halves of roughly `n/2`.
2. Enumerate every subset sum within each half — `2^(n/2)` values each.
3. Sort one half's sums.
4. For each sum `s` in the other half, binary search for `W - s` in the sorted list.

Total work is `O(2^(n/2) · n)`, which is staggering compared to `2ⁿ` but perfectly happy with `n = 40`. Note the trade: this approach relies on small `n` and doesn't care how large the weights are. The DP is the opposite — it doesn't care how large `n` is and dies when `W` is large.

Greedy is not an option either way. Taking the largest items first looks reasonable and fails immediately: with `nums = [5, 4, 4]` and `W = 8`, a greedy pass grabs 5, then 4, overshoots to 9, and reports failure — while `4 + 4 = 8` was right there. Sorting by value-per-weight doesn't rescue it in general, which is the standard motivating example in algorithms notes on why knapsack needs DP.

## The bugs that cost people the interview

**Not asking how large `W` is.** This is the single highest-value question before writing any code, and interviewers tend to read it as a signal. The answer determines whether you're writing a 10-line DP or a meet-in-the-middle routine with a sort and a binary search.

**Wrong loop direction.** Ascending inner loop in the 0/1 version silently allows reuse. The resulting code passes small tests and fails on anything with repeated values.

**Confusing decision with counting.** "Is there a subset?" returns a boolean and short-circuits. "How many subsets?" returns a count that needs modular arithmetic if the numbers get big, and needs `dp[0] = 1` as the base case rather than `True`.

**Quoting `O(n·W)` as polynomial.** Covered above, and it's asked often enough to prepare a one-sentence answer: it's pseudo-polynomial, because `W` contributes `log W` to the input size.

**One table, two dimensions, wrong base row.** `m(i, 0) = true` for every `i` (empty subset sums to 0) and `m(0, v) = false` for `v > 0`. Getting these backwards produces off-by-one answers that look plausible.

**Forgetting the total-sum pruning.** If `target > sum(nums)`, return false before allocating anything. It costs one line and saves a wasted table in the capacity variant.

## Where pattern-based drilling fits

Recognizing "this is a subset/knapsack table" under time pressure is the part that doesn't come from reading explanations. The tell is usually a small set of items plus a numeric bound with a modest range — items and a target, items and a capacity, numbers that must be split into two equal groups. Once you've seen the pattern applied to partition, target sum, and last-stone problems, the recurrence stops being something you reconstruct from scratch.

That recognition step is what [👉 AlgoMonster's dynamic programming pattern track](https://algo.monster?fpr=li69) is built around: the lesson set groups the subset-family problems under one pattern template and uses decision flowcharts to route a new problem to the right approach, with the code in a browser editor. It's text and diagrams rather than video, which is either the reason it works for you or the reason it doesn't — there's no video content at any price tier, and the problem bank runs in the low hundreds rather than the thousands, so most people supplement it with a separate problem pool for volume. The AI assistant and Speedrun modes are unlimited on paid plans, and the AI walks through submitted code and points at the error rather than handing over the answer.

### AlgoMonster plans

The checkout side is simpler than most prep platforms, mostly because you can pay once. Prices move with whatever promotion is running, so the numbers below are what the subscribe page showed when this was checked.

| Plan | What it covers | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample patterns and problems — enough to judge the text-and-diagram format, not enough to prep seriously | $0 | — | [ Start on the free tier](https://algo.monster?fpr=li69) |
| Pro – Monthly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company questions refreshed monthly; bonus courses included | $18.99/mo | Billed monthly, cancel or upgrade anytime | [ Check the monthly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro – Yearly | Everything in Monthly plus 12 months of full access and a member discount on live bootcamps | $69.99/year ($5.84/mo) | Billed annually | [ See the yearly price](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in Yearly plus lifetime updates — no renewals, new content included as it ships | $89 once | One-time payment | [ Get lifetime access](https://algo.monster/subscribe?fpr=li69) |

Two honest caveats on those numbers. AlgoMonster discounts aggressively and rotates sales, so the checkout total is the only figure that counts — other snapshots of the same page have displayed the yearly plan at $99 ($8.25/mo) and lifetime at $189, and third-party coupon trackers currently list those. And there's no team or enterprise plan; this is built for one person studying on their own.

Which one makes sense depends on your timeline. If you have an interview in six weeks, the monthly plan at $18.99 covers it and you can stop paying. The yearly plan is cheaper than four months of monthly if you expect to keep prepping across a search that runs longer than a few weeks. Lifetime is a one-time $89, which is less than five months of monthly billing — it pays for itself if you ever expect to interview again, and most engineers do, every two to three years.

## Quick answers

**Is subset sum NP-complete?** Yes, weakly NP-complete. There's no known polynomial-time algorithm, and one exists only if P = NP.

**What's the DP complexity?** `O(n·W)` time and `O(W)` space with the rolling array, `O(n·W)` space if you keep the full table for reconstruction.

**Why pseudo-polynomial?** Because `W` takes `log W` bits to write down, and the running time is linear in `W`, not in `log W`.

**Why iterate the sum backwards?** So `dp[v - x]` still reflects the state before the current item was considered. Forwards turns the 0/1 problem into the unbounded one.

**What if the weights are huge?** Drop the DP and use meet-in-the-middle, `O(2^(n/2) · n)`, which depends on `n` instead of `W`.

**Does the 2D table handle negative numbers?** Not directly. Offset the indices or switch to a set of reachable sums.

**Can I use the subset table for "max sum without exceeding W"?** Yes. Replace `OR` with `max` and initialize the first row to zeros. That's 0/1 knapsack with value equal to weight, which is why the two problems are taught together.

The version of this problem you'll be handed in an interview is rarely the plain one. It'll be dressed up as partitioning a set, hitting a target with `+` and `-` signs, or splitting stones — and the recognition step, not the recurrence, is where the time goes.
