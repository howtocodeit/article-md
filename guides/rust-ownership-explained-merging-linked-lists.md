---
title: "Ownership by example: merging linked lists in Rust"
meta_title: "Ownership By Example: Merging Linked Lists in Rust"
slug: rust-ownership-explained-merging-linked-lists
description: Grok Rust's ownership model with a "simple" linked list coding challenge.
meta_description: Grok Rust's ownership model with a "simple" linked list coding challenge.
color: orange
tags: [rust, ownership, data structures, algorithms]
version: 1.1.0
---

"Angus", you say, "you promised us practical, better-than-production Rust. You're not about to walk through a LeetCode question, are you?".

Well... yes. But stay with me! I have a very good reason.

Coding challenges are more likely to advance your knowledge in Rust than in any other language – because Rust makes them _harder_ than other languages. 

Getting down low and hand-coding linked lists, trees and graphs sets up violent altercations with the borrow checker.

It's quite easy to hide a surface-level knowledge of Rust's ownership model if you focus on high-level packages like web frameworks. But if you're not 100% proficient with `std::mem::swap`,  `Rc`, `RefCell` and their friends, coding challenges will weed you out.

If you aspire to be a strong systems programmer or make quality contributions to popular libraries, you need this knowledge. It should be automatic.

I have just the task to help you close those gaps using clean, idiomatic Rust.

> Email sign-up

## Understanding Rust's ownership model with singly linked lists

Linked lists fascinate me. It takes a moment to appreciate why Rust's safety guarantees actually make this trivial, entry-level data structure _hard_ to implement.

Aria Beingessner's [*Learning Rust With Entirely Too Many Linked Lists*](https://rust-unofficial.github.io/too-many-lists/) is one of my favourite Rust resources. If you haven't worked through it, please do – you'll leave with a fine appreciation for why the borrow checker seems to get in our way, and even less confidence in working with `unsafe` code. This is a good thing, since overconfidence is the enemy of `unsafe`.

> Overconfidence is the enemy of `unsafe`.

When confronted with the following challenge, you know that Rust won't make it straightforward:

_"Given two sorted, singly linked lists, merge them in sorted order, and return the sorted list."_

A list is represented simply as a  `ListNode` with an optional reference to the next node in the chain:

```rust
#[derive(PartialEq, Eq, Clone, Debug)]  
pub struct ListNode {  
    pub val: i32,  
    pub next: Option<Box<ListNode>>,  
}
```

And the signature of the merge function is as follows:

```rust
pub fn merge(  
    mut list1: Option<Box<ListNode>>,  
    mut list2: Option<Box<ListNode>>,  
) -> Option<Box<ListNode>>
```

Does this surprise you? It surprised me.

When working with safe representations of linked lists and trees, I always expect some flavor of `Option<Rc<RefCell<Node>>>`. `Rc` supports multiple ownership of child nodes, so you can point two heads to the same child, or access a child element without taking ownership away from its containing list. `RefCell` provides interior mutability, allowing us to change a node's `next` pointer through a shared reference.

However, `Box<ListNode>` means each node has a single owner. That makes it harder to do what I'd normally do in a language like Go or C++, assembling the output list by comparing `node.val` for the first two nodes of the input list, appending the node with the lowest value to the output list, and walking down the remaining chains.

## Breaking all of Rust's ownership rules

Here's a direct translation of that approach into "Rust":

```rust
pub fn merge(  
    mut list1: Option<Box<ListNode>>,  
    mut list2: Option<Box<ListNode>>,  
) -> Option<Box<ListNode>> {  
    let pre_head = Some(Box::new(ListNode { val: 0, next: None }));  ^1
    let mut tail = pre_head;  ^2
  
    while list1.is_some() && list2.is_some() {  ^3
        if list1.unwrap().val < list2.unwrap().val { ^4
            tail.unwrap().next = list1; 
            list1 = list1.unwrap().next;  ^5
        } else {  
            tail.unwrap().next = list2;
            list2 = list2.unwrap().next;  
        };  
  
        tail = tail.unwrap().next;  
    }  
  
    tail.unwrap().next = if list1.is_some() { list1 } else { list2 };  ^6
  
    pre_head.unwrap().next   ^7
}
```

`pre_head` is declared as a dummy node that points to the real head of the output list `^1`. In Go, this trick allows us to avoid checking that the output list is non-`nil` before attempting to append new nodes – we know for sure that `pre_head` is non-`nil`, or `Some` in our case. At the end of the function, we return `pre_head`'s `next` node `^7` to get the actual head.

Will this technique carry over to Rust? I think we both know the answer to that, but let's keep the fantasy alive a little longer.

