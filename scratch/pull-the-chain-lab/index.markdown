---
title: 'Lab 2: Pull the Chain'
author: Ken Arnold
date: '2021-03-05'
slug: lab-2
categories: []
tags:
  - lab
subtitle: ''
summary: 'Understand backprop and gradient descent using physical analogies'
authors: []
lastmod: '2021-03-05'
featured: no
dates:
  topic: '2021-03-05'
  continue: '2021-03-08'
  postlab: '2021-03-11'
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---

In this activity, we'll be working with a physical analogy for backpropagation and gradient descent.

Imagine a chain of paperclips laying flat on a table, with one end anchored. Suppose we idealize the situation slightly: each link is attached to the next by a peg that lets them rotate but not slide. So the *angles* that each chain-link makes with the previous one tells us everything about the position of the chain. Imagine this chain laying haphazardly on the table.

Now suppose we attach a string to the end of the chain, and tie a small weight to the other end of the string. So far the chain doesn't move. Hold the end of the chain while you push the weight over the edge of the table. The weight drops down until there's no slack in the string.

Now let go of the end of the chain.

What happens to the chain?

-   The string pulls on the end of the chain.
-   That force on the end of the chain is *propagated* back through the chain all the way to the anchor.
-   The angles of the chain segments to each other change until the chain is straight.

The forces of each link on each other are analogous to backpropagation.

What happens to the weight? It drops down as the chain straightens.

