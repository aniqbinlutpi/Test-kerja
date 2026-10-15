
## Section A: Theory and Architecture


<img width="1264" height="216" alt="image" src="https://github.com/user-attachments/assets/b6557889-e4ad-44df-9ee9-a40fb00d1a46" />

### Question 1: Next.js Architecture

<img width="1270" height="82" alt="image" src="https://github.com/user-attachments/assets/f26bbf60-6429-4dcb-8b7d-d91a15b8d8ca" />

**Server Component**
- runs only on the server side. it's never send javascript to the browser (users cant know), like logical/flow of the system.
- programmer fetching data, query db, logic, cant be exposed.

**Client Component ('use client')** 
- it's javascript is downloaded and executed inside the user's browser.
- andles UI interactions, user clicks, and client-side data (useState/useQuery)

<img width="418" height="60" alt="image" src="https://github.com/user-attachments/assets/733737e5-26ba-428f-ae9a-a9006ea99f8d" />

**Use Server Components**
- fetching data directly from database or backend API (no need create extra API routes).
- static pages or layout that dont need user interaction (header, footer,text).
- heavy packages/libraries, so user browser doesnt have to download big files.

**Use Client Components**
- any user interaction and clicks (buttons, form submit, typing inputs).
- managing UI state and live data (useState / fetching using TanStack Query).
- browser-only features (accessing localStorage, window screen size, cookies).
- popup modals, dropdown menu or anything that users open and close on the screen.

<img width="1092" height="72" alt="image" src="https://github.com/user-attachments/assets/1e680b34-22b5-4e05-8f78-faa3f4a99524" />

- Client component will download all its javascript code straight into user's browser.
- any user can just press F12 or open Inspect Element/Console (in network tabs) and easily see your secret keys (like Stripe keys, OpenAI API key or database password).
- if hackers find these secrets, they can easily steal user data or drop/delete your whole database.
that why anything secret and database queries must always stay on server side / backend only.

### Question 2: Database Design

<img width="1330" height="154" alt="image" src="https://github.com/user-attachments/assets/00189e7e-60df-4ad1-8fad-f070152013a4" />

#### 1. SQL Schema (PostgreSQL)

```sql
-- Customers table
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Invoices table (one customer can have many invoices)
CREATE TABLE invoices (
    id SERIAL PRIMARY KEY,
    customer_id INT NOT NULL,
    amount NUMERIC(10, 2) NOT NULL,
    status VARCHAR(50) DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    -- Foreign key linking to customers table
    CONSTRAINT fk_customer 
        FOREIGN KEY (customer_id) 
        REFERENCES customers(id) 
        ON DELETE RESTRICT
);
```
<img width="1356" height="104" alt="image" src="https://github.com/user-attachments/assets/4a8acda5-042d-49f4-8b7f-2d9971a97121" />

- i will use ON DELETE RESTRICT, never CASCADE for financial records.
- invoices are money records and audit trails. if we use CASCADE deleting a customer will accidentally delete all their past invoices and payment history.
- if invoices get deleted, company will have accounting and tax audit issues because we lose the proof.
- with RESTRICT, the database will block anyone from deleting the customer if they still have invoice history.
- if we really need to delete or disable the customer, it's better to use soft delete 
- example: set is_active = false or deleted_at = now()) so the invoice history stays safe.

---

### Question 3: Payment Webhooks

#### 1. Handling Duplicate Webhook Events (Idempotency)

> **ELI5:**
> Imagine a mail carrier delivers the exact same parcel twice because they were worried you didn't receive it. Before accepting the second parcel, you check your receipt list: *"I already signed and received this package 5 minutes ago!"* and politely decline the duplicate.

To ensure the invoice is updated only once and the customer receives only one email:
1. **Check Status Before Updating (State Guard):**
   * When the webhook arrives, check the current invoice status in the database.
   * If the status is already marked as `PAID`, immediately return a `200 OK` response to the gateway and stop execution (skip updating and skip sending the email).
2. **Event Idempotency Table:**
   * Store the unique webhook event ID (e.g., `evt_123` from Stripe) in a `processed_webhook_events` table with a unique constraint.
   * If an incoming event ID already exists in this table, safely discard it.
3. **Database Transactions:**
   * Wrap the status update and the event logging inside a database transaction (`BEGIN ... COMMIT`) to prevent race conditions if two webhooks hit at the exact same millisecond.

#### 2. Verifying Request Authenticity (Preventing Imposters)

> **ELI5:**
> Like an official wax seal on an envelope with a royal stamp that only the real king has the signet ring to produce.

* **Webhook Signature Verification:**
  1. The payment gateway shares a secret key (known only to the gateway and your backend server, stored securely in `.env`).
  2. With every webhook POST request, the gateway creates a cryptographic hash (HMAC-SHA256) of the raw request payload using that secret key, and sends it in the HTTP headers (e.g., `Stripe-Signature` or `X-Signature`).
  3. Your server reads the raw body, generates the HMAC signature using your secret key, and compares the two signatures (`crypto.timingSafeEqual`).
  4. If they match, the request genuinely came from the gateway. If they don't, it is an attacker pretending to be the gateway, and your server responds with `400 Bad Request` or `401 Unauthorized`.

