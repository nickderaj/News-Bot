<h1 align="center">News</h1>

<p align="center">
  A private, phone-first reader for the owner's FT and Bloomberg subscriptions, with questions answered from the articles and any article read aloud on demand.
</p>

## How it works

A scheduled job signs in to each subscription with a browser session the owner
set up once, collects new articles, and stores them in SQLite. The web app shows
them in a clean reader. **Listen** narrates an article with xAI's Zagan voice;
each clip is generated only on request, once, and kept. **Ask** answers
questions about the stored articles through Hermes, citing the articles it used.

Articles, clips and sessions stay on the host; nothing licensed is ever
committed here.

## Status and docs

Being built: see the [plan](docs/PLAN.md). Every change passes the
[quality gate](docs/DEVELOPMENT.md).
