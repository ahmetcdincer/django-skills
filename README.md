# django-skills

A Claude Code skill that teaches Django best practices through wrong vs. correct code examples. Covers 27 topics and 213 sub-topics, from project structure to deployment.

When installed, Claude Code automatically applies these patterns when working on Django projects — writing better queries, avoiding N+1 problems, using correct field types, following security best practices, and more.

## Installation

```bash
npx skills add Reyretee/django-skills --skill django-best-practices
```

## What It Does

This skill provides Claude Code with a senior Django developer's knowledge base. Every topic includes:

- A **wrong** example showing the common mistake and why it fails
- A **correct** example showing the production-ready approach
- A short **explanation** of why the correct approach is better

Claude uses this knowledge automatically when generating, reviewing, or refactoring Django code.

## Topics Covered

| #  | Section                    | What It Covers                                                        |
|----|----------------------------|-----------------------------------------------------------------------|
| 1  | Django Core                | Project structure, app layout, settings split, URL config, WSGI/ASGI  |
| 2  | Models (ORM)               | Field types, ForeignKey, M2M, abstract models, proxy models, managers |
| 3  | Django ORM                 | QuerySets, Q/F objects, aggregation, subqueries, select/prefetch      |
| 4  | Migrations                 | Safe migrations, data migrations, squashing, zero-downtime ops        |
| 5  | Django Admin               | ModelAdmin, inlines, actions, fieldsets, custom views, performance    |
| 6  | Views                      | FBV, CBV, generic views, mixins, when to use which                   |
| 7  | URL Routing                | path/re_path, namespaces, reverse, custom converters                 |
| 8  | Templates                  | Inheritance, tags, filters, custom tags, context processors, security |
| 9  | Forms                      | Forms, ModelForms, validation, formsets, file upload security         |
| 10 | Authentication             | Custom user model, backends, permissions, groups, password reset      |
| 11 | Sessions & Cookies         | Backend selection, cookie security, session expiry and rotation       |
| 12 | Middleware                 | Request lifecycle, ordering, custom middleware, async middleware       |
| 13 | Static & Media Files       | STATIC_ROOT vs STATICFILES_DIRS, WhiteNoise, S3/CDN, collectstatic  |
| 14 | Django Security            | CSRF, XSS, SQL injection, HTTPS, password hashing, SECRET_KEY        |
| 15 | Django Signals             | pre/post_save, m2m_changed, custom signals, when to avoid signals    |
| 16 | Caching                    | Redis, per-view cache, fragment cache, low-level API, invalidation   |
| 17 | File Handling              | FileField, storage API, upload security, Pillow                      |
| 18 | Internationalization       | i18n settings, gettext_lazy, template tags, timezone support          |
| 19 | Testing                    | TestCase types, fixtures, pytest-django, FactoryBoy, mocking         |
| 20 | Django REST Framework      | Serializers, ViewSets, auth, permissions, throttling, filtering      |
| 21 | Background Tasks           | Celery, Django-Q, Huey, Django 6.0 tasks, idempotency                |
| 22 | Deployment                 | Gunicorn, Nginx, Docker, CI/CD, zero-downtime, health checks         |
| 23 | Performance                | Debug Toolbar, N+1, indexing, caching, deferred fields, async views  |
| 24 | Django Channels            | ASGI, WebSocket consumers, channel layers, group messaging            |
| 25 | Django Ecosystem           | DRF, Allauth, Storages, Guardian, Unfold, django-redis, and more     |
| 26 | Advanced Django            | Custom fields, multi-DB, database routers, custom management commands |
| 27 | Django Architecture        | MTV pattern, service layer, DDD, SOLID, anti-patterns                |

## Example

Every sub-topic follows this structure:

```markdown
### select_related

Wrong:
  comments = Comment.objects.all()
  for comment in comments:
      print(comment.post.title)  # N+1 queries

Correct:
  comments = Comment.objects.select_related('post').all()
  for comment in comments:
      print(comment.post.title)  # Single query with JOIN

Why: select_related uses SQL JOINs to fetch related ForeignKey/OneToOne
objects in a single query. Use it whenever you access FK relations in a loop.
```

## File Structure

```
skills/
  django-best-practices/
    SKILL.md          # The skill file (8,000+ lines, 213 sub-topics)
```

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- [skills.sh](https://skills.sh) ecosystem (`npx skills`)

## License

MIT
