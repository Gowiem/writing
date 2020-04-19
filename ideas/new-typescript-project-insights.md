# Things to know before starting a TypeScript Project

1. Always check for @types packages
1. How to build and export typings properly
1. Accept that it is a compiled language and write it as so.
  - Wrote things half functionally and half OO. Should have gone all in on OOP.
  - Avoided dependency injection, typedi is great though and worth the investment.
1. Odd to know what needs types and what doesn't... follow this pattern:
  - All functions are strongly typed. Local vars can be untyped if explicitly given a value.
