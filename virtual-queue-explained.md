# Virtual Queue Platform: The Idea and the Technology

## 1. The Core Idea

Waiting in line is a problem every organization has, but each one has it differently. A bakery needs a single line. A government bureau needs separate lines for passports, ID cards, and payments. A clinic needs to know why you came. A bank has several tellers serving the same line.

This project is a **universal virtual queue platform**: one system that adapts to any of these, so customers can take a digital ticket on their own phone instead of standing in a physical line.

The platform is built on three ideas:

**Tabs.** A Tab is an independent virtual waiting line. An organization creates as many Tabs as its subscription allows:

- **Free / Starter:** 1 Tab (a small coffee shop)
- **Pro:** up to 5 Tabs (a bureau with Passports, ID Cards, Payments, General)
- **Enterprise:** unlimited Tabs across multiple service desks

**No app, no account.** A tablet or TV at the entrance shows a QR code. The customer scans it with their phone camera and gets a live ticket in their browser. Nothing to download and nothing to sign up for.

**Real-time everything.** Nobody refreshes a page. When staff call the next person, every screen involved (the kiosk, the called customer, and everyone still waiting) updates instantly.

---

## 2. The Three Screens

The whole thing is one React web application, but it shows three different experiences depending on who opens the URL.

### The Kiosk Screen (entrance display)
Left open full-screen on a tablet or TV. It shows a large QR code and a "Now Serving" board listing which ticket numbers are being called to which counters.

The QR code is **dynamic**. It regenerates every 30 seconds, and also immediately after someone scans it. This is the anti-cheating mechanism: someone can't photograph the code and join the line from their couch the next day, because the token inside it has already expired.

### The Customer Ticket Screen (mobile browser)
When a customer scans the QR code, their phone opens the site and checks how the organization configured that Tab:

- **Mode A, Instant Ticket:** no extra info needed (like a fast-food pickup counter). The ticket is created immediately, with zero clicks.
- **Mode B, Guided Intake Form:** the organization defined custom questions ("Which office are you visiting?", "What is your reason for visit?"). The customer fills out a short form, and the system routes the ticket to the matching Tab and assigns the right prefix (for example, P-014 for Passports).

After that, the customer sees their ticket number, how many people are ahead, and an estimated wait time. When they're one or two spots from being called, the phone plays a chime, flashes the screen, and sends a browser notification. The ticket ID is saved in the browser's localStorage, so if they accidentally close Safari or Chrome, reopening the link restores their exact place in line.

### The Staff and Admin Dashboard (laptop or counter)
- **Admin:** manages the subscription, creates or renames Tabs, turns custom intake questions on or off per Tab, and views analytics (busiest hours, average wait times, no-show rates).
- **Staff:** opens their assigned Tab(s) and sees the live list of waiting people, plus any notes the customer entered (like "Reason: Renewal"). They can click **Call Next**, flag someone as urgent/VIP to jump the queue, or mark someone as a **No-Show** if they don't walk up within a few minutes.

---

## 3. The Technologies and Why Each One Is Here

The backend is a Node.js server acting as the traffic controller between the screens and two specialized databases. Each technology solves a specific problem that the others can't solve well.

### React (the frontend)
React builds the user interface. Its component model suits this project well: the same app can render a kiosk board, a customer ticket, and a staff dashboard by reusing building blocks and switching which screen to show. React's state updates also pair naturally with real-time data. When new data arrives, the screen redraws itself.

### Node.js + Express (the REST API)
Express handles **request and response** interactions, where the browser asks for something once and waits for the answer. Examples:

- Logging in to an account
- Creating a new Tab
- Submitting a customer intake form
- Loading weekly analytics charts

Node.js is a good fit because it handles many simultaneous connections efficiently, and a queue system has many phones, kiosks, and dashboards connected at once.

### Socket.io (the real-time walkie-talkie)
Normal websites only update when the user hits Refresh, because HTTP is one-shot: ask, answer, done. That's useless for a live queue.

**Socket.io** keeps a permanent, two-way connection open between each browser and the server. Either side can send a message at any time. The server can *push* updates to a phone without the phone asking.

**Rooms** are the key feature. Socket.io lets you group connected browsers into isolated rooms, for example `org_5_tab_2` (organization 5, Tab 2). When a staff member clicks Call Next in Tab 2, the server broadcasts only to that room, and in one instant:

- The kiosk updates its Now Serving board
- The next customer's phone vibrates and turns green
- Everyone else in line sees their position drop by one

Customers in other Tabs or other organizations never hear about it. Rooms are what keep a multi-tenant system both efficient and private.

