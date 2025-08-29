---
title: bash
tags:
  - "#reference"
---
Topics:  [[DevOps]]

---

## Basic template
```sh
#!/usr/bin/env bash
set -euo pipefail
```
-   `set -e`: exit on error
-   `set -u`: using undefined vars aborts script
-   `set -o pipefail`: catch pipeline failures. If one command fails, whole pipeline fails

