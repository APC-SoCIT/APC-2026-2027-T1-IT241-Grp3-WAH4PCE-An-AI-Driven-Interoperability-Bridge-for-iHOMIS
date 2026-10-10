# Component Diagram (Low Level) - Core Services [Package]

| C4 Component | Implementation File | Description |
| :--- | :--- | :--- |
| 📦 **API Layer [Package]** | [`src/app/api/`](./src/app/api) | Next.js API Routes representing the Golang HTTP services. |
| 🔄 **Provide Health Data Orchestrator** | [`src/app/api/ingest/route.ts`](./src/app/api/ingest/route.ts) | Coordinates Bronze, Silver, and Gold transformations. |
| ✅ **Validate Schema & Completeness** | [`src/lib/validator.ts`](./src/lib/validator.ts) | `validateFHIRBundle()` & `validateHL7V2Payload()`. |
| 🔐 **Privacy Consent Gatekeeper** | [`src/app/api/ingest/route.ts`](./src/app/api/ingest/route.ts) | Verifies patient consent inside the orchestrator. |
| 🛡️ **Deterministic Syntactic Fallback** | [`src/lib/mapping-calc.ts`](./src/lib/mapping-calc.ts) | 390+ lines of fallback mapping algorithms. |
| 🤖 **Semantic AI Translator** | [`src/lib/ai.ts`](./src/lib/ai.ts) | `transformWithAI()` execution chain. |
| 📝 **Sanitized Transaction Logger** | [`src/app/api/ingest/route.ts`](./src/app/api/ingest/route.ts) | Final stage Supabase write operations. |
| 🚫 **Quarantine Queue Manager** | [`src/app/api/decline/route.ts`](./src/app/api/decline/route.ts) | Logic routing failed payloads to QUARANTINED state. |
| 🧠 **Local Gemma 4 LLM Engine** | [`src/lib/ai.ts`](./src/lib/ai.ts) | Local Gemma 4 provider initialization. |
| 💾 **MongoDB Database** | [`src/lib/supabase.ts`](./src/lib/supabase.ts) | Supabase/Postgres equivalent. |

### `<<package>> API Layer [Package]`
**Directory:** [`apps/adapt-ipaas/src/app/api/`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/) — Contains `ingest/`, `decline/`, `request/`, `metrics/`, `transactions/`, `auth/`

### `<<component>> Provide Health Data Orchestrator [orchestrator.go]`
**Same as Section 2 — Provide Health Data Pipeline Engine:** [`api/ingest/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/ingest/route.ts) Lines 83–157 (Bronze → Silver → Gold pipeline)

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

### `<<component>> Validate Schema & Completeness [schema_validator.go]`
**Same as Section 2:** [`lib/validator.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/lib/validator.ts) — `validateFHIRBundle()` + `validateHL7V2Payload()`

```typescript
/**
 * FHIR & HL7v2 Payload Validator
 * Checks for mandatory fields before forwarding
 */

import type { TransformDirection } from './ai';

interface ValidationResult {
  valid: boolean;
  errors: string[];
}

/**
 * Validate a FHIR Bundle output (→ FHIR R4 transformation)
 */
export function validateFHIRBundle(data: Record<string, unknown>): ValidationResult {
  const errors: string[] = [];

  // Check it's a Bundle
  if (data.resourceType !== 'Bundle') {
    errors.push('Root resourceType must be "Bundle"');
  }

  // Check it has entries
  const entries = data.entry as Array<Record<string, unknown>> | undefined;
  if (!entries || !Array.isArray(entries) || entries.length === 0) {
    errors.push('Bundle must contain at least one entry');
  } else {
    // Check for required resource types
    const resourceTypes = entries.map(
      (e) => (e.resource as Record<string, unknown>)?.resourceType
    );

    if (!resourceTypes.includes('Patient')) {
      errors.push('Bundle must contain a Patient resource');
    }
    if (!resourceTypes.includes('Encounter')) {
      errors.push('Bundle must contain an Encounter resource');
    }
    if (!resourceTypes.includes('Condition')) {
      errors.push('Bundle must contain a Condition resource');
    }

    // Check Patient has PhilHealth ID
    const patientEntry = entries.find(
      (e) => (e.resource as Record<string, unknown>)?.resourceType === 'Patient'
    );
    if (patientEntry) {
      const patient = patientEntry.resource as Record<string, unknown>;
      const identifiers = patient.identifier as Array<Record<string, unknown>> | undefined;
      const hasPhilHealth = identifiers?.some(
        (id) => id.system === 'https://www.philhealth.gov.ph/memberid' && id.value
      );
      if (!hasPhilHealth) {
        errors.push('Patient must have a PhilHealth identifier');
      }
    }
  }

  return { valid: errors.length === 0, errors };
}

/**
 * Validate an HL7v2 flat JSON output (FHIR → HL7v2 transformation)
 */
export function validateHL7V2Payload(data: Record<string, unknown>): ValidationResult {
  const errors: string[] = [];

  const requiredFields = [
    'patient_fname',
    'patient_lname',
    'dob',
    'sex',
    'philhealth_no',
    'diagnosis_code',
    'referring_facility_name',
  ];

  for (const field of requiredFields) {
    if (!data[field]) {
      errors.push(`Missing required field: ${field}`);
    }
  }

  // Check vitals object exists
  if (!data.vitals || typeof data.vitals !== 'object') {
    errors.push('Missing or invalid vitals object');
  } else {
    const vitals = data.vitals as Record<string, unknown>;
    const requiredVitals = ['bp_systolic', 'bp_diastolic', 'heart_rate', 'temperature'];
    for (const vital of requiredVitals) {
      if (vitals[vital] === undefined || vitals[vital] === null) {
        errors.push(`Missing vital sign: ${vital}`);
      }
    }
  }

  // Validate PhilHealth number format (basic check)
  if (data.philhealth_no && typeof data.philhealth_no === 'string') {
    if (data.philhealth_no.length < 6) {
      errors.push('PhilHealth number appears invalid (too short)');
    }
  }

  return { valid: errors.length === 0, errors };
}

// Legacy alias for backward compatibility
export function validateIHOMISPayload(data: Record<string, unknown>): ValidationResult {
  return validateHL7V2Payload(data);
}

/**
 * Validate based on transformation direction
 */
export function validateTransformation(
  data: Record<string, unknown>,
  direction: TransformDirection | 'IHOMIS_TO_FHIR' | 'FHIR_TO_IHOMIS'
): ValidationResult {
  switch (direction) {
    case 'HL7V2_TO_FHIR_R4':
    case 'IHOMIS_TO_FHIR':
      return validateFHIRBundle(data);
    case 'FHIR_R4_TO_HL7V2':
    case 'FHIR_TO_IHOMIS':
      return validateHL7V2Payload(data);
    default:
      return validateFHIRBundle(data);
  }
}

```

