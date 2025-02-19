---
title: "Input functions"
teaching: 40
exercises: 30
questions:
- "What is a Python function?"
- "How can Python functions help in defining Snakemake rules?"
objectives:
- "Understand some components of Python useful for working in Snakemake"
- "Use an input function to supply complex inputs to a rule"
- "Use an input function to set a rule parameter"
keypoints:
- "Python functions are needed to resolve some complex input/output relationships"
- "Input functions for rules always take a single argument which contains the keywords for the specific job"
- "Input functions may also be used to determine parameters"
---
*For reference, [this is the final Snakefile from episodes 1 to 6](../code/ep06.Snakefile) you may use to
start this episode.*

## Workflows can get really complex

If you turn your entire analysis project into a Snakemake workflow, you may end up with something like this.

![A flying Snakemake spaghetti monster][fig-spaghetti]

Sometimes, the standard Snakemake filename matching is not enough to express the connections you need
to make, especially with complex combining steps. In these cases we can use **input functions**. Before
seeing how these work, we need to talk about Python functions in general.

## Python functions in general

We've already made use of utility functions like `expand()` and `glob_wildcards()` in writing Snakefiles. Now we
are going to write some new functions of our own.

If you are already a Python programmer this will be familiar to you. If not, we're just going to cover the essentials
needed for use in Snakemake. Functions in Python look like this:

~~~
def myfunc(in_val1, in_val2):
    result = expand( "foo {x}", x = [in_val1, in_val2] )
    return result
~~~
{: .language python}

There are many things to note in these three lines:

* The keyword `def`, as well as the parentheses and the colon, are always used when defining a Python function
* As with rules, the function has a name of our choosing - `myfunc`
* The function has *arguments*, ie. placeholders for values to pass in, which are also named - `in_val1` and `in_val2`
* The function body is indented and consists of one or more statements to be run when the function is called
* The body may set *local variables* - here `result`
* Other functions may be called within the function body - here `expand()`
* The keyword **return** exits the function with a result

The simplest way to test the function is to put it in a file and run it with Snakemake. We'll use a couple of `print()`
lines to test out the function.

~~~
# Save as myfunc_test.py then run "snakemake -n -s myfunc_test.py"
def myfunc(in_val1, in_val2):
    result = expand( "foo {x}", x = [in_val1, in_val2] )
    return result

print( myfunc("cobra", "adder") )
print( myfunc("one thing", "another thing") )
~~~
{: .language-python}

Having put the above lines into `myfunc_test.py`, we see that each time the function is called it produces a 2-item list.
Remember that lists of things are denoted by square bracket notation.
Since there are no rules in the file, Snakemake has nothing to put in the DAG and so it stops after running the two `print()`
statements.

~~~
$ snakemake -n -s myfunc_test.py
['foo cobra', 'foo adder']
['foo one thing', 'foo another thing']
Building DAG of jobs...
Nothing to be done.
~~~

> ## Note
>
> You could also run the above file directly in Python - `$ python3 myfunc_test.py` - but to make it work you'd
> also need to add the line
>
> ~~~
> from snakemake.io import glob_wildcards, expand
> ~~~
>
> to the top. When the code is run via Snakemake these functions, among other things, are imported for you.
>
{: .callout}

> ## Challenge
>
> The above *myfunc* function just generates useless pairs of strings, but we're going to be generating lists
> of input file names for our rules. Write a function that takes a single argument called `sample_name` and returns
> a list of all the *read 1* files for that sample. So, for example, if your function is called like:
>
> ~~~
> print(yourfunc('etoh60'))
> ~~~
>
> the result should be: `['reads/etoh60_1_1.fq', 'reads/etoh60_2_1.fq', 'reads/etoh60_3_1.fq']`
>
> You may want to use `expand()` and/or `glob_wildcards()` within your function. You should also know that concatenating
> strings in Python is done by using `+`, so `"reads/" + sample_name + "_1_1.fq"` will give you the first file name.
>
> > ## Solution
> >
> > This is the most flexible solution, using `glob_wildcards()` to scan for all the available repeats.
> >
> > ~~~
> > def yourfunc(sample_name):
> >   repeats = glob_wildcards("reads/" + sample_name +"_{rep}_1.fq").rep
> >   result = expand("reads/{sample_name}_{rep}_1.fq", sample_name=sample_name, rep=repeats)
> >   return result
> >
> > print(yourfunc('etoh60'))
> > ~~~
> > {: .language-python}
> >
> > And when run:
> >
> > ~~~
> > $ snakemake -n -s yourfunc_test.py
> > ['reads/etoh60_1_1.fq', 'reads/etoh60_3_1.fq', 'reads/etoh60_2_1.fq']
> > Building DAG of jobs...
> > Nothing to be done.
> > ~~~
> >
> > You may see the list in a different order. To make it be sorted, simply change the last line of the function to:
> >
> > ~~~
> > return sorted(result)
> > ~~~
> >
> {: .solution}
{: .challenge}


