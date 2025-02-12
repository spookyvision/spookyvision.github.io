+++
title = "Leaving Rust gamedev after 3 years: a measured response"
slug = "rust-gamedev"
+++

# TODO
- update playground to use new lock primitives? or at least RWLock
- or maybe the lock primitives were sth else... anyway mention https://blog.rust-lang.org/2024/07/25/Rust-1.80.0.html

[This article](https://loglog.games/blog/leaving-rust-gamedev/) went pretty viral when it , and for good reason. It's written by an experienced developer who devoted a lot of time to Rust, really tried to like it, and ultimately didn't. 
As such it provides deep insight into shortcomings of both the Rust language and available game engines. A lot of work went into it, and I honestly think it's great that it exists - even though I like Rust a lot more than the author, valid criticism is important for any kind of system to stay healthy and improve.

That said, I don't agree with all the points made, and even though my game development experience is much more limited (not *purely* armchair, though), I feel like adding my perspective. I don't like this to be a one-sided effort though, so I'll be quote-replying to sections where I disagree but also ones where I'm on the same page.

Let's dive in. If you're short on time, check out the TODO LINK conclusion!

## Intro

>  I'm looking at things from the perspective of "I want to make a game in 3-12 months maximum and release it so that people can play it and I can make some money from it.". 

This is a valid warning. Especially if you're new to Rust you're likely not going to be super productive in it at first - it's not the easiest language to pick up. However it is definitely possible to ship a successful indie game in it, even as a solo developer with no previous Rust experience, as evidenced by [(the) Gnorp Apologue](https://www.reddit.com/r/rust/comments/18ul125/writing_a_game_in_rust_the_gnorp_apologue/).

> The community as a whole is overwhelmingly focused on tech, to the point where the "game" part of game development is secondary. As an example of this, I remember one time a discussion around the Rust Gamedev Meetup, that while was probably done jokingly was still imo illustrative of the issue, with something like "someone wants to present a game at the meetup, is that even allowed?"

A bit of a sweeping generalization, but fair enough. Also the question is pretty funny. I do want to add though that this isn't a Rust specific issue by far, everyone wants to write game engines. Guilty as charged myself!
I just wouldn't go so far as to imply or outright state you can't use Rust for anything more than tech demos. 
Now, would I *recommend* doing game development in Rust? Not wholeheartedly! For many kinds of project I'd say, first check out e.g. Godot - a feature rich editor can be a very valuable asset.

## Once you get good at Rust all of these problems will go away

> The most fundamental issue is that the borrow checker forces a refactor at the most inconvenient times. Rust users consider this to be a positive, because it makes them "write good code", but the more time I spend with the language the more I doubt how much of this is true. Good code is written by iterating on an idea and trying things out, and while the borrow checker can force more iterations, that does not mean that this is a desirable way to write code.

I actually don't consider a forced refactoring a 100% positive thing, but I can see where this attitude comes from. In a grand scheme of things I agree with the author's point: it's often not exactly easy to try something out quickly in Rust. You need to do a *lot* more typing than in for example Python (though I usually don't mind this *much*, thanks to rust-analyzer), and the language demands you be a bit of an architect pretty much at all times. On top of that build times can become an issue with larger frameworks, turning repeated iteration on ideas into a chore. Dynamic linking (explained in Bevy docs) can help, but it's still not all roses. The holy grail of hot reloading remains elusive, though recently a [WASM based experiment](https://github.com/chamons/game-hotreload-example) came out of the woodwork - curious about future developments, WASM is a platform I hold in pretty high regard generally. All in all it's a very valid point though.

On the other hand I've never had a more pleasant experience doing refactorings than in Rust, thanks to the type system catching all my mistakes and enabling me to do it with an absolutely minimal amount of brainpower. I've literally ported firmware to a different microcontroller platform being extremely sleep deprived and it worked on the first successful build. The articles states pretty much this later on too though, so we're not arguing.

And regarding lifetime anno(y)tations in particular, if you're not on a performance critical path it's always an option to use `(A)Rc`, `Box`, `clone()`, a `Context` object, etc. I do 100% agree that lifetime related refactoring is a big PITA, so they should really be avoided as long as you can.

> In Rust, sometimes just doing a thing is not possible, because the thing you might need to do is not available in the place where you're doing the thing, and you end up being *made* to refactor by the compiler, even if you know the code is mostly throwaway.

You can totally have shared mutable globals in Rust. [Here, I wrote you one](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=65a9114b61727ba76407eb922de2ec10). Granted: `std::sync::OnceLock` is kinda new. Before it appeared we had to rely on 3rd party crates like `lazy_static` or `once_cell`, and it's admittedly hard to keep track of all useful (or even essential) crates. This is indicative of a young ecosystem where not everything important has a best practice yet, solutions might be clumsy or not even established yet. Definitely in general an area where Rust can still improve a lot!

