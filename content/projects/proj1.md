+++
title = "Lox Interpreter"
description = "A Interpreter for the Lox programing language"
weight = 1
date = 2026-01-01

[extra]
github = "https://github.com/TallCheesecake/lox"
tags = ["rust", "languages"]

[taxonomies]
tags=["rust", "lox"]
+++

# Lox Interpreter
This is a Rust interpreter for the Lox programming language. It is broken down
into a front end, middle end, and back end.

The front end consists of a scanner, parser, and resolver. The middle end is
responsible for IR generation and optimization. The IR I am planning to use is
a linear SSA form, similar to a simplified version of what LLVM does. The back
end is not finished yet, but will be a simple stack-based VM.

{{ note( header="Note", body='For all Lox-related blogs, please refer to the
"lox" tag under "/tags".') }}
