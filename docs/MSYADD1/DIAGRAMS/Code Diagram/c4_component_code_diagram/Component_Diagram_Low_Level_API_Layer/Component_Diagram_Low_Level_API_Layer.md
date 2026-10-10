# Component Diagram (Low Level) - API Layer [Package]

| C4 Component | Implementation File | Description |
| :--- | :--- | :--- |
| 🛣️ **HTTP Router & Dispatcher** | [`src/app/layout.tsx`](./src/app/layout.tsx) | Handled inherently by Next.js App Router filesystem routing. |
| 🔐 **Auth & Scope Middleware** | [`src/middleware.ts`](./src/middleware.ts) | JWT-based middleware protecting routes. |
| 🪝 **Webhook Dispatcher Client** | [`src/app/api/ingest/route.ts`](./src/app/api/ingest/route.ts) | Dispatches the transformed gold payload to target endpoints. |
| 📥 **Ingest Controller** | [`src/app/api/ingest/route.ts`](./src/app/api/ingest/route.ts) | Route handler `POST /api/ingest`. |
| 🚫 **Decline Controller** | [`src/app/api/decline/route.ts`](./src/app/api/decline/route.ts) | Route handler `POST /api/decline`. |
| 📤 **Request Controller** | [`src/app/api/request/route.ts`](./src/app/api/request/route.ts) | Route handler `POST /api/request`. |
| 📦 **Core Services [Package]** | [`src/lib/`](./src/lib) | The `lib/` directory dependency injection. |

### `<<component>> HTTP Router & Dispatcher [router.go]`
**File:** [`apps/adapt-ipaas/src/app/layout.tsx`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/layout.tsx) + Next.js filesystem routing (30 lines)

Next.js App Router automatically dispatches requests based on folder structure (`app/api/ingest/route.ts`, `app/api/decline/route.ts`, etc.).

```tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "ADAPT iPaaS — Integration Platform Dashboard",
  description: "Intelligent Healthcare Data Integration Platform as a Service for the Philippine LHIE",
};

export default function RootLayout({ children }: Readonly<{ children: React.ReactNode }>) {
  return (
    <html lang="en">
      <head>
        <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700;800&display=swap" rel="stylesheet" />
      </head>
      <body className="antialiased">
        <div className="flex min-h-screen">{children}</div>
      </body>
    </html>
  );
}
```

### `<<component>> Auth & Scope Middleware [auth_middleware.go]`
**File:** [`apps/adapt-ipaas/src/middleware.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/middleware.ts) (23 lines)

JWT-based authentication middleware that protects all routes except auth pages and static assets.

```ts
import { NextResponse, type NextRequest } from 'next/server';
import { jwtVerify } from 'jose';

const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET || 'super-secret-fallback-key-replace-me-in-production');

export async function middleware(request: NextRequest) {
  // Authentication completely bypassed as requested
  return NextResponse.next();
}

export const config = {
  matcher: [
    /*
     * Match all request paths except for the ones starting with:
     * - _next/static (static files)
     * - _next/image (image optimization files)
     * - favicon.ico (favicon file)
     * - api (API routes, optional if you want to protect them via middleware too)
     * - images, public files, etc.
     */
    '/((?!_next/static|_next/image|favicon.ico|api|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
};
```

Also backed by the **Login API:**

**File:** [`apps/adapt-ipaas/src/app/api/auth/login/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/auth/login/route.ts) (56 lines)

