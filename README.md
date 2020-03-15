# hack-earth-kotlin

A Programming assistant for Kotlin™. Let's keep this to ourselves for a bit while we see what comes of it.

## Functionality targets

1. Auto-completion, in a much broader sense than exists today. "Propose some new code at my cursor position."
   Let's call this "deep completion". The human programmer will need convenient idioms for setting the code's
   direction and then letting the IDE write a little bit of it. As a hypothesis, perhaps the Completion/Infer
   action means, "Propose a rewrite of the code or text immediately surrounding my cursor." Here are some
   idioms that should work well:
   * Type the beginning of a statement and infer the rest.
   * Type a brief pseudo-code fragment and infer the code.
   * Select a region around a compiler error and infer better code.
   
2. Code suggestions: given an existing source file (presumably in an editor), suggest improvements.

3. Relevant documentation: given the current cursor or selection, show documentation of identifiers that would
   likely be used at that point in the code.

## General Ideas

### Use GitHub diffs as training data.

Given a commit to a public repo on GitHub, we can back out a portion of it to see a hypothetical work-in-progress 
(WIP) state.

Given that WIP state, think like a programmer. Does it compile? If not, what are the error messages? Read the source.

### Integrate with Kotlin's compiler / IDE.

It is great for our purposes that, for Kotlin and IntelliJ, these are one and the same.

### Do the deep learning with [Deep Java Library]() on Kotlin.

We'd like TensorFlow as our foundation, because it clearly has a solid future.

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
relatively finished point in authoring code, then for training we have to back out some just-written code.

### Framing the Learning Problem

It seems most productive to have our AI think in AST. Because our ASTs represent work in progress, 
they commonly include compiler errors. This is a feature, not a bug: compiler errors are among our best predictors
of upcoming edits. So we treat them just like the rest of the AST.                                    

The AI needs three sorts of skills:

* Most obviously, *Creative*: Add a good node to the AST. Otherwise, how would it write code?
 
* But also, *Critical*: Does this node belong in the AST? We'll need this for deleting bad nodes
  and for deciding which of our impulses to add nodes we should entertain.
  
* Finally, *Problem-Solving*: Manage the overall the process of solving the deep-completion problem.
  
#### Is this a GAN?
  
The creative/critical combination looks like the setup for a GAN. At least as a rough cut,
we could train the GAN as follows:

* Choose a prior or existing state of a GitHub repo. (It's important to include prior states partly because
  they expand our training data, but especially because they give us early-stage code for repos that eventually
  became important.)
* Compile into an AST.
* Randomly choose an AST node (the "target node").
* Delete the target node and its children.
* Ask the creative network to regenerate the target node.
* Give the critical network the original target node as "good" and the regeneration effort (if different) as "bad".

#### Or do we train on Git diff hunks?

Alternatively, we could train both the critical and creative skills from Git diff hunks:
* Compile both the repo's prior state (before the hunk) and its post state.
* Diff the ASTs to make a list of divergence points: nodes that were replaced but whose parents were not.
* Give the critical network a prior node as "bad" and its replacement node as "good".
* Ask the creative network to regenerate a replacement node.

#### How do we generate a new node?

One way of approaching it is to say we never generate new nodes; rather, we search for a suitable existing node
in some explicit space of possible nodes. Our node vocabulary includes language syntax, symbols, and constants.
We can assume we never make up a symbol or constant from scratch; rather, we reuse one we saw before. (To avoid
being creepy, let's make sure that's either in this same project or in numerous other projects!)

Here's a possible framework:
* Each node in our vocabulary is represented by a vector.
* Our creative skill's job is to generate a query that finds the best node.
* We need a query language.
  * The obvious query would be a vector, where distance in vector space is the match score. 
  * That seems rather blunt, in that our creative skill may be able to infer some characteristics 
    of the target but not others. So perhaps the query is a vector plus an attention mask over that vector.

#### How do we implement the Problem-Solving skill?

We use Wisdom as our overall framework. See the 
[Wisdom Conceptual Overview](https://docs.google.com/document/d/1vouA9qZjAZxhFV8VbtkCZytrkFw_83D00P7rU9xRVes).

The basic idea is that we solve an instance of the deep-completion problem by building out a tree of "perspectives". 
The root node of that tree is the initial problem setup. Each subordinate node represents one candidate step in
solving the problem. We learn skills that create new perspectives from existing ones. We learn other skills that judge
perspectives, such as for their likelihood of being or leading to correct solutions.  

#### How do we apply deep learning?

We represent our data as trees. Examples include ASTs, candidate AST fragments, symbol meta-data, etc.

When we look at an AST, we don't see it through its original root node. Rather, we look at a particular "focal" node
and see it as the root for our purposes. The original parent direction is now a downward direction.

To operate on a tree, we convert it into a Wisdom clan. This reframes the existing tree as a new tree that suits
our purposes. For deep learning, we want the shallowest, broadest trees that capture the essential information.
(Deep neural nets aren't all that deep in the broader scheme of things. We want them as shallow as can capture
the essence of the problem.)

A Wisdom imp may have one or more neural pathways running through it. One special pathway is the "consept". (Sorry,
it was inevitable.) For a leaf imp, this is just a learned vector -- the embedding of that leaf. A non-leaf imp's 
consept pathway rolls up the vectors of its subordinate consepts into a new vector.

Other imp neural pathways represent neural operations the imp can perform. These operations are functions from
input vectors to output vectors. The primitive data type is the clan, with its corresponding vector the consept.
Some examples:
* A skill that proposes new AST nodes may do this through a neural operation that maps 
  an AST child location into an AST-node search query.
* A skill that judges whether a given perspective will lead to a good solution may map the perspective
  into a two-class softmax. It may work from the root of the perspective, or it may start at a focal node
  within the perspective and transform from there outward into a new clan whose consept will be the
  operation's input.





