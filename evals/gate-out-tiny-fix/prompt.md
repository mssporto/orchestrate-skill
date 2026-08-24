---
tags: [orchestrate, gate-out]
runs: 1
scaffold_script: |
  #!/usr/bin/env bash
  set -euo pipefail
  mkdir -p src/auth
  cat > src/auth/passwordReset.js <<'EOF'
  function handleExpiredResetToken() {
    // TODO: this message is stale and confuses users who just want to
    // request a new link.
    return { status: 400, error: "token invalid" };
  }

  module.exports = { handleExpiredResetToken };
  EOF
---

The error message returned by `handleExpiredResetToken()` in `src/auth/passwordReset.js` currently says `"token invalid"`. Update it to say something like "This reset link has expired — request a new one." That's the only change needed; it's a single string in one file.
