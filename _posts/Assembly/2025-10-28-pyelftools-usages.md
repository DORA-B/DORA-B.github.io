---
title: Usage of pyelftools
date: 2025-10-28
categories: [Assembly]
tags: [readelf]     # TAG names should always be lowercase
published: false
---

> Problem: I need to find all the references of all the entries of the GOT, and convert the instruction with their references with the correct labels --> normalization.

> Problem: there are different types of the data loading in the arm assembly file, so I need to identify them and try to map all the corresponding symbols to them.


1. Mapping from the symbols to different data
- Some data are in
  - .got section  --> 
  - .data section --> 
  - .bss section  --> NOBITS


the call to the rel.plt and rel.dyn.
the call routine? from call site to plt stub (in plt section), and then in plt stub call to real rel.plt in .got
