# Coherence and Orphan Rules in Rust

This repo is an unofficial, experimental attempt to explain the design challenges of coherence and the orphan rules in Rust, and provide clear places where the community can aggregate specific kinds of feedback on this complex issue (inspired by and suggested on https://internals.rust-lang.org/t/blog-post-proposal-for-a-staged-rfc-process/7766). We do not expect any prior knowledge of type theory or other programming languages; if you know most of what's in [The Book](https://doc.rust-lang.org/book/), then everything here should make sense (if not, please open an issue).

Discussion on this repo might lead to PRs to improve the compiler, the development of third-party crates, the drafting of new RFCs, convincing everyone that the status quo is the least of many evils, or merely providing a convenient link the next time someone asks about these problems. Any of those would be considered a success.

# What is Coherence?

In Rust, "trait coherence" (or simply "coherence") is the property that there is **at most one implementation** of a trait for any given type.

Any programming language that has a feature like traits or interfaces must either:
- Enforce coherence by simply refusing to compile programs that contain conflicting implementations
- Embrace incoherence and give programmers a way to manually choose an implementation when there's a conflict

Rust chooses to enforce coherence.

## What's wrong with incoherence?

We could provide some kind of disambiguation syntax like `<Type as Trait using impl from Crate>::method()` if we wanted to support incoherence. But there are deeper problems with incoherence than the need for some funky syntax:

- The incoherence may occur very far away from the crates with conflicting `impl`s, forcing this choice on a user that doesn't and shouldn't even know that any of these traits, types or crates exist, much less what will happen if they ask for CrateA's `impl` instead of CrateB's.

- The oft-cited "hashtable problem": If the trait method is being invoked by some other crate on your behalf, you can't use the explicit disambiguation syntax because the call isn't in your code. The classic concrete example is `std::collections::HashMap` invoking `Hash` `impl`s on your types.

	- It's been suggested that we could allow something like `prefer CrateA impl of Trait for Type;` in your code to affect the behavior of code in all crates you depend on. That doesn't actually solve the problem, but only pushes it down one level. As soon as two crates write conflicting `prefer` statements, you've got a conflict again.

# What are the Orphan Rules?

Rust enforces trait coherence through two sets of rules:

- The "overlap rules" forbid you from writing two `impl` blocks that "overlap", i.e. apply to some of the same types. For example, `impl<T: Debug> Trait for T` overlaps with `impl<T: Display> Trait for T` because some types might implement both `Debug` and `Display`, so you can't write both.

	- The current plan is to dramatically relax these rules with a feature called "specialization".

- The "orphan rules", very roughly speaking, forbid you from writing an `impl` where both the trait and the type are defined in a different crate. This is meant to:

	- Prevent "dependency hell" problems where multiple crates write conflicting `impl`s, so nobody can ever use both crates in the same program, and the crate authors don't even realize they've created this footgun until someone tries to combine them.

	- Allow crates to add `impl`s for their traits and types without it being considered a breaking change. Without the orphan rules, no crate would ever be allowed to add an `impl` in a minor or patch version because that would break any program that contained an overlapping `impl`.

The precise statement of the orphan rules is rather technical because of generics like `impl Trait<Foo, Bar> for Type<Baz, Quux>` (see ["Little Orphan Impls"](http://smallcultfollowing.com/babysteps/blog/2015/01/14/little-orphan-impls/)), and [the #[fundamental] attribute](https://github.com/rust-lang/rfcs/pull/1023), and [OIBITs/auto traits](https://github.com/rust-lang/rfcs/pull/127), and probably something else I'm forgetting about. Plus, [we might be changing them again soon](https://github.com/rust-lang/rfcs/pull/2451).

Most Rustaceans stick to the simplification that "either the type or trait must be from the same crate" since it's very easy to remember and is accurate in the most typical cases.

# Why are the Orphan Rules controversial?

In short: there are a lot of `impl`s that people want to write, but they currently cannot write because of these rules.

This is where things get interesting, and where **we want your input**.

In many cases this desire is an "[XY problem](https://meta.stackexchange.com/a/66378/279182)", and there's actually a much better solution that doesn't involve anyone writing an orphan impl. **I'm hoping this repo can help clarify which use cases are XY problems, which are merely theoretical and which are genuine pain points**. I suspect that alone will make it obvious what, if anything, Rust should change.

- In some cases, the `impl` really does need to be provided by `Type`'s crate or `Trait`'s crate for reasons unrelated to the orphan rules. This includes most `std` traits like `Ord` and `Eq`, as well as `serde`'s `Serialize` and `Deserialize`. See the [core traits issue](https://github.com/Ixrec/rust-orphan-rules/issues/2) for details and feedback.

- In many cases, the "newtype pattern" is theoretically the best solution. This means creating a brand new type `NewType` within your crate that is nothing but a wrapper around `Type`, so you can provide whatever `impl`s you want for it. This trivially and elegantly solves "the hashtable problem" because newtypes are completely separate types, so there's simply no ambiguity about what code is supposed to call what impl, no matter how many other crates create similar newtypes or even newtypes of your newtypes.

  - There are many use cases where creating such a newtype is tedious or fragile because several methods and traits must be reimplemented for it by simply delegating to the original type. If you have such a use case, please describe it on the [tedious newtypes issue](https://github.com/Ixrec/rust-orphan-rules/issues/3).

  - There are proposals for adding a "delegation" feature to Rust, which should help reduce the tedium of newtyping. Please post on the [How much would delegation help?](https://github.com/Ixrec/rust-orphan-rules/issues/4) issue whether or not delegation would solve your use case.

  - As far as I know, there are no practical use cases where newtyping is _impossible_, only cases where it's tedious, fragile, etc. If you believe you know of a counterexample, please post on the [Are newtypes always an option?](https://github.com/Ixrec/rust-orphan-rules/issues/5) issue.

- In some cases, the desired `impl` probably should exist, and client code should not have to provide it via a newtype, but there's no obvious "home" for the `impl` because the trait and type are defined in separate, unrelated crates.

  - If the trait's crate is "clearly more fundamental" than the type's crate, or vice versa, then the "less fundamental" crate is probably the best home for that impl. For example, if I write a `linear-algebra` crate with `Matrix` and `Vector` types, I should probably provide impls for `num-traits` like `MulAdd` and `Pow`. The one problem with this case (that all users of `linear-algebra` now depend on `num-traits`) will be solved by [RFC #1787 "Automatic Features"](https://github.com/rust-lang/rfcs/pull/1787).

  - Sometimes neither the type's crate or the trait's crate is a good home. For example, `chrono` types should implement `diesel` traits, so that you can easily pass date/time types to your database code, but it's generally agreed that those `impl`s don't belong in either crate. As far as I know, **Rust has no good answer for this today**. Please post on the [official orphans issue](https://github.com/Ixrec/rust-orphan-rules/issues/7) if you know of any compelling use cases that fit this category or have any thoughts on how we might solve it.

- In some cases, the use case will likely be solved by specialization. Please post on the [How much does specialization help?](https://github.com/Ixrec/rust-orphan-rules/issues/6) issue if you know of such a use case.

- In some complex cases, the desired `impl` doesn't "feel" like an orphan impl, although it technically is one by today's rules. These are the cases that probably should be addressed by changing the precise statement of the orphan rules. See https://github.com/rust-lang/rfcs/issues/1856 and https://github.com/rust-lang/rfcs/pull/2451 for very recent, unfinished discussions about these cases. Since this discussion is already active and I don't want to fragment it, I'm going to avoid creating dedicated issues about these details here (at least until the dust settles on RFC #2451).

That's all the high-level cases I'm aware of. If I missed something, please open an issue.

# Miscellaneous

https://github.com/Ixrec/rust-orphan-rules/issues/1 is a timeline of Rust RFCs, issues and blog posts relevant to the orphan rules, with many links

https://github.com/Ixrec/rust-orphan-rules/issues/9 describes how other languages handle these issues

## License

This repository is licensed under either of

- Apache License, Version 2.0, (LICENSE-APACHE or http://www.apache.org/licenses/LICENSE-2.0)
- MIT license (LICENSE-MIT or http://opensource.org/licenses/MIT)

at your option.

## Contributions

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions.