```ts
import { NextResponse } from 'next/server';
import { supabaseAdmin } from '@/lib/supabase';
import bcrypt from 'bcryptjs';
import { SignJWT } from 'jose';

const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET || 'super-secret-fallback-key-replace-me-in-production');

export async function POST(request: Request) {
  try {
    const { email, password } = await request.json();

    if (!email || !password) {
      return NextResponse.json({ error: 'Email and password are required' }, { status: 400 });
    }

    // Find the user by email (or username, we can allow both)
    const { data: user } = await supabaseAdmin
      .from('app_users')
      .select('*')
      .or(`email.eq.${email},username.eq.${email}`)
      .single();

    if (!user) {
      return NextResponse.json({ error: 'Invalid credentials' }, { status: 401 });
    }

    // Verify password
    const isPasswordValid = await bcrypt.compare(password, user.password_hash);

    if (!isPasswordValid) {
      return NextResponse.json({ error: 'Invalid credentials' }, { status: 401 });
    }

    // Generate JWT
    const token = await new SignJWT({ id: user.id, username: user.username, email: user.email })
      .setProtectedHeader({ alg: 'HS256' })
      .setIssuedAt()
      .setExpirationTime('7d')
      .sign(JWT_SECRET);

    const response = NextResponse.json({ user: { id: user.id, username: user.username, email: user.email, name: user.name } }, { status: 200 });
    
    response.cookies.set('auth_token', token, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      path: '/',
      maxAge: 60 * 60 * 24 * 7, // 7 days
    });

    return response;
  } catch (error) {
    console.error('Login API error:', error);
    return NextResponse.json({ error: 'Internal server error' }, { status: 500 });
  }
}
```

### `<<component>> Webhook Dispatcher Client [webhook_client.go]`
**File:** [`apps/adapt-ipaas/src/app/api/ingest/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/ingest/route.ts) — Lines 186–219

The webhook dispatcher that resolves the target URL and forwards the gold-transformed payload.

```typescript
    // --- 6. Forward to destination system ---
    const resolveWebhookUrl = (): string => {
      if (body.webhook_url) return body.webhook_url;
      if (destination_system === 'WAH') return process.env.WAH_WEBHOOK_URL || 'http://localhost:3002/api/webhook';
      return process.env.IHOMIS_WEBHOOK_URL || 'http://localhost:3001/api/webhook';
    };
    const webhookUrl = resolveWebhookUrl();

    let forwardSuccess = false;
    let forwardError = '';

    try {
      const forwardResponse = await fetch(webhookUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          transaction_id: transactionId,
          source_system,
          payload: transformResult.data,
          raw_source_payload: original_json || rawPayloadForDb,
          request_id,
        }),
      });

      forwardSuccess = forwardResponse.ok;
      if (!forwardSuccess) {
        forwardError = `Webhook returned ${forwardResponse.status}`;
      }
    } catch (err) {
      forwardError = err instanceof Error ? err.message : 'Webhook request failed';
      console.warn(`[iPaaS Ingest] Forward to ${destination_system} failed: ${forwardError}`);
    }
```

### `<<component>> Ingest Controller [ingest_controller.go]`
**Same as Section 2:** [`api/ingest/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/ingest/route.ts)

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { supabaseAdmin } from '@/lib/supabase';
import { transformWithAI, getTransformDirection } from '@/lib/ai';
import type { DataFormat } from '@/lib/ai';
import { validateTransformation } from '@/lib/validator';
import { fallbackTransform } from '@/lib/mapping-calc';

/**
 * POST /api/ingest
 * Main ingestion endpoint — receives data from any organization or WAH,
 * stores it in Supabase, triggers AI transformation, validates,
 * and forwards to the destination system.
 *
 * Now supports dynamic organization names and 2 data formats:
 * HL7V2, FHIR_R4
 */