## Rust being great at big refactorings solves a largely self-inflicted issues with the borrow checker

This section mostly restates refactoring woes from the previous one so I won't rehash those.

> I'd argue as far as *maintainability being the wrong value for indie games*, as what we should strive for is iteration speed. 

Ideally we'd have both (you don't want a terrible spaghetti situation when your project becomes an unexpected hit), but in terms of priorities: yes - and iteration speed is a pain point, as discussed before.

> Other languages allow much easier workarounds for immediate problems without necessarily sacrificing code quality. In Rust, it's always a choice of *do I add an 11th parameter to this function, or add another `Lazy<AtomicRefCell<T>>`, or do I put this in another god object, or do I add indirection and worsen my iteration experience, or do I spend time redesigning this part of code yet again.*

I would really have loved 1-2 concrete examples here, especially a comparison with the author's preferred approach because I do think this particular kind of pain is somewhat overstated and *can* typically be solved in a satisfactory way. I mean: if you're used to just having mutable pointers to everything you want then yes, the required object hierarchy in Rust *will* make you sad but I'd argue that's not exactly good code quality on your end. I do sympathize: sometimes you just want to hack something in and rustc won't let you. It can be frustrating. It can be a case of "I know this is safe, just let me do it", and maybe you're even right (but oh boy will things explode if you're not. I spent months chasing a double free in a C++ codebase that always manifested in my component but ultimately turned out to be in the underlying cocos2d-x library).

## Indirection only solves some problems, and always at the cost of dev ergonomics

I like this section a lot - it is rather Bevy-specific though, so less indicative of a problem with the language and more with the ecosystem. Still, it's a great concrete example of a situation where an ECS might cause you a lot of frustration. Since one of the main Bevy maintainers was quite sympathetic to the article I hope this is one of the parts that will lead to more sophisticated APIs in Bevy, or even the development a mature non-ECS game engine.
Bevy is a very impressive project, but in terms of game engines it is *very* young, and especially if LogLog Games tried to work with it 3 years ago it must've really been too immature to work with. A [draft roadmap to version 1.0](https://mastodon.gamedev.place/@alice_i_cecile/112515740546806334) was just published in May 2024!

## ECS solves the wrong kind problem

"If all you have is a hammer..." is surely a mentality prone to causing problems. Though I wouldn't go so far as to say ECS is *never* the right solution either.

> `Rc<RefCell<T>>` combined with weak pointers. While this could work, in games performance matters, and the overhead of these is non-trivial due to memory locality.

This could be a very interesting research project - can we optimize locality somehow? It is an important aspect for hot loops. But I wouldn't rule this solution out entirely, at least not without benchmarking. And let's not rule out `Rc<Refcell<Vec<T>>>` for locality.

What follows is a discussion of generational arenas, which sound great for certain kinds of situations. Definitely a tool to keep in your belt, along with bump allocators or other custom memory management solutions. Impatiently waiting for [`allocator_api`](https://github.com/rust-lang/rust/issues/32838) to get stabilized…

Back to ECS:

> It should also be noted that the approach of "splitting up components into as small as possible for maximum reuse" is something that is very often cited as a virtue. I've been in countless arguments where someone tried to convince me how I absolutely should be separating `Position` and `Health` out of my objects, and how my code is spaghetti if I'm not doing that. 
> Having tried those approaches quite a few times, I'm now very much on the *hard disagree* side in complete generality, unless maximum performance is of concern, and then I'd concede on this point only for those entities where this performance concern matters. 

Philosophically, to me, components are pretty similar to *fields*, and you wouldn't want a combined `position_health` field either, no? It can (and should) be argued that a single global namespace for those fields isn't ideal, it certainly doesn't sound scalable. Also I feel coming up with a "perfect" component design can become something done just for its own sake with no tangible benefits.

Let's see what the future brings.

## Conclusion 
TODO
    - some good points, some bad
    - bevy centric
    - weird hangups about perceived attitude
        - but some devs are assholes, eg lots of python hate whyyyy
        - let's RIIR with love bruh
    - The elephant in the room: dispatch in Rust can be really annoying
TODO 
- lots of non intuitive front loading required
- rustc fights you tooth and nail against quick hacks (but you CAN have them)
- Rust is immature 
    - 1.80 LazyCell bla
    - tooling: cargo, debug, bla - RA amazing but debugging hm bleh (in general mention downsides the article did NOT go into;)
    - libraries, C/++ reigns supreme
        - juce
        - fmod
        - (game) UI - https://mastodon.gamedev.place/@alice_i_cecile/112837834583301070
            - no a10y
