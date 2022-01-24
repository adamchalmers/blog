+++
title = "Parsing bitstreams with Nom"
date = 2022-01-05
description = "How to use Nom to parse binary protocols at the level of individual bits"
draft = true

[taxonomies]
tags = ["rust", "programming", "nom", "parsing"]
+++

In our previous Nom adventure we [parsed text files](/nom-chars.md). But Nom can easily parse binary too! Nom has parsing primitives for reading individual bits. Then, using the same Nom parser combinators from the last post, you can combine those simple bitwise parsers into a more complex parser than parses some binary protocol. In this tutorial, we'll build a parser for the binary protocol in [Advent of Code 2021, day 16](https://adventofcode.com/2021/day/16). 

<!-- more -->