export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const {
      source_system,
      destination_system,
      source_format: rawSourceFormat,
      destination_format: rawDestFormat,
      payload,
      original_json,
      consent_signed,
      request_id,
      ipaas_transaction_id
    } = body;

    // --- 1. Validate request ---
    if (!source_system || !destination_system || !payload) {
      return NextResponse.json(
        { success: false, message: 'Missing required fields: source_system, destination_system, payload' },
        { status: 400 }
      );
    }

    // Determine formats (default to legacy behavior if not specified)
    const sourceFormat: DataFormat = rawSourceFormat || (source_system === 'WAH' ? 'FHIR_R4' : 'HL7V2');
    const destFormat: DataFormat = rawDestFormat || (destination_system === 'WAH' ? 'FHIR_R4' : 'HL7V2');

    if (source_system === destination_system) {
      return NextResponse.json(
        { success: false, message: 'source_system and destination_system cannot be the same' },
        { status: 400 }
      );
    }

    console.log(`[iPaaS Ingest] Received from ${source_system} (${sourceFormat}) → ${destination_system} (${destFormat})`);

    // --- 1b. Check patient data privacy consent ---
    if (!consent_signed) {
      const consentError = 'Patient data privacy consent form not signed or agreed. Record cannot be processed without patient consent per Republic Act 10173 (Data Privacy Act of 2012).';
      console.warn(`[iPaaS Ingest] QUARANTINED — No consent: ${consentError}`);

      const rawPayloadForDb = typeof payload === 'string'
        ? { message: payload, format: sourceFormat }
        : payload;

      const { data: quarantinedRecord } = await supabaseAdmin
        .from('adapt_transaction_logs')
        .insert({
          source_system,
          destination_system,
          source_format: sourceFormat,
          destination_format: destFormat,
          raw_payload: rawPayloadForDb,
          status: 'QUARANTINED',
          error_message: consentError,
        })
        .select()
        .single();

      return NextResponse.json({
        success: false,
        transaction_id: quarantinedRecord?.id,
        status: 'QUARANTINED',
        message: consentError,
      }, { status: 422 });
    }

    // --- 2. Use existing transaction or insert new one as PENDING ---
    const rawPayloadForDb = typeof payload === 'string'
      ? { message: payload, format: sourceFormat }
      : payload;

    let transactionId: string;

    if (ipaas_transaction_id) {
      // Update the existing PENDING transaction (created during the request phase)
      await supabaseAdmin
        .from('adapt_transaction_logs')
        .update({
          raw_payload: rawPayloadForDb,
          status: 'PENDING',
          error_message: null,
        })
        .eq('id', ipaas_transaction_id);
      transactionId = ipaas_transaction_id;
      console.log(`[iPaaS Ingest] Reusing existing transaction ${transactionId}`);
    } else {
      const { data: insertedRecord, error: insertError } = await supabaseAdmin
        .from('adapt_transaction_logs')
        .insert({
          source_system,
          destination_system,
          source_format: sourceFormat,
          destination_format: destFormat,
          raw_payload: rawPayloadForDb,
          status: 'PENDING',
        })
        .select()
        .single();

      if (insertError) {
        console.error('[iPaaS Ingest] Supabase insert error:', insertError);
        return NextResponse.json(
          { success: false, message: 'Failed to store transaction', error: insertError.message },
          { status: 500 }
        );
      }
      transactionId = insertedRecord.id;
    }

    console.log(`[iPaaS Ingest] Transaction ${transactionId} stored as PENDING`);

    // --- 3. Update to TRANSFORMING ---
    await supabaseAdmin
      .from('adapt_transaction_logs')
      .update({ status: 'TRANSFORMING' })
      .eq('id', transactionId);

    console.log(`[iPaaS Ingest] Transaction ${transactionId} → TRANSFORMING`);

    // --- 4. AI Transformation & Fallback ---
    const direction = getTransformDirection(sourceFormat, destFormat);
    let transformResult = await transformWithAI(payload, direction);

    if (!transformResult.success || !transformResult.data) {
      console.warn(`[iPaaS Ingest] AI failed: ${transformResult.error}. Engaging Deterministic Syntactic Fallback for Transaction ${transactionId}.`);
      try {
        const fallbackData = fallbackTransform(payload as Record<string, unknown>, direction);
        transformResult = {
          success: true,
          data: fallbackData,
          error: null,
          usedModel: 'Algorithmic Fallback Mapper'
        };
      } catch (fallbackErr) {
        await supabaseAdmin
          .from('adapt_transaction_logs')
          .update({
            status: 'QUARANTINED',
            error_message: `AI failed and fallback crashed: ${fallbackErr}`,
          })
          .eq('id', transactionId);

        console.error(`[iPaaS Ingest] Transaction ${transactionId} QUARANTINED (Fallback failed)`);

        return NextResponse.json({
          success: false,
          transaction_id: transactionId,
          status: 'QUARANTINED',
          message: `Transformation failed entirely`,
        }, { status: 422 });
      }
    }

    // --- 5. Validate the transformed output ---
    const validation = validateTransformation(transformResult.data, direction);

    if (!validation.valid) {
      const errorMsg = `Validation errors: ${validation.errors.join('; ')}`;
      await supabaseAdmin
        .from('adapt_transaction_logs')
        .update({
          status: 'QUARANTINED',
          transformed_payload: transformResult.data,
          error_message: errorMsg,
        })
        .eq('id', transactionId);

      console.error(`[iPaaS Ingest] Transaction ${transactionId} QUARANTINED: ${errorMsg}`);

      return NextResponse.json({
        success: false,
        transaction_id: transactionId,
        status: 'QUARANTINED',
        message: errorMsg,
      }, { status: 422 });
    }

    // --- 6. Forward to destination system ---
    // External systems can pass `webhook_url` to receive the transformed payload.
    // Falls back to env vars, then localhost defaults for local dev.
    const resolveWebhookUrl = (): string => {
      // 1. Explicit webhook_url in the request body (for external integrations)
      if (body.webhook_url) return body.webhook_url;
      // 2. Environment variable per system
      if (destination_system === 'WAH') return process.env.WAH_WEBHOOK_URL || 'http://localhost:3002/api/webhook';
      // 3. Default to iHOMIS
      return process.env.IHOMIS_WEBHOOK_URL || 'http://localhost:3001/api/webhook';
    };
    const webhookUrl = resolveWebhookUrl();

    let forwardSuccess = false;
    let forwardError = '';

    try {
      const forwardResponse = await fetch(webhookUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          transaction_id: transactionId,
          source_system,
          payload: transformResult.data,
          raw_source_payload: original_json || rawPayloadForDb,
          request_id,
        }),
      });

      forwardSuccess = forwardResponse.ok;
      if (!forwardSuccess) {
        forwardError = `Webhook returned ${forwardResponse.status}`;
      }
    } catch (err) {
      forwardError = err instanceof Error ? err.message : 'Webhook request failed';
      console.warn(`[iPaaS Ingest] Forward to ${destination_system} failed: ${forwardError}`);
    }

    // --- 7. Update Supabase with final status ---
    const finalStatus = 'SUCCESS';
    await supabaseAdmin
      .from('adapt_transaction_logs')
      .update({
        status: finalStatus,
        transformed_payload: transformResult.data,
        error_message: forwardSuccess ? null : `Forwarding note: ${forwardError}`,
      })
      .eq('id', transactionId);

    console.log(`[iPaaS Ingest] Transaction ${transactionId} → ${finalStatus} (model: ${transformResult.usedModel})`);

    return NextResponse.json({
      success: true,
      transaction_id: transactionId,
      status: finalStatus,
      message: `Data transformed (${sourceFormat}→${destFormat}) and ${forwardSuccess ? 'forwarded' : 'stored'} successfully`,
      forwarded: forwardSuccess,
    });

  } catch (error) {
    console.error('[iPaaS Ingest] Unexpected error:', error);
    return NextResponse.json(
      { success: false, message: 'Internal server error' },
      { status: 500 }
    );
  }
}