## A rule in need of an input function

Previously we defined a rule `kallisto_quant` like so:

~~~
rule kallisto_quant:
    output:
        directory("kallisto.{sample}")
    input:
        index = "Saccharomyces_cerevisiae.R64-1-1.kallisto_index",
        fq1   = "trimmed/{sample}_1.fq",
        fq2   = "trimmed/{sample}_2.fq",
    shell:
        "kallisto quant -i {input.index} -o kallisto.{wildcards.sample} {input.fq1} {input.fq2}"
~~~
{: .language}

This rule can perform quantification on any pair of FASTQ reads, and assumes each file pair corresponds to one
sample. But what if all three reads were technical replicates from the same sample?
In this case we would want to feed all of these to Kallisto at once.

~~~
# In a new file named kallisto_all.Snakefile...
rule kallisto_quant_all:
    output:
        outdir = directory("kallisto_all.{sample}"),
    input:
        index = "Saccharomyces_cerevisiae.R64-1-1.kallisto_index",
        fq_pairs = [ "trimmed/{sample}_1_1.fq", "trimmed/{sample}_1_2.fq",
                     "trimmed/{sample}_2_1.fq", "trimmed/{sample}_2_2.fq",
                     "trimmed/{sample}_3_1.fq", "trimmed/{sample}_3_2.fq" ]
    shell:
        "kallisto quant -i {input.index} -o {output.outdir} {input.fq_pairs}"
~~~
{: .language}