### `<<component>> Privacy Consent Gatekeeper [consent_gatekeeper.go]`
**Same as Section 2:** [`api/ingest/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/ingest/route.ts) Lines 52–81

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

### `<<component>> Deterministic Syntactic Fallback [fallback_mapper.go]`
**Same as Section 2:** [`lib/mapping-calc.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/lib/mapping-calc.ts)

```typescript
/**
 * Shared mapping completeness calculator.
 * Reused by both the Data Mapper page and the Dashboard to compute
 * how many destination fields were successfully filled by the AI transformation.
 */
import type { TransformDirection } from './ai';


// ─── Templates ───

const IHOMIS_TEMPLATE: { category: string; label: string }[] = [
  { category: 'Patient Demographics', label: 'First Name' },
  { category: 'Patient Demographics', label: 'Middle Name' },
  { category: 'Patient Demographics', label: 'Last Name' },
  { category: 'Patient Demographics', label: 'Suffix' },
  { category: 'Patient Demographics', label: 'Date of Birth' },
  { category: 'Patient Demographics', label: 'Sex' },
  { category: 'Patient Demographics', label: 'Civil Status' },
  { category: 'Patient Demographics', label: 'PhilHealth No.' },
  { category: 'Patient Demographics', label: 'Contact No.' },
  { category: 'Patient Demographics', label: 'Street' },
  { category: 'Patient Demographics', label: 'Barangay' },
  { category: 'Patient Demographics', label: 'City' },
  { category: 'Patient Demographics', label: 'Province' },
  { category: 'Vital Signs', label: 'BP Systolic' },
  { category: 'Vital Signs', label: 'BP Diastolic' },
  { category: 'Vital Signs', label: 'Heart Rate' },
  { category: 'Vital Signs', label: 'Temperature' },
  { category: 'Vital Signs', label: 'Respiratory Rate' },
  { category: 'Vital Signs', label: 'SpO2' },
  { category: 'Vital Signs', label: 'Weight (kg)' },
  { category: 'Vital Signs', label: 'Height (cm)' },
  { category: 'Diagnosis & Referral', label: 'Chief Complaint' },
  { category: 'Diagnosis & Referral', label: 'ICD-10 Code' },
  { category: 'Diagnosis & Referral', label: 'Diagnosis Description' },
  { category: 'Diagnosis & Referral', label: 'Priority' },
  { category: 'Diagnosis & Referral', label: 'Referring Facility' },
  { category: 'Diagnosis & Referral', label: 'Physician' },
];

const WAH_TEMPLATE: { category: string; label: string }[] = [
  { category: 'Patient Resource', label: 'Given Name' },
  { category: 'Patient Resource', label: 'Middle Name' },
  { category: 'Patient Resource', label: 'Family Name' },
  { category: 'Patient Resource', label: 'Birth Date' },
  { category: 'Patient Resource', label: 'Gender' },
  { category: 'Patient Resource', label: 'PhilHealth ID' },
  { category: 'Patient Resource', label: 'Phone' },
  { category: 'Patient Resource', label: 'Address' },
  { category: 'Patient Resource', label: 'City' },
  { category: 'Encounter', label: 'Class' },
  { category: 'Encounter', label: 'Priority' },
  { category: 'Encounter', label: 'Facility' },
  { category: 'Encounter', label: 'Physician' },
  { category: 'Encounter', label: 'Reason' },
  { category: 'Observations (Vitals)', label: 'BP Systolic' },
  { category: 'Observations (Vitals)', label: 'BP Diastolic' },
  { category: 'Observations (Vitals)', label: 'Heart Rate' },
  { category: 'Observations (Vitals)', label: 'Temperature' },
  { category: 'Observations (Vitals)', label: 'Respiratory Rate' },
  { category: 'Observations (Vitals)', label: 'SpO2' },
  { category: 'Observations (Vitals)', label: 'Weight (kg)' },
  { category: 'Observations (Vitals)', label: 'Height (cm)' },
  { category: 'Condition', label: 'ICD-10 Code' },
  { category: 'Condition', label: 'Display' },
  { category: 'Condition', label: 'Chief Complaint' },
  { category: 'Condition', label: 'Clinical Status' },
];

// ─── Aliases (same as mapper) ───

const EXTRACTOR_ALIASES: Record<string, string[]> = {
  'Given Name': ['Given Name'],
  'Family Name': ['Family Name'],
  'Birth Date': ['Birth Date'],
  'Gender': ['Gender'],
  'PhilHealth ID': ['PhilHealth No.', 'PhilHealth ID'],
  'Phone': ['Phone', 'Contact No.'],
  'Address': ['Address Line', 'Street', 'Address'],
  'Class': ['Class'],
  'Reason': ['Reason'],
  'Facility': ['Facility', 'Referring Facility'],
  'Physician': ['Physician', 'Requester'],
  'BP Systolic': ['BP Systolic', 'Systolic blood pressure', 'Systolic Blood Pressure'],
  'BP Diastolic': ['BP Diastolic', 'Diastolic blood pressure', 'Diastolic Blood Pressure'],
  'Heart Rate': ['Heart Rate', 'Heart rate'],
  'Temperature': ['Temperature', 'Body temperature', 'Body Temperature'],
  'Respiratory Rate': ['Respiratory Rate', 'Respiratory rate'],
  'SpO2': ['SpO2', 'Oxygen saturation', 'Oxygen Saturation'],
  'Weight (kg)': ['Weight (kg)', 'Body weight', 'Body Weight'],
  'Height (cm)': ['Height (cm)', 'Body height', 'Body Height'],
  'Display': ['Display', 'Description', 'Diagnosis Description'],
  'Clinical Status': ['Clinical Status', 'Clinical Notes'],
  'Chief Complaint': ['Chief Complaint'],
  'ICD-10 Code': ['ICD-10 Code'],
  'Middle Name': ['Middle Name'],
  'City': ['City'],
  'Priority': ['Priority'],
  'First Name': ['First Name', 'Given Name'],
  'Last Name': ['Last Name', 'Family Name'],
  'Date of Birth': ['Date of Birth', 'Birth Date'],
  'Sex': ['Sex', 'Gender'],
  'Civil Status': ['Civil Status', 'Marital Status'],
  'Contact No.': ['Contact No.', 'Phone'],
  'Street': ['Street', 'Address Line', 'Address'],
  'Province': ['Province', 'Province/State'],
  'Barangay': ['Barangay'],
  'Suffix': ['Suffix'],
  'PhilHealth No.': ['PhilHealth No.', 'PhilHealth ID'],
  'Diagnosis Description': ['Diagnosis Description', 'Description', 'Display'],
  'Referring Facility': ['Referring Facility', 'Facility'],
};

const FIELD_MAP: [string, string][] = [
  ['First Name', 'Given Name'],
  ['Last Name', 'Family Name'],
  ['Middle Name', 'Middle Name'],
  ['Date of Birth', 'Birth Date'],
  ['Sex', 'Gender'],
  ['PhilHealth No.', 'PhilHealth ID'],
  ['Contact No.', 'Phone'],
  ['Street', 'Address'],
  ['City', 'City'],
  ['BP Systolic', 'BP Systolic'],
  ['BP Diastolic', 'BP Diastolic'],
  ['Heart Rate', 'Heart Rate'],
  ['Temperature', 'Temperature'],
  ['Respiratory Rate', 'Respiratory Rate'],
  ['SpO2', 'SpO2'],
  ['Weight (kg)', 'Weight (kg)'],
  ['Height (cm)', 'Height (cm)'],
  ['Chief Complaint', 'Chief Complaint'],
  ['ICD-10 Code', 'ICD-10 Code'],
  ['Diagnosis Description', 'Display'],
  ['Priority', 'Priority'],
  ['Physician', 'Physician'],
  ['Referring Facility', 'Facility'],
];

// ─── Field Extractors ───

type Row = { category: string; label: string; value: string };

function extractHL7Data(payload: Record<string, unknown>): Row[] {
  const vitalsObj = (payload.vitals && typeof payload.vitals === 'object') ? payload.vitals as Record<string, unknown> : null;
  const g = (k: string) => {
    if (payload[k] != null && String(payload[k]) !== '') return String(payload[k]);
    if (vitalsObj && vitalsObj[k] != null && String(vitalsObj[k]) !== '' && String(vitalsObj[k]) !== '0') return String(vitalsObj[k]);
    return '';
  };
  const rows: Row[] = [];
  const add = (cat: string, label: string, val: string) => { if (val) rows.push({ category: cat, label, value: val }); };

  add('Patient', 'First Name', g('patient_fname'));
  add('Patient', 'Last Name', g('patient_lname'));
  add('Patient', 'Middle Name', g('patient_mname'));
  add('Patient', 'Suffix', g('patient_suffix'));
  add('Patient', 'Date of Birth', g('dob'));
  add('Patient', 'Sex', g('sex'));
  add('Patient', 'Civil Status', g('civil_status'));
  add('Patient', 'PhilHealth No.', g('philhealth_no'));
  add('Patient', 'Contact No.', g('contact_no'));
  add('Patient', 'Street', g('address_street'));
  add('Patient', 'Barangay', g('address_barangay'));
  add('Patient', 'City', g('address_city'));
  add('Patient', 'Province', g('address_province'));
  add('Vitals', 'BP Systolic', g('bp_systolic'));
  add('Vitals', 'BP Diastolic', g('bp_diastolic'));
  add('Vitals', 'Heart Rate', g('heart_rate'));
  add('Vitals', 'Temperature', g('temperature'));
  add('Vitals', 'Respiratory Rate', g('respiratory_rate'));
  add('Vitals', 'SpO2', g('oxygen_saturation'));
  add('Vitals', 'Weight (kg)', g('weight_kg'));
  add('Vitals', 'Height (cm)', g('height_cm'));
  add('Diagnosis', 'Chief Complaint', g('chief_complaint'));
  add('Diagnosis', 'ICD-10 Code', g('diagnosis_code'));
  add('Diagnosis', 'Description', g('diagnosis_desc'));
  add('Diagnosis', 'Priority', g('priority'));
  add('Referral', 'Facility', g('referring_facility_name'));
  add('Referral', 'Physician', g('referring_physician'));

  return rows;
}

// eslint-disable-next-line @typescript-eslint/no-explicit-any
function extractFHIRData(payload: Record<string, unknown>): Row[] {
  const rows: Row[] = [];
  const add = (cat: string, label: string, val: unknown) => { if (val != null && String(val).trim()) rows.push({ category: cat, label, value: String(val).trim() }); };

  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  const entries = (payload as any)?.entry || [];
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  const resources = entries.map((e: any) => e?.resource).filter(Boolean);
  if (resources.length === 0 && (payload as Record<string, unknown>)?.resourceType) {
    resources.push(payload);
  }

  let chiefComplaintFound = false;

  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  for (const res of resources as any[]) {
    const rt = res?.resourceType;
    if (rt === 'Patient') {
      const name = res.name?.[0] || {};
      const givenArr = name.given || [];
      add('Patient', 'Given Name', givenArr[0]);
      add('Patient', 'Middle Name', givenArr.length > 1 ? givenArr.slice(1).join(' ') : null);
      add('Patient', 'Family Name', name.family);
      add('Patient', 'Suffix', name.suffix?.[0]);
      add('Patient', 'Birth Date', res.birthDate);
      add('Patient', 'Gender', res.gender);
      add('Patient', 'Marital Status', res.maritalStatus?.text || res.maritalStatus?.coding?.[0]?.display);
      for (const id of (res.identifier || [])) {
        if (id.system?.includes('philhealth') || id.type?.coding?.[0]?.code === 'SB') {
          add('Patient', 'PhilHealth No.', id.value);
        }
      }
      for (const t of (res.telecom || [])) {
        add('Patient', 'Phone', t.value);
      }
      const addr = res.address?.[0] || {};
      add('Patient', 'Address Line', (addr.line || []).join(', '));
      add('Patient', 'City', addr.city);
      add('Patient', 'Province/State', addr.state || addr.district);
    }
    if (rt === 'Encounter') {
      add('Encounter', 'Class', res.class?.display || res.class?.code);
      add('Encounter', 'Priority', res.priority?.coding?.[0]?.display || res.priority?.coding?.[0]?.code || res.priority?.text || res.priority);
      const reason = res.reasonCode?.[0]?.text || res.reasonCode?.[0]?.coding?.[0]?.display || res.reason?.[0]?.concept?.text;
      add('Encounter', 'Reason', reason);
      if (reason && !chiefComplaintFound) { /* fallback later */ }
      add('Encounter', 'Facility', res.serviceProvider?.display || res.serviceProvider?.reference || res.location?.[0]?.location?.display);
      add('Encounter', 'Physician', res.participant?.[0]?.individual?.display || res.participant?.[0]?.actor?.display);
    }
    if (rt === 'Observation') {
      const display = res.code?.text || res.code?.coding?.[0]?.display || 'Observation';
      if (res.component) {
        for (const comp of res.component) {
          const compName = comp.code?.coding?.[0]?.display || 'Component';
          add('Vitals', compName, comp.valueQuantity?.value);
        }
      } else if (res.valueQuantity) {
        add('Vitals', display, res.valueQuantity.value);
      }
    }
    if (rt === 'Condition') {
      add('Diagnosis', 'ICD-10 Code', res.code?.coding?.[0]?.code);
      add('Diagnosis', 'Description', res.code?.coding?.[0]?.display || res.code?.text);
      add('Diagnosis', 'Clinical Status', res.clinicalStatus?.coding?.[0]?.code);
      const complaint = res.note?.[0]?.text || res.category?.[0]?.text;
      if (complaint) {
        add('Diagnosis', 'Chief Complaint', complaint);
        chiefComplaintFound = true;
      }
    }
  }

  if (!chiefComplaintFound) {
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    for (const res of resources as any[]) {
      if (res?.resourceType === 'Encounter') {
        const reason = res.reasonCode?.[0]?.text || res.reasonCode?.[0]?.coding?.[0]?.display || res.reason?.[0]?.concept?.text;
        if (reason) {
          add('Diagnosis', 'Chief Complaint', reason);
          break;
        }
      }
    }
  }

  return rows;
}

function extractDataFields(payload: Record<string, unknown>): Row[] {
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  if ((payload as any)?.resourceType === 'Bundle' || (payload as any)?.entry) return extractFHIRData(payload);
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  if ((payload as any)?.resourceType) return extractFHIRData(payload);
  // eslint-disable-next-line @typescript-eslint/no-explicit-any
  if ((payload as any)?.patient_fname || (payload as any)?.patient_lname || (payload as any)?.philhealth_no || (payload as any)?.bp_systolic) return extractHL7Data(payload);
  const hl7 = extractHL7Data(payload);
  if (hl7.length > 0) return hl7;
  return extractFHIRData(payload);
}

// ─── Matching Logic ───

function findValueForTemplateField(
  templateLabel: string,
  extractedFields: Row[],
  isDestWAH: boolean
): string | null {
  const direct = extractedFields.find(f => f.label === templateLabel);
  if (direct) return direct.value;

  const aliases = EXTRACTOR_ALIASES[templateLabel];
  if (aliases) {
    for (const alias of aliases) {
      const found = extractedFields.find(f => f.label === alias);
      if (found) return found.value;
    }
  }

  const lower = templateLabel.toLowerCase();
  const ci = extractedFields.find(f => f.label.toLowerCase() === lower);
  if (ci) return ci.value;

  for (const [iLabel, wLabel] of FIELD_MAP) {
    const destLabel = isDestWAH ? wLabel : iLabel;
    const srcLabel = isDestWAH ? iLabel : wLabel;
    if (destLabel === templateLabel) {
      const found = extractedFields.find(f => f.label === srcLabel);
      if (found) return found.value;
    }
  }
  return null;
}

// ─── Public API ───

export interface MappingResult {
  totalFields: number;
  filledFields: number;
  emptyFields: number;
  percentage: number;
}

/**
 * Calculate the mapping completeness percentage for a transaction.
 * @param transformedPayload The AI-transformed output payload
 * @param destinationSystem The destination system name (e.g. 'WAH', 'iHOMIS')
 */
export function calculateMappingPercentage(
  transformedPayload: Record<string, unknown> | null,
  destinationSystem: string
): MappingResult {
  if (!transformedPayload) {
    return { totalFields: 0, filledFields: 0, emptyFields: 0, percentage: 0 };
  }

  const isDestWAH = destinationSystem.toLowerCase().includes('wah');
  const template = isDestWAH ? WAH_TEMPLATE : IHOMIS_TEMPLATE;
  const extracted = extractDataFields(transformedPayload);

  let filled = 0;
  for (const tf of template) {
    const val = findValueForTemplateField(tf.label, extracted, isDestWAH);
    if (val) filled++;
  }

  const total = template.length;
  const pct = total > 0 ? Number(((filled / total) * 100).toFixed(1)) : 0;

  return {
    totalFields: total,
    filledFields: filled,
    emptyFields: total - filled,
    percentage: pct,
  };
}

/**
 * Calculate source-side field fill count.
 */
export function calculateSourceFillCount(
  rawPayload: Record<string, unknown> | null,
  sourceSystem: string
): MappingResult {
  if (!rawPayload) {
    return { totalFields: 0, filledFields: 0, emptyFields: 0, percentage: 0 };
  }

  const isSrcWAH = sourceSystem.toLowerCase().includes('wah');
  const template = isSrcWAH ? WAH_TEMPLATE : IHOMIS_TEMPLATE;
  const extracted = extractDataFields(rawPayload);

  let filled = 0;
  for (const tf of template) {
    const val = findValueForTemplateField(tf.label, extracted, isSrcWAH);
    if (val) filled++;
  }

  const total = template.length;
  const pct = total > 0 ? Number(((filled / total) * 100).toFixed(1)) : 0;

  return {
    totalFields: total,
    filledFields: filled,
    emptyFields: total - filled,
    percentage: pct,
  };
}

/**
 * Deterministic algorithmic fallback mapper
 * Used when AI models fail or rate limit
 */
export function fallbackTransform(
  payload: Record<string, unknown>,
  direction: TransformDirection
): Record<string, unknown> {
  const extracted = extractDataFields(payload);
  const isDestWAH = direction === 'HL7V2_TO_FHIR_R4' || direction === 'IHOMIS_TO_FHIR';
  
  const getVal = (label: string) => {
    return findValueForTemplateField(label, extracted, isDestWAH) || '';
  };

  if (isDestWAH) {
    const bundle: Record<string, unknown> = {
      resourceType: 'Bundle',
      type: 'transaction',
      entry: [
        {
          resource: {
            resourceType: 'Patient',
            identifier: [
              { system: 'https://www.philhealth.gov.ph/memberid', value: getVal('PhilHealth ID') }
            ],
            name: [{
              family: getVal('Family Name'),
              given: [getVal('Given Name')],
              // Use direct search for Suffix as it's not in the main WAH template
              suffix: [extracted.find(f => f.label === 'Suffix')?.value || '']
            }],
            gender: getVal('Gender'),
            birthDate: getVal('Birth Date'),
            telecom: [{ value: getVal('Phone') }],
            address: [{
              line: [getVal('Address')],
              city: getVal('City')
            }]
          }
        },
        {
          resource: {
            resourceType: 'Encounter',
            class: { code: getVal('Class') || 'AMB' },
            priority: { text: getVal('Priority') },
            reasonCode: [{ text: getVal('Reason') }],
            serviceProvider: { display: getVal('Facility') },
            participant: [{ individual: { display: getVal('Physician') } }]
          }
        },
        {
          resource: {
            resourceType: 'Condition',
            code: {
              coding: [{ code: getVal('ICD-10 Code'), display: getVal('Display') }]
            },
            clinicalStatus: { coding: [{ code: getVal('Clinical Status') }] },
            note: [{ text: getVal('Chief Complaint') }]
          }
        }
      ]
    };

    const addVital = (label: string, code: string, display: string) => {
      const val = getVal(label);
      if (val) {
        (bundle.entry as Array<unknown>).push({
          resource: {
            resourceType: 'Observation',
            code: { coding: [{ code, display }] },
            valueQuantity: { value: Number(val) || val }
          }
        });
      }
    };

    addVital('BP Systolic', '8480-6', 'Systolic blood pressure');
    addVital('BP Diastolic', '8462-4', 'Diastolic blood pressure');
    addVital('Heart Rate', '8867-4', 'Heart rate');
    addVital('Temperature', '8310-5', 'Body temperature');
    addVital('Respiratory Rate', '9279-1', 'Respiratory rate');
    addVital('SpO2', '2708-6', 'Oxygen saturation');
    addVital('Weight (kg)', '29463-7', 'Body weight');
    addVital('Height (cm)', '8302-2', 'Body height');

    return bundle;
  } else {
    // FHIR_R4_TO_HL7V2 or FHIR_TO_IHOMIS
    return {
      patient_fname: getVal('First Name'),
      patient_lname: getVal('Last Name'),
      patient_mname: getVal('Middle Name'),
      dob: getVal('Date of Birth'),
      sex: getVal('Sex'),
      civil_status: getVal('Civil Status'),
      philhealth_no: getVal('PhilHealth No.'),
      contact_no: getVal('Contact No.'),
      address_street: getVal('Street'),
      address_city: getVal('City'),
      vitals: {
        bp_systolic: getVal('BP Systolic'),
        bp_diastolic: getVal('BP Diastolic'),
        heart_rate: getVal('Heart Rate'),
        temperature: getVal('Temperature'),
        respiratory_rate: getVal('Respiratory Rate'),
        oxygen_saturation: getVal('SpO2'),
        weight_kg: getVal('Weight (kg)'),
        height_cm: getVal('Height (cm)')
      },
      chief_complaint: getVal('Chief Complaint'),
      diagnosis_code: getVal('ICD-10 Code'),
      diagnosis_desc: getVal('Diagnosis Description'),
      priority: getVal('Priority'),
      referring_facility_name: getVal('Referring Facility'),
      referring_physician: getVal('Physician')
    };
  }
}

```

