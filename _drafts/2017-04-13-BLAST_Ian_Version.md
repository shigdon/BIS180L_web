---
layout: lab
title: BLAST for paralogs and orthologs
hidden: true    <!-- To prevent post from being displayed as regular blog post -->
tags:
- Linux
- sequence
---


## Part 1: Orthologs ##

In the last lab, you learned that

* There is always a best local alignment, even in random sequences
* The distribution of random alignment scores is not normal
* The random background depends on
	* The lengths of the sequences
	* The composition of the sequences
	* The scoring parameters (matrix, gaps)
* The significance of a score is difficult to assess

We previously called the best match for the C.elegans protein B0213.10 in the A. thaliana and D.
melanogaster proteomes the ortholog. but the ortholog is operationally
defined as the best reciprocal match between proteomes. That is, after
finding the best match to the fly proteome, one must take the fly protein
and search it against the worm proteome to determine if it finds
B0213.10. If if it does, the proteins are orthologous.

Let's do something ambitious. Let's find ALL the orthologous proteins
between worm and fly. How long will it take to search every protein
against every other protein? You made this calculation in Lab 1 with the
`water` program and it was outrageously long. Let's start with the same
task but using `blastp`. To begin, let's organize our thoughts and files
in a new directory.  Make sure that directory is in your git assignments repository.

	cd ~/BIS180L_Assignments_Julin.Maloof/
	mkdir Assignments_2
	cd Assignments_2
	touch lab2_notebook.md

Keep a list of your the commands that you used in your lab_notebook.md file.  Answer the questions in the asiignment2-worksheet.md file [template available on the website]( {{site.baseurl}}/Assignments/assignment2-worksheet.md)

Before you can use `blastp` you need to format the BLAST database. A
FASTA file can be turned into a BLAST database with `formatdb`. Try the
following command to see the command options.

	formatdb --help

As it turns out, we only need a couple of these. Let's make an alias to
a FASTA proteome, format it as a BLAST database, and then observe the
new files.

	ln -s ~/Data/Species/D.melanogaster/protein.fa ./flypep
	formatdb -i flypep -p T

When you try this with wormpep you will get many error messages.  If you get sick of seeing all the error messages, type control-C. `blastp` expects the FASTA headers to be formatted in a very specific manner. It turns out that almost everyone formats the header differently even though there are certain conventions that should be followed. That's what real-world bioinformatics looks like sometimes. Here's how to quiet those error messages.

	formatdb -i flypep -p T 2> /dev/null

The `2>` token redirects the stderr stream to a file instead of the
screen. `/dev/null` is the name of a dummy file that is never made. It's
like a black hole for data: stuff goes in but never comes out. Now let's
observe that there are a bunch of new files you just made with the
`formatdb` command.

	ls -lrt

Now let's try aligning our previous favorite protein (`B0213.10`) to the
fly proteome. Make an alias here and call it `P450`. We know from Lab 1
that comparing a single C. elegans protein to a entire proteome can take
a couple minutes, and that if you attempted to align two whole
proteomes, it would take weeks. Let's align P450 to the entire D.
melanogaster proteome with BLAST and see how long it takes to run.

	time blastp -db flypep -query P450 > default.blast

That was fast! `blastp` is clearly much faster than `water`. Not only
that, if you look at the files, you will see that a BLAST report gives
you some statistics about the search. The E-value is sort of similar to
what you would get if you did some shuffling experiments and determined
how different the alignment was from the random background.

There are a lot of that control the speed of BLAST. The most important
of these are the seeding parameters. The default `blastp` search uses a
word_size of 3 and a threshold of 11. Let's experiment with the
threshold value from 10 to 15.

Instead of typing the command 6 times, lets use a __for loop__.   If you haven't already done so, take a minute to read up on [for loops]({{site.baseurl}}/{% post_url 2017-04-13-bash_for_loops %}).  __Exercise one__ in the for loop document is to be turned in as the answer to question one for this lab.  You can stop reading the for loop post after exercise one (although you are free to continue with it if you want to.)

The statement below will assign the variable $wlength the value "1" and then run the code between "do" an "done", subsitituing "1" for wherever "${wlength}" is written.  Then wlength will be assigned "2" and the code between do and done will be run again.  Then 3 and so on.  The result is that blastp is run 6 times, each with a different word length.

    for T in 10 11 12 13 14 15
        do
            echo "Starting blastp with threshold ${T}"
            time blastp -db flypep -query P450 -threshold ${T} > ${T}.blast
        done

Look at the files created and their size. Why are some larger than others?

    ls -lrt

Now calculate how long it will take to search a proteome against another
proteome using the parameters above and add these to your notebook and make a table to answer the assignment question.

---------------------------------------------------------------------------

When you're searching 1 protein against a database, the human readable
format of BLAST looks good. But if you're going to post-process the file
with Unix tools, you want the file to be easily parseable. So we're
going to change the output format to something different. Go back and do
the previous BLAST search of P450 vs flypep, but add the following
command line option `-outfmt 7` (this changes the output format to be
tab-delimited fields with a little header information that tells you
what each column is). Feel free to explore other output formats (because
you're a scientist and scientists are curious creatures and not robots).

Now add another option: `-culling_limit 1`. This will make it so that
BLAST only reports the single best alignment.

Time to set up the searches. First thing will be to make a `wormpep`
database because you'll need to search the worm against the fly and
vice-versa. So go ahead and `formatdb` a wormpep file. Then search one
proteome against the other. We're going to switch to output format 6
because we don't really care about the header lines.

	blastp -query wormpep -db flypep -outfmt 7 -culling_limit 1

If you're impatient, you can speed up the search by changing
`-threshold` as you did above. But is that a good idea? You decide. You
can also speed up the program by telling it to use more threads with the
`-num_threads` option. If you're curious about these options (and
others), it's best to try them on something smaller than searching one
proteome against the other. You could make miniature versions of each
using `head`, for example.


While you're waiting for the jobs to complete, you might want to tackle
the 2nd half of this lab farther below.

-------------------------------------------------------------------------

Once the jobs are complete use `less` to examine the outputs of each
(actually, you can do this even before the jobs are complete). Use the
`less` search feature (forward slash key) to find proteins in one file
that match proteins in the other file. Find some putative orthologs.

## Part 2: Paralogs

In this part, we are going to look for the **paralogs** of a couple worm
genes: T21B10.2a and B0213.10. That is, we are going to search the these
proteins against the C. elegans proteome to look for sequences that
arose by duplication within the same genome. A gene may have from 0 to
many paralogs. A simple, but flawed way to determine which alignments
represent paralogs is to use an E-value cutoff. But it's a good place to
start. Create the files as needed given the command lines below and then
do the searches.

	blastp -db wormpep -query T21B10.2a > T21B10.2a.blastp
	blastp -db wormpep -query B0213.10 > B0213.10.blastp

Inspect the BLAST report with `less`. Do all the sequences look like
they are highly related? You may want to change the E-value cutoff to
something higher or lower than 1e-10.
