---
layout: post
published: false
title: "Deep challenges to dynamic documentation in daily workflows"
category: open-science
tags:
- blog
- open-science
- workflow
- R

---

We often discuss dynamic documents such as Sweave and knitr in reference
to final products such as publications or software package vignettes.
In this case, all the elements involved are already fixed: external
functions, code, text, and so forth.  The dynamic documentation
engine is really just a tool to combine them (knit them together).
Using dynamic documentation on a day-to-day basis on ongoing research
presents a compelling opportunity but a rather more complex challenge
as well.  The code base grows, some of it gets turned into external
custom functions where it continues to change. One analysis script
branches into multiple that vary this or that.  The text and figures
are likewise subject to the same revision as the code, expanding and
contracting, or being removed or shunted off into an appendix.

Structuring a dynamic document when all the parts are morphing and moving
is one of the major opportunities for the dynamic approach, but also the
most challenging.  Here I describe some of those challenges along with
various tricks I have adopted to deal with them, mostly in hopes that
someone with a better strategy might be inspired to fill me in.



What works: better than the old way
-----------------------------------

For a while now I have been using the [knitr](http://yihui.name/knitr)
dynamic documentation/reproducible research software for my project
workflow.  Most discussion of dynamic documentation focuses on 'finished'
products such as journal articles or reports.  Over the past year, I
have found the dynamic documentation framework to be particularly useful
as I develop ideas, and remarkably more challenging to then integrate
into a final paper in a way that really takes advantage of its features.
I explain both in some detail here.

My former workflow followed a pattern no doubt familiar to many:

* Bash away in an R terminal, paste useful bits into an R script...
* Write manuscript separately, pasitng in figures, tables, and inline
values returned from R.

This doesn't leave much of a record of what I did or why, which is
particularly frustrating when some discussion reminds me of an earlier
idea.

When I begin a new project, I start off writing a `.Rmd` file, intermixing
notes to myself and code chunks.  Chunks break up the code into conceptual
elements, markdown gives me a more expressive way to write notes than
comment lines do.  Output figures, tables, and inline values inserted.
So far so good.  I version manage this creature in git/Github.  Great,
now I have a trackable history of what is going on, and all is well:

1. Document my thinking and code as I go along on a single file
scratch-pad

2. Version-stamped history of what I put in and what I got out on each
step of the way

3. Rich markup with equations, figures, tables, embedded.

4. Caching of script chunks, allowing me to tweak and rerun an analysis
without having to execute the whole script.  While we can of course
duplicate that behavior with careful save and load commands in a script,
in knitr this comes for free.


Limitations to .Rmd scripts alone
----------------------------------

1. As I go along, the .Rmd files starts getting too big and cluttered
to easily follow the big picture of what I'm trying to do.

2. Before long, my investigation branches.  Having followed one `.Rmd`
script to some interesting results, I start a new `.Rmd` script
representing a new line of investigation.  This new direction will
nevertheless want to re-use large amounts of code from the first file.

A solution? The R package "research compendium" approach
---------------------------------------------------------

I start abstracting tasks performed in chunks into functions, so I
can re-use these things elsewhere, loop over them, and document them
carefully somewhere I can reference that won't be in the way of what
I'm thinking. I start to move thse functions into `R/` directory of an R
package structure, documenting with Roxygen. I write unit tests for these
functions (in `inst/tests`) to have quick tests to check their sanity
without running my big scripts (recent habit).  The package structure
helps me:

* Reuse the same code between two analyses without copy-paste or getting
our of sync
* Document complicated algorithms outside of my working scripts
* Test complicated algorithms outside of my working scripts (`devtools::check` and/or unit tests)
* Manage dependencies on other packages (DESCRIPTION, NAMESPACE),
including other projects of mine


This runs into trouble in several ways.


Problem 1: Reuse of code chunks
-------------------------------


What to do with code I want to reuse across blocks but do not want to write as a function, document, or test?

_Perhaps this category of problem doesn't exist, except in my laziness._

This situation arises all the time, usually through the following mechanism: almost
any script performs several steps that are best represented as chunks calling different
functions, such as `load_data`, `set_fixed_parameters`, `fit_model`, `plot_fits`, etc. I
then want to re-run almost the same script, but with a slightly different configuration
(such as a different data set or extra iterations in the fixed parameters).  For just
a few such cases, it doesn't make sense to write these into a single function,[^1]
instead, I copy this script to a new file and make the changes there.

This is great until I want to change something in about the way both scripts behave
that cannot be handled just by changing the `R/` functions they share.  Plotting
options are a good example of this (I tend to avoid wrapping `ggplot` calls as
separate functions, as it seems to obfuscate what is otherwise a rather semantic
and widely recognized, if sometimes verbose, function call).


[^1]: If I have a lot of different configurations, it may make sense to
wrap up all these steps into a single function that takes input data
and/or parameters as it's argument and outputs a data frame with the
results and inputs.


I have explored using knitr's support for external chunk inclusion, which
allows me to maintain a single R script with all commonly used chunks,
and then import these chunks into multiple .Rmd files.  An example of
this can be seen in my nonparametric-bayes repo, where several files
(in the same directory) draw most of their code from [external-chunks.R](https://github.com/cboettig/nonparametric-bayes/blob/9232dfd814c40e3c48c5a837be110a870d8639da/inst/examples/BUGS/external-chunks.R).



Problem 2: package-level reproducibility
-----------------------------------------

_Minor/relatively easy to fix._

Separate files can frustrate reproducibility of a given commit.
As I change the functions in `R/`, the `.Rmd` file can give different results despite
being unchanged.  (Or fail to reflect changes because it is caching
chunks and does not recognize the function definitions have changed
underneath it).  Git provides a solution to this: since the `.Rmd`
file lives in the same git repository (`inst/examples`) as the package,
I can make sure the whole repository matches the hash of the `.Rmd` file:
`install_github("packagename", "cboettig", "hash")`.

This solution is not fail-safe: the installed version, the potentially
uncommitted (but possibly installed) version of the R functions in the
working directory, and the R functions present at the commit of the `.Rmd`
file (and thus matching the hash) could all be different.  If we commit
and install before every `knit`, we can avoid these potential errors
(at the cost of some computational overhead), restoring reproducibility
to the chain.

Problem 3: Synthesizing results into a manuscript
--------------------------------------------------

In some ways this is the easiest part, since the codebase is relatively
static and it is just a matter of selecting which results and figures
to include and what code is necessary to generate it.  A few organizational
challenges remain:

While we generally want `knitr` code chunks for the figures and tables
that will appear, we usually aren't interested in displaying much,
if any, of the actual code in the document text (unlike the examples
until this point, where this was a major advantage of the knitr approach).
In principle, this is as simple as setting `echo=FALSE` in the global
chunk options.  In practice, it means there is little benefit to having
the chunks interwoven in the document.  What I tend to want is having
all the chunks run at the beginning, such that any variables or results
can easily be added (and their appearance tweaked by editing the code)
as figure chunks or inline expressions. The only purpose of maintaining
chunks instead of a simple script is the piecewise caching of chunk
dependencies which can help debugging.

Since displaying the code is surpressed, we are then left with the somewhat
ironic challenge of how best to present code as a supplement.  One option
is simply to point to the source `.Rmd`, another is to use the `tangle()`
option to extract all the code as a separate `.R` file.  In either case,
the user must also identify the correct version of the R package itself
for the external `R/` functions.

Problem 4: Branching into other projects
-----------------------------------------

Things get most complicated when projects begin to branch into other projects.
In an ideal world this is simple: a new idea can be explored on a new branch
of the version control system and merged back in when necessary, and an
entirely new project can be built as a new R package in a different repo that
depends on the existing project. After several examples of each, I have learned
that it is not so simple.



