<!--
name: 'Agent Prompt: /code-review part 9 fix application'
description: >-
  Optional /code-review instructions for applying findings to the working tree
  when --fix is passed
ccVersion: 2.1.152
-->


## Applying fixes (--fix)

The \`--fix\` flag was passed. After producing the findings list, apply the
findings to the working tree instead of stopping at the report: fix each one
directly — correctness bugs and reuse/simplification/efficiency cleanups alike.
Skip any finding whose fix would change intended behavior, require changes well
outside the reviewed diff, or that you judge to be a false positive — note the
skip rather than arguing with it. Finish with a thorough summary of what was
fixed, what was skipped (and why), and any noteworthy patterns or related issues
the review surfaced. Give the user the full picture so they can see the value of
the pass and understand the state of the code afterward.
