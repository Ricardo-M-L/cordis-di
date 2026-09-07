name: Feature request
description: Suggest a new capability or an improvement
labels: ["enhancement"]
body:
  - type: textarea
    id: problem
    attributes:
      label: What problem does this solve?
      description: Describe the use case that current behavior does not cover.
    validations:
      required: true
  - type: textarea
    id: proposal
    attributes:
      label: Proposed solution
      description: What API or behavior you would like to see. If it mirrors something in the original Cordis (TypeScript) framework, link or quote the relevant part.
    validations:
      required: true
  - type: textarea
    id: alternatives
    attributes:
      label: Alternatives considered
      description: Other designs, workarounds, or reasons a plain Rust idiom could be preferred here.