We want to keep track of the tail of the output so we can append each node directly to the end of the list. Starting from the head and traversing the whole list for each node we appended would result in quadratic runtime. So I guess we just initialize `tail` as `pre_head` `^2`? Sure, seems legit. 🫠

While there are still nodes in both `list1` and `list2`, we execute the main body of the algorithm `^3`. If only one list has nodes left in it, we know that all of these nodes have values greater than any of the nodes we've already appended to the output. We just tack the remaining list onto the output tail `^6`! 

The loop is quite simple. We walk both lists, updating tail as we go.

And this algorithm is correct! But the Rust is garbage.

```text
error[E0382]: borrow of moved value: `list1`
   --> src/merge_two_sorted_lists.rs:32:11
    |
24  |     mut list1: Option<Box<ListNode>>,
    |     --------- move occurs because `list1` has type `Option<Box<ListNode>>`, which does not implement the `Copy` trait
...
32  |     while list1.is_some() && list2.is_some() {
    |     ------^^^^^-----------------------------
    |     |     |
    |     |     value borrowed here after move
    |     inside of this loop
33  |         if list1.unwrap().val < list2.unwrap().val {
    |                  -------- `list1` moved due to this method call, in previous iteration of loop
34  |             tail.unwrap().next = list1;
35  |             list1 = list1.unwrap().next;
    |             ----- this reinitialization might get skipped
    |
note: `Option::<T>::unwrap` takes ownership of the receiver `self`, which moves `list1`

# Many, many lines of compiler errors elided

error[E0382]: borrow of moved value: `list2`
error[E0382]: use of moved value: `list1`
error[E0382]: use of moved value: `list2`
error[E0382]: use of moved value: `tail`
error[E0382]: use of moved value: `pre_head`
```


@@@info
Once you've written enough Rust, working in languages without ownership semantics starts to feel uncomfortable. 

You worry you might be breaking the law, or that the borrow checker will jump out from behind a bin and shank you.
@@@


The moment we assign `pre_head` to `tail` `^2`, it's over. We've given away ownership of `pre_head`, and have nothing to return at `^7`.

When we unwrap the heads of `list1` and `list2` to compare them `^4`, we doom our loop to failure too:

```rust
pub fn unwrap(self) -> T
```

`Option::unwrap` takes `self` by value. That means we take ownership of both input list heads, but only attempt to put one back `^5`.

And I say "attempt", because that's fubared too. If the node at the head of `list1` contains the smallest value, we give ownership of `list1` to `tail.next` (taking ownership of `tail` in the process). But to move the head of  `list1` along to its next node, we need access to `list1`... which we've just given away.

## Merging linked lists with (almost) idiomatic Rust

Let's stop this nonsense. You understand the challenge that Rust's ownership model presents to this sort of algorithm. Let me walk you through a more idiomatic way to do it.

A correct solution reminds us to be alert to problems that can be solved without reaching for heavy-duty tools like `Rc` and `RefCell`, which both have runtime overhead.


@@@warning
`RefCell` in particular should be chosen as a last resort, because it shifts borrow checking from compile time to runtime. 

I prefer a program that won't compile when I'm inevitably wrong about lifetimes.
@@@

Overuse of these types is a telltale sign of an inexperienced Rust developer. Often, being smart with references and swaps achieves the desired result in a way that can be checked statically.

Behold:

```rust
pub fn merge_two_lists_loop_2(  
    mut list1: Option<Box<ListNode>>,  
    mut list2: Option<Box<ListNode>>,  
) -> Option<Box<ListNode>> {  
    let mut head = None;  
    let mut next_tail = &mut head;  ^8
  
    while list1.is_some() && list2.is_some() {  ^9
        let head1 = &mut list1;  
        let head2 = &mut list2;  ^10
  
        let smallest_head = if head1.as_ref().unwrap().val < head2.as_ref().unwrap().val {  ^11
            head1  
        } else {  
            head2  
        };  
  
        std::mem::swap(smallest_head, next_tail); ^12 
             
        let new_next_tail = &mut next_tail.as_mut().unwrap().next;
		std::mem::swap(smallest_head, new_next_tail); ^13
		next_tail = new_next_tail  ^14
    }  
  
    *next_tail = if list1.is_some() { list1 } else { list2 };  ^15
  
    head  
}
```

This is quite remarkable. It turns out that we can merge two linked lists without ever taking ownership of a node.

At `^8`, we initialize  `head` as `None`. This isn't equivalent to `pre_head`. This is the spot in memory where the true head of our output list will live.

`next_tail` is a mutable reference to this `Option::None`. That is, it is a reference to the place in memory where the next node should go.

The loop condition is the same – `Option::is_some()` takes `&self`, so there's no move here `^9`. But this time we start each iteration by taking mutable references to the head of each list `^10`. We don't have ownership of the input heads, we just point to them.