To see where gradient descent comes in, think about the *potential energy* of the system (we'll see that it is analogous to the *loss*). The potential energy of the chain doesn't change; only the energy of the weight changes. At the start, when the chain is haphazard, the weight has high potential energy. As it drops, the potential energy drops: the system moves so as to reduce its potential energy.

Let's assume that things move slowly enough that we can ignore the kinetic energy. Therefore, the forces along the entire system are exactly those that act to reduce its potential energy. (... or so I think. Someone who knows more physics than me could derive the Lagrangian mechanics and check this.)

## Objectives

-   Explain how backpropagation and gradient descent work by a physical analogy
-   Practice using automatic differentiation to implement gradient descent using PyTorch
-   Practice working with PyTorch `tensor`s

## Getting Started

Partner up (optional but encouraged) and create a notebook. You may use Colab or the lab machines.

### Setup code

If using Colab, create and run a code cell with the following (to update fastai):

    !pip install -Uq fastbook

Then import the usual functionality with this code cell:

``` python
from fastai.vision.all import *
```

### Modeling our chain

Create a 10-link chain by defining the 10 angles that each segment makes with the previous one:

``` python
set_seed(0)
n = 10
orig_angles = torch.randn(n) * np.pi / 4
angles = orig_angles.clone()
```

Copy in this function to compute the positions of each endpoint in the chain. (You don't need to understand how it works.)

    def compute_positions(relative_angles):
        angles = relative_angles.cumsum(dim=0)
        deltas = torch.stack([angles.cos(), angles.sin()])
        end_positions = deltas.cumsum(dim=1)
        return torch.cat([torch.zeros(2, 1), end_positions], dim=1).T

And here is a function to show the chain:

    def show_chain(positions, **kw):
        n = len(positions)
        xs, ys = positions.T
        line = plt.plot(xs, ys, marker='o', **kw)
        plt.gca().set_xlim(-n, n); plt.gca().set_ylim(-n, n)
        return line

Try running these two code chunks to test the above.

    positions = compute_positions(angles); positions

    show_chain(positions);

Notice that `positions` is a Torch tensor. **Look at `positions.shape` and explain each number.** Now look at `positions` itself and see if you understand the shape of what you see.

### Pulling the Chain

Can we pull the end of the chain towards the right? We'll do this by tugging on the end position.

Make a new code cell that we're going to build up into a *training loop* step by step. (All of the remaining instructions will go within that same code cell.)

First, create a variable called `end_position` that gives the `x` and `y` coordinates of the end of the chain (the last position in `positions`). Try to figure out the syntax for this yourself, but if it takes more than 1 minute **please ask for a hint**. Then, **create a variable called `loss`** (we'll see why we use that name soon) that gets the value of `n` minus the `x` coordinate of the right of the chain. (Convince yourself that `loss` will get a value of 0 if the chain is straight out to the right, and greater than 0 otherwise).

Now we want to see how we might change `angles` in order to reduce `loss`. We'll do this by *backpropagating* `loss` through the chain. Within this same code cell:

1.  Try running `loss.backward()` and notice that it fails. Comment it out and leave it at the bottom of the cell.

2.  At the top of the cell, use `angles = orig_angles.clone()` so that `angles` starts from the same thing each time you run the cell.

3.  Tell `angles` that you want its gradient. (`requires_grad_()`)

4.  Uncomment `loss.backward()` now. It will still fail, but now you can fix it: you have an `angles` that requires gradient, so use *that* to compute `loss`. (You'll only need to move code around; you should not need to change your `loss =` calculation, or the functions `compute_positions` or `show_chain`.) Test that `backward` succeeds.

5.  Print out `loss`. You'll notice that it's a `tensor`, but there's one extra thing; what is it?

6.  Print out `angles.grad`. How does its shape compare with `angles`?

7.  Add a line at the end to step down the gradient of `angles`:

    `angles.data -= angles.grad * learning_rate`.

    (Define the `learning_rate` to be .01 at the top of the cell.)

8.  Plot the new position of the chain. You'll need to use `show_chain(positions.detach())` so PyTorch doesn't try to take the gradient of the plotting function.

9.  Put this whole process in a loop: compute loss, backprop, update. You'll need to think about what sequence these operations need to be done in. **Don't forget to `.zero()` the `.grad`**! 500 updates should be more than enough.

Here's a template for the loop, which also includes code to plot the loss:

``` python
fig, (ax1, ax2) = plt.subplots(ncols=2, figsize=(16, 5))
plt.sca(ax1) # use the left plot to show the chain
angles = orig_angles.clone().requires_grad_()
losses = []
n_iter = 500
for i in range(n_iter):

    # ... some code ...
    color = [1 - i / n_iter] * 3 if i != n_iter - 1 else 'blue'
    show_chain(positions.detach(), color = color)
    
    # ... some code ...
    loss = #...
    losses.append(loss.item())
    
    # ... some code ...

# Plot the losses
ax2.plot(losses); ax2.set(ylabel="Loss", xlabel="Step");
```

You should see the chain end up straight to the right, and the loss should smoothly approach 0.

Hints:

-   `end_position` is the `-1`th element of `position`.

-   The x position of the end is `end_position[0]`.

-   Use `print` statements or similar to check these values:

    -   `positions`, `end_position`, and `loss` all have a `grad_fn`.
    -   `end_position` has 2 numbers (an *x* and *y* coordinate).
    -   `loss` is a single number (rather than a matrix)

### Analysis

-   Try changing the learning rate. What do you notice when the learning rate is too big? Too small?

    -   We sometimes talk about the gradient descent as having *converged* or *diverged*. Can you guess what these might mean? Identify a learning rate that corresponds to each situation, and describe the behavior of the loss and the angles in each situation.

-   Try changing the definition of `loss`:

    -   Try having both the *x* and *y* coordinate contribute equally to the `loss`. What does this correspond to in the physical system? What happens to the chain? What changes if you multiple the *x* by a big constant? A small constant?
    -   Tie another string to the middle of the chain: add a term to `loss` that uses one of the intermediate positions. Where does the chain end up?
    -   *optional* Add a term that penalizes the max absolute value of `angles`. What happens to the chain?

-   For each of the following terms, identify what element of the code you created corresponds to it (variable names, methods, etc.), and describe the connection.

    -   loss function

    -   parameters

    -   backpropagation

    -   gradient descent (what would you change if you wanted to do gradient *ascent* instead?)

    -   learning rate

-   The analogy in this lab didn't include the *data* that are used to train a model. We could extend the analogy by imagining each data point as its own little string pulling the chain in potentially different directions. Which of the following would correspond to *stochastic* gradient descent?

    -   all of the strings pulling at the same time

    -   only letting one string pull at a time, in systematic order

    -   only letting one string pull at a time, in random order

## Wrap-up

Make sure that both partners have a copy of the notebook.
