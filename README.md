
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

---

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

---

### Question 3: Payment Webhooks
<img width="1948" height="296" alt="image" src="https://github.com/user-attachments/assets/b0c6784b-0379-496f-b1eb-7106646f6503" />

1. check status first before update
- when webhook comes in, check the invoice status in database first.
- if status is already PAID, just return 200 OK and exit early. 
- no need to update db again and dont send duplicate email to cust

2. save event ID
- stripe always sends a unique event_id.
- we can save this ID into our database table with unique constraint.
- if the same event ID hit our server again, our system knows it is a duplicate and simply ignores it.

3. use database transaction - prevent race condition
- wrap the update and event logging inside a transaction (BEGIN ... COMMIT).
- this prevents race condition if the gateway accidentally sends two requests at the exact same millisecond.

<img width="1948" height="72" alt="image" src="https://github.com/user-attachments/assets/16b7c33e-feab-498b-b664-639ca85f4f06" />

**Verify Webhook Signature**
-payment gateway will give us a *Webhook Secret Key* which we save safely inside our backend *.env* file.
- everytime gateway sends a webhook request, they include a signature in the header (like stripe-signature).
- in our code, we use that secret key to verify the signature with the request body.
- if the signature not matching or missing, it means someone is pretending or trying to fake a payment
- so we immediately reject the request.

---

## Section B: Code Review

<img width="2008" height="832" alt="image" src="https://github.com/user-attachments/assets/ed962aac-17ab-4f1a-a5a6-319ca815b343" />


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

**1. SQL Injection (Security Problem)**
- **What could go wrong:** Values `${newStatus}` and `${invoiceId}` are injected directly into the SQL string.
-anyone can pass malicious SQL (like `' OR '1'='1` or `; DROP TABLE invoices; --`) and compromise, steal, or delete the entire database.
- **How to fix:** use parameterized queries / prepared statements (e.g. `$1, $2`) so the database engine treats input strictly as data, never as executable code.

**2. Illegal State Transitions (Logic & Business)**
- **What could go wrong:** An invoice is a state machine, but this code blindly accepts any `newStatus`. For example, an invoice that is already `PAID` or `CANCELLED` could be illegally changed back to `DRAFT`.
- **How to fix:** Validate allowed transitions in business logic
  example
  - `DRAFT` can go to `PENDING`, 
  - `PENDING` can go to `PAID`, 
  - *but `PAID` cannot go back to `DRAFT`*

**3. Missing Authentication & Authorization (Architecture & Data )**
- **What could go wrong:** The code takes `userId` from the body but never checks if the caller is actually logged in, nor does it verify if that user owns the invoice.
- normal user can call this API and mark any invoice as `PAID` for free and easily
- **How to fix:** Get user identity from server session/token (never trust client body), check ownership in database (`WHERE id = $1 AND customer_id = $2`)
- and ensure only admins or payment webhooks can mark an invoice as `PAID`.

---

#### Fixed Version of the Route:

```typescript
// app/api/invoice/update/route.ts
import { NextResponse } from "next/server";
import { z } from "zod";
import { db } from "@/lib/db";
import { getAuthUser } from "@/lib/auth"; // verify session

// 1. Schema validation
const updateSchema = z.object({
  invoiceId: z.coerce.number().int().positive(),
  newStatus: z.enum(["DRAFT", "PENDING", "PAID", "CANCELLED"]),
});

// 2. Business logic: State machine transitions
const ALLOWED_TRANSITIONS: Record<string, string[]> = {
  DRAFT: ["PENDING", "CANCELLED"],
  PENDING: ["PAID", "CANCELLED"],
  PAID: ["CANCELLED"], // once paid, cannot go back to draft
  CANCELLED: [],       // final state
};

export async function POST(req: Request) {
  try {
    // Check if user is logged in
    const user = await getAuthUser(req);
    if (!user) {
      return NextResponse.json({ success: false, error: "Unauthorized" }, { status: 401 });
    }

    const body = await req.json().catch(() => null);
    const parsed = updateSchema.safeParse(body);
    if (!parsed.success) {
      return NextResponse.json({ success: false, error: parsed.error.flatten().fieldErrors }, { status: 400 });
    }

    const { invoiceId, newStatus } = parsed.data;

    // Fetch existing invoice to check ownership and state transition
    const existing = await db.query(
      `SELECT id, status, customer_id FROM invoices WHERE id = $1 AND customer_id = $2`,
      [invoiceId, user.id]
    );

    if (existing.rowCount === 0) {
      return NextResponse.json({ success: false, error: "Invoice not found or forbidden" }, { status: 404 });
    }

    const currentStatus = existing.rows[0].status;

    // Check illegal state transition
    const allowed = ALLOWED_TRANSITIONS[currentStatus] || [];
    if (!allowed.includes(newStatus)) {
      return NextResponse.json(
        { success: false, error: `Cannot transition invoice from ${currentStatus} to ${newStatus}` },
        { status: 400 }
      );
    }

    // Parameterized query to prevent SQL Injection
    const updated = await db.query(
      `UPDATE invoices 
       SET status = $1, updated_at = NOW() 
       WHERE id = $2 
       RETURNING *`,
      [newStatus, invoiceId]
    );

    return NextResponse.json({ success: true, data: updated.rows[0] });

  } catch (error) {
    console.error("Invoice update error:", error);
    return NextResponse.json({ success: false, error: "Internal server error" }, { status: 500 });
  }
}
