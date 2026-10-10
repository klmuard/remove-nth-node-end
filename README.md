# remove nth node from end of list: One-Pass Two Pointers, a Dummy Node, and the Edge Cases That Actually Break Submissions

This problem has a reputation problem. It's tagged Medium on LeetCode, it sits at roughly a 49% acceptance rate, and most people read the description once, think "find the nth from the end, unlink it," and start typing. That works fine on `[1,2,3,4,5], n=2`. It breaks on `[1], n=1` and `[1,2], n=2`, which is exactly where the interviewer is watching.

So the interesting part of this problem isn't the pointer movement. It's the two structural decisions that make the pointer movement safe: the **dummy node** and the **gap of n versus n+1**. Get those wrong and you'll pass the samples and crash on the hidden tests.

Below is the full working solution in the languages that show up in interviews, the reasoning behind each decision, and the four specific bugs that account for most failed submissions.

## What the problem actually asks for

Given the head of a singly linked list and an integer `n`, remove the nth node from the end and return the head of the modified list.


Input:  head = [1,2,3,4,5], n = 2
Output: [1,2,3,5]

Input:  head = [1], n = 1
Output: []

Input:  head = [1,2], n = 1
Output: [1]


Constraints are small (`1 <= sz <= 30`, `1 <= n <= sz`), which is a hint that the interviewer doesn't care about input size at all. They care about whether you handle the boundary. Two follow-ups usually come after: "can you do it in one pass?" and "what if you can't modify the list?"

Note the asymmetry in the problem that catches people: `n` is guaranteed valid, but the node you're removing might be the head. The head has no predecessor. In a singly linked list, deleting a node requires its predecessor. That's the whole tension of this question.

## Approach 1: two passes (correct, but not what they're asking)

The obvious solution: walk the list once to get its length `L`, then walk again to position `L - n - 1` (the node before the target) and splice.

python
class Solution:
    def removeNthFromEnd(self, head: Optional[ListNode], n: int) -> Optional[ListNode]:
        length = 0
        cur = head
        while cur:
            length += 1
            cur = cur.next

        if length == n:          # removing the head
            return head.next

        steps = length - n - 1   # steps from head to the node before target
        cur = head
        for _ in range(steps):
            cur = cur.next
        cur.next = cur.next.next
        return head


Time O(L), space O(1). It's fine, and if you blank on the fast/slow trick in an interview, writing this first is not a mistake. It also demonstrates that you noticed the head-removal case (`if length == n`).

But it has a special case baked into it, and special cases are what interviewers probe. The one-pass version removes the need for that `if`.

## Approach 2: one pass, fixed gap, dummy node

The insight: "nth from the end" is really "n nodes away from the boundary." If two pointers maintain a gap of exactly `n` nodes, then when the leading pointer hits the end, the trailing pointer is sitting exactly where you need it.

python
class Solution:
    def removeNthFromEnd(self, head: Optional[ListNode], n: int) -> Optional[ListNode]:
        dummy = ListNode(0, head)
        fast = slow = dummy

        for _ in range(n):
            fast = fast.next

        while fast.next:
            slow = slow.next
            fast = fast.next

        slow.next = slow.next.next
        return dummy.next


That's the whole thing. Six meaningful lines.

### Walk through `[1,2,3,4,5], n = 2`

| Step | `fast` at | `slow` at | List state |
| --- | --- | --- | --- |
| Start | dummy | dummy | `dummy → 1 → 2 → 3 → 4 → 5` |
| Advance `fast` n=2 | 2 | dummy | unchanged |
| Move both | 3 | 1 | unchanged |
| Move both | 4 | 2 | unchanged |
| Move both | 5 | 3 | unchanged |
| `fast.next` is None → stop | 5 | 3 | — |
| Splice | 5 | 3 | `1 → 2 → 3 → 5` |

`slow` lands on node `3`, which is directly before node `4`. One pointer rewiring and node 4 is gone. The list is traversed once.

