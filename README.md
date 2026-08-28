# Silverstripe Gate Tasks

Reach a [SilverGate](https://github.com/atwx/silverstripe-gate-client) managed
site from your own command line: run a build task on it, or pull its database
and assets down to where you are working.

Install it where you develop. Nothing needs it on the site you are aiming at
beyond `gate-client`, which is already there for the website login.

```bash
composer require --dev atwx/silverstripe-gate-tasks
```

## What has to be reachable

Neither SilverGate module is a Composer dependency of this one — everything here
talks HTTP. What it needs is that the pieces exist:

| Where | What | Why |
| --- | --- | --- |
| the machine you work at | this module | the tasks and the callback route |
| the manager | `gate-manager` | signs the JWT, and runs the consent flow |
| the manager | a token endpoint | see below — **not** shipped by any module |
| the site you are aiming at | `gate-client` | trades that JWT for a session |
| the site you pull from | `atwx/silverstripe-projectinfo` | serves the endpoints `pull-live` reads |

**The token endpoint is yours to provide.** `--intranet-url` defaults to
`https://intra.atw.io/_api/token`, which is application code at ATW rather than
anything a module installs. Point the option elsewhere and implement the other
end: it takes a `domain` plus either a `token` form field or an
`Authorization: Bearer` header, resolves that to a member, and answers with

```json
{ "jwt": "…", "domain": "www.example.com", "scope": "read", "member": "someone@example.com" }
```

The `domain` it returns is the spelling the manager holds, which the tasks then
use — see [www](#www).

`tasks:remote` additionally needs the account the manager signs for to hold
`ADMIN` on the target site, because `/dev/tasks` is gated on it.

## tasks:pull-live

Copies a live site onto the machine you are sitting at: database first, then
assets, then imports both.

```bash
sake tasks:pull-live -u www.example.com
sake tasks:pull-live -u www.example.com --only-db
sake tasks:pull-live -u www.example.com --only-assets
```

The dump lands in `_livedata/db/`, the assets in `_livedata/assets/`, and both
are then imported over the local database and `ASSETS_PATH`. The download step
talks to the export endpoints above, so the module has to be installed on the
site being pulled from as well.

| Option | |
| --- | --- |
| `-u`, `--url` | the site to pull from, with or without the `www.` |
| `--only-db` / `--only-assets` | skip the other half |
| `--http-user` / `--http-pass` | HTTP basic auth, if the site is behind it |
| `-t`, `--token` | a personal access token instead of the browser flow |
| `-i`, `--intranet-url` | a manager other than the default |

Importing runs `mysql` against the credentials in the local environment. It
overwrites the local database without asking.

## tasks:remote

Runs a build task **on** a managed site.

```bash
sake tasks:remote SendMessagesTask -u www.example.com
sake tasks:remote ExportEmployers -u www.example.com > employers.csv
sake tasks:remote ExportEmployers -u www.example.com -- --with-offers
```

There is no API behind this and none is needed: the site's own `/dev/tasks`
runner does the work, reached with the session the module already establishes.
Silverstripe streams task output as it is produced, so it arrives line by line
rather than in one lump at the end. Everything the module itself has to say —
progress, the authorisation link, errors — goes to **stderr**, which is what
makes redirecting the output with a plain `>` give you just the task's own
output.

Anything after a `--` is passed on to the remote task as options.

> **Options with a shortcut cannot be passed by their long name.**
> `HttpRequestInput` in the framework overwrites the long name's value with the
> shortcut's, so `--only-db` arrives and `--url=…` does not, while `-u=…` does.
> Declare options without a shortcut on tasks meant to be run this way.

A task whose name the site does not know is reported as a failure. The runner
answers `200` either way and says so only in the page, so the exit code comes
from reading that.

## Authorisation

Both tasks need to prove who they are to the manager, which then signs a token
for the target site. There are two ways.

**Through the browser, by default.** The first call prints a link to the
manager's consent screen; approving it once yields a grant that renews itself
quietly from then on. Nothing to copy, nothing to keep in an environment file.
`tasks:pull-live` asks for read access, `tasks:remote` for write.

**With a personal access token**, by passing `--token`. Useful where no browser
is available, in cron or CI. These rotate, so they have to be fetched again each
time.

The grant lives in `~/.silvergate/<manager-host>.json`, one file per manager and
readable only by its owner. Deleting it asks for consent again.

### How the code gets back to the command line

A command line process cannot receive a browser redirect, and a loopback port
inside a container is not reachable from the browser on the host. The
development site is, so it is used as the landing spot:

```
CLI      prints the consent link, waits
browser  you approve at the manager
manager  redirects to https://<your-dev-site>/_silvergatecallback?code=…
site     OAuthCallbackController leaves the code in the temp directory
CLI      picks it up, exchanges it, carries on
```

The command line process and the web server run as the same user on the same
machine, so a file is all the two of them need. The code on its own is worth
nothing: exchanging it takes the PKCE verifier, which never leaves the process
that started the flow.

That route is registered **only in dev mode**, and the controller checks again
before answering. It has no business existing on a production site.

The callback URL is taken from `DDEV_PRIMARY_URL`, falling back to
`SS_BASE_URL`. `Director::absoluteBaseURL()` is deliberately last: on the command
line it reports the container's internal hostname, which is neither reachable
from a browser nor `https`.

### www

Sites are recorded at the manager with or without a `www.`, and `-u` accepts
either. The manager answers with the spelling it actually knows, and the tasks
follow it from there — putting a `www.` in front of a site that has none tends
to fail the certificate check, so guessing is not an option.

## Requirements

PHP 8.3 and Silverstripe 6. `tasks:pull-live` shells out to `mysql` to import
what it downloaded, so that has to be on the path of whatever runs it.