We'd like to avoid owning a node, so to perform the comparison between values `^11`, we change `head1` and `head2` from `&Option<Box<ListNode>>` into `Option<&Box<ListNode>>` using `Option::as_ref`.

We unwrap the `Option<&Box<ListNode>>` (which is guaranteed to be `Some` thanks to the loop condition), and compare their `val` fields through references.


@@@info
`PartialOrd::partial_cmp` (which provides the implementation for the comparison operators `<`, `>`, etc.) also takes `&self`. Again, no ownership required.
@@@


The *reference* to the lesser of the two nodes is assigned to `smallest_head`, which represents the node that we want to append to our output list. At this point, `smallest_head` is either a mutable reference to the head of `list1`, or a mutable reference to the head of `list2`.

Now the fun part.

At `^12`, we swap the node that `smallest_head` references with the node that `next_tail` references. In other words, we put the node `smallest_head` points to into the slot indicated by `next_tail`.

Rust won't allow us to create dangling pointers – `smallest_head` must still point to something even after we put its referent into place. That's what `swap` is good for. We initialized `next_tail` as `None`, so we take that `None`, and stick it into the space previously occupied by the smallest head.

Imagine that `list1` contained the smallest head on any given iteration of the loop. We've just swapped `&mut list1` and `next_tail`. At this point, `list1` is `None`. `next_tail` points to what used to be `list1`. `head` contains the merged list built up so far _and the rest of the list formerly known as `list1`_.

> `head` contains the merged list built up so far and the rest of the list formerly known as `list1`.

Not ideal. Let's fix it with another swap `^13`. The first argument to `swap` is `smallest_head` again, which is a reference to either `list1` or `list2`. Recall that this now points to `None`. The second argument is a mutable reference to the unsorted _tail_ of the list that we just swapped into `next_tail`: `&mut next_tail.as_mut().unwrap().next`. This is a reference to `next_tail`'s tail, which we'll call `new_next_tail`.

Breaking it down, `next_tail.as_mut()` turns `&mut Option<Box<ListNode>>` into `Option<&mut Box<ListNode>>`, which we unwrap, giving us access to the `next` node of the current tail. Remember, this `next` node represents the new head of one of the input lists.

If we tried the assignment `let new_next_tail = next_tail.as_mut().unwrap().next`, Rust would interpret this as an attempt to move `next_tail.next` out of `next_tail` through a mutable reference. This isn't allowed. Hence, we take a mutable reference to `next_tail.next` explicitly:

```rust
`let new_next_tail = &mut next_tail.as_mut().unwrap().next`
```

Now we've got something swappable `^13`. `smallest_head` was pointing to `None` (the previous value referenced by `next_tail`), so swapping causes `new_next_tail` to once again point to `None`.

`new_next_tail` was pointing to the new head of one of the input lists. This swap puts the new head back into the list it came from, since `smallest_head` is just a mutable reference to either `list1` or `list2`, depending on which head had the smaller value.

`next_tail` is still `Some` though. If we continued the loop here, the value we just added to the output would start being swapped around in the next iteration.

Thanks to our swaps, we know that `new_next_tail` is `None`, and that this is the exact spot in memory where the next node should be placed. We can just assign `new_next_tail` to `next_tail` at the bottom of the loop `^14`.

By doing this, we also guarantee that `next_tail` will always point to `None` at the top of the loop, and that the final node in the output list will correctly point its `next` field to `None`.


@@@warning
Setting `*next_tail = None` wouldn't work.

It's critical to remember that `next_tail` is not "the next node" but "the place in memory where the next node should go".

`*next_tail = None` would cause the node we just put there to be dropped. The next iteration of the loop would put the next node in the exact same place, which would again be dropped, leading to a final output list of `None`.

We must advance `next_tail` to point to the place where the _next_ node belongs.
@@@


Once one of the lists is exhausted, we just whack the remaining list onto the end of the output `^15`. It's safe to assign directly to `*next_tail` here, because we advanced `next_tail` at the end of the final loop iteration.

Return the head, and there you have it – an iterative, efficient solution that avoids working with owned input nodes.

## Merging linked lists readably with recursion

It's possible to solve this problem recursively, and more readably than the iterative approach.

```rust
pub fn merge(  
    mut list1: Option<Box<ListNode>>,  
    mut list2: Option<Box<ListNode>>,  
) -> Option<Box<ListNode>> {  
    match (list1, list2) {  ^16
        (None, None) => None,  ^17
        (Some(node), None) | (None, Some(node)) => Some(node),  ^18
        (Some(mut node1), Some(mut node2)) => {  ^19
            if node1.val < node2.val {  
                node1.next = merge(node1.next, Some(node2)); ^20
                Some(node1)  
            } else {  
                node2.next = merge(Some(node1), node2.next);
                Some(node2)  
            }  
        }  
    }  
}
```