### Why the dummy node is not optional

Trace `[1], n = 1` without a dummy, starting `fast = slow = head`:

- Advance `fast` one step. `fast` is now `None`.
- The loop condition `fast.next` immediately tries to read `.next` on `None`.

That's your crash. The dummy node guarantees a predecessor always exists, even when the node being removed is the original head. With the dummy in place, the `n == length` case needs no special handling at all: `slow` ends up on the dummy itself, `slow.next = slow.next.next` skips the real head, and `dummy.next` becomes the new head. Uniform logic, no branch.

If you're writing C++, this is also where you decide whether to free the removed node. Not required for LeetCode to pass, but a good habit to mention out loud in an interview.

cpp
ListNode* removeNthFromEnd(ListNode* head, int n) {
    ListNode dummy(0, head);
    ListNode* fast = &dummy;
    ListNode* slow = &dummy;

    while (n--) fast = fast->next;
    while (fast->next) {
        slow = slow->next;
        fast = fast->next;
    }

    ListNode* toDelete = slow->next;
    slow->next = slow->next->next;
    delete toDelete;
    return dummy.next;
}


And the Java version, which is what most interview screens still run:

java
class Solution {
    public ListNode removeNthFromEnd(ListNode head, int n) {
        ListNode dummy = new ListNode(0, head);
        ListNode fast = dummy, slow = dummy;

        for (int i = 0; i < n; i++) fast = fast.next;

        while (fast.next != null) {
            slow = slow.next;
            fast = fast.next;
        }

        slow.next = slow.next.next;
        return dummy.next;
    }
}


## Complexity

Time is O(L), where L is the number of nodes. The fast pointer takes `n` steps forward and then both pointers move together for another `L - n` steps, so it's one linear pass plus a constant. Space is O(1): a dummy node and two pointer variables, independent of list length.

There's a subtlety worth knowing because it comes up. A common variant advances `fast` by `n + 1` steps and then loops `while fast:` instead of `while fast.next:`. Both are correct. The `n + 1` version terminates with `fast == null`; the `n` version terminates with `fast` on the last real node and `slow` on the predecessor of the target. Mixing the two forms is mistake #2 below.

## The four bugs that actually cost people this problem

These show up over and over in submissions and in interview write-ups.

**1. Advancing the gap by `n + 1` while still looping on `fast.next`.** You create a gap one node too wide, `slow` lands one node too early, and you delete the wrong node. The samples `[1,2,3,4,5], n=2` will still sometimes pass if you're lucky. The hidden tests won't. Pick one convention and commit: gap of `n`, loop on `fast.next`, `slow` ends on the predecessor.

**2. No dummy node, so head removal crashes.** Covered above. The tell is `AttributeError: 'NoneType' object has no attribute 'next'` on a one-element list. If a solution works on `[1,2,3,4,5]` and throws on `[1]`, this is why.

**3. Off-by-one in the loop condition.** `while fast:` instead of `while fast.next:` moves `slow` one node too far, and now `slow.next` is the node *after* the target. The output looks almost right, which makes it worse — you'll stare at it for ten minutes.

**4. Rarely, no input validation.** LeetCode guarantees `1 <= n <= sz`. Production code doesn't get that guarantee. If `n` exceeds the list length, `fast` becomes `None` mid-advance and you crash. Worth one sentence in an interview: check `fast.next is not None` inside the advance loop and bail or raise.

If you want a check on whether you've internalized it, the reference walkthrough with the step-by-step visualization is worth running through once: [👉 See the full AlgoMonster walkthrough for Remove Nth Node From End of List](https://algo.monster/liteproblems/19?fpr=li69).

## Where this problem sits in the pattern map

This isn't a standalone trick. It's the fixed-gap variant of two pointers, and the recognition signal is specific: any problem asking for "the kth element from the end," "the node before X," or "one pass, no extra space" on a linked list is a gap-pointer problem. The same skeleton solves:

