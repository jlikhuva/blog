# Managing Data from Large Scale Single Cell Experiments

## Introduction

- sequencing become a dominant method in molecular biology
- Reduction to sequencing
- Such experiments generate a lot of heterogenous data.
- Currently, no canonical tools or pipelines for processing and managing this data.
- Document the ad hoc processes that we've developed in our lab.. At the end encapsulate those processes into  a `CLI` tool and a RabbitMQ based pipeline.

## Wet Lab Protocols

### General

- Before jumping in, it may be useful to spend some time understanding how the data is generated. These present some of the most remarkable breakthroughs in micro-fluidics in the past two decades.
- Talk about single cell, but we often deal with single nuclei, not cells (because we are often not working with fresh tissue samples). We won't make a distinction nonetheless
- Single Cell Profiling procedures:
- Cell Isolation — DropSeq
- How to identify cells — Cellular Barcodes
- How to identify transcripts — UMIs
- Problems that may arise: Doublets/Multiplets, Dead/Stressed cells, Empty droplets

### GEX

- Explanation of reverse transcription
- Difference between nuclear and cytoplasmic RNA
  - Intron splicing
  - Poly-A tail
- Ambient RNA

### ATAC

- Chromatic Accessibility and its importance.
- What are Transposases and what do they do?

### Reduction to Sequencing

- PeturbSeq
- Spatial Sequencing

Protocols often make use of CellRanger suite of reagents and machines. Regardless of the specific protocol employed, end result is a library that can be sequenced on HTS machines (often Illumina). Before we can use this data to make molecular discoveries, we need to preprocess it.The rest of this note will explain how to do that in a robust, reproducible, and efficient manner.

## Data Pre-processing

### Goals

- Use the CellRanger suite of computational tools to manage and preprocess the data.
- Memory and Compute hungry. Cannot be done on local machines. Make use of cloud compute instances
- Produce copious amounts of intermediate and final data -- easy to exhaust disk space. Make use of Cloud Storage. Explain using GCS, but S3 achieves the same.
- Judiciously keep logs to help keep track of long running experiments.

Begin by explaining how to accomplish these tasks in a manual non scalable fashion. Then use the knowledge we gain from doing so to build an automated pipeline.

### BCL to FASTQ conversion and Archival

```rust
//! TODO
```

### Generating the Expression Matrices

```rust
//! TODO
```

### Automating the Pipelines

## Conclusion & Future Work