---

## Section B: Code Review

### Original Code:

```typescript
// app/api/invoice/update/route.ts
import { NextResponse } from "next/server";
import { db } from "@/lib/db"; // hypothetical db client

export async function POST(req: Request) {
  try {
    const body = await req.json();
    const { invoiceId, newStatus, userId } = body;
    // Update invoice
    const updatedInvoice = await db.query(
      `UPDATE invoices SET status = '${newStatus}' WHERE id = ${invoiceId}`
    );
    return NextResponse.json({ success: true, data: updatedInvoice });
  } catch (error) {
    return NextResponse.json({ success: false, error: error.message }, { status: 500 });
  }
}
```

---

### Identified Issues:

| # | Issue Category | Problem & What Could Go Wrong | How to Fix It |
|---|---|---|---|
| **1** | **Security (Critical)** | **SQL Injection Vulnerability**<br>Values (`${newStatus}` and `${invoiceId}`) are concatenated directly into the raw SQL string. An attacker can inject arbitrary SQL commands (e.g., `' OR '1'='1` or `; DROP TABLE invoices; --`) to bypass checks or wipe the database. | Use **Parameterized Queries / Prepared Statements** (e.g., `$1, $2`) so the database engine treats input strictly as data, never as executable code. |
| **2** | **Security & Logic** | **Missing Authentication & Authorization (IDOR / Broken Access Control)**<br>The route trusts `userId` from the request body without verifying user session credentials, and doesn't even verify if that user owns the invoice. Anyone can change any customer's invoice status to `PAID`. | Authenticate the user from a secure server-side session/token. Ensure the update query checks ownership (e.g., `WHERE id = $1 AND customer_id = $2`) or requires admin privileges. |
| **3** | **Data Integrity** | **Lack of Input Validation**<br>No validation on `newStatus` or `invoiceId`. A user could supply an invalid status (e.g., `"BANANA"` or an empty string) or invalid data types (`invoiceId = "abc"`), causing database errors or corrupting state. | Validate inputs against an allowed whitelist/enum (e.g., `['DRAFT', 'PENDING', 'PAID', 'CANCELLED']`) and verify that `invoiceId` is a positive integer using a validator like **Zod**. |
| **4** | **Security / Ops** | **Information Disclosure in Error Handling**<br>Returning `error.message` directly in HTTP 500 responses exposes internal database schemas and server stack traces to clients. | Log the detailed error internally (`console.error`) and return a generic, safe error message to the client. |

---

### Refactored & Fixed Route:

```typescript
// app/api/invoice/update/route.ts
import { NextResponse } from "next/server";
import { z } from "zod";
import { db } from "@/lib/db";
import { getAuthUser } from "@/lib/auth"; // Your server-side session helper

// 1. Strict input validation schema
const updateInvoiceSchema = z.object({
  invoiceId: z.coerce.number().int().positive({
    message: "invoiceId must be a positive integer",
  }),
  newStatus: z.enum(["DRAFT", "PENDING", "PAID", "CANCELLED"], {
    errorMap: () => ({ message: "Invalid invoice status" }),
  }),
});

export async function POST(req: Request) {
  try {
    // 2. Authentication: Verify caller's identity via server-side session
    const currentUser = await getAuthUser(req);
    if (!currentUser) {
      return NextResponse.json(
        { success: false, error: "Unauthorized. Please log in." },
        { status: 401 }
      );
    }

    // 3. Parse and validate body
    const json = await req.json().catch(() => null);
    const parsed = updateInvoiceSchema.safeParse(json);

    if (!parsed.success) {
      return NextResponse.json(
        {
          success: false,
          error: "Validation failed",
          details: parsed.error.flatten().fieldErrors,
        },
        { status: 400 }
      );
    }

    const { invoiceId, newStatus } = parsed.data;

    // 4. Parameterized Query (Prevents SQL Injection) + Authorization check
    // Ensures regular users can only update their own invoices
    const result = await db.query(
      `UPDATE invoices 
       SET status = $1, updated_at = NOW() 
       WHERE id = $2 AND customer_id = $3
       RETURNING id, customer_id, amount, status, updated_at`,
      [newStatus, invoiceId, currentUser.id]
    );

    // 5. Handle invoice not found or unauthorized access
    if (result.rowCount === 0) {
      return NextResponse.json(
        {
          success: false,
          error: "Invoice not found or you do not have permission to modify it.",
        },
        { status: 404 }
      );
    }

    // 6. Return sanitized, successful response
    return NextResponse.json({
      success: true,
      data: result.rows[0],
    });

  } catch (error) {
    // 7. Safe error handling: Log internally, don't leak internals to client
    console.error("[API_INVOICE_UPDATE_ERROR]:", error);

    return NextResponse.json(
      { success: false, error: "Internal server error" },
      { status: 500 }
    );
  }
}
```
