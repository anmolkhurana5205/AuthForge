# ⚡ Server Actions — Deep Dive

> *A complete explanation of what Server Actions are, why they exist, and how this project uses them — with real code walkthroughs.*

---

## 1. The Problem They Solve

Before Server Actions, the classic full-stack flow for a form submission looked like this:

```
[Client Component]          [API Route]              [Database]
     |                           |                       |
     |-- fetch POST /api/login ->|                       |
     |   { body: JSON.stringify  |                       |
     |     (email, password) }   |                       |
     |                           |-- await db.query() -->|
     |                           |<-- user row ----------|
     |<-- { status: 200,         |                       |
     |      json: { success } }  |                       |
     |                           |                       |
  parse response,                                        
  handle errors,                                         
  update UI                                              
```

You had to manually:
- Write a `fetch()` call with method, headers, body
- Build a `NextResponse` on the server with status codes
- Parse `req.json()` on the server side
- Handle errors in two places (network + business logic)
- Duplicate your TypeScript types across client and server

**Server Actions collapse all of this into a single function call.**

---

## 2. What is a Server Action?

A **Server Action** is an `async` function marked with the `"use server"` directive that:

- **Runs exclusively on the server** — never in the browser, no matter where it is called from
- **Can be called from a Client Component** as if it were a normal local function
- **Has full access to** the database, environment secrets, Node.js APIs, and file system
- **Returns plain JavaScript objects** — no `Response`, no status codes, no JSON serialization needed

Under the hood, Next.js creates a hidden POST endpoint for each Server Action and handles the serialization/deserialization automatically. You never see this HTTP request — it is completely transparent.

### The `"use server"` Directive

```ts
"use server";   // ← This single line transforms this file

export const myAction = async (data: SomeType) => {
  // This code ONLY ever runs on the server
  // You can safely use: DB, secrets, bcrypt, file system, etc.
  return { success: "Done!" };
};
```

> **Rule:** Every file in `actions/` starts with `"use server"`. Any function exported from such a file becomes a Server Action.

---

## 3. How Calling a Server Action Works (Mental Model)

```
[LoginForm.tsx — Client Component]
  ↓  imports and calls:
  const result = await login(formValues);
        ↑
        └── Looks like a regular function call
            But Next.js intercepts it,
            sends a POST request to the server,
            runs login() there,
            and returns the result back to the client

[actions/login.ts — Server]
  ↓  receives the call
  ↓  runs all server-side logic
  ↓  returns { success: "..." } or { error: "..." }
        ↑
        └── A plain JS object, not a Response or JSON string
```

The client gets back a typed object — same TypeScript types, no casting, no `response.json()`.

---

## 4. Server Actions in This Project

This project has **7 Server Actions**, one per authentication concern, all living in the `actions/` folder:

```
actions/
├── login.ts            ← Credential login + 2FA handling
├── register.ts         ← User registration + email verification trigger
├── reset.ts            ← Password reset request (sends email)
├── new-password.ts     ← Sets a new password using a reset token
├── new-verification.ts ← Confirms email ownership via token link
├── settings.ts         ← Updates authenticated user's profile settings
└── admin.ts            ← Admin-only action (role-based guard demo)
```

> **Design Decision:** This project deliberately avoids API routes for form submissions. The only API route (`/api/admin/route.ts`) exists purely as a demo comparison. Everything else uses Server Actions.

---

## 5. Anatomy of a Server Action — `register.ts`

The simplest action to understand is registration:

