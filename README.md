# liftoff — Django + Celery on Railway

A minimal Django 5 project wired up with Celery (worker + beat), Redis as the
broker/result backend, and Postgres, deployed on Railway with the Railpack
builder.

## Services

This one repo backs three Railway services:

| Service  | Start command                        |
| -------- | ------------------------------------ |
| `web`    | `gunicorn liftoff.wsgi`              |
| `worker` | see `railway.worker.json`            |
| `beat`   | see `railway.beat.json`              |

The `worker` and `beat` services read their start command from a config file
instead of the dashboard. Point each one at its file under
**Settings → Config-as-code → Railway Config File**:

- `worker` → `railway.worker.json`
- `beat` → `railway.beat.json`

There is deliberately no `railway.json` at the repo root, because Railway
applies that path to every service by default and it would clobber the `web`
service's own settings.

## Why the Celery services pass `--uid`/`--gid`

Deploying a Celery worker on Railway logs this on every boot:

```
SecurityWarning: You're running the worker with superuser privileges: this is
absolutely not recommended!
Please specify a different user using the --uid option.
User information: uid=0 euid=0 gid=0 egid=0
```

Railpack's final image is built from `debian:trixie-slim` with no `USER`
directive, and `railpack.json` has no field for setting one — so the container
process starts as root and Celery notices.

`celery worker` and `celery beat` both accept `--uid`/`--gid` and call
`maybe_drop_privileges()` before the worker boots, so switching to the
unprivileged `nobody`/`nogroup` account (uid/gid 65534, present in the Debian
base image) both silences the warning and actually fixes what it is warning
about.

Two things worth knowing:

- `C_FORCE_ROOT` does **not** silence this. That env var only gates the hard
  `SecurityError` raised when running as root with a pickle-based serializer.
  The warning above fires unconditionally whenever `uid == 0`.
- `nobody` cannot write to `/app`. That is fine here — the worker writes
  nothing to disk, and beat uses `django_celery_beat.schedulers:DatabaseScheduler`
  (set via `CELERY_BEAT_SCHEDULER` in `liftoff/settings.py`), so its schedule
  lives in Postgres rather than in a `celerybeat-schedule` file. If you ever
  switch back to the default `PersistentScheduler`, give it a writable path with
  `--schedule=/tmp/celerybeat-schedule`.

If you would rather have the whole container run unprivileged, swap Railpack for
a `Dockerfile` and add a `USER` directive after the dependency install. Railway
picks up a root `Dockerfile` automatically. Applying `--user nobody --group
nogroup` to the `gunicorn` command gives the `web` service the same treatment
without leaving Railpack.

## Local development

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver
```

In separate shells:

```bash
celery -A liftoff worker -l info
celery -A liftoff beat -l info
```

No `--uid` locally — you are not running as root.