### `<<component>> Semantic AI Translator [ai_translator.go]`
**Same as Section 1:** [`lib/ai.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/lib/ai.ts) — `transformWithAI()`

```typescript
import { GoogleGenerativeAI } from '@google/generative-ai';
import Groq from 'groq-sdk';

const genAI = process.env.GEMINI_API_KEY ? new GoogleGenerativeAI(process.env.GEMINI_API_KEY) : null;
const groq = process.env.GROQ_API_KEY ? new Groq({ apiKey: process.env.GROQ_API_KEY }) : null;

// Model fallback chain — try Gemini first, then juggle to Groq
export const MODEL_FALLBACKS = [
  { provider: 'gemini', model: process.env.GEMINI_MODEL || 'gemini-3.1-flash-lite' },
  { provider: 'gemini', model: 'gemini-2.5-flash-lite' },
  { provider: 'gemini', model: 'gemini-2.5-flash' },
  { provider: 'groq', model: 'llama-3.3-70b-versatile' },
  { provider: 'groq', model: 'mixtral-8x7b-32768' },
  { provider: 'groq', model: 'llama3-70b-8192' }
];

// ============================================
// System Prompts for all format pairs
// ============================================

const HL7V2_TO_FHIR_PROMPT = `You are a healthcare data transformation engine for the Philippine Local Health Information Exchange (LHIE).

Your task: Convert the input flat JSON patient record (representing a simplified HL7 v2 payload) into a FULLY VALID PH Core HL7 FHIR R4 Transaction Bundle.

The output FHIR Bundle MUST contain:
1. **Patient**:
   - name.given[0]: patient_fname
   - name.given[1]: patient_mname
   - name.family: patient_lname
   - identifier: PhilHealth (system: "https://www.philhealth.gov.ph/memberid") from philhealth_no
   - telecom[0]: phone (system: "phone", value: contact_no)
   - address: line[0]: address_street, line[1]: address_barangay, city: address_city, district: address_province, postalCode: address_zip
   - gender: M=male, F=female
   - birthDate: dob
2. **Encounter**:
   - status: "finished", class: "AMB"
   - serviceProvider.display: referring_facility_name
   - participant[0].individual.display: referring_physician
   - reasonCode[0].text: referral_reason or chief_complaint
   - priority: ROUTINE/URGENT/EMERGENCY
3. **Observation** (Create a resource for EACH valid numeric vital sign in the "vitals" object using these LOINC codes):
   - bp_systolic: 8480-6 (Systolic blood pressure)
   - bp_diastolic: 8462-4 (Diastolic blood pressure)
   - heart_rate: 8867-4 (Heart rate)
   - temperature: 8310-5 (Body temperature)
   - respiratory_rate: 9279-1 (Respiratory rate)
   - oxygen_saturation: 2708-6 (Oxygen saturation)
   - weight_kg: 29463-7 (Body weight)
   - height_cm: 8302-2 (Body height)
   *Ensure each Observation includes valueQuantity with the numeric value and appropriate unit.*
4. **Condition**:
   - code: ICD-10 coding from diagnosis_code
   - note[0].text: chief_complaint

Bundle: type "transaction", fullUrl using "urn:uuid:" format, request with method "POST".
Output ONLY valid JSON. No markdown, no code fences, no explanation.`;

const FHIR_TO_HL7V2_PROMPT = `You are a healthcare data transformation engine for the Philippine Local Health Information Exchange (LHIE).

Your task: Convert the following PH Core HL7 FHIR R4 Bundle into a flat JSON format compatible with HL7 v2 systems.

Extract data from the FHIR Bundle resources (Patient, Encounter, Observation, Condition) and map them to this EXACT structure:

{
  "patient_fname": "from Patient.name[0].given[0]",
  "patient_lname": "from Patient.name[0].family",
  "patient_mname": "from Patient.name[0].given[1] or empty string",
  "patient_suffix": "from Patient.name[0].suffix[0] or empty string",
  "dob": "Patient.birthDate in YYYY-MM-DD",
  "sex": "M or F from Patient.gender (male=M, female=F)",
  "civil_status": "S/M/W/D from Patient.maritalStatus",
  "philhealth_no": "from Patient.identifier where system contains philhealth",
  "contact_no": "from Patient.telecom where system is phone",
  "address_street": "from Patient.address[0].line[0]",
  "address_barangay": "from Patient.address[0].line[1] or empty",
  "address_city": "from Patient.address[0].city",
  "address_province": "from Patient.address[0].district",
  "address_zip": "from Patient.address[0].postalCode",
  "vitals": {
    "bp_systolic": "number from BP component LOINC 8480-6",
    "bp_diastolic": "number from BP component LOINC 8462-4",
    "heart_rate": "number from LOINC 8867-4",
    "temperature": "number from LOINC 8310-5",
    "respiratory_rate": "number from LOINC 9279-1",
    "oxygen_saturation": "number from LOINC 2708-6 or null",
    "weight_kg": "number from LOINC 29463-7",
    "height_cm": "number from LOINC 8302-2"
  },
  "chief_complaint": "from Condition.note or Encounter.reasonCode",
  "diagnosis_code": "ICD-10 code from Condition.code.coding",
  "diagnosis_desc": "display from Condition.code",
  "diagnosis_type": "admitting/final/working from Condition.verificationStatus",
  "referring_facility_code": "from Encounter.serviceProvider or generate",
  "referring_facility_name": "from Encounter.serviceProvider display",
  "referring_physician": "from Encounter.participant display",
  "referring_physician_license": "from identifier or empty",
  "referral_reason": "from Encounter.reasonCode text",
  "priority": "ROUTINE/URGENT/EMERGENCY from Encounter.priority"
}

All numeric vitals must be numbers, not strings. Missing fields should use empty string or 0.
Output ONLY valid JSON. No markdown, no code fences, no explanation.`;


// ============================================
// Format type definitions
// ============================================
export type DataFormat = 'HL7V2' | 'FHIR_R4';
export type TransformDirection =
  | 'HL7V2_TO_FHIR_R4'
  | 'FHIR_R4_TO_HL7V2'
  // Legacy aliases
  | 'IHOMIS_TO_FHIR'
  | 'FHIR_TO_IHOMIS';

function getPromptForDirection(direction: TransformDirection): string {
  switch (direction) {
    case 'HL7V2_TO_FHIR_R4':
    case 'IHOMIS_TO_FHIR':
      return HL7V2_TO_FHIR_PROMPT;
    case 'FHIR_R4_TO_HL7V2':
    case 'FHIR_TO_IHOMIS':
      return FHIR_TO_HL7V2_PROMPT;
    default:
      return HL7V2_TO_FHIR_PROMPT;
  }
}

/**
 * Determine the transformation direction from source and destination formats.
 */
export function getTransformDirection(sourceFormat: DataFormat, destFormat: DataFormat): TransformDirection {
  const key = `${sourceFormat}_TO_${destFormat}`;
  const validDirections: Record<string, TransformDirection> = {
    'HL7V2_TO_FHIR_R4': 'HL7V2_TO_FHIR_R4',
    'FHIR_R4_TO_HL7V2': 'FHIR_R4_TO_HL7V2',
  };
  return validDirections[key] || 'HL7V2_TO_FHIR_R4';
}

/**
 * Transform data using AI with automatic model fallback juggling.
 * If Gemini hits quota, it instantly falls back to Groq LPU models.
 */
export async function transformWithAI(
  payload: unknown,
  direction: TransformDirection
): Promise<{ success: boolean; data: Record<string, unknown> | null; error: string | null; usedModel?: string }> {
  const systemPrompt = getPromptForDirection(direction);

  const inputData = typeof payload === 'string'
    ? payload
    : JSON.stringify(payload, null, 2);

  var prompt = systemPrompt + '\n\n## Code Flow Diagram\n```mermaid
graph TD
    A[API Layer] -->|Routes to| B[Provide Health Data Orchestrator]
    B -->|Validates| C[Privacy Consent Gatekeeper]
    B -->|Translates| D[Semantic AI Translator]
    D <-->|Inference| E[Local Gemma 4 LLM Engine]
    B -->|Fallback| F[Deterministic Syntactic Fallback]
    B -->|Checks Schema| G[Validate Schema & Completeness]
    B -->|Logs| H[Sanitized Transaction Logger]
    B -->|Quarantines| I[Quarantine Queue Manager]
    H --> J[(MongoDB Database)]
    I --> J

    classDef default fill:#f9f9f9,stroke:#333,stroke-width:2px;
```\n\n\nInput Data:\n' + inputData;

  // Deduplicate model list while preserving order
  var models = MODEL_FALLBACKS.filter(function(v, i, a) { return a.findIndex(function(t) { return t.model === v.model; }) === i; });

  for (var idx = 0; idx < models.length; idx++) {
    var provider = models[idx].provider;
    var modelName = models[idx].model;
    try {
      console.log('[AI] Trying ' + provider + ' model: ' + modelName + ' for ' + direction + '...');

      var responseText = '';

      if (provider === 'gemini' && genAI) {
        var model = genAI.getGenerativeModel({
          model: modelName,
          generationConfig: { responseMimeType: 'application/json', temperature: 0.1 },
        });
        var result = await model.generateContent(prompt);
        responseText = result.response.text();
      } else if (provider === 'groq' && groq) {
        var completion = await groq.chat.completions.create({
          messages: [
            { role: 'system' as const, content: systemPrompt },
            { role: 'user' as const, content: 'Input Data:\n' + inputData }
          ],
          model: modelName,
          temperature: 0.1,
          response_format: { type: 'json_object' as const },
        });
        responseText = completion.choices[0]?.message?.content || '';
      } else {
        console.warn('[AI] Provider ' + provider + ' not configured (missing API key)');
        continue;
      }

      if (!responseText) throw new Error('Empty response');

      var parsedData = JSON.parse(responseText);
      console.log('[AI] Transformation successful using ' + provider + ' (' + modelName + ')');
      return { success: true, data: parsedData, error: null, usedModel: modelName };

    } catch (error) {
      var errorMessage = error instanceof Error ? error.message : 'Unknown error';
      console.warn('[AI] ' + provider + ' (' + modelName + ') failed: ' + errorMessage + '. Juggling to next...');
      continue;
    }
  }

  // All models exhausted
  return {
    success: false,
    data: null,
    error: 'All AI models (Gemini and Groq) exhausted or failed. Check API keys or wait for quota reset.',
  };
}

```

### `<<component>> Sanitized Transaction Logger [audit_logger.go]`
**File:** [`apps/adapt-ipaas/src/app/api/ingest/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/ingest/route.ts) — Lines 221–231

The audit logging happens at the final step of the pipeline when the transaction status and transformed payload are written.

```typescript
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
```

### `<<component>> Quarantine Queue Manager [quarantine.go]`
**Same as Section 3 — Decline API:** [`api/decline/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/decline/route.ts) (74 lines — see full code above)

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

Also triggered inline in [`api/ingest/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/ingest/route.ts) at Lines 140–157 and 162–181 when transformation or validation fails.

### `<<component>> Local Gemma 4 LLM Engine [Inference Runtime]`
**Same as Section 1:** [`lib/ai.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/lib/ai.ts)

```typescript
import { GoogleGenerativeAI } from '@google/generative-ai';
import Groq from 'groq-sdk';

const genAI = process.env.GEMINI_API_KEY ? new GoogleGenerativeAI(process.env.GEMINI_API_KEY) : null;
const groq = process.env.GROQ_API_KEY ? new Groq({ apiKey: process.env.GROQ_API_KEY }) : null;

// Model fallback chain — try Gemini first, then juggle to Groq
export const MODEL_FALLBACKS = [
  { provider: 'gemini', model: process.env.GEMINI_MODEL || 'gemini-3.1-flash-lite' },
  { provider: 'gemini', model: 'gemini-2.5-flash-lite' },
  { provider: 'gemini', model: 'gemini-2.5-flash' },
  { provider: 'groq', model: 'llama-3.3-70b-versatile' },
  { provider: 'groq', model: 'mixtral-8x7b-32768' },
  { provider: 'groq', model: 'llama3-70b-8192' }
];

// ============================================
// System Prompts for all format pairs
// ============================================

const HL7V2_TO_FHIR_PROMPT = `You are a healthcare data transformation engine for the Philippine Local Health Information Exchange (LHIE).

Your task: Convert the input flat JSON patient record (representing a simplified HL7 v2 payload) into a FULLY VALID PH Core HL7 FHIR R4 Transaction Bundle.

The output FHIR Bundle MUST contain:
1. **Patient**:
   - name.given[0]: patient_fname
   - name.given[1]: patient_mname
   - name.family: patient_lname
   - identifier: PhilHealth (system: "https://www.philhealth.gov.ph/memberid") from philhealth_no
   - telecom[0]: phone (system: "phone", value: contact_no)
   - address: line[0]: address_street, line[1]: address_barangay, city: address_city, district: address_province, postalCode: address_zip
   - gender: M=male, F=female
   - birthDate: dob
2. **Encounter**:
   - status: "finished", class: "AMB"
   - serviceProvider.display: referring_facility_name
   - participant[0].individual.display: referring_physician
   - reasonCode[0].text: referral_reason or chief_complaint
   - priority: ROUTINE/URGENT/EMERGENCY
3. **Observation** (Create a resource for EACH valid numeric vital sign in the "vitals" object using these LOINC codes):
   - bp_systolic: 8480-6 (Systolic blood pressure)
   - bp_diastolic: 8462-4 (Diastolic blood pressure)
   - heart_rate: 8867-4 (Heart rate)
   - temperature: 8310-5 (Body temperature)
   - respiratory_rate: 9279-1 (Respiratory rate)
   - oxygen_saturation: 2708-6 (Oxygen saturation)
   - weight_kg: 29463-7 (Body weight)
   - height_cm: 8302-2 (Body height)
   *Ensure each Observation includes valueQuantity with the numeric value and appropriate unit.*
4. **Condition**:
   - code: ICD-10 coding from diagnosis_code
   - note[0].text: chief_complaint

Bundle: type "transaction", fullUrl using "urn:uuid:" format, request with method "POST".
Output ONLY valid JSON. No markdown, no code fences, no explanation.`;

const FHIR_TO_HL7V2_PROMPT = `You are a healthcare data transformation engine for the Philippine Local Health Information Exchange (LHIE).

Your task: Convert the following PH Core HL7 FHIR R4 Bundle into a flat JSON format compatible with HL7 v2 systems.

Extract data from the FHIR Bundle resources (Patient, Encounter, Observation, Condition) and map them to this EXACT structure:

{
  "patient_fname": "from Patient.name[0].given[0]",
  "patient_lname": "from Patient.name[0].family",
  "patient_mname": "from Patient.name[0].given[1] or empty string",
  "patient_suffix": "from Patient.name[0].suffix[0] or empty string",
  "dob": "Patient.birthDate in YYYY-MM-DD",
  "sex": "M or F from Patient.gender (male=M, female=F)",
  "civil_status": "S/M/W/D from Patient.maritalStatus",
  "philhealth_no": "from Patient.identifier where system contains philhealth",
  "contact_no": "from Patient.telecom where system is phone",
  "address_street": "from Patient.address[0].line[0]",
  "address_barangay": "from Patient.address[0].line[1] or empty",
  "address_city": "from Patient.address[0].city",
  "address_province": "from Patient.address[0].district",
  "address_zip": "from Patient.address[0].postalCode",
  "vitals": {
    "bp_systolic": "number from BP component LOINC 8480-6",
    "bp_diastolic": "number from BP component LOINC 8462-4",
    "heart_rate": "number from LOINC 8867-4",
    "temperature": "number from LOINC 8310-5",
    "respiratory_rate": "number from LOINC 9279-1",
    "oxygen_saturation": "number from LOINC 2708-6 or null",
    "weight_kg": "number from LOINC 29463-7",
    "height_cm": "number from LOINC 8302-2"
  },
  "chief_complaint": "from Condition.note or Encounter.reasonCode",
  "diagnosis_code": "ICD-10 code from Condition.code.coding",
  "diagnosis_desc": "display from Condition.code",
  "diagnosis_type": "admitting/final/working from Condition.verificationStatus",
  "referring_facility_code": "from Encounter.serviceProvider or generate",
  "referring_facility_name": "from Encounter.serviceProvider display",
  "referring_physician": "from Encounter.participant display",
  "referring_physician_license": "from identifier or empty",
  "referral_reason": "from Encounter.reasonCode text",
  "priority": "ROUTINE/URGENT/EMERGENCY from Encounter.priority"
}

All numeric vitals must be numbers, not strings. Missing fields should use empty string or 0.
Output ONLY valid JSON. No markdown, no code fences, no explanation.`;


// ============================================
// Format type definitions
// ============================================
export type DataFormat = 'HL7V2' | 'FHIR_R4';
export type TransformDirection =
  | 'HL7V2_TO_FHIR_R4'
  | 'FHIR_R4_TO_HL7V2'
  // Legacy aliases
  | 'IHOMIS_TO_FHIR'
  | 'FHIR_TO_IHOMIS';

function getPromptForDirection(direction: TransformDirection): string {
  switch (direction) {
    case 'HL7V2_TO_FHIR_R4':
    case 'IHOMIS_TO_FHIR':
      return HL7V2_TO_FHIR_PROMPT;
    case 'FHIR_R4_TO_HL7V2':
    case 'FHIR_TO_IHOMIS':
      return FHIR_TO_HL7V2_PROMPT;
    default:
      return HL7V2_TO_FHIR_PROMPT;
  }
}

/**
 * Determine the transformation direction from source and destination formats.
 */
export function getTransformDirection(sourceFormat: DataFormat, destFormat: DataFormat): TransformDirection {
  const key = `${sourceFormat}_TO_${destFormat}`;
  const validDirections: Record<string, TransformDirection> = {
    'HL7V2_TO_FHIR_R4': 'HL7V2_TO_FHIR_R4',
    'FHIR_R4_TO_HL7V2': 'FHIR_R4_TO_HL7V2',
  };
  return validDirections[key] || 'HL7V2_TO_FHIR_R4';
}

/**
 * Transform data using AI with automatic model fallback juggling.
 * If Gemini hits quota, it instantly falls back to Groq LPU models.
 */
export async function transformWithAI(
  payload: unknown,
  direction: TransformDirection
): Promise<{ success: boolean; data: Record<string, unknown> | null; error: string | null; usedModel?: string }> {
  const systemPrompt = getPromptForDirection(direction);

  const inputData = typeof payload === 'string'
    ? payload
    : JSON.stringify(payload, null, 2);

  var prompt = systemPrompt + '\n\nInput Data:\n' + inputData;

  // Deduplicate model list while preserving order
  var models = MODEL_FALLBACKS.filter(function(v, i, a) { return a.findIndex(function(t) { return t.model === v.model; }) === i; });

  for (var idx = 0; idx < models.length; idx++) {
    var provider = models[idx].provider;
    var modelName = models[idx].model;
    try {
      console.log('[AI] Trying ' + provider + ' model: ' + modelName + ' for ' + direction + '...');

      var responseText = '';

      if (provider === 'gemini' && genAI) {
        var model = genAI.getGenerativeModel({
          model: modelName,
          generationConfig: { responseMimeType: 'application/json', temperature: 0.1 },
        });
        var result = await model.generateContent(prompt);
        responseText = result.response.text();
      } else if (provider === 'groq' && groq) {
        var completion = await groq.chat.completions.create({
          messages: [
            { role: 'system' as const, content: systemPrompt },
            { role: 'user' as const, content: 'Input Data:\n' + inputData }
          ],
          model: modelName,
          temperature: 0.1,
          response_format: { type: 'json_object' as const },
        });
        responseText = completion.choices[0]?.message?.content || '';
      } else {
        console.warn('[AI] Provider ' + provider + ' not configured (missing API key)');
        continue;
      }

      if (!responseText) throw new Error('Empty response');

      var parsedData = JSON.parse(responseText);
      console.log('[AI] Transformation successful using ' + provider + ' (' + modelName + ')');
      return { success: true, data: parsedData, error: null, usedModel: modelName };

    } catch (error) {
      var errorMessage = error instanceof Error ? error.message : 'Unknown error';
      console.warn('[AI] ' + provider + ' (' + modelName + ') failed: ' + errorMessage + '. Juggling to next...');
      continue;
    }
  }

  // All models exhausted
  return {
    success: false,
    data: null,
    error: 'All AI models (Gemini and Groq) exhausted or failed. Check API keys or wait for quota reset.',
  };
}

```

### `<<component>> MongoDB Database [Staging & Audit]`
**Same as Section 1:** [`lib/supabase.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/lib/supabase.ts)

```typescript
import { createClient, SupabaseClient } from '@supabase/supabase-js';

const supabaseUrl = process.env.NEXT_PUBLIC_SUPABASE_URL || '';
const supabaseAnonKey = process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY || '';

let _supabase: SupabaseClient | null = null;

function getSupabase(): SupabaseClient {
  if (!_supabase) {
    if (!supabaseUrl || !supabaseAnonKey || supabaseUrl.includes('your-')) {
      throw new Error('Supabase credentials not configured. Update .env.local with real values.');
    }
    _supabase = createClient(supabaseUrl, supabaseAnonKey);
  }
  return _supabase;
}

// eslint-disable-next-line @typescript-eslint/no-explicit-any
export const supabase = new Proxy({} as SupabaseClient, {
  get: (_, prop) => {
    const client = getSupabase();
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    const value = (client as any)[prop as string];
    return typeof value === 'function' ? value.bind(client) : value;
  },
});

export const supabaseAdmin = supabase;

```

---