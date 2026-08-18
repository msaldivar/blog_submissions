---
title: How do I add authentication to a Next.js app with SuperTokens?
description: "Learn how to add authentication to your Next.js app. Step-by-step guide on login, signup, and sessions with SuperTokens."
date: "2026-08-20"
cover: "TODO.png"
category: "next.js, auth, guide"
author: "Maurice Saldivar"
---


# How do I add authentication to a Next.js application?

Authentication is a solved problem in most frameworks. In Next.js it keeps getting re-solved, because the framework splits your application across more execution contexts than almost anything else in mainstream web development. Server components, client components, middleware, and route handlers all need to know who the user is, and each one reads cookies and validates sessions differently. A session hook that works in a client component won't run in a server component. Middleware defaults to the edge runtime, where plenty of Node libraries won't load. Layer in the App Router and Pages Router split, SSR hydration, and aggressive caching on platforms like Vercel, and "add a login page" quietly becomes an architecture decision.

You have three realistic paths. Build auth yourself and own token rotation, CSRF protection, and refresh logic forever. Hand it to a managed provider and accept per-user pricing and vendor lock-in. Or run an open source framework inside your own stack.

SuperTokens takes the third path. It's an open source, extensible auth framework: a core service you can self-host for free, backend and frontend SDKs that embed directly into your Next.js codebase, and prebuilt UI for login, signup, and session management. Your user data stays in your database, and recipe overrides let you customize almost any part of the flow.

This guide walks through adding authentication to a Next.js app with SuperTokens: setup, login, signup, and sessions. If you're specifically on the App Router, our companion walkthrough, [Adding login to your Next.js app using the App Directory and SuperTokens](https://supertokens.com/blog/adding-login-to-your-nextjs-app-using-the-app-directory-and-supertokens), goes deeper on that pattern.

## What do I need before adding authentication to my Next.js app?

Not much. If you've shipped anything with Next.js before, you probably have everything on this list already.

**Node.js and a Next.js project.** Install the current LTS release of Node.js; recent Next.js versions enforce a minimum runtime and refuse to start on anything older. If you don't have a project yet, `npx create-next-app@latest` scaffolds one in under a minute. SuperTokens supports both the App Router and the Pages Router. The examples in this guide use the App Router.

**Working React and Next.js knowledge.** You don't need to be an expert, but you should be comfortable with components, hooks, and the difference between client and server components. Authentication in Next.js lives exactly at that boundary, so knowing which side of `'use client'` your code runs on will save you real debugging time. If you can write a route handler under `app/api/`, you know enough.

**Environment variables for your SuperTokens config.** SuperTokens needs a connection URI pointing at a core instance, an API key if your core requires one, and OAuth credentials for any social providers you add. Put them in `.env.local`, which Next.js loads automatically:

```bash
# .env.local
SUPERTOKENS_CONNECTION_URI=https://try.supertokens.io
# Only needed for the managed service or a self-hosted core with API keys enabled
SUPERTOKENS_API_KEY=your-api-key
GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
```

The `try.supertokens.io` core is a free shared instance meant for development. It works for this guide, but swap in your own self-hosted core or a managed instance before production. One Next.js-specific warning: anything prefixed with `NEXT_PUBLIC_` gets bundled into the browser. Your API key and client secrets should never carry that prefix.

## How do I install SuperTokens in my Next.js project?

One command covers it:

```bash
npm install supertokens-node supertokens-auth-react supertokens-web-js nextjs-cors
```

Starting a project from scratch instead? `npx create-supertokens-app@latest` scaffolds a Next.js app with all of this wired up. If you're adding auth to an existing codebase, it's worth knowing what each package does, because the split mirrors where authentication work actually happens.

**`supertokens-node` is the backend SDK.** It runs inside your route handlers under `app/api/`, exposes the auth endpoints (sign up, sign in, sign out, session refresh) so you never write them by hand, and verifies sessions in server components and protected API routes. It's also the only piece that talks to the SuperTokens core, which means your connection URI and API key stay server-side.

**`supertokens-auth-react` is the frontend SDK.** It renders the prebuilt login and signup UI, tracks session state in the browser, and refreshes tokens transparently when they expire. You'll initialize it once in a client component and mostly forget it exists.

The other two are supporting cast. `supertokens-web-js` is the lower-level browser SDK that `supertokens-auth-react` builds on: npm pulls it in automatically as a peer dependency, but yarn and pnpm want it listed explicitly, and it's the package you'd reach for if you build a fully custom UI. `nextjs-cors` sets CORS headers on your API routes, which matters if your frontend and API ever live on different domains and costs nothing when they don't.

In most stacks the backend and frontend SDKs live in separate repositories. Next.js is the exception: everything installs into one project, because the framework serves your React code and your API routes from the same codebase. Note what that command does not install: the SuperTokens core. The core is a standalone service, not an npm dependency. During development, the shared `try.supertokens.io` instance fills that role; in production, your connection URI points at a core you run in Docker or a managed instance.