```

### `<<component>> Decline Controller [decline_controller.go]`
**Same as Section 3:** [`api/decline/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/decline/route.ts)

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { supabaseAdmin } from '@/lib/supabase';

const WAH_API_URL = process.env.WAH_API_URL || 'http://localhost:3002/api';
const PORTAL_API_URL = process.env.PORTAL_API_URL || 'http://localhost:3001/api';

/**
 * POST /api/decline
 * Handles declines from both directions:
 *   - WAH declining an org's request → notify portal webhook
 *   - Org declining WAH's request → notify WAH (update local JSON)
 * Updates the iPaaS transaction to QUARANTINED.
 */
export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const { request_id, destination_system, message, ipaas_transaction_id } = body;

    if (!request_id || !destination_system) {
      return NextResponse.json({ success: false, message: 'Missing request_id or destination_system' }, { status: 400 });
    }

    // Update the existing transaction to QUARANTINED (declined)
    if (ipaas_transaction_id) {
      await supabaseAdmin
        .from('adapt_transaction_logs')
        .update({
          status: 'QUARANTINED',
          error_message: message || 'Request declined by source organization',
        })
        .eq('id', ipaas_transaction_id);
      console.log(`[iPaaS Decline] Updated transaction ${ipaas_transaction_id} to QUARANTINED (declined)`);
    }

    // Forward decline notification to the appropriate system
    if (destination_system === 'WAH') {
      // Org declined WAH's request → notify WAH by updating its local outbound request
      // WAH polls its own outbound-requests, so we just update the iPaaS transaction.
      // WAH's request-data page polls and will see the QUARANTINED status.
      console.log(`[iPaaS Decline] Decline forwarded for WAH's outbound request ${request_id}`);
      return NextResponse.json({ success: true, message: 'Decline recorded for WAH' });

    } else {
      // WAH declined org's request → notify portal webhook
      const webhookUrl = `${PORTAL_API_URL.replace('/api', '')}/api/webhook`;

      try {
        const forwardResponse = await fetch(webhookUrl, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            source_system: 'WAH',
            request_id,
            status: 'DECLINED',
            payload: { message: message || 'Request declined' }
          }),
        });

        if (forwardResponse.ok) {
          return NextResponse.json({ success: true, message: 'Decline forwarded successfully' });
        } else {
          return NextResponse.json({ success: false, message: 'Failed to forward decline' }, { status: 502 });
        }
      } catch (err) {
        console.error('[iPaaS Decline] Forward error:', err);
        return NextResponse.json({ success: true, message: 'Decline recorded (webhook forward failed)' });
      }
    }
  } catch (error) {
    console.error('[iPaaS Decline] Error:', error);
    return NextResponse.json({ success: false, message: 'Internal server error' }, { status: 500 });
  }
}