```ts
"use server";

import { RegisterSchema } from "@/schemas";        // Zod schema
import { db } from "@/lib/db";                     // Prisma client
import bcrypt from "bcryptjs";                     // Password hashing
import { getUserByEmail } from "@/data/user";      // DB query helper
import { generateVerificationToken } from "@/lib/tokens";
import { sendVerificationEmail } from "@/lib/mail";

export const register = async (values: z.infer<typeof RegisterSchema>) => {

  // STEP 1: Validate input with Zod
  const validatedFields = RegisterSchema.safeParse(values);
  if (!validatedFields.success) {
    return { error: "Invalid fields!" };
  }

  const { email, password, name } = validatedFields.data;

  // STEP 2: Check if user already exists
  const existingUser = await getUserByEmail(email);
  if (existingUser) {
    return { error: "Email already in use!" };
  }

  // STEP 3: Hash the password (ONLY safe on the server)
  const hashedPassword = await bcrypt.hash(password, 10);

  // STEP 4: Create user in PostgreSQL via Prisma
  await db.user.create({ data: { name, email, password: hashedPassword } });

  // STEP 5: Generate a verification token and store it in DB
  const verificationToken = await generateVerificationToken(email);

  // STEP 6: Send the verification email via Resend API
  await sendVerificationEmail(verificationToken.email, verificationToken.token);

  return { success: "Confirmation Email Sent!" };
};
```

### Why each step MUST be on the server:
| Step | Why it can't be on the client |
|---|---|
| Zod validation | Can be duplicated client-side for UX, but server is the source of truth |
| `getUserByEmail()` | Direct DB query — browser has no DB access |
| `bcrypt.hash()` | Hashing is CPU-intensive and the salt rounds must be server-controlled |
| `db.user.create()` | Direct Prisma/DB write — browser cannot touch the DB |
| `generateVerificationToken()` | Writes token to DB — must be server-side |
| `sendVerificationEmail()` | Uses `RESEND_API_KEY` from `.env` — secrets cannot go to the browser |

---

## 6. The Complex Action — `login.ts` (Full Flow)

`login.ts` is the most sophisticated action in the project. It handles 4 completely different scenarios inside a single function:

```
User submits { email, password, code? }
        ↓
┌─────────────────────────────────────────────────────────┐
│  STEP 1: Zod Validation                                 │
│  LoginSchema.safeParse(values)                          │
│  → FAIL: return { error: "Invalid fields!" }            │
└────────────────────────┬────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  STEP 2: User Lookup                                    │
│  getUserByEmail(email)                                  │
│  → NOT FOUND: return { error: "Email doesn't exist" }  │
└────────────────────────┬────────────────────────────────┘
                         ↓
        ┌────────────────┴────────────────┐
        ↓                                ↓
  Email NOT verified            Email IS verified
        ↓                                ↓
  generateVerificationToken()    2FA enabled?
  sendVerificationEmail()         ↓          ↓
  return { success:             YES          NO
    "Confirmation email sent!" }  ↓           ↓
  [STOP — don't log in]      Code         Go straight to
                            submitted?    signIn()
                           ↓       ↓
                          YES      NO
                           ↓       ↓
                     Validate   generateTwoFactorToken()
                     token      send2FAEmail()
                       ↓        return { twoFactor: true }
                     Delete     [STOP — UI shows code input]
                     token
                       ↓
                     Create TwoFactorConfirmation
                     in DB (Auth.js will check this)
                       ↓
                     signIn("credentials", ...)
                       ↓
                  Auth.js signIn callback runs
                  (checks emailVerified +
                   TwoFactorConfirmation,
                   then deletes confirmation)
                       ↓
                  return { success: "Logged in!" }
```

### The 2FA Handshake (Key Pattern)

The 2FA flow uses a clever two-step handshake between the Server Action and Auth.js:

1. **Server Action** validates the 6-digit code, then **creates a `TwoFactorConfirmation` record** in the DB as a "voucher"
2. **Server Action** then calls `signIn()` which triggers Auth.js's `signIn` callback
3. **Auth.js `signIn` callback** checks for that `TwoFactorConfirmation` record — if it exists, the login is allowed
4. **Auth.js** immediately **deletes the confirmation** so it is one-time use only

This is why there is no `return` statement after the 2FA confirmation is created — the code intentionally falls through to `signIn()`.

---

## 7. The Auth-Protected Action — `settings.ts`

Actions on protected pages also verify the current session on the server:

```ts
"use server";

export const settings = async (values: z.infer<typeof SettingsSchema>) => {

  // STEP 1: Get the currently logged-in user from the JWT session
  const user = await currentUser();
  if (!user) return { error: "Unauthorized" };

  // STEP 2: Verify they still exist in the DB
  const dbUser = await getUserById(user.id!);
  if (!dbUser) return { error: "Unauthorized!" };

  // STEP 3: Validate input
  const validatedFields = SettingsSchema.safeParse(values);
  if (!validatedFields.success) return { error: "Invalid fields" };

  // STEP 4: Update the user in the DB
  await db.user.update({
    where: { id: dbUser.id },
    data: { ...validatedFields.data },
  });

  return { success: "Settings Updated" };
};
```

> **Important:** Even though the settings page is already protected by middleware, the Server Action checks authentication **again**. You cannot trust that only authenticated users will call a Server Action — someone could call the endpoint directly. Always verify on the server.

This is the principle of **"never trust the client"** applied to Server Actions.

---

## 8. Server Actions vs. API Routes — Side by Side

This project has both, making the comparison concrete:

### API Route — `app/api/admin/route.ts`
```ts
import { currentRole } from "@/lib/auth";
import { UserRole } from "@prisma/client";
import { NextResponse } from "next/server";

export async function GET() {
  const role = await currentRole();
  if (role === UserRole.ADMIN) {
    return new NextResponse(null, { status: 200 });
  }
  return new NextResponse(null, { status: 403 });
}
```

**To call this from the client:**
```ts
const response = await fetch("/api/admin");
if (response.ok) {
  // handle success
} else if (response.status === 403) {
  // handle forbidden
}
```

---

### Server Action — `actions/admin.ts`
```ts
"use server";

export const adminAction = async () => {
  const role = await currentRole();
  if (role === UserRole.ADMIN) return { success: "Allowed" };
  return { error: "Forbidden!" };
};
```

**To call this from the client:**
```ts
const result = await adminAction();
if (result.success) { /* ... */ }
if (result.error) { /* ... */ }
```

---

### Comparison Table

| Concern | Server Action | API Route |
|---|---|---|
| **How you call it** | `login(formData)` — direct function call | `fetch('/api/login', { method: 'POST', body: ... })` |
| **Request building** | None — Next.js handles it | Manual: headers, method, JSON.stringify |
| **Response parsing** | None — returns plain JS object | Manual: `await response.json()` |
| **Type safety** | ✅ End-to-end — same TypeScript types | ❌ Manual — types don't cross the boundary |
| **Error handling** | Return `{ error: "..." }` | Return `NextResponse` with status codes |
| **Boilerplate** | Minimal | More verbose |
| **When to use** | Form submissions, mutations | Public APIs, webhooks, GET requests, integrations |

---

## 9. The Security Rule — Double Validation

Every Server Action in this project follows the same security pattern:

```
1. Validate input shape      → Zod schema
2. Verify authentication     → currentUser() / currentRole()
3. Verify DB state           → getUserById() etc.
4. Perform the operation     → db.create() / db.update()
5. Return typed result       → { success } or { error }
```

This matters because **Server Actions are still HTTP endpoints** under the hood. Middleware only protects page navigation — it does not prevent someone from directly calling your Server Action's POST endpoint. Always validate and authenticate inside the action itself.

---

## 10. Summary

> **Server Actions = Server-side logic with client-side ergonomics.**

| What | Detail |
|---|---|
| **Directive** | `"use server"` at the top of the file |
| **Lives in** | `actions/` folder |
| **Runs on** | Server only — always |
| **Called from** | Client Components, Server Components, other Server Actions |
| **Returns** | Plain JS objects — `{ success }` or `{ error }` |
| **Has access to** | DB, secrets, bcrypt, emails, Node.js APIs |
| **Does NOT need** | `fetch`, `req`, `res`, `NextResponse`, status codes |
| **Used for** | All 7 auth flows in this project |
| **Security** | Validate + authenticate inside every action — never trust the client |
