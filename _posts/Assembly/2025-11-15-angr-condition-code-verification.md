---
title: Angr condition Code Verification
date: 2025-11-15
categories: [Assembly]
tags: [assembly,python,angr]     # TAG names should always be lowercase
published: false
---

> Problem: I need to verify different condition codes between guset and hose symbolic machines, but how to verify them across different isas?

# Overviews

## guest side
- 1. init symbol machine
- 2. run symbol machine
- 3. extract final results from guest machine. (g_state)


## host side
- 1. init host side according to the guest state's initialized state.
- 2. run host machine
- 3. extract final state from host machine. (h_state)

## compare guest and host state
