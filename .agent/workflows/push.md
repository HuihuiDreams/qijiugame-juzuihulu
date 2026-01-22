---
description: Automatically commit and push changes to GitHub after every edit.
---

// turbo-all

1. Execute the following command to commit and push safely (PowerShell compatible). Do not use '&&' as it is not supported in all PowerShell versions.
   `git add .; if ($?) { git commit -m "Auto-commit: [brief description of changes]"; if ($?) { git push origin master } }`
