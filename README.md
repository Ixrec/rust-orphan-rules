# Coherence and Orphan Rules in Rust

This repo is an unofficial, experimental attempt to explain what the Coherence / Orphan Rules Dilemma is, and provide clear places where the community can aggregate specific kinds of feedback on this complex issue, inspired by and suggested on https://internals.rust-lang.org/t/blog-post-proposal-for-a-staged-rfc-process/7766.

We do not expect any prior knowledge of type theory or other programming languages. The summaries and explanations in this repo are aimed at anyone with a basic understanding of Rust. Specifically, if you know most of what's in [The Book](https://doc.rust-lang.org/book/), then everything here should make sense, and your feedback is welcome (and if something doesn't make sense, please open an issue).

Discussion on this repo might lead to PRs to improve the compiler, the development of third-party crates, the drafting of new RFCs, convincing everyone that the status quo is the least of many evils, or merely providing a convenient link the next time someone asks about these problems. Any of those would be considered a success.

# What is Coherence?

In Rust, "Trait Coherence" (or simply "Coherence") is the property that there is **at most one implementation** of a trait for any given type.

Any programming language that has a feature like traits or interfaces must either:
- Enforce coherence by simply refusing to compile programs that contain conflicting implementations
- Embrace incoherence and give programmers a way to manually choose an implementation when there's a conflict

Rust chooses to enforce coherence.

## What's wrong with incoherence?

We could provide some kind of disambiguation syntax like `<Type as Trait using impl from Crate>::method()` if we wanted to support incoherence. But there are deeper problems with incoherence than the need for some funky syntax:

- The incoherence may occur very far away from the crates with conflicting `impl`s, forcing this choice on a user that doesn't and shouldn't even know that any of these traits, types or crates exist, much less what will happen if they ask for CrateA's `impl` instead of CrateB's.

- The oft-cited "hashtable problem": If the trait method is being invoked by some other crate on your behalf, you can't use the explicit disambiguation syntax because the call isn't in your code. The classic concrete example is `std::collections::HashMap` invoking `Hash` `impl`s on your types.

	- It's been suggested that we could allow something like `prefer CrateA impl of Trait for Type;` in your code to affect the behavior of code in all crates you depend on. That doesn't actually solve the problem, but only pushes it down one level. As soon as two crates write conflicting `prefer` statements, you've got a conflict again.