The trick here is to see that we take ownership of _every_ node in the input lists on the way down, and give it to the output list on the way back up. There's no faffing around with borrows, because we never borrow anything.

At each recursion, we consider the heads of the two inputs lists as a tuple of nodes `^16`.

If both are `None`, then both lists are empty – we return `None` to take the final spot in the output list  `^17`.

In practice, this only happens if both input lists are empty to begin with. Each iteration removes at most one node from one of the input lists. The second match arm catches the case where one list is empty and one isn't `^18`. In this case, we just return the matched node, and it will bring the remainder of its list with it.

The interesting case is when both heads are `Some` `^19`. We identify the node with the lowest value, and set its `next` field to the result of a recursive `merge` call, passing `node.next` as one input, and the entirety of the list with the larger head as the second.

Note that matching the heads of each list takes ownership. We give that ownership away when we call `merge` recursively `^20`, but the immediate reassignment to `nodeN.next` means that the complete node is ours to pass back up the call chain once the recursion returns.

This more readable code comes at the cost of the stack space required to hold the recursive calls (i.e. it has *O(n)* space complexity).

However, more expensive code that performs _well enough_ for your use case is typically a better option than high-performance code that only half the team understands.

> Code that performs well enough is better than code half the team understands.

You're on _How To Code It_, so naturally you understand both, but keep this trade-off in mind when coding collaboratively.

## The best of both worlds: idiomatic, ergonomic iteration

With Rust, we can have nice things.

Aesthetes AnthonyMikh and calebsander teamed up on third solution [in the Discussion](https://github.com/howtocodeit/comments/discussions/6#discussioncomment-9930479). It's a good one, combining the readability of `match`-based recursion with the space complexity of a loop:

```rust
pub fn merge_two_lists(
    mut list1: Option<Box<ListNode>>,
    mut list2: Option<Box<ListNode>>
) -> Option<Box<ListNode>> {
    let mut head = None;
    let mut next_tail = &mut head;
    
    loop {
        match (list1, list2) {
            (Some(mut head1), Some(mut head2)) => { ^21
                let smallest_head = if head1.val < head2.val {
                    list1 = head1.next.take(); ^22
                    list2 = Some(head2); ^23
                    head1
                } else {
                    list2 = head2.next.take();
                    list1 = Some(head1);
                    head2
                };
                next_tail = &mut next_tail.insert(smallest_head).next; ^24
            }
            (tail, None) | (None, tail) => { ^25
                *next_tail = tail;
                return head
            }
        }
    }
}
```

Like the recursive implementation, this one takes ownership of the nodes. Unlike the recursive implementation, it does so in a loop.

Ownership of the heads of both lists is acquired at the top of each iteration `^21`, which requires us to put something back before the start of the next.

Let's assume `list1`'s head has the smaller value. We assign `head1`'s `next` node directly to `list1` `^22`, setting up `list1` for the next iteration and keeping `head1` for ourselves. `Option::take` gives us ownership of everything after `head1` and leaves `None` in its place. I find this more intuitive than `mem::swap`.

@@@info
`Option::take` is in fact implemented in terms of `mem::replace`:

```rust rust src/core/option.rs
impl<T> Option<T>
    #[inline]
    #[stable(feature = "rust1", since = "1.0.0")]
    #[cfg_attr(bootstrap, rustc_allow_const_fn_unstable(const_mut_refs))]
    #[rustc_const_stable(feature = "const_option", since = "1.83.0")]
    pub const fn take(&mut self) -> Option<T> {
        // FIXME(const-hack) replace `mem::replace` by `mem::take` when the latter is const ready
        mem::replace(self, None)
    }
}
```
@@@

Having no further use for the larger `head2`, we hand ownership straight back to `list2` `^23`.

Rather than do the `swap` dance with `next_tail`, we invoke `Option::insert` to put `smallest_head` in its place. We know that `next_tail` is `None` at the top of the loop, so there's no risk of overwriting a node. Finally, we assign a mutable reference to the new tail's `next` node to `next_tail`, setting up the next iteration `^24`.

In the case where either of the two lists is `None` `^25`, our work is done. Just slot the
whole of the remaining list into the spot indicated by `next_tail`, and return the head of the sorted list.

Beautiful work – thank you for the contribution.

No exercises this time – head on over to [LeetCode](https://leetcode.com/) or [HackerRank](https://www.hackerrank.com/) and sit with the discomfort of your Rust not being quite as strong as you hoped. It's a journey we all have to take.
