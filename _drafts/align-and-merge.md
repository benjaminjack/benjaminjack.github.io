---
title: Aligning site-specific data to an amino acid sequence
author: Benjamin R. Jack
layout: post
excerpt: Most biologists today would define synthetic biology as the integration...
---

- Paragraph 1: on structural bioinformatics, framing the problem

In bioinformatics, we typically manipulate, align, and generally munge large sets of nucleotide sequences. In structural bioinformatics, we place a special emphasis on protein _structure_ in addition to the nucleotide sequences that code for such proteins. Studying the relationship between protein sequence and protein structure commonly involves taking some site-specific metric, like evolutionary rate, and mapping it precisely to an amino-acid. Often times, a single amino-acid may have several metrics associated with it. Mapping several site-specific metrics to a sequence presents a unique set of challenges. To make these challenges more clear, let us say we want to calculate evolutionary rate and solvent accessibility for a given protein. We might start by calculating site-wise evolutionary rates. Then we calculate solvent accessibility. We quickly discover the protein structure contains two cross-linked cysteine residues for which it is impossible to calculate solvent accessibility. If we remove the cysteine residues, our calculations will run, but the resulting set of solvent accessibilities will be two amino-acids shorter than our set of site-wise evolutionary rates. How do we align them correctly? In an ideal world we would just calculate evolutionary rates again without the cysteines so that everything lines up appropriately. Sometimes this isn't possible, however, because doing all of the recalculations may require a lot of computational power. Next imagine that at some point later you want to compute local packing density, catalytic residue information, and and other site-specific metrics. At each step of the process you risk misaligning site-specific metrics with the amino-acid sequence. 

How do we solve the problem of possibly mis-aligning site-specific metrics with a sequence? With a multiple sequence alignment of course!

- Paragraph 2: explaining the script conceptually
- Paragraph 3: examples of actual code
- Paragraph 4: summary, caveats