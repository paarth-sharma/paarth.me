---
title: "ComplianceOS"
date: 2026-10-25
summary: "One workspace for Indian CA firms, replacing the WhatsApp + Excel + five-tools mess. Built offline-first so it works on 2G and a Windows 7 laptop."
status: "current"
---

**Indian CA firms run on WhatsApp threads and Excel sheets. I'm building the tool they should have had.**

## Overview

A Chartered Accountant in India juggles dozens of clients, each with their own GST, TDS, income tax and MCA deadlines. The documents arrive over WhatsApp, the tasks live in a spreadsheet, and nobody is quite sure who is waiting on whom.

ComplianceOS is a multi-tenant SaaS practice-management platform that pulls all of that into one place: a compliance calendar, client document collection, task workflow, DSC tracking and billing. It isn't a filing tool or accounting software. It's the workflow layer that sits above GSTN, the Income Tax portal, MCA and Tally.

It's aimed at solo CAs and small firms managing 30 to 200 client entities. The number I'm designing around is the time from "document request sent" to "filing completed", and I want to cut it by 60% against the WhatsApp baseline.

The constraints are what make it interesting:

- **Works on 2G and Windows 7 with Chrome 109.** Plenty of firms run old machines on bad connections, so performance is a feature, with a budget of under 80 KB of JavaScript per route.
- **Offline-first.** IndexedDB is the source of truth and the network is a bonus.
- **Indian all the way down.** Rupee grouping, GSTIN/PAN/CIN validation, and real compliance codes like GSTR-3B, AOC-4 and TDS 26Q.
- **Tenant isolation in the ORM.** Every query is scoped by firm, with no exceptions.
- **Firm-branded client portal.** Clients see their CA's name, not mine.

Right now the foundation is built: the design system, multi-tenant database with an audit log, OTP login with rotating tokens, the offline layer, and a PWA shell. The compliance-calendar rules engine and the rest of the features come next.

## Technologies

- Next.js 16 and React 19, in strict TypeScript
- Tailwind CSS v4 with a typed design system
- PostgreSQL with Prisma 7, plus Redis
- Custom `jose` dual-token auth (OTP login, refresh rotation, revocation)
- Serwist PWA, Dexie (IndexedDB) and TanStack Query for offline-first
- pnpm monorepo, Docker Compose for local infra, Sentry for monitoring
- Planned: Cloudflare R2, BullMQ, Resend, Razorpay


## What I Learned

- **A tight constraint makes decisions for you.** I held myself to the most primitive worst case I came across - "Must run on Windows 7 and 2G" ruled out webfonts, heavy bundles and a lot of fashionable libraries before I wrote a line of UI.
- **Offline-first changes everything.** Once IndexedDB is the source of truth, auth, retries and error handling all have to be designed around the network being flaky, not an afterthought.
- **Off-the-shelf isn't always the fit.** I evaluated Auth.js and dropped it for a custom token scheme, because it didn't cleanly handle refresh rotation, revocation and offline re-auth together.
- **Isolation belongs at the data layer.** Scoping every query by tenant in one place is far safer than trusting every route to remember it.

## Links

The repo is private for now. If you'd like a walkthrough, [email me](mailto:jobs.paarth@gmail.com).