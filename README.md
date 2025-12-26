# Inside the Engine Room: Building the Xandeum Network Analytics Dashboard

There is a massive difference between *knowing* a network is running and actually *seeing* it breathe.

That was the itch we wanted to scratch with the new **Xandeum Validator Network** dashboard. We didn't just want a static list of IPs; we wanted a living "source of truth"—a place where anyone could jump in and instantly get the pulse of the network, from storage capacity to economic activity.

Here’s a look under the hood at how we built a high-performance analytics platform that can keep up with hundreds of gossiping nodes without breaking a sweat.

## 🚀 The Headache (aka The Challenge)

The Xandeum network is noisy in the best way possible. It consists of hundreds of validator nodes ("pods") that are constantly gossiping data back and forth. Building a dashboard on top of this presented a few specific hurdles:

1. **Sync Reliability:** How do you capture the state of a decentralized network without getting "partial" data? We needed atomic snapshots.
2. **Performance at Scale:** We wanted the dashboard to feel instant. No loading spinners while we query a massive database.
3. **Time Travel:** It's not enough to see what’s happening *now*. We needed to track historical performance (uptime, credits) to spot trends over time.

## 🏗️ The Stack

We opted for a **serverless, event-driven architecture**. We wanted to keep DevOps overhead low while maximizing read throughput.

* **The Face:** Next.js 14 (App Router) with Tailwind for styling and Recharts for the visuals.
* **The Brains:** Next.js API Routes (Serverless Functions).
* **The Memory:** PostgreSQL (via Supabase) paired with Prisma ORM.
* **The Clock:** GitHub Actions for handling cron triggers.

## 🛠️ How We Built It

We leaned heavily on **Next.js 14** and **PostgreSQL**, but the magic is really in how we handle data consistency.

### 1. Atomic Gossip Sync (The Heartbeat)

Early on, we decided to move away from simple API polling. Instead, we built a **Transaction-Based Sync** mechanism.

Every 5 minutes, a GitHub Action wakes up and triggers our sync endpoint. But we don't just write data as it comes in. We use a strictly ACID approach:

1. Fetch the latest gossip from RPC nodes.
2. Open a database transaction.
3. **Mark everyone as inactive** (innocent until proven guilty, or rather, offline until proven online).
4. Upsert the active nodes from the new snapshot.
5. Commit everything at once.

This is crucial because it prevents the "half-updated" state. When you load the dashboard, you are never seeing a partial sync. You see the network exactly as it was at the last heartbeat.

### 2. Caching for Speed

Direct database queries are great, but they aren't fast enough for the user experience we wanted. We implemented a custom **In-Memory Caching Layer**.

We use a "write-through" strategy. Immediately after the cron job commits data to Postgres, the service layer proactively refreshes the global cache. When you visit the site, you aren't hitting the database; you're hitting RAM. This keeps our response times in the milliseconds, even under heavy load.

*Dev Note: For the charts, we use dynamic bucketing. If you look at the last hour, you see high-res 30-second intervals. If you look at the last month, we switch to daily aggregates to keep the payload light.*

### 3. The "Priority Scorer"

Here’s the thing about "performance": it’s subjective.

Some users care about **Credits** (economic stake). Others care about raw **Uptime** or **Storage** capacity. Instead of forcing our opinion on the user, we built a **Client-Side Priority Scorer**.

You can use a slider to define your own weights (e.g., *I care 70% about Uptime and 30% about Storage*). The app normalizes these values into a 0.00 - 1.00 score for every validator and color-codes them (Green/Yellow/Gray) in real-time. We even save these preferences to your browser’s `localStorage`, so your view remains personalized every time you come back.

## 🔗 Talking to the Network

Interacting with the Xandeum RPC layer was a fun challenge in normalization. We pull data primarily from `get-pods-with-stats` for discovery and `get-credits` to map public keys to stake.

One small but annoying detail we solved: IP normalization. The RPC returns IPs in various formats, so we standardized everything by stripping ports for storage keys (keeping it clean as `X.X.X.X`) and appending them dynamically only when we need to make a transport call.

## 🔮 What's Next?

This dashboard is just the foundation. We’re already looking at adding predictive uptime modeling (using that historical data we're collecting) and more detailed geographic distribution maps.

For now, though, we’re just happy to see the lights blinking green.

* Draft a README.md file for this project based on this architecture?
* Generate the Prisma Schema code that would support this specific data model?
* Write a Twitter/X thread summarizing this post to share with the community?
