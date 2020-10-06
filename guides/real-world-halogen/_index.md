+++
title = "Real World Halogen [Draft]"
category = "Development"
date = "2019-01-01"
layout = "guide-landing"
description = "This guide demonstrates how to build a real world single-page application in PureScript and its most popular framework, Halogen. It comes with over 2,000 lines of commented code so you can see exactly how the ideas presented here translate to idiomatic PureScript."
+++

Functional languages like PureScript, ReasonML, and Elm offer powerful features to manage complexity and help you reliably design, build, and refactor apps of any size. Yet, despite functional programming's long history, there are few resources that teach how to build a non-trivial applications with these languages.

This guide demonstrates how to build a real world single-page application in PureScript and its most popular framework, Halogen. It comes with over [2,000 lines of commented code](https://github.com/thomashoneyman/purescript-halogen-realworld) so you can see exactly how the ideas presented here translate to idiomatic PureScript.

I'm a senior developer at [Awake Security](https://awakesecurity.com) (acquired by Arista Networks) and previously worked at [CitizenNet](https://citizennet.com) (acquired by Conde Nast). Both companies leverage PureScript to build single-page applications. I'm convinced it's the best language for the web available today, and by the end of this guide you'll be well-equipped to use it to build reliable applications of your own.

{{< subscribe >}}

### A word of caution

This rough draft contains example code that may not be up-to-date with the [thoroughly-commented source code](https://github.com/thomashoneyman/purescript-halogen-realworld). The examples in this guide should work, but refer to the source code for the up-to-date code.

### Prerequisites

This is not a gentle introduction to PureScript or Halogen. It is intended for advanced beginners or intermediate PureScript developers who can build small Halogen apps but don't yet feel comfortable building real world applications with the language & framework. If you feel lost when you begin reading, I recommend checking out learning resources including:

- [PureScript By Example](https://github.com/dwhitney/purescript-book)
- [The official Halogen guide](https://github.com/purescript-halogen/purescript-halogen/)
- Jordan Martinez's [learning repository](https://github.com/JordanMartinez/purescript-jordans-reference/) and notes on [getting started with PureScript](https://github.com/JordanMartinez/purescript-jordans-reference/blob/latestRelease/00-Getting-Started/01-Install-Guide.md) and [learning Halogen from the bottom up](https://github.com/JordanMartinez/purescript-jordans-reference/blob/latestRelease/21-Hello-World/09-Projects/src/01-Node-and-Halogen/02-Halogen.md)