Running the above in dry-run mode produces the command as expected (there's no need to try this):

~~~
$ snakemake -s kallisto_all.Snakefile -pn -- kallisto_all.etoh60
...
kallisto quant -i Saccharomyces_cerevisiae.R64-1-1.kallisto_index -o kallisto_all.etoh60 trimmed/etoh60_1_1.fq trimmed/etoh60_1_2.fq trimmed/etoh60_2_1.fq trimmed/etoh60_2_2.fq trimmed/etoh60_3_1.fq trimmed/etoh60_3_2.fq
~~~
{: .language-bash}

The problem with this rule is that it's inflexible. It only works if there are exactly three sets of paired files
for every sample. We can fix this with a use of `expand()` and `glob_wildcards()`, but we need to use a function
like the one above. To see why, try:

~~~
rule kallisto_quant_all:
    output:
        outdir = directory("kallisto_all.{sample}"),
    input:
        index = "Saccharomyces_cerevisiae.R64-1-1.kallisto_index",
        fq_pairs = expand( "trimmed/{sample}_{rep}_{end}.fq",
                                sample = "{sample}",
                                rep = glob_wildcards("reads/{sample}_{rep}_1.fq").rep,
                                end = [1, 2] )
    shell:
        "kallisto quant -i {input.index} -o {output.outdir} {input.fq_pairs}"
~~~

This runs, but the result is no good - the command has too many input files:

~~~
kallisto quant -i Saccharomyces_cerevisiae.R64-1-1.kallisto_index -o kallisto_all.etoh60 trimmed/etoh60_3_1.fq trimmed/etoh60_3_2.fq trimmed/etoh60_2_1.fq trimmed/etoh60_2_2.fq trimmed/etoh60_1_1.fq trimmed/etoh60_1_2.fq trimmed/etoh60_3_1.fq trimmed/etoh60_3_2.fq trimmed/etoh60_2_1.fq trimmed/etoh60_2_2.fq trimmed/etoh60_1_1.fq trimmed/etoh60_1_2.fq trimmed/etoh60_2_1.fq trimmed/etoh60_2_2.fq trimmed/etoh60_1_1.fq trimmed/etoh60_1_2.fq trimmed/etoh60_3_1.fq trimmed/etoh60_3_2.fq
~~~

> ## Question
>
> What is happening here? And is there a way to fix it? This example is getting quite complex, so take the time to think carefully about it.
>
> > ## Solution
> >
> > The `rep = glob_wildcards(...)` in the rule has two wildcards: `{sample}` and `{rep}`, so it matches every rep of *every* sample.
> > In the solution to the exercise above, we used concatenation to incorporate the sample name into the pattern, so we need to do the
> > same here: `rep = glob_wildcards("reads/" + wildcards.sample + "_{rep}_1.fq").rep`.
> >
> > But this gets us: `NameError: name 'wildcards' is not defined`
> >
> > The fundamental problem is that Snakemake will try to run the `expand(...)` function at the point where it is building the rule
> > definition, and this is well before it knows the output and the wildcards. Indeed, the rule is probably going to be applied to
> > multiple samples, and so we want Snakemake to calculate `input.fq_pairs` and run `expand()` and `glob_wildcards()` for each of
> > those samples, ie. for each job *(remember: a job = a rule + keywords)*, not just for the rule as a whole.
> >
> > This fundamental problem means we can't just fix this by teaking the syntax. We need to tell Snakemake that the `fq_pairs` input
> > of this rule is to be provided by an **input function**.
> {: .solution}
{: .challenge}

# Putting it together

An input function replaces Snakemake's normal way of calculating the inputs to a rule by plugging wildcards into templates. In most
cases, you can (and should) use carefully chosen file naming to avoid the need for input functions, but in this case there is no way
to do it while keeping the rule generic.

For a function to be an input function in Snakemake, the function needs to be defined with a single argument named **wildcards**,
so we get:

~~~
def make_fq_pairs(wildcards):
    sample_name = wildcards.sample
    reps = sorted(glob_wildcards("reads/" + sample_name + "_{rep}_1.fq").rep)

    return expand( "trimmed/{sample}_{rep}_{end}.fq", sample = sample_name,
                                                      rep = reps,
                                                      end = [1, 2] )
~~~

Looking at this line by line:

* The function is named `make_fq_pairs`
* The first line assigns the value from `wildcards.sample` to a local variable `sample_name`
* The second line uses uses `glob_wildcards()` to find the read pairs for this sample...
 * The sample name is inserted into the pattern template using the `+` operator to concatenate the strings
 * This means the only wildcard is `{rep}`
 * The results of `glob_wildcards()` come out in an arbitrary order, so we use the `sorted()` function
   to make the list be sorted
 * The resulting list is assigned to the `reps` local variable
* Finally, `expand()` function gives the full list of files that will be needed as input to the job

Note that `glob_wildcards()` is used on the reads directory, because there won't be any files to search for
in the `trimmed/` directory until the pipeline has run. However, the input files for this rule are the trimmed ones
so we use the prefix `trimmed/` in the `expand()` part.

Having made the function, we can tell Snakemake that this is to be used to determine the inputs to the rule:

~~~
rule kallisto_quant_all:
    output:
        outdir = directory("kallisto_all.{sample}"),
    input:
        index = "Saccharomyces_cerevisiae.R64-1-1.kallisto_index",
        fq_pairs = make_fq_pairs
    shell:
        "kallisto quant -i {input.index} -o {output.outdir} {input.fq_pairs}"
~~~

~~~
$ snakemake -s kallisto_all.Snakefile -pn -- kallisto_all.etoh60
...
kallisto quant -i Saccharomyces_cerevisiae.R64-1-1.kallisto_index -o kallisto_all.etoh60 trimmed/etoh60_1_1.fq trimmed/etoh60_1_2.fq trimmed/etoh60_2_1.fq trimmed/etoh60_2_2.fq trimmed/etoh60_3_1.fq trimmed/etoh60_3_2.fq
~~~
{: .language-bash}

Note that there are no parentheses, just the plain function name. Snakemake stores the function and only runs it
when it determines that the rule is needed to make some particular output, and at this point it passes in the
*wildcards* values for that job.


> ## Exercise
>
> Input functions can also be used to set **params** values. Alter the definition of the **trimreads** function
> so that **min_length** is "100" for read 1 of any FASTQ pair, but "80" for read 2.
>
> As a reminder, here's the basic version of the rule with params added:
>
> ~~~
> rule trimreads:
>   output: "trimmed/{sample}.fq"
>   input:  "reads/{sample}.fq"
>   params:
>     qual_threshold = "22",
>     min_length     = "100",
>   shell:
>     "fastq_quality_trimmer -t {params.qual_threshold} -l {params.min_length} -o {output} <{input}"
> ~~~
>
> Within your function, you'll need to make an if/else decision. The syntax to do this is:
>
> ~~~
> min_length = "100" if read_name.endswith("1") else "80"
> ~~~
>
> > ## Solution
> >
> > This is one way to write the function:
> >
> > ~~~
> > def min_length_func(wildcards):
> >     read_name = wildcards.sample
> >     min_length = "100" if read_name.endswith("1") else "80"
> >     return min_length
> > ~~~
> >
> > And in the *trimreads* rule, the *params* will now be:
> >
> > ~~~
> > params:
> >   qual_threshold = "22",
> >   min_length     = min_length_func,
> > ~~~
> >
> {: .solution}
{: .challenge}

> ## Testing input functions
>
> If you want to test an input function directly, you need to make your own `wildcards` object to pass in to it. The syntax is:
>
> ~~~
> test_wildcards = snakemake.io.Wildcards(fromdict=dict(sample = 'etoh60'))
> print(yourfunc(test_wildcards))
> ~~~
>
{: .callout}

*For reference, [this is a Snakefile](../code/ep09.Snakefile) incorporating the changes made in this episode.*

[fig-spaghetti]: ../fig/rulegraph_complex.svg
{% comment %}
Leif Wigge, Rasmus Ågren and John Sundh SciLifeLab, National Bioinformatics Infrastructure Sweden (NBIS), Bioinformatics Long-term Support
https://nbis-reproducible-research.readthedocs.io/en/course_1911/snakemake/
MIT License
{% endcomment %}

{% include links.md %}
