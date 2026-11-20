# moodle-teacher-mcp — plan

MCP server for Moodle teachers: review submissions, grade with feedback, manage deadlines and announce — safely, from Claude.

## 0. Positioning

- Teacher-focused. Most existing Moodle MCP servers are read-only and student-oriented.
- **Core Moodle web services only, no companion plugin.** Works on any Moodle 4.5+ with REST enabled.
- Limitation: core has no web service to create an assignment (`core_courseformat_create_module` only covers quick-create modules; `assign` is not one — verify). Creating sections/activities is milestone M6, as an optional companion plugin.

## 1. Tools (v1)

### Read (`readOnlyHint`)

| Tool | Moodle WS function |
|---|---|
| `list_courses` | `core_enrol_get_users_courses` |
| `get_course_contents` | `core_course_get_contents` |
| `list_participants` | `core_enrol_get_enrolled_users` |
| `list_assignments` | `mod_assign_get_assignments` |
| `list_submissions` (filters: not submitted / not graded / late) | `mod_assign_get_submissions` + `mod_assign_list_participants` |
| `get_submission` (online text + files; `.py`/`.sql` as text) | `mod_assign_get_submission_status` + `webservice/pluginfile.php` |
| `get_grades` | `gradereport_user_get_grade_items` |

### Write (`destructiveHint`; bulk operations default to `dry_run=true`)

| Tool | Moodle WS function |
|---|---|
| `grade_submission` (grade + feedback) | `mod_assign_save_grade` |
| `grade_submissions_bulk` | `mod_assign_save_grades` |
| `grant_extension` | `mod_assign_save_user_extensions` |
| `revert_to_draft` | `mod_assign_revert_submissions_to_draft` |
| `post_announcement` | `mod_forum_add_discussion` |
| `message_students` | `core_message_send_instant_messages` |

### Safety rules

- Bulk operations return a preview first ("12 grades, 3 students without a submission"); they apply only when called again with `confirm=<preview id>`.
- `notify_students=false` by default.
- If the assignment uses marking workflow, grades go to "in review", not released.

## 2. Repo layout

```
src/moodle_teacher_mcp/
  __main__.py      # CLI: stdio (default) | --http --port
  server.py        # MCPServer, tool registration
  config.py        # pydantic-settings: MOODLE_URL, MOODLE_TOKEN
  client.py        # async httpx WS client; Moodle errors -> clear ToolError
  models.py        # pydantic models of WS responses (only the fields we use)
  tools/  courses.py  assignments.py  grading.py  communication.py
  safety.py        # dry-run, previews, confirm tokens
tests/
  unit/            # respx mocks, fast
  integration/     # real Moodle in Docker, @pytest.mark.moodle
docker/            # compose: Moodle + Postgres + seed (course, teacher, 5 students, submissions)
.github/workflows/ ci.yml  release.yml
server.json        # MCP Registry
README.md  CHANGELOG.md  LICENSE (MIT)
```

## 3. Auth

- **stdio:** `MOODLE_URL` + `MOODLE_TOKEN` from env.
- **HTTP:** token per request (`Authorization: Bearer`), one process for many teachers. No MCP OAuth for now: Moodle core is not an OAuth provider (document as a limitation).
- README: where to get a token — Profile → Security keys (`moodle_mobile_app` service) or a token from the admin. Check on the Docker Moodle which functions above are in the mobile service; if `mod_assign_save_grade` is not, document how an admin creates an external service with the needed functions.

## 4. Tests and CI

- **unit:** pytest + respx, no network.
- **integration:** Docker Moodle (reuse `dev/` from moodle-ai-teacher). Seed via `admin/tool/generator` or WS. Scenario: list missing submissions → grade → grade visible in gradebook.
- **CI:** `ruff`, `pyright --strict`, unit; integration on a matrix of Moodle **4.5 LTS** and **5.x**.
- **release.yml:** on tag `v*` → `uv build` → PyPI trusted publishing (no token secrets) → MCP Registry.

## 5. Milestones

| # | Scope | Done when |
|---|---|---|
| M0 | Skeleton, `uv`, ruff/pyright, CI with unit tests | green CI on an empty server |
| M1 | client.py + read tools + stdio | Claude Code lists your courses on the Docker Moodle |
| M2 | Docker Moodle + seed + integration tests in CI | 4.5 / 5.x matrix green |
| M3 | Grades, feedback, extensions, dry-run/confirm | full grading scenario in an integration test |
| M4 | Announcements, messages, HTTP transport | `claude mcp add --transport http …` works |
| M5 | README with GIF, PyPI, MCP Registry, v0.1.0 | `uvx moodle-teacher-mcp` installs from scratch |
| M6 (optional) | Companion plugin to create sections and assignments | separate repo, GPLv3 |

After M5: pin the repo on the GitHub profile and link it from the profile README.