### Redis (ultra-fast temporary memory)
Redis is a database that lives **entirely in the server's RAM** instead of on a hard drive. RAM is extremely fast (responses in well under a millisecond), so Redis handles everything that changes quickly or is meant to disappear. The trade-off is that RAM is not designed for permanent storage, so Redis is used for temporary data only. It does three important jobs here:

**1. Expiring QR tokens (TTL).** When the kiosk generates a QR code, Node.js saves the secret token in Redis with a 30-second **Time To Live**. After 30 seconds, Redis deletes it automatically, with no cleanup code needed. When a customer scans a valid token, Redis reserves it for that phone for 2 minutes, which gives them time to fill out a form without being kicked out. Meanwhile the kiosk immediately shows a brand-new QR code for the next person.

**2. The live waiting lines.** Each Tab's queue is a list in Redis, kept in strict first-in, first-out order. Adding a ticket to the back and taking one from the front are both instant operations.

**3. Preventing double-calls (atomic operations).** Imagine a bank with three tellers on the same Tab, and two of them click Call Next at the exact same millisecond. Without protection, both could be handed the same customer. Redis processes commands strictly **one at a time**, so each operation completes fully before the next begins. Teller 1 gets A-005 and Teller 2 gets A-006, never the same person twice. This property is called being *atomic*, and it's one of the most important reasons Redis is used here instead of a regular database table.

### PostgreSQL (the permanent filing cabinet)
PostgreSQL writes data to disk, so nothing is lost if the server restarts. It stores everything that needs to last:

- **Accounts and subscriptions:** business logins, plans, and how many Tabs each organization is allowed
- **Tab rules and custom questions:** instant ticket or intake form, the dropdown options, and the ticket prefix (A-, B-, VIP-)
- **Tickets with flexible data (JSONB):** a bakery needs no extra fields while a clinic needs Symptoms and Doctor Name. A JSONB column stores each organization's custom answers in whatever shape they need, so one table serves every industry without structural changes.
- **Analytics and wait-time history:** every time a ticket is completed or marked no-show, its timestamps (joined, called, finished) are saved. The wait-time estimator uses recent completion speeds from this history, divided by the number of active staff counters, to give new customers realistic estimates.

### Why two databases?

The split is a deliberate trade-off between **speed** and **safety**:

| | Redis | PostgreSQL |
|---|---|---|
| Storage | RAM | Disk |
| Speed | Sub-millisecond | Slower, but fast enough |
| Durability | Temporary | Permanent |
| Best for | Live queues, expiring tokens, atomic operations | Accounts, rules, tickets, history, analytics |

Redis makes the live experience instant. PostgreSQL makes sure the business data is never lost. Neither would do both jobs well alone.

---

## 4. A Single Customer Visit, in Words

1. **Setup.** A bureau subscribes to the Pro plan, creates three Tabs (General, Passports, Payments), and adds a dropdown asking which service the customer needs. They open the Kiosk URL on an entrance tablet.
2. **QR generation.** The kiosk asks Node.js for a QR code. Node.js creates a random token, saves it in Redis for 30 seconds, and sends it to the kiosk over Socket.io.
3. **The scan.** A customer scans the code. Node.js checks the token in Redis, locks it to that customer's phone, and immediately pushes a fresh QR code to the kiosk so the next person can scan right away.
4. **Ticket creation.** The customer picks "Passports" and submits. Node.js saves the ticket in PostgreSQL (including the custom answer as JSONB), adds the ticket ID to the Passports waiting list in Redis, and joins the customer's phone to the Passports Socket.io room.
5. **Live waiting.** As staff serve earlier customers, Socket.io updates the customer's screen in real time: "2 people ahead of you. Est. wait: 6 mins."
6. **Turn called.** A staff member clicks Call Next. Redis removes the ticket from the front of the line, Socket.io fires an alert on the customer's phone and updates the kiosk, and PostgreSQL records the final timestamps for analytics.

---

## 5. Why This Project Stands Out

Most student or portfolio projects are CRUD apps: basic HTTP requests and a single database. This one combines React, Node.js, Socket.io, Redis, and PostgreSQL to demonstrate real systems engineering:

- **Concurrency:** two staff members clicking at once without conflict
- **Ephemeral security:** self-destructing QR tokens
- **Pub/sub room broadcasting:** targeted real-time updates
- **Hybrid data storage:** the right database for each kind of data
- **Multi-tenant design:** one system serving many different industries

---

## 6. Where to Start

Build the **"Ugly Walking Skeleton"** first. Ignore subscriptions, analytics, and custom forms for week one. Get a single hardcoded Tab working where:

1. Scanning a QR code puts a ticket into Redis
2. Clicking "Next" removes it from Redis
3. Socket.io updates both screens live

Once that heartbeat works, layering on PostgreSQL accounts, Tab limits, custom JSONB forms, and analytics becomes straightforward, because the hardest architectural piece is already proven.
