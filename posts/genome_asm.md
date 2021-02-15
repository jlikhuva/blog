# Genome Assembly

To sequence a genome, scientists begin by collecting tissue samples. The samples are then treated with biochemical agents that break down the DNA in the cells into _very many_ fragments. This is called the _Fragmentation_ step. Sequencing machines produce _reads_ (of the same size) by looking at these fragments. In spite of the rapid advancements in the field, modern sequencing machines are still not able to generate entire genomes in a single read. Therefore, the final step in any sequencing pipeline involves computationally reconstructing the genome from the collection short reads. This final step, called genome assembly,  will be the focus of this note. We'll begin by examining the complexity that arises when trying to reconstruct the genome from _reads_. We'll then explore two ways of characterizing the genome assembly problem using graphs. Eventually, we'll derive and implement an algorithm for genome assembly that depends on finding an Eulerian Path in a De Bruijn Graph.

## The nature of Fragments & Reads

Right off the bat, we can note a few key things. First, we have no idea

## The Overlap Graph

## The De Bruijn Graph

## Euler's Theorem

## Finding Eulerian Cycles

