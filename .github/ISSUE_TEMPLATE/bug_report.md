name: Bug report
description: Something behaves incorrectly or panics
labels: ["bug"]
body:
  - type: textarea
    id: what-happened
    attributes:
      label: What happened?
      description: A clear description of the bug, including any panic message or unexpected behavior.
      placeholder: Describe what you observed vs. what you expected.
    validations:
      required: true
  - type: textarea
    id: reproduction
    attributes:
      label: Steps to reproduce
      description: A minimal code sample (```rust block) or command sequence.
      placeholder: |
        ```rust
        // minimal code that triggers the issue
        ```
    validations:
      required: true
  - type: input
    id: crate
    attributes:
      label: Affected crate
      description: Which workspace member is involved (cordis-core, cordis-loader, cordis-hmr, ...)?
      placeholder: cordis-core
  - type: textarea
    id: environment
    attributes:
      label: Environment
      description: `rustc -V` output, OS, and how the crate was obtained (git checkout, generated project).
    validations:
      required: true