- Find the middle of a linked list (gap of 1, fast moves twice as fast)
- Detect a cycle (Floyd's algorithm)
- Remove duplicates in a sorted list
- Find the kth-to-last element without deleting it

The mental model that makes it stick: two runners on a track. Start one `n` steps ahead. When the leader reaches the finish line, the trailer is exactly `n` steps behind. That's the entire derivation. Everything else — the dummy node, the loop condition — is implementation detail for making the boundary safe.

Once you can state the gap invariant out loud ("after setup and at the top of every iteration, `fast` is exactly `n` nodes ahead of `slow`"), you can re-derive the code from scratch in an interview instead of recalling it. That's the difference between memorizing `removeNthFromEnd` and knowing the pattern.

## Practicing this the efficient way

Grinding random problems after reading one editorial is where most people stall. If your actual gap is "I can write the code but I don't recognize which pattern this problem wants in the first three minutes," then the useful unit of study isn't more problems, it's the pattern-to-problem mapping.

AlgoMonster is built around that idea. It was put together by ex-Google engineers, and the format is text plus interactive diagrams rather than video, with a decision flowchart that routes a problem description to a pattern. For this specific problem, the flowchart path is short: is it a graph? No. Is it a kth-smallest/largest problem? No, it's positional. Is it a linked list? Yes. Fast/slow or fixed-gap pointers? Yes.

That routing is the thing being tested in a real interview — the deciding costs you 15 of your 45 minutes if it goes badly.

The subscribe page currently lists access to 325 lessons and problems, 678 illustrations and 3,000+ editorials, with content covering the core patterns plus introductory system design and behavioral material. Two honest limitations to weigh before paying: there's no video content at all, and the system design coverage is introductory, so if you're targeting L5+ loops, you'll need something else for that round.

### Current plans and pricing

AlgoMonster doesn't offer monthly billing. The options are free, annual, or lifetime. Prices on the subscribe page move with promotions, so treat the numbers below as the currently listed rates rather than permanent ones.

| Plan | What you get | Price | Billing cycle | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems; enough to test whether the text-and-diagram style works for you | $0 | No expiry | [ Start on the free tier](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company-sourced questions updated monthly; 325 lessons and problems, 678 illustrations, 3,000+ editorials; 12 months of full access | $99/year (listed at about $8.25/month billed annually) | Annual, with a 3-day free trial before the first charge | [ Start the 3-day free trial on Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan, plus lifetime updates and an internal referral to a top tech company for qualified candidates | $189 | One-time payment, permanent access | [ Get Pro Lifetime for $189 one-time](https://algo.monster/subscribe?fpr=li69) |

The break-even math is straightforward. If you expect to run interview prep more than once in the next few years — which most engineers do, given how job-hopping works — lifetime at $189 is cheaper than two annual renewals at $99. If you have one loop in eight weeks and you're done, the annual plan is the cheaper commitment, and the 3-day trial means you can confirm the format suits you before paying anything.

Coupon codes do circulate through creators and partner channels, and they can stack on top of whatever sale is running. The codes change constantly and expiry dates are unreliable in third-party listings, so check at checkout instead of trusting a code you found in a blog post from six months ago.

One structural note: because there's no monthly option, the free tier is the only way to preview the teaching style before committing. Spend a few days there. If the pattern-first, text-based format doesn't click for how you learn, no discount makes it worth the purchase — and that's true of the $99 and the $189 alike.

## Quick reference

The solution, in the form you'd write on a whiteboard:


1. dummy = ListNode(0, head)
2. fast = slow = dummy
3. advance fast n times
4. while fast.next: advance fast and slow together
5. slow.next = slow.next.next
6. return dummy.next


Two decisions carry the whole solution: the dummy node makes head removal a non-case, and the gap of exactly `n` (with the loop on `fast.next`) puts `slow` on the predecessor instead of the target. Everything else is bookkeeping. Test it against `[1], n=1` and `[1,2], n=2` before you call it done — those two inputs catch every version of this bug.
