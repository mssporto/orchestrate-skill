---
tags: [orchestrate, gate-in]
runs: 1
scaffold_script: |
  #!/usr/bin/env bash
  set -euo pipefail
  mkdir -p src/api/routes/__tests__ src/dashboard/forms/__tests__

  cat > src/api/routes/teams.js <<'EOF'
  const express = require('express');
  const router = express.Router();

  // Existing pattern: validate body, 400 on bad input, 409 on duplicate.
  router.post('/teams', (req, res) => {
    const { name, ownerId } = req.body || {};
    if (!name || !ownerId) {
      return res.status(400).json({ error: 'name and ownerId are required' });
    }
    if (teamsByName.has(name)) {
      return res.status(409).json({ error: 'a team with this name already exists' });
    }
    const team = { id: `team_${Date.now()}`, name, ownerId };
    teamsByName.set(name, team);
    return res.status(201).json(team);
  });

  const teamsByName = new Map();

  module.exports = router;
  EOF

  cat > src/api/routes/__tests__/teams.test.js <<'EOF'
  const request = require('supertest');
  const express = require('express');
  const teamsRouter = require('../teams');

  function buildApp() {
    const app = express();
    app.use(express.json());
    app.use(teamsRouter);
    return app;
  }

  describe('POST /teams', () => {
    it('rejects a request missing required fields', async () => {
      const res = await request(buildApp()).post('/teams').send({});
      expect(res.status).toBe(400);
    });

    it('creates a team given valid input', async () => {
      const res = await request(buildApp())
        .post('/teams')
        .send({ name: 'Rockets', ownerId: 'user_1' });
      expect(res.status).toBe(201);
      expect(res.body.name).toBe('Rockets');
    });
  });
  EOF

  cat > src/dashboard/forms/CreateTeamForm.jsx <<'EOF'
  import { useState } from 'react';

  // Existing pattern: local state for the field, a submitting flag, and a
  // slot to surface a server-side error string back to the user.
  export function CreateTeamForm({ onSubmit }) {
    const [name, setName] = useState('');
    const [submitting, setSubmitting] = useState(false);
    const [error, setError] = useState(null);

    async function handleSubmit(event) {
      event.preventDefault();
      if (!name.trim()) {
        setError('Team name is required');
        return;
      }
      setSubmitting(true);
      setError(null);
      try {
        await onSubmit({ name });
      } catch (err) {
        setError(err.message || 'Something went wrong');
      } finally {
        setSubmitting(false);
      }
    }

    return (
      <form onSubmit={handleSubmit}>
        <label htmlFor="team-name">Team name</label>
        <input
          id="team-name"
          value={name}
          onChange={(event) => setName(event.target.value)}
        />
        {error && <p role="alert">{error}</p>}
        <button type="submit" disabled={submitting}>
          {submitting ? 'Creating…' : 'Create team'}
        </button>
      </form>
    );
  }
  EOF

  cat > src/dashboard/forms/__tests__/CreateTeamForm.test.jsx <<'EOF'
  import { render, screen, fireEvent } from '@testing-library/react';
  import { CreateTeamForm } from '../CreateTeamForm';

  test('shows a validation error when the name is empty', () => {
    render(<CreateTeamForm onSubmit={jest.fn()} />);
    fireEvent.click(screen.getByRole('button', { name: /create team/i }));
    expect(screen.getByRole('alert')).toHaveTextContent(/required/i);
  });
  EOF
---

Add a `POST /invites` endpoint to the API with request validation (email, role, and an optional expiry), a matching "Invite teammate" form in the admin dashboard that calls it, and test coverage for both the endpoint and the form.

This codebase already has a similar endpoint, form, and test pair you should follow the pattern of:
- `src/api/routes/teams.js` and its test at `src/api/routes/__tests__/teams.test.js` — the existing shape for a validated, duplicate-checking POST route.
- `src/dashboard/forms/CreateTeamForm.jsx` and its test at `src/dashboard/forms/__tests__/CreateTeamForm.test.jsx` — the existing shape for a form with client-side validation, a submitting state, and a way to surface a server error.

Requirements:
- The `POST /invites` endpoint should validate the request body (email, role, optional expiry) and return a clear 4xx error for invalid input or a duplicate invite for the same email, following the pattern in `teams.js`.
- The new "Invite teammate" form needs client-side validation, a loading state while the request is in flight, and a way to surface a server-side error (e.g. duplicate invite) to the person filling it in, following the pattern in `CreateTeamForm.jsx`.
- Add tests for the endpoint's validation/duplicate-handling logic and for the form's submit/error states, following the pattern in the existing test files.

This touches the API layer, the dashboard UI, and two separate test suites — please get all three pieces done.
