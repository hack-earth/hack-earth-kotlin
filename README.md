# hack-earth-kotlin

A Programming assistant for Kotlin&trade;. Let's keep this to ourselves for a bit while we see what comes of it.

## Functionality targets

1. Auto-completion, in a much broader sense than exists today. "Propose some new code at my cursor position."
   For example, the following usage idioms should be supported well:
   * Writing a function header, putting the cursor inside the brackets, and auto-completing the function body.
   * Writing the names of one or several variables, simply separated by spaces, selecting those, and 
     auto-completing a chunk of code that uses them.
   * Writing a rough semblance of some code, selecting it, and auto-completing it into real code.
   
2. Code suggestions: given an existing source file (presumably in an editor), suggest improvements.

3. Relevant documentation: given the current cursor or selection, show documentation of identifiers that would
   likely be used at that point in the code.

## General Ideas

### Use GitHub diffs as training data.

Given a commit to a public repo on GitHub, we can back out a portion of a commit to see a hypothetical work-in-progress (WIP) state.

Given that WIP state, think like a programmer. Does it compile? If not, what are the error messages? Read the source.

### Use Kotlin's compiler as a library.

We will need deep, peculiar integration with Kotlin's AST and code analysis.

### Do the deep learning with [Deep Java Library]() on Kotlin.

### Deploy our functionality as an IntelliJ IDEA plugin.

## Ideas for Learning

### Code Context

At any given point in the code, we want a deep context. This corresponds closely to information that a human programmer
might consider important when writing the code. Here are some examples:
* For each identifier in scope:
  * "Meaning" of the name. (When we say "meaning", imagine an embedding.)
  * Type. (Our information about the type should be roughly the same as our information about any identifier in scope.)
  * Scope.
  * How recently the declaration was written (per the Git history) and updated.
* The surrounding scopes, at roughly the same level as for identifiers in scope.
* The immediately preceding code.

One complication for learning is that we want the context in which the code is being written, not the
context in which it already exists. If we are training from GitHub commits, which usually represent a 
relatively finished point in authoring code, then for training we have to back out the code being written.

### Flow of Training

#### WIP ASTs and AST Diffs

We need special ASTs that we'll call "WIP ASTs", because we must approximate the situation when our end-user is actively editing.
Obtain these by replaying an arbitrary portion of a given commit, and then "compiling" the resulting, unfinished code.
(This isn't a full compile. Rather, it is the compiler's syntactic and semantic analysis, intercepted in memory,
with no code generation.)

We need to learn a transformation from a WIP AST to a later one obtained by continued edits to the source code.
We need this for at least two reasons:
* For training, we can't afford a "recompile" (even in our partial, intra-compiler way) for every observation we collect.
* We have essentially the same situation in our end-user application: we can't afford to "recompile" 
  every time our user types a character.
    
Our overall data-capture process is roughly as follows:
* Choose the "before" state of a commit as our starting point.
* "Imagine" a series of one-character edits that would have been one way to make that diff.
  (In the long run, we may learn this from watching user edits occur, but for now we make it up.)
* Replay a prefix of those edits.
* Capture a WIP AST at this point: the "pre AST".
* Replay some more edits. Capture these as the "applied edits".
* Generate a WIP AST at this later point: the "post AST".
* Capture a delta from the pre AST to the post AST. This is the "WIP AST delta".
* Also capture the next edits to be applied from the diff. These are the "upcoming edits".

This defines our first learning task, which we will call our "AST edit model": given a pre AST and
applied edits, generate a WIP AST delta. It's an unusual application of deep learning: 
we are training a model to approximate something we could accomplish directly with traditional tools]
by recompiling. But yeah, for performance both in training and inapplication, we need to do that.
It will be a whole lot easier to train a model to do it than to add the compability by hand into
the Kotlin compiler.

#### Compiler Errors as First-Class AST Citizens

Note that because our WIP ASTs represent work in progress, they commonly include compiler errors.
This is a feature, not a bug: compiler errors are among our best predictors of upcoming edits. So we 
treat them just like the rest of the AST, including predicting which compiler errors go away and which new 
ones appear in our post AST.

#### Auto-Completion Problem Setup

Given the framework described above, the problem setup for our auto-completion model is straightforward: 
* Start with a WIP AST representing a recent compiler pass, plus "applied edits" representing 
  the user's editing work since that compilation.
* Use our AST edit model to generate a post AST for the present moment.
* Predict some upcoming edits. This could be
  * the next token,
  * upcoming tokens that resolve all nearby compiler errors,
  * or a replacement for selected text that retains some of those existing fragments
    while resolving all nearby compiler errors.

