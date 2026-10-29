---
title: "SUDO"
date: 2025-11-10
summary: "Suck it Up and DO. A real-time kanban board where any card can become its own board, login is a one-time code, and the delete button actually deletes."
status: "current"
---

**A kanban board for people who'd rather finish the task than configure the tool.**

[Try it](https://sudo-kanban.up.railway.app) · [Source on GitHub](https://github.com/paarth-sharma/sudo)

## Overview

The name is the whole pitch: Suck it Up and DO.

SUDO is a kanban board with what you need and not much else. Make a board, add columns, drag cards across them, invite people. You log in with a one-time code sent to your email, so there's no password to forget.

A few things I cared about building in:

- **Cards that grow up.** When a task turns out to be a project, turn it into its own sub-board instead of cramming a checklist into the description.
- **Live collaboration.** Moves show up for everyone on the board as they happen, and you can see who's online.
- **Data you can take back.** Delete your account and everything goes with it. Nothing is kept on my side.

The rest is what you'd expect: multiple assignees, deadlines, priorities, global search, dark and light mode. Emails and OTPs are encrypted with AES-256-GCM, and Postgres row-level security guards access to boards.

Under the hood it's server-rendered. Go builds the HTML, HTMX handles the interactivity, and WebSockets carry the live updates, so there's very little JavaScript to maintain.

## Technologies Used

- Go with Gin
- HTMX and Templ for server-rendered, type-safe templates
- TailwindCSS
- PostgreSQL on Supabase, with row-level security
- WebSockets for real-time updates
- Resend for OTP emails
- Docker and nginx for self-hosting

## Outcome

SUDO is live and I'm still working on it. It's self-hostable with Docker, and I run a copy for friends and family. The license is MIT with the Commons Clause, so it's free to use and host yourself but not to resell.

What I took from it:

- HTMX plus server-side rendering gets you a real-time app with far less frontend code than I expected.
- Security is mostly a pile of unglamorous decisions: sessions, hashed and encrypted codes, access rules at the database level.
- Putting code in public as a fresh grad is humbling and useful. The repo is open to pointers and contributors.

Next up: a proper test suite, performance work, file attachments, labels and a Gantt view.