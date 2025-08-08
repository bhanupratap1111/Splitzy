
# Splitzy – Product & Technical Overview

Splitzy is an intuitive platform that allows users to track shared expenses, split bills with precision, and settle up balances seamlessly. Whether it's a one-on-one dinner or a group vacation, Splitzy keeps a clear record of who owes whom, enhanced with automated reminders and personalized spending insights powered by AI.

## 1) Features

- Track 1:1 and group expenses  
- Split bills equally, by percentage, or exact amounts  
- Clerk authentication and Convex realtime backend
- Dashboard analytics with monthly charts
- Receive automated payment reminders and monthly spending insights  

Public landing at `/` with auth‑gated app (`dashboard`, `contacts`, `expenses`, `groups`, `settlements`).

---

## 2) Tech Stack

- **Frontend:** Next.js 15 (App Router), React 19, Tailwind (via shadcn/ui components), Recharts  
- **Auth:** Clerk  
- **Backend & DB:** Convex (queries, mutations, actions)  
- **Jobs/Workflows:** Inngest (cron scheduled)  
- **Email:** Resend (via Convex action)  
- **AI:** Google Generative AI (Gemini)  
- **Tooling:** Zod, React Hook Form, Sonner (toasts)  

### Key Env Vars:

```env
CONVEX_DEPLOY_KEY
CONVEX_DEPLOYMENT
NEXT_PUBLIC_CONVEX_URL
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY
CLERK_SECRET_KEY
NEXT_PUBLIC_CLERK_SIGN_IN_URL
NEXT_PUBLIC_CLERK_SIGN_UP_URL
CLERK_JWT_ISSUER_DOMAIN
RESEND_API_KEY
GEMINI_API_KEY
```

---

## 3) App Structure

- **Root layout and providers:** `app/layout.js` (Clerk + Convex providers, global Toaster)  
- **Public landing:** `app/page.jsx` (marketing sections powered by `lib/landing.js`)  
- **Auth pages:** `app/(auth)/sign-in/...`, `app/(auth)/sign-up/...`  
- **Protected app:** wrapped by `app/(main)/layout.jsx` and `middleware.js`  

### Pages:

- Dashboard: `app/(main)/dashboard/page.jsx`  
- Contacts (people & groups): `app/(main)/contacts/page.jsx` (+ `create-group-modal`)  
- New Expense: `app/(main)/expenses/new/page.jsx` (+ form components)  
- Group detail: `app/(main)/groups/[id]/page.jsx`  
- Person detail: `app/(main)/person/[id]/page.jsx`  
- Settlements: `app/(main)/settlements/[type]/[id]/page.jsx`  
- Header and navigation: `components/header.jsx` (Clerk auth state, deep links)  

---

## 4) Data Model (Convex `convex/schema.js`)

### users

- Fields: `name`, `email`, `tokenIdentifier`, `imageUrl`  
- Indexes: `by_token`, `by_email`, search indexes for name and email  

### expenses

- Fields: `description`, `amount`, `category?`, `date`, `paidByUserId`, `splitType`, `splits[]`, `groupId?`, `createdBy`  
- Indexes: `by_group`, `by_user_and_group`, `by_date`  

### settlements

- Fields: `amount`, `note?`, `date`, `paidByUserId`, `receivedByUserId`, `groupId?`, `relatedExpenseIds?`, `createdBy`  
- Indexes: `by_group`, `by_user_and_group`, `by_receiver_and_group`, `by_date`  

### groups

- Fields: `name`, `description?`, `createdBy`, `members[] { userId, role, joinedAt }`  

---

## 5) Backend Modules (Convex)

### users.js

- `store`: persists Clerk identity → users table  
- `getCurrentUser`: lookup via `by_token`  
- `searchUsers`: combines name and email search indexes, excludes current user  

### contacts.js

- `getAllContacts`: deduces people from 1:1 expenses; lists user’s groups  
- `createGroup`: validates members and inserts group with roles  

### groups.js

- `getGroupOrMembers`: fetches user’s groups and optionally members for a selected group  
- `getGroupExpenses`: returns group, members, expenses, settlements, computed ledgers/balances, `userLookupMap`  

### expenses.js

- `createExpense`: validates splits sum; group membership; inserts expense  
- `getExpensesBetweenUsers`: 1:1 expense and settlement history, with running balance  
- `deleteExpense`: permission check; cleans up `relatedExpenseIds` in settlements  

### settlements.js

- `createSettlement`: validation (`amount>0`, payer ≠ receiver, membership), inserts settlement  
- `getSettlementData`: builds UI payloads for user or group settlement flows  

### dashboard.js

- `getUserBalances`: tallies 1:1 net positions and grouped lists (you owe / you are owed)  
- `getTotalSpent`: year‑to‑date personal share  
- `getMonthlySpending`: monthly totals for current year (for charts)  

### email.js (Convex action)

- `sendEmail`: uses Resend with `from: "Splitzy <onboarding@resend.dev>"`, accepts subject/html/text/apiKey  

### inngest.js (Convex queries for jobs)

- `getUsersWithOutstandingDebts`: computes per‑counterparty positive net debts for 1:1  
- `getUsersWithExpenses`, `getUserMonthlyExpenses`: sources for spending‑insights job  

### seed.js

- Developer utility to seed users/groups/expenses/settlements  

---

## 6) Jobs & Integrations

### Inngest API route

- Bound in `app/api/inngest/route.js`  
- Client: `lib/inngest/client.js` (`id: "splitzy"`, `name: "Splitzy"`)  

### Payment Reminders

- `lib/inngest/payment-reminders.js`  
- **Cron:** `0 10 * * *` (daily at 10:00 UTC)  
- **Flow:** Fetch users with positive 1:1 debts → build HTML table → call `api.email.sendEmail` via Convex action using `RESEND_API_KEY`  

### Spending Insights

- `lib/inngest/spending-insights.js`  
- **Cron:** `0 8 1 * *` (8:00 UTC on the 1st each month)  
- **Flow:** For each user, gather last‑month expenses → prompt Gemini → email results via Convex action  

---

## 7) Frontend Flows

### Expense Creation

- File: `app/(main)/expenses/new/components/expense-form.jsx`  
- Validates with Zod  
- Supports equal, percentage, exact splits  
- Ensures `sum(splits) ≈ amount` (tolerance 0.01)  
- On success, routes to counterpart or group  

### Splitting UI

- Components: `split-selector.jsx`, `participant-selector.jsx`, `group-selector.jsx`, `category-selector.jsx`  

### Dashboard

- Aggregates totals and charts from Convex queries  
- Lists groups and navigation actions  

### Person & Group Pages

- **Person:** 1:1 ledger, settlements, quick "Settle up"  
- **Group:** member list, ledger, expenses, settlements tabs  

---

## 8) Auth & Security

- Clerk enforced via `middleware.js` route matcher on protected routes:
  `/dashboard`, `/expenses`, `/contacts`, `/groups`, `/person`, `/settlements`  
- `useStoreUser` hook stores Clerk identity in Convex on first load  
- Server‑side access always via `internal.users.getCurrentUser` for identity and authorization checks  

---

## 9) Email & AI

- **Email:** Resend via Convex action `api.email.sendEmail`  
- **AI:** Gemini renders monthly spending insights email content  

---

## 10) Developer Experience

- Scripts: `npm run dev`, `npm run build`, `npm run start`, `npm run lint`  
- Seeding: `npx convex run seed:seedDatabase`  
- Component library: `shadcn/ui` (used in `components/ui/*`)  
- Hooks: typed wrappers for Convex (`hooks/use-convex-query.jsx`, `hooks/use-store-user.jsx`)  

---

## 11) Operational Considerations

- Ensure env vars set in deployment for Convex, Clerk, Resend, Gemini  
- Cron jobs run in UTC  
- Email sender/domain verification in Resend  
- Observability: Email action logs to console; Inngest step results include per-user outcomes  