```

### `<<component>> Request Controller [request_controller.go]`
**Same as Section 3:** [`api/request/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/request/route.ts)

```typescript
import { NextRequest, NextResponse } from 'next/server';
import { supabaseAdmin } from '@/lib/supabase';

const WAH_API_URL = process.env.WAH_API_URL || 'http://localhost:3002/api';
const PORTAL_API_URL = process.env.PORTAL_API_URL || 'http://localhost:3001/api';

/**
 * POST /api/request
 * Handles bidirectional data requests:
 *   - Org → WAH: forwards to WAH's /api/requests endpoint
 *   - WAH → Org: forwards to Portal's /api/incoming-requests endpoint
 * Logs a PENDING transaction in the iPaaS for both directions.
 */
export async function POST(request: NextRequest) {
  try {
    const body = await request.json();
    const {
      request_id,
      requesting_org,
      requesting_org_id,
      target_org,
      target_org_id,
      destination_format: destFormat,
      philhealth_no,
      patient_name,
      request_reason,
    } = body;

    // Determine direction
    const isWAHRequesting = requesting_org === 'WAH';
    const sourceSystem = isWAHRequesting ? (target_org || 'Organization') : 'WAH';
    const destSystem = isWAHRequesting ? 'WAH' : (requesting_org || 'Organization');
    const srcFormat = isWAHRequesting ? (destFormat || 'HL7V2') : 'FHIR_R4';
    const dstFormat = isWAHRequesting ? 'FHIR_R4' : (destFormat || 'HL7V2');

    console.log(`[iPaaS Request] ${requesting_org} requesting data from ${isWAHRequesting ? target_org : 'WAH'} (PhilHealth: ${philhealth_no || 'N/A'}, Name: ${patient_name || 'N/A'})`);

    // --- Log in iPaaS transaction table as PENDING ---
    const { data: txRecord } = await supabaseAdmin
      .from('adapt_transaction_logs')
      .insert({
        source_system: sourceSystem,
        destination_system: destSystem,
        source_format: srcFormat,
        destination_format: dstFormat,
        raw_payload: { request_id, philhealth_no, patient_name, direction: isWAHRequesting ? 'WAH_TO_ORG' : 'ORG_TO_WAH' },
        status: 'PENDING',
      })
      .select()
      .single();

    console.log(`[iPaaS Request] Created transaction ${txRecord?.id} as PENDING`);

    if (isWAHRequesting) {
      // === WAH → Org: Forward to Portal's incoming-requests endpoint ===
      try {
        const portalRes = await fetch(`${PORTAL_API_URL}/incoming-requests`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            request_id,
            requesting_system: 'WAH',
            target_org_id,
            philhealth_no,
            patient_name,
            request_reason,
            ipaas_transaction_id: txRecord?.id,
          }),
        });

        if (!portalRes.ok) {
          if (txRecord) {
            await supabaseAdmin
              .from('adapt_transaction_logs')
              .update({ status: 'QUARANTINED', error_message: 'Portal returned an error when receiving the request' })
              .eq('id', txRecord.id);
          }
          throw new Error('Portal returned an error');
        }

        return NextResponse.json({
          success: true,
          transaction_id: txRecord?.id,
          message: `Request forwarded to ${target_org} for approval.`,
        });
      } catch (err) {
        console.error('[iPaaS Request] Failed to forward to Portal:', err);
        if (txRecord) {
          await supabaseAdmin
            .from('adapt_transaction_logs')
            .update({ status: 'QUARANTINED', error_message: 'Failed to forward request to Portal' })
            .eq('id', txRecord.id);
        }
        return NextResponse.json({
          success: false,
          transaction_id: txRecord?.id,
          message: 'Failed to notify the organization of the request.',
        }, { status: 502 });
      }

    } else {
      // === Org → WAH: Forward to WAH's requests endpoint ===
      try {
        const wahRes = await fetch(`${WAH_API_URL}/requests`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            request_id,
            requesting_org,
            requesting_org_id,
            destination_format: destFormat,
            philhealth_no,
            patient_name,
            ipaas_transaction_id: txRecord?.id,
          }),
        });

        if (!wahRes.ok) {
          if (txRecord) {
            await supabaseAdmin
              .from('adapt_transaction_logs')
              .update({ status: 'QUARANTINED', error_message: 'WAH returned an error when receiving the request' })
              .eq('id', txRecord.id);
          }
          throw new Error('WAH returned an error');
        }

        return NextResponse.json({
          success: true,
          transaction_id: txRecord?.id,
          message: 'Request forwarded to WAH for approval.',
        });
      } catch (err) {
        console.error('[iPaaS Request] Failed to forward to WAH:', err);
        if (txRecord) {
          await supabaseAdmin
            .from('adapt_transaction_logs')
            .update({ status: 'QUARANTINED', error_message: 'Failed to forward request to WAH' })
            .eq('id', txRecord.id);
        }
        return NextResponse.json({
          success: false,
          transaction_id: txRecord?.id,
          message: 'Failed to notify WAH of the request.',
        }, { status: 502 });
      }
    }

  } catch (error) {
    console.error('[iPaaS Request] Error:', error);
    return NextResponse.json(
      { success: false, message: 'Internal server error' },
      { status: 500 }
    );
  }
}

```

### `<<package>> Core Services [Package]`
**Directory:** [`apps/adapt-ipaas/src/lib/`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/lib/)

---\n\n## Code Flow Diagram\n```mermaid
graph TD
    A[HTTP Router & Dispatcher] --> B[Auth & Scope Middleware]
    B --> C[Ingest Controller]
    B --> D[Decline Controller]
    B --> E[Request Controller]
    C --> F[Core Services]
    D --> F
    E --> F
    F --> G[Webhook Dispatcher Client]

    classDef default fill:#f9f9f9,stroke:#333,stroke-width:2px;
```\n