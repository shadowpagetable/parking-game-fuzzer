# parking-game-fuzzer

This is a template for a fuzzer for [parking-game](https://github.com/addisoncrump/parking-game) puzzles.
While originally built as a homework assignment, this is released publicly for folks looking to learn how to implement
their own components in [LibAFL](https://github.com/AFLplusplus/LibAFL).

## Purpose

This repo serves as examples for implementing various components of LibAFL yourself, as well as working through some of
the logic you may need to consider in your own applications.
This targets a puzzle game, for which fuzzing is almost certainly a poor application, but the game highlights certain
design patterns that crop up in the testing of real programs.
Notably, this repo teaches you how to:

- Build custom executors (including integration with custom observer types)
- Build custom observers
- Build custom feedbacks which inspect the results of those observers and interact with user statistics
- Build custom input types
- Build custom mutators for those input types
- Basic snapshot fuzzing strategies and mutators specialized for them

Obviously, since we are working with a game, the components are somewhat contrived; in real applications, you will face
other difficulties specific to your target.
This is just around to help people get their bearings with the design of LibAFL and why we implement things the way we
do, and the challenges therein are to give a sense of how to implement new components.

### Anti-Purpose

This repo is **not** designed to teach you how to fuzz real programs.
The execution and feedback mechanisms here are deeply contrived and do not represent optimal strategies.
This strictly serves as a way to think about building your own LibAFL components, not fuzzing as a whole.

### Intended Audience

This is intended for practitioners who are at least already aware of fuzzing, have a basic knowledge of Rust, and are
looking to build custom fuzzing components with LibAFL.
For folks wanting to learn fuzzing strategies for specific applications, consider reading
the [Fuzzing Book](https://www.fuzzingbook.org/) or trying to apply existing LibAFL executors.

## License

The code within this repository is licensed under CC0.
Reuse how you wish, but please give credit back as possible.

The maps provided in the [maps](maps/) directory are ported
from [Jim Storer's personal page](https://www.cs.brandeis.edu/~storer/JimPuzzles/ZPAGES/zzzTokyoParking.html).
I could not find an associated license.
Redistribute at your own risk.
The maps are distributed here as educational materials for non-commercial use.

## Exercises

These exercises center around solving puzzles from Tokyo Parking (now licensed as Thinkfun's Rush Hour).
Watch [the promotional video on Rush Hour](https://www.youtube.com/watch?v=HI0rlp7tiZ0) to get a sense of how to solve
these puzzles, and maybe try solving them yourself.
They aren't so simple!

The [parking-game](https://docs.rs/parking-game/latest/parking_game/) crate implements the rules of movement for these
puzzles.
The implementation ensures that the board stays in a "sane" state by applying modifications to a view over the state.
A loader for human-readable puzzles is provided for you in the fuzzer template so that you can load the original Tokyo
Parking board layouts into the fuzzer by providing their path as command-line arguments.

These exercises will walk you through the stages of building a fuzzer for a new target with LibAFL.
They start out easy and guided, but get harder and with less direction over time.
First, you will implement the components for fuzzing these puzzles; tests are provided to ensure that your
implementations are reasonably sound, but are not extensive.
Then, you will link together the fuzzer logic in [main.rs](src/main.rs) to fuzz these puzzles.
Already, this simple implementation will be able to solve most puzzles reasonably quickly, but you will develop extra
feedback mechanisms that show that there are still opportunities for improved performance.

In the last remaining parts, we explore deeper optimizations that have real-world correspondences to optimizations in
certain targets.
First, you will improve your mutator to avoid "wasting" executions on invalid inputs.
With this implemented, a new optimization becomes available: resumed execution, or snapshot fuzzing.
The final exercise deals with the optimization of snapshot fuzzing, and discusses real-world correspondences.

### Exercise 0: Components

Before completing this exercise, ensure that your environment is correctly configured by running `cargo test`.
You should see that the following tests fail:

- executor::test::simple_run_check
- feedbacks::test::example_observation
- feedbacks::test::simple_solved
- feedbacks::test::simple_unsolved
- observers::test::distinguish_states
- observers::test::simple_observation

These are the tests for the components that you are about to implement.

Tasks for this section are denoted as comments in the code as `TODO(pt.0)` or as `todo!("(pt.0) ...")` macros.
Make sure to review the other comments to get a better sense of what each part of the boilerplate does.
If you ever get lost with what each component does, you can run `cargo doc --open` to get an overview of each.
These may be implemented in any order, as the components are tested individually.
Nevertheless, I recommend implementing the components in the order listed below.

#### Executor

The [executor](https://docs.rs/libafl/latest/libafl/executors/trait.Executor.html) component in LibAFL represents
exactly what it sounds like: the execution of the input in the target.
In our case, we need a custom executor to "run" the input (i.e., apply the sequence of moves) on the puzzle.
Your first task is to implement the executor in [executor.rs](src/executor.rs) by simply applying the moves in-order on
the board.
The corresponding boilerplate is provided for you.

Once implemented successfully, the test `executor::test::simple_run_check` should pass.

#### Guidance

An [observer](https://docs.rs/libafl/latest/libafl/observers/trait.Observer.html) in LibAFL serves to collect
information about an execution to be processed later by
a [feedback](https://docs.rs/libafl/latest/libafl/feedbacks/trait.Feedback.html).
The purpose of these observers can be anything to augment the fuzzer's progress, but we'll start by investigating the
use of observers for _guiding_ the fuzzer -- in this case, getting closer to a puzzle solution.

Guidance for mutational fuzzers, like the one we're building here, effectively boils down to identifying inputs which
are "interesting" and therefore should be retained for further mutation.
The core idea of this strategy is that _novelty_ is a strong indicator for where other new behavior might be observed.
Here, since we don't have a good idea of what "getting closer" to a puzzle solution looks like, we'll start by simply
saying that _any_ previously unseen puzzle state is interesting.
The easiest way to do that is simply to check if the hash of the puzzle has been observed -- meaning we treat any new
state as interesting.
There might be better guidance mechanisms, but for now, we'll rely on this.

LibAFL provides existing utilities for measuring if a previous hash has been observed with
the [ObserverWithHashField](https://docs.rs/libafl/latest/libafl/observers/trait.ObserverWithHashField.html) trait.
For this step, complete the ObserverWithHashField implementation for the FinalStateObserver located
in [observers.rs](src/observers.rs).
Avoid hashing data which is redundant (i.e., the car's lengths never change).

Once implemented successfully, the test `observers::test::distinguish_states` should pass.

#### Objective

Some observers and feedbacks are used for determining if an _objective_ has been reached rather than for guidance.
An objective in classical fuzzing is simply a crash, but can be anything that represents what we want to find.
In this case, our objective is to be able to drive the "objective car" out of the board.

In this task, you need to implement ViewObserver -- an observer which measures what each car can see.
We will use this for our objective by determining if the objective car can drive out of the board (i.e., it can see the
edge of the board ahead of it).
The corresponding SolvedFeedback is already implemented for you; all you need to do is implement the view functionality.

Once implemented correctly, the `observers::test::simple_observation` test and all `feedbacks` tests should pass.
Be sure to review the feedbacks tests to understand how the observers and feedbacks interact.

#### Mutator

Without trying different inputs, the fuzzer can't make any progress.
To start simple, let's make a mutator that knows nothing about the current state of the board and just moves a random
car in a random direction at a random time.
There are no tests for this, but if your implementation is incorrect, the next exercise will not be achievable.

To implement this mutator, follow the steps provided in the TODO in PGRandMutator.
The exact implementation is not important, but you should ensure that any index can be selected (including the end!) and
that any direction can be selected.
You'll need to interact with the input, so make sure to review the documentation on how to do so.

### Exercise 1: Basic Fuzzer

Your next exercise is to link together all the components that you've just implemented.
Complete all of the `TODO(pt.1)` items presented in [main.rs](src/main.rs).
If you ever get lost, review the test code; many TODOs are completed within the tests from part 0.

Once this is done, go ahead and run the fuzzer with `cargo run -- maps/tokyo1.map`.
If implemented correctly, the fuzzer should complete within a few seconds.

#### Measuring effectiveness

Unfortunately, our fuzzer is not very fast at solving harder puzzles.
Try running `cargo run -- maps/tokyo36.map`.
For a 6x6 puzzle, we barely make progress.
Can we do better?

We need to evaluate what about our fuzzer is currently holding us back.
To start, think about how our components interact: our executor applies the moves until either the moves are exhausted
or an error occurs (i.e., an invalid move is attempted).
If your executor is implemented efficiently, you should be observing upwards of 250,000 executions per second on modern
hardware.
How much execution time is being wasted on invalid inputs as a result of our oversimplified mutator?

To measure this, let's make a feedback which measures the rate of erroneous inputs.
Go to [feedbacks.rs](src/feedbacks.rs) and implement CrashRateFeedback based on the TODO(pt.1) comments.
Some boilerplate is provided, but for this one, you're mostly on your own.
Make sure to review other implementations and search the documentation of LibAFL as needed.

Note particularly here that any mutable data is stored within metadata.
While not relevant for our single-threaded fuzzer that never has to restore from disk, any data which is not contained
within the state (i.e., within the metadata), will be lost whenever we recover from a crash on actual targets.
It is therefore good practice to always put the mutable data within metadata, as outlined in the template.

#### Documenting limitations

Reflect on what about our mutator could be causing so many failures.
Add a comment on PGRandMutator in [mutators.rs](src/mutators.rs) as to why it underperforms; there are at least two
major contributing factors.

### Exercise 2: Avoiding Wasted Executions

Most of our execution waste is coming from executing inputs which have no hope of succeeding.
Let's fix that by making our mutations smarter, at the cost of a bit more complexity.

#### Context enriching feedbacks

Some feedbacks just collect metadata for later use.
ViewFeedback associates metadata to individual testcases which tell us how many cars the moves can make at each point.
Start by including this in your feedbacks in [main.rs](src/main.rs).
This will not have any effect at the start.

#### Smart(er) mutators

Since we now have metadata for each testcase that tells us the number of moves that each car can make and in which
directions, we can build a mutator that takes advantage of this information.
Implement PGTailMutator based on the `TODO(pt.2)` comments in [mutators.rs](src/mutators.rs).
This is likely the most difficult task so far; take your time and review what you've already done to complete this.
Feel free to ask
questions [in the discussions](https://github.com/addisoncrump/parking-game-fuzzer/discussions/categories/q-a) if you
get really stuck, but avoid giving any spoilers.

Once this is done, replace your mutator in [main.rs](src/main.rs).
Your crash rate should now be 0% for all maps.

### Exercise 3: Snapshot Fuzzing

Review your implementation of executor.
Our target is very fast -- several hundred thousand executions per second -- but this is not representative of real
targets.
Let's slow things down a bit.

In [executor.rs](src/executor.rs), use the commented `sleep` call to insert an artificial delay of 1µs per move.
By increasing the cost of the moves (which is more consistent with a real target), we see that our executions slow down
over time as we have more moves per testcase.
But, since we only mutate the tail of the input now, we are effectively wasting execution time on the common prefix.
Can we resume from the last state that we executed?

To do this, add the FinalStateFeedback to your feedbacks in [main.rs](src/main.rs).
This will save metadata that stashes the final state after an execution is completed.
Implement this functionality in FinalStateFeedback by addressing the `TODO(pt.3)`s in [observers.rs](src/observers.rs).
Then, all we need to do is load that state instead of the initial state in our executor.

Load the snapshot from testcase metadata in your executor following the TODOs outlined
in [executor.rs](src/executor.rs).
Ensure that the moves that are executed after the snapshot are only those which were not already executed.

After implementing this, you should notice that your performance once again shoots up -- probably by a few times.

#### Investigating the speedup

Remove the artificial delay from the previous section.
You'll notice that the speedup is significantly less than previously observed -- to be expected, since we artificially
inflated the execution time of moves.
Keep this in mind for situations where snapshots are more expensive than simply resetting and running the whole input
again.

Moreover, we could only apply this optimization because of the unusual input scenario we are in: the input is processed
one part at a time.
This does happen in practice (e.g., with asynchronous embedded targets awaiting input from peripherals), but is
generally rare.
Snapshot fuzzing is not always the answer.

### Exercise 4: Endgame

Recall the implementation of PGTailMutator from earlier.
Notice how we recompute the possible mutations every time.
The number of valid mutations at any given state are not only finite -- they are few, and exactly computed by
PGTailMutator.
We can exhaust the whole mutation space at once and avoid redundant re-execution of mutations we've already observed.

To do this, we will now implement a stage to replace the mutation stage that exhausts the input space without
redundancy.
To do so, we will loosely reimplement the logic from PGTailMutator, but we won't randomly choose, we will only ever take
one step in any direction, and we won't execute with our old executor anymore.
This is the most difficult task; take your time, and remember you can ask for
help [in the discussions](https://github.com/addisoncrump/parking-game-fuzzer/discussions/categories/q-a).
Follow the `TODO(pt.4)` sections from [stages.rs](src/stages.rs) to complete this task.
Once completed, you can then replace the mutation stage in [main.rs](src/main.rs); you may need to reorder some
statements to keep the borrow checker happy!

### Exercise 5: Reflect

Note that in part 4, we removed all randomness from the search.
Is this still a fuzzer?
Most people who build fuzzers would probably say no; there's no random element to the search!
But, when did this stop being a fuzzer, exactly?
I leave this last exercise to the reader.

Nevertheless, there is a lesson here: fuzzing only makes sense when there is a clear guidance mechanism and when there
is a need for randomness due to the enormity of the search space.
Fuzzing is, fundamentally, just a randomized search process guided by (in most cases) novelty.
So, when you're designing your fuzzers going forward, take the optimizations as they make themselves known to you -- but
don't lose your exploration ability along the way.

-----

## Addendum 1: Using existing LibAFL input types

- If you are building a fuzzer expecting composite inputs outside of this exercise, you may want take a look at
[LibAFL's `ListInput`](https://github.com/AFLplusplus/LibAFL/blob/main/crates/libafl/src/inputs/list.rs). It provides
additional functionality around interacting with and mutating list-like inputs.
[This discussion](https://github.com/addisoncrump/parking-game-fuzzer/discussions/2) explores how this could be used in
the context of this project and provides further details on its features.
