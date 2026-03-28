# django-skills

A Claude Code skill and agent for Django best practices. Covers 27 topics and 213 sub-topics through wrong vs. correct code examples — from project structure to deployment.

When installed, Claude Code automatically applies these patterns when working on Django projects — writing better queries, avoiding N+1 problems, using correct field types, following security best practices, and more.

## Installation

### Skill Only

```bash
npx skills add Reyretee/django-skills --skill django-best-practices
```

### Skill + Agent

```bash
npx skills add Reyretee/django-skills --skill django-best-practices
mkdir -p .claude/agents && curl -o .claude/agents/django-developer.md https://raw.githubusercontent.com/Reyretee/django-skills/main/agents/django-developer.md
```

> **Note:** The `npx skills add` command only installs skills. The Django developer agent must be downloaded separately as shown above.

## What It Does

This skill provides Claude Code with a senior Django developer's knowledge base. Every topic includes:

- A **wrong** example showing the common mistake and why it fails
- A **correct** example showing the production-ready approach
- A short **explanation** of why the correct approach is better

Claude uses this knowledge automatically when generating, reviewing, or refactoring Django code.

## File Structure

```
django-skills/
├── skills/
│   └── django-best-practices/
│       ├── SKILL.md                    # Workflow, rules, and reference index
│       └── references/
│           ├── core.md                 # Project structure, settings, WSGI/ASGI
│           ├── models.md               # Field types, relations, Meta, managers
│           ├── queries.md              # QuerySets, Q/F objects, aggregation, N+1
│           ├── migrations.md           # Safe migrations, data migrations, squashing
│           ├── admin.md                # ModelAdmin, inlines, actions, performance
│           ├── views.md                # FBV, CBV, generic views, mixins
│           ├── urls.md                 # path/re_path, namespaces, converters
│           ├── templates.md            # Inheritance, tags, filters, security
│           ├── forms.md                # ModelForms, validation, formsets
│           ├── auth.md                 # User models, permissions, sessions
│           ├── middleware.md            # Request lifecycle, ordering, async
│           ├── static-media.md         # WhiteNoise, S3/CDN, file uploads
│           ├── security.md             # CSRF, XSS, SQL injection, HTTPS
│           ├── signals.md              # pre/post_save, custom signals
│           ├── caching.md              # Redis, per-view, fragment, invalidation
│           ├── i18n.md                 # gettext_lazy, timezone support
│           ├── testing.md              # pytest, factory_boy, mocking, coverage
│           ├── drf.md                  # Serializers, viewsets, permissions
│           ├── celery.md               # Task definition, retries, periodic tasks
│           ├── deployment.md           # Gunicorn, Docker, CI/CD, async views
│           ├── channels.md             # WebSocket consumers, channel layers
│           ├── ecosystem.md            # Allauth, storages, debug toolbar
│           └── architecture.md         # Service layer, DDD, SOLID, multi-DB
└── agents/
    └── django-developer.md             # Senior Django developer agent
```

## Architecture

This repo follows the **progressive disclosure** pattern:

- **SKILL.md** — Slim workflow file with rules and a reference index. No inline code examples.
- **references/** — 23 self-contained reference files with wrong/correct code patterns per topic.
- **AGENT.md** — A senior Django developer agent that preloads the skill and applies structured workflows.

Claude reads only what it needs: SKILL.md identifies the topic, then loads the relevant reference file on demand.

## Topics Covered

| #  | Topic                      | Reference File         |
|----|----------------------------|------------------------|
| 1  | Project Structure          | `references/core.md`   |
| 2  | Models & ORM               | `references/models.md` |
| 3  | ORM Queries                | `references/queries.md` |
| 4  | Migrations                 | `references/migrations.md` |
| 5  | Django Admin               | `references/admin.md` |
| 6  | Views                      | `references/views.md` |
| 7  | URL Routing                | `references/urls.md` |
| 8  | Templates                  | `references/templates.md` |
| 9  | Forms                      | `references/forms.md` |
| 10 | Authentication & Sessions  | `references/auth.md` |
| 11 | Middleware                 | `references/middleware.md` |
| 12 | Static & Media Files       | `references/static-media.md` |
| 13 | Security                   | `references/security.md` |
| 14 | Signals                    | `references/signals.md` |
| 15 | Caching                    | `references/caching.md` |
| 16 | Internationalization       | `references/i18n.md` |
| 17 | Testing                    | `references/testing.md` |
| 18 | Django REST Framework      | `references/drf.md` |
| 19 | Background Tasks           | `references/celery.md` |
| 20 | Deployment & Performance   | `references/deployment.md` |
| 21 | Django Channels            | `references/channels.md` |
| 22 | Django Ecosystem           | `references/ecosystem.md` |
| 23 | Architecture Patterns      | `references/architecture.md` |

## Example

Every sub-topic in the reference files follows this structure:

```markdown
### select_related

**Wrong:**
  comments = Comment.objects.all()
  for comment in comments:
      print(comment.post.title)  # N+1 queries

**Correct:**
  comments = Comment.objects.select_related('post').all()
  for comment in comments:
      print(comment.post.title)  # Single query with JOIN

> **Why:** select_related uses SQL JOINs to fetch related ForeignKey/OneToOne
> objects in a single query. Use it whenever you access FK relations in a loop.
```