Incidentally, Googling "the hashtable problem" will turn up a lot of the early Rust design discussions surrounding coherence and orphan rules (as well as generics and trait objects and a whole bunch of other questions that hadn't been settled back then).

# What are the Orphan Rules?

Rust enforces trait coherence through two sets of rules:

- The "overlap rules" forbid you from writing two `impl` blocks that "overlap", i.e. apply to some of the same types. For example, `impl<T: Debug> Quack for T` overlaps with `impl<T: Display> Quack for T` because some types might implement both Debug and Display, so you can't write both.

	- The current plan is to dramatically relax these rules with a feature called "specialization". We will not discuss the details of specialization here, but it seems uncontroversial that we want some form specialization in Rust, so it's worth keeping in mind.

- The "orphan rules", very roughly speaking, forbid you from writing an `impl` where both the trait and the type are defined in a different crate. This is meant to:

	- Prevent "dependency hell" problems where multiple crates write conflicting `impl`s, so nobody can ever use both crates in the same program, and the crate authors don't even realize they've created this footgun until someone tries to combine them.

	- Allow crates to add `impl`s for their traits and types without it being considered a breaking change. Without the orphan rules, no crate would ever be allowed to add an `impl` in a minor or patch version because that would break any program that contained an overlapping `impl`.

The precise statement of the orphan rules is rather technical because you have to deal with generics like `impl Trait<Foo, Bar> for Type<Baz, Quux>` (["Little Orphan Impls"](http://smallcultfollowing.com/babysteps/blog/2015/01/14/little-orphan-impls/) is a good introduction to this design space), and [the #[fundamental] attribute](https://github.com/rust-lang/rfcs/pull/1023). Plus, [we might be changing them again soon](https://github.com/rust-lang/rfcs/pull/2451).

Most Rustaceans stick to the simplification that "either the type or trait must be from the same crate" since it's very easy to remember and is accurate in the most typical cases. Here, we'll only bring up the technical details for specific issues where they do become relevant.

# Why are the Orphan Rules controversial?

This is where things get interesting, and where we want your input.

To dramatically oversimplify it: there are a lot of `impl`s that people want to write, but they currently cannot write because of these rules.

However, in many cases this desire is an "XY problem", and there's actually a much better solution that doesn't involve writing an orphan impl at all. **One of my biggest hopes with this repo is to clear up which use cases are XY problems, which are merely theoretical and which are genuine pain points**. I suspect that alone will make it a lot clearer what, if anything, Rust should change.

- In some cases, the `impl` really does need to be provided by `Type`'s crate or `Trait`'s crate for reasons unrelated to the orphan rules, so the orphan rules complaint is simply a red herring. See the [core traits issue] for detailed explanations (and feedback if you disagree), but in short:

  - I believe this applies to `std` traits like `Ord`, `Eq`, `Hash`, etc, as well as serde's `Serialize` and `Deserialize` (which seem to be the most common source of orphan rule complaints).

  - This is why the Rust API Guidelines [recommend that everyone implement these traits](https://rust-lang-nursery.github.io/api-guidelines/interoperability.html) when appropriate.

- In many cases, the "newtype pattern" is theoretically the best solution. This means creating a brand new type `NewType` within your crate that is nothing but a wrapper around `Type`, so you can provide whatever `impl`s you want for it. This trivially and elegantly solves "the hashtable problem" because newtypes are completely separate types, so there's simply no ambiguity about what code is supposed to call what impl, no matter how many other crates create similar newtypes or even newtypes of your newtypes.

  - There are many use cases where creating such a newtype is tedious because several methods and traits must be reimplemented for it by simply delegating to the original type. If you have such a use case, please describe it on the [tedious newtypes issue]().

  - There are proposals for adding a "delegation" feature to Rust, which should help reduce the tedium of newtyping. Please post on the [how much does delegation help?]() issue whether or not delegation would solve your use case.

  - Cases where newtypes are the theoretically correct solution are often cases where, in languages like C++ or Java with "traditional inheritance", you would use inheritance to create a subclass implementing the same interface as the base class. Today, traditional inheritance often seems more convenient than Rust's trait system because it delegates all the methods by default. This is part of the reason delegation is expected to improve newtyping.

  - As far as I know, there are no practical use cases where newtyping is _impossible_, only cases where it's tedious. If you believe you know of a counterexample, please post on the [are newtypes always an option?]() issue.

- In some cases, the desired `impl` probably should exist, and client code should not have to provide it via a newtype, but there's no obvious "home" for the `impl` because the trait and type are defined in separate crates.

  - If the trait's crate is "clearly more fundamental" than the type's crate, or vice versa, then the "less fundamental" crate is probably the best home for that impl. For example, if I write a `linear-algebra` crate with `Matrix` and `Vector` types, I should probably provide impls for `num-traits` like `MulAdd` and `Pow`.

    - One downside is that this forces all users of `linear-algebra` to depend on `num-traits`, even if they don't care about those `impl`s. The best known proposal for solving this is [RFC #1787 "Automatic Features"](https://github.com/rust-lang/rfcs/pull/1787), which was postponed not due to any problems with the RFC itself but because cargo features in general need a rethink.

  - Sometimes neither the type's crate or the trait's crate is a good home. For example, `chrono` types should implement `diesel` traits, so that you can easily pass date/time types to your database code, but it's generally agreed that those `impl`s don't belong in either crate. As far as I know, **Rust has no good answer for this today**, not even a proposal. Please post on the [official orphans issue]() if you know of any compelling use cases that fit this category or have any thoughts on how we might solve it.

- In some cases, the use case will likely be solved by specialization. Please post on the [How much does specialization help?]() issue if you know of such a use case.

- In some complex cases, the desired `impl` doesn't "feel" like an orphan impl, although it technically is one by today's rules. These are the cases that probably should be addressed by changing the precise statement of the orphan rules.

  - See https://github.com/rust-lang/rfcs/issues/1856 and https://github.com/rust-lang/rfcs/pull/2451 for very recent, unfinished discussions about these cases.

That's all the high-level cases I'm aware of. If you think I missed something, please open an issue.

# A Brief History of Rust's Orphan Rules

I'm sure this is incomplete. Pull requests welcome!

Mar 2014 - [RFC #19 "Opt-in builtin traits" is posted](https://github.com/rust-lang/rfcs/pull/19) and [accepted](https://github.com/rust-lang/rfcs/pull/19#issuecomment-38627909). This RFC discussed the special case in the coherence rules we need for OIBITs/auto traits.

Apr 2014 - [RFC #48 "Trait reform"](https://github.com/rust-lang/rfcs/pull/48) covers a _lot_ of ground, but one of its many goals was to "Expand coherence rules to operate recursively and distinguish orphans more carefully." I think this is the first RFC to explicitly define and discuss coherence, and state that it's enforced through a combination of overlap and orphan rules.

Jun 2014 - [RFC #127 "Opt-in builtin traits, take 2: default and negative impls" is posted](https://github.com/rust-lang/rfcs/pull/127). At least for me, it's much easier to understand the special case here than it was in RFC #19: "Negative impls are subject to the usual orphan rules, but they are permitting to be overlapping. This makes sense because negative impls are not providing an implementation ..."

Sep 2014 - [RFC #127 "Opt-in builtin traits, take 2: default and negative impls" is accepted](https://github.com/rust-lang/rfcs/pull/127).

Jan 2015 - [RFC #586 "Negative bounds" is posted)[https://github.com/rust-lang/rfcs/pull/586]. Technically only changes the overlap rules, not the orphan rules, but this might cover some of the use cases that make people suggest orphan rule changes.

Jan 2015 - [Niko's "Little Orphan Impls" blog post](http://smallcultfollowing.com/babysteps/blog/2015/01/14/little-orphan-impls/). If you're still wondering why coherence/orphan rules are such a hard problem, this is the one link you should definitely read.

Mar 2015 - [RFC #1023 "Rebalancing Coherence" is posted](https://github.com/rust-lang/rfcs/pull/1023). This is the RFC that introduced the #[fundamental] attribute. It was accepted and implemented a month later in April.

Apr 2015 - [RFC #586 "Negative bounds" is postponed)[https://github.com/rust-lang/rfcs/pull/586#issuecomment-91562382] because "There is too much in flight."

Apr 2015 - [RFC #1023 "Rebalancing Coherence" is accepted](https://github.com/rust-lang/rfcs/pull/1023#issuecomment-88278874) and implemented.

May 2015 - Rust 1.0 released.

Jun 2015 - [RFC #1148 "Mutually exclusive traits" is posted](https://github.com/rust-lang/rfcs/pull/1148).

Jun 2015 - [RFC #1210 "impl specialization" is posted](https://github.com/rust-lang/rfcs/pull/1210). Technically only changes the overlap rules, not the orphan rules, but this might cover some of the use cases that make people suggest orphan rule changes.

Feb 2016 - [RFC #1210 "impl specialization" is accepted](https://github.com/rust-lang/rfcs/pull/1210#issuecomment-187777838).

Apr 2016 - [RFC #1148 "Mutually exclusive traits" is postponed](https://github.com/rust-lang/rfcs/pull/1148#issuecomment-207610749) because "it makes sense to wait until the specialization implementation has progressed a bit farther before we make any decision here." but "there will be a role for negative reasoning", and "the broad outlines of this RFC are correct."

Jan 2017 - [Rust issue #1856](https://github.com/rust-lang/rfcs/issues/1856) "Orphan rules are stricter than we would like" is created, and most of the discussion on it happens this same month. This appears to contain the most recent opinions of several core team members on the precise orphan rules and how to improve them.

May 2017 - [RFC #2451 "Re-Rebalancing Coherence" is posted](https://github.com/rust-lang/rfcs/pull/2451). As of this writing, this RFC is still open.

Jul 2017 - [RFC #2052 "Evolving Rust through Epochs" is posted](https://github.com/rust-lang/rfcs/pull/2052). I'm including this because of the explicit statement that "Trait coherence rules, like the "orphan" rule" cannot be changed via editions.

Apr 2018 - [Aturon's "Sound and ergonomic specialization for Rust" blog post](http://aturon.github.io/2018/04/05/sound-specialization/). There are many other blog posts about why specialization soundness is hard, but I think this is the most recent one and represents the current status.

## License

This repository is licensed under either of

- Apache License, Version 2.0, (LICENSE-APACHE or http://www.apache.org/licenses/LICENSE-2.0)
- MIT license (LICENSE-MIT or http://opensource.org/licenses/MIT)

at your option.

## Contributions

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions.
