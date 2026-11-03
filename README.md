# Handbook

How I build and ship software: the stack I reach for, how CI is shaped, how things get
deployed, and why. It applies to every project I maintain, whichever GitHub org it lives in;
a project's own docs only record where it differs and why.

| Doc | Read it before |
|---|---|
| [`stack.md`](stack.md) | starting a project, adding a dependency or a service, replacing a library |
| [`ci.md`](ci.md) | adding or changing a workflow, a test suite or a coverage threshold |
| [`deploy.md`](deploy.md) | changing how a project builds images, deploys or rolls back |
| [`decisions.md`](decisions.md) | changing any of the above: the reasons are there |

## For agents

These docs are written to be followed by coding agents as much as by people. Read the doc that
matches the change before making it. If a rule here fights the task, say so in the PR rather
than breaking it silently, and propose the change here.

## What does not belong here

This repo is public. Nothing that names a server, a host, a customer, a person or a secret goes
in: those live in the private infra repo and in each project's config. Neither do the current
gaps of a given project ("X still deploys without tests"): those are issues in that project.

## Changing it

A PR that edits the doc and adds a dated entry to `decisions.md` saying what changed and why.
Rewrite the text that became wrong; the decisions log is the only place that keeps history.
