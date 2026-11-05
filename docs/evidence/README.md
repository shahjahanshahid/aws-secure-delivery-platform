# Evidence

Proof that a control or feature works, kept alongside the claims it supports.
Anything marked "in place" elsewhere in the docs should link to evidence here,
to a pull request, or to a CI run.

## Rules

- Show both sides: a positive test (the allowed thing works) and a negative
  test (the blocked thing is refused).
- Sanitize before committing. No IP addresses, account IDs, hostnames outside
  the documented `role-NN` names, tokens, keys or personal details.
- Prefer text output over screenshots so it can be searched and reviewed.
- One file per capability, named `YYYY-MM-DD-short-name.md`, using the
  [manifest template](manifest-template.md).
