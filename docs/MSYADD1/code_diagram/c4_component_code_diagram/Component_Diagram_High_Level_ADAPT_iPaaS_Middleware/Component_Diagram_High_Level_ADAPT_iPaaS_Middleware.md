# Component Diagram (High Level) for ADAPT iPaaS Middleware

| C4 Component | Implementation File | Description |
| :--- | :--- | :--- |
| 📊 **Admin Dashboard** | [`src/app/page.tsx`](./src/app/page.tsx) | The WAH4PCE Admin Dashboard UI (React). |
| 📥 **Ingest API** | [`src/app/api/ingest/route.ts`](./src/app/api/ingest/route.ts) | Standardized ingest API endpoints. |
| 🚫 **Decline API** | [`src/app/api/decline/route.ts`](./src/app/api/decline/route.ts) | API for isolating malformed or declined payloads. |
| 📤 **Request API** | [`src/app/api/request/route.ts`](./src/app/api/request/route.ts) | Endpoints to query and negotiate specific health records. |
| 📈 **Metrics Service** | [`src/app/api/metrics/route.ts`](./src/app/api/metrics/route.ts) | Service returning telemetry (throughput, success rates). |
| 🤖 **AI Transformation Service** | [`src/lib/ai.ts`](./src/lib/ai.ts) | The service executing system prompts for data mapping. |
| 🛡️ **Deterministic Fallback** | [`src/lib/mapping-calc.ts`](./src/lib/mapping-calc.ts) | Service providing rule-based transformations. |
| ✅ **Validation Service** | [`src/lib/validator.ts`](./src/lib/validator.ts) | Service for structural integrity checks. |
| 📋 **Transaction Log Service** | [`src/app/api/transactions/route.ts`](./src/app/api/transactions/route.ts) | API service returning sanitized transaction histories. |
| 🧠 **Local LLM Engine** | [`src/lib/ai.ts`](./src/lib/ai.ts) | AI Inference Engine. |
| 💾 **Staging & Audit Database** | [`src/lib/supabase.ts`](./src/lib/supabase.ts) | The persistence layer. |

### `<<component>> Admin Dashboard [Next.js UI]`
**Same as Section 1:** [`page.tsx`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/page.tsx) (256 lines — full dashboard with metrics, charts, and transaction table)

```tsx
'use client';
import { useEffect, useState, useMemo } from 'react';
import Sidebar from '@/components/Sidebar';
import { calculateMappingPercentage, calculateSourceFillCount } from '@/lib/mapping-calc';

async function safeFetch(url: string) {
  const res = await fetch(url);
  const text = await res.text();
  try { return JSON.parse(text); } catch { return { success: false }; }
}

interface Metrics {
  total_records: number; success_count: number; pending_count: number;
  quarantined_count: number; transforming_count: number; success_rate: number;
  org_to_wah: number; wah_to_org: number;
  hl7v2_count: number; fhir_count: number;
}

interface Transaction {
  id: string; source_system: string; destination_system: string;
  source_format: string; destination_format: string;
  status: string; created_at: string;
  raw_payload?: Record<string, unknown> | null;
  transformed_payload?: Record<string, unknown> | null;
}

export default function Dashboard() {
  const [metrics, setMetrics] = useState<Metrics | null>(null);
  const [recentTx, setRecentTx] = useState<Transaction[]>([]);
  const [loading, setLoading] = useState(true);

  const fetchData = async () => {
    const [metricsData, txData] = await Promise.all([
      safeFetch('/api/metrics'), safeFetch('/api/transactions?limit=50'),
    ]);
    if (metricsData.success) setMetrics(metricsData.metrics);
    if (txData.success) setRecentTx(txData.data || []);
    setLoading(false);
  };

  useEffect(() => { fetchData(); const i = setInterval(fetchData, 10000); return () => clearInterval(i); }, []);

  // Pre-compute mapping percentages for all transactions
  const txMappings = useMemo(() => {
    const map: Record<string, { src: number; dest: number; srcFilled: number; srcTotal: number; destFilled: number; destTotal: number }> = {};
    for (const tx of recentTx) {
      if (tx.status !== 'SUCCESS') {
        map[tx.id] = { src: 0, dest: 0, srcFilled: 0, srcTotal: 0, destFilled: 0, destTotal: 0 };
        continue;
      }
      const srcResult = calculateSourceFillCount(tx.raw_payload || null, tx.source_system);
      const destResult = calculateMappingPercentage(tx.transformed_payload || null, tx.destination_system);
      map[tx.id] = {
        src: srcResult.percentage,
        dest: destResult.percentage,
        srcFilled: srcResult.filledFields,
        srcTotal: srcResult.totalFields,
        destFilled: destResult.filledFields,
        destTotal: destResult.totalFields,
      };
    }
    return map;
  }, [recentTx]);

  // Average mapping % across successful transactions
  const avgMapping = useMemo(() => {
    const successTx = recentTx.filter(tx => tx.status === 'SUCCESS' && txMappings[tx.id]?.destTotal > 0);
    if (successTx.length === 0) return 0;
    const sum = successTx.reduce((acc, tx) => acc + (txMappings[tx.id]?.dest || 0), 0);
    return Number((sum / successTx.length).toFixed(1));
  }, [recentTx, txMappings]);

  const statusStyle = (s: string) => {
    const m: Record<string, { bg: string; color: string }> = {
      SUCCESS: { bg: 'rgba(5,150,105,0.08)', color: '#059669' },
      PENDING: { bg: 'rgba(217,119,6,0.08)', color: '#d97706' },
      TRANSFORMING: { bg: 'rgba(37,99,235,0.08)', color: '#2563eb' },
      QUARANTINED: { bg: 'rgba(220,38,38,0.08)', color: '#dc2626' },
    };
    return m[s] || m.PENDING;
  };

  const formatBadgeStyle = (fmt: string) => {
    const m: Record<string, { bg: string; color: string }> = {
      HL7V2: { bg: 'rgba(59,130,246,0.08)', color: '#3b82f6' },
      FHIR_R4: { bg: 'rgba(16,185,129,0.08)', color: '#10b981' },
    };
    return m[fmt] || m.HL7V2;
  };

  const formatLabel = (fmt: string) => {
    const m: Record<string, string> = { HL7V2: 'HL7v2', FHIR_R4: 'FHIR R4' };
    return m[fmt] || fmt;
  };

  // Color for mapping percentage
  const pctColor = (pct: number) => {
    if (pct >= 85) return '#059669'; // green
    if (pct >= 60) return '#d97706'; // amber
    return '#dc2626'; // red
  };

  const pctBg = (pct: number) => {
    if (pct >= 85) return 'rgba(5,150,105,0.08)';
    if (pct >= 60) return 'rgba(217,119,6,0.08)';
    return 'rgba(220,38,38,0.08)';
  };

  return (
    <>
      <Sidebar />
      <main className="flex-1 p-6 overflow-auto">
        <div className="mb-6">
          <h1 className="text-lg font-semibold">Dashboard</h1>
          <p className="text-sm" style={{ color: 'var(--color-text-secondary)' }}>Real-time monitoring of multi-format health data transformations</p>
        </div>

        {loading ? (
          <div className="flex items-center justify-center h-48">
            <div className="w-6 h-6 border-2 border-t-transparent rounded-full animate-spin" style={{ borderColor: 'var(--color-accent-bright)' }} />
          </div>
        ) : (
          <>
            {/* Main Metrics */}
            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-5 gap-4 mb-6">
              {[
                { label: 'Total Records', value: metrics?.total_records || 0, color: '#8b5cf6', icon: <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5"><path d="M21 12a9 9 0 11-6.22-8.56"/></svg> },
                { label: 'Success Rate', value: `${metrics?.success_rate || 0}%`, color: '#059669', icon: <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5"><polyline points="20 6 9 17 4 12"/></svg> },
                { label: 'Avg Mapping', value: `${avgMapping}%`, color: pctColor(avgMapping), icon: <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5"><path d="M22 12h-4l-3 9L9 3l-3 9H2"/></svg> },
                { label: 'Pending', value: (metrics?.pending_count || 0) + (metrics?.transforming_count || 0), color: '#d97706', icon: <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5"><circle cx="12" cy="12" r="10"/><polyline points="12 6 12 12 16 14"/></svg> },
                { label: 'Quarantined', value: metrics?.quarantined_count || 0, color: '#dc2626', icon: <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5"><path d="M10.29 3.86L1.82 18a2 2 0 001.71 3h16.94a2 2 0 001.71-3L13.71 3.86a2 2 0 00-3.42 0z"/><line x1="12" y1="9" x2="12" y2="13"/><line x1="12" y1="17" x2="12.01" y2="17"/></svg> },
              ].map(m => (
                <div key={m.label} className="ipaas-card p-5" style={{ borderLeft: `3px solid ${m.color}` }}>
                  <div className="flex items-center gap-2 mb-2" style={{ color: m.color }}>{m.icon}<span className="text-xs font-medium uppercase tracking-wide" style={{ color: 'var(--color-text-muted)' }}>{m.label}</span></div>
                  <p className="text-2xl font-bold">{m.value}</p>
                </div>
              ))}
            </div>

            {/* Direction + Format Stats */}
            <div className="grid grid-cols-1 lg:grid-cols-2 gap-4 mb-6">
              {/* Direction Cards */}
              <div className="ipaas-card p-5">
                <div className="flex items-center gap-2 mb-3">
                  <div className="w-7 h-7 rounded-md flex items-center justify-center" style={{ background: 'rgba(37,99,235,0.08)' }}>
                    <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="#2563eb" strokeWidth="1.5"><path d="M5 12h14M12 5l7 7-7 7"/></svg>
                  </div>
                  <div><p className="text-sm font-medium">Org → WAH</p><p className="text-xs" style={{ color: 'var(--color-text-muted)' }}>HL7v2 → FHIR R4</p></div>
                </div>
                <p className="text-3xl font-bold" style={{ color: '#2563eb' }}>{metrics?.org_to_wah || 0}</p>
                <p className="text-xs mt-1" style={{ color: 'var(--color-text-muted)' }}>records transformed</p>
              </div>
              <div className="ipaas-card p-5">
                <div className="flex items-center gap-2 mb-3">
                  <div className="w-7 h-7 rounded-md flex items-center justify-center" style={{ background: 'rgba(139,92,246,0.08)' }}>
                    <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="#8b5cf6" strokeWidth="1.5"><path d="M19 12H5M12 19l-7-7 7-7"/></svg>
                  </div>
                  <div><p className="text-sm font-medium">WAH → Org</p><p className="text-xs" style={{ color: 'var(--color-text-muted)' }}>FHIR R4 → HL7v2</p></div>
                </div>
                <p className="text-3xl font-bold" style={{ color: '#8b5cf6' }}>{metrics?.wah_to_org || 0}</p>
                <p className="text-xs mt-1" style={{ color: 'var(--color-text-muted)' }}>records transformed</p>
              </div>
            </div>

            {/* Format Breakdown */}
            <div className="grid grid-cols-2 gap-4 mb-6">
              {[
                { label: 'HL7 v2.x', count: metrics?.hl7v2_count || 0, color: '#3b82f6', bg: 'rgba(59,130,246,0.08)' },
                { label: 'FHIR R4', count: metrics?.fhir_count || 0, color: '#10b981', bg: 'rgba(16,185,129,0.08)' },
              ].map(f => (
                <div key={f.label} className="ipaas-card p-4">
                  <div className="flex items-center gap-2 mb-2">
                    <span className="text-xs font-bold px-2 py-0.5 rounded" style={{ background: f.bg, color: f.color }}>{f.label}</span>
                  </div>
                  <p className="text-xl font-bold" style={{ color: f.color }}>{f.count}</p>
                  <p className="text-xs" style={{ color: 'var(--color-text-muted)' }}>transformations</p>
                </div>
              ))}
            </div>

            {/* Recent Activity */}
            <div className="ipaas-card overflow-hidden">
              <div className="px-5 py-4 border-b" style={{ borderColor: 'var(--color-border)' }}>
                <h2 className="text-sm font-semibold">Recent Activity</h2>
                <p className="text-xs" style={{ color: 'var(--color-text-muted)' }}>All transactions ({recentTx.length})</p>
              </div>
              {recentTx.length === 0 ? (
                <div className="p-10 text-center">
                  <svg width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="var(--color-text-muted)" strokeWidth="1" className="mx-auto mb-3"><path d="M14 2H6a2 2 0 00-2 2v16a2 2 0 002 2h12a2 2 0 002-2V8z"/><polyline points="14 2 14 8 20 8"/></svg>
                  <p className="text-sm" style={{ color: 'var(--color-text-muted)' }}>No transactions yet. Send data from an organization or WAH.</p>
                </div>
              ) : (
              <div style={{ maxHeight: '360px', overflowY: 'auto' }}>
                <table className="data-table">
                  <thead><tr><th>Transaction ID</th><th>Direction</th><th>Formats</th><th>Sent</th><th>Received</th><th>Status</th><th>Date</th></tr></thead>
                  <tbody>
                    {recentTx.map(tx => {
                      const st = statusStyle(tx.status);
                      const srcFmt = formatBadgeStyle(tx.source_format);
                      const dstFmt = formatBadgeStyle(tx.destination_format);
                      const mapping = txMappings[tx.id];
                      const hasMappingData = tx.status === 'SUCCESS' && mapping && mapping.destTotal > 0;
                      return (
                        <tr key={tx.id} onClick={() => window.location.href = `/mapper?id=${tx.id}`} style={{ cursor: 'pointer' }}>
                          <td className="font-mono text-xs" style={{ color: 'var(--color-accent-bright)' }}>{tx.id.slice(0, 8)}...</td>
                          <td className="text-sm">{tx.source_system} → {tx.destination_system}</td>
                          <td>
                            <span className="text-[10px] font-bold px-1.5 py-0.5 rounded mr-1" style={{ background: srcFmt.bg, color: srcFmt.color }}>{formatLabel(tx.source_format)}</span>
                            <span style={{ color: 'var(--color-text-muted)' }}>→</span>
                            <span className="text-[10px] font-bold px-1.5 py-0.5 rounded ml-1" style={{ background: dstFmt.bg, color: dstFmt.color }}>{formatLabel(tx.destination_format)}</span>
                          </td>
                          {/* Source mapping % */}
                          <td>
                            {hasMappingData ? (
                              <div className="flex items-center gap-1.5">
                                <div className="w-14 h-1.5 rounded-full overflow-hidden" style={{ background: 'rgba(0,0,0,0.06)' }}>
                                  <div className="h-full rounded-full transition-all" style={{ width: `${mapping.src}%`, background: pctColor(mapping.src) }} />
                                </div>
                                <span className="text-[10px] font-bold" style={{ color: pctColor(mapping.src) }}>{mapping.src}%</span>
                                <span className="text-[9px]" style={{ color: 'var(--color-text-muted)' }}>{mapping.srcFilled}/{mapping.srcTotal}</span>
                              </div>
                            ) : (
                              <span className="text-[10px]" style={{ color: 'var(--color-text-muted)' }}>—</span>
                            )}
                          </td>
                          {/* Destination mapping % */}
                          <td>
                            {hasMappingData ? (
                              <div className="flex items-center gap-1.5">
                                <div className="w-14 h-1.5 rounded-full overflow-hidden" style={{ background: 'rgba(0,0,0,0.06)' }}>
                                  <div className="h-full rounded-full transition-all" style={{ width: `${mapping.dest}%`, background: pctColor(mapping.dest) }} />
                                </div>
                                <span className="text-[10px] font-bold" style={{ color: pctColor(mapping.dest) }}>{mapping.dest}%</span>
                                <span className="text-[9px]" style={{ color: 'var(--color-text-muted)' }}>{mapping.destFilled}/{mapping.destTotal}</span>
                              </div>
                            ) : (
                              <span className="text-[10px]" style={{ color: 'var(--color-text-muted)' }}>—</span>
                            )}
                          </td>
                          <td><span className="ipaas-badge" style={{ background: st.bg, color: st.color }}>{tx.status}</span></td>
                          <td className="text-xs" style={{ color: 'var(--color-text-muted)' }}>{new Date(tx.created_at).toLocaleString()}</td>
                        </tr>
                      );
                    })}
                  </tbody>
                </table>
              </div>
              )}
            </div>
          </>
        )}
      </main>
    </>
  );
}

```

### `<<component>> Ingest API [API]`
**Same as Section 2:** [`api/ingest/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/ingest/route.ts) (250 lines — full pipeline)

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

### `<<component>> Decline API [API]`
**File:** [`apps/adapt-ipaas/src/app/api/decline/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/decline/route.ts) (79 lines)

Handles bidirectional declines (WAH → Org or Org → WAH), quarantines the transaction, and forwards to the appropriate webhook.

```ts
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
      const { data, error } = await supabaseAdmin
        .from('adapt_transaction_logs')
        .update({
          status: 'QUARANTINED',
          error_message: message || 'Request declined by source organization',
        })
        .eq('id', ipaas_transaction_id)
        .select();

      if (error || !data || data.length === 0) {
        return NextResponse.json({ success: false, message: 'Transaction record not found' }, { status: 404 });
      }

      console.log(`[System (Internal)] Updated transaction ${ipaas_transaction_id} to QUARANTINED (declined)`);
    }

    // Forward decline notification to the appropriate system
    if (destination_system === 'WAH') {
      // Org declined WAH's request → notify WAH by updating its local outbound request
      // WAH polls its own outbound-requests, so we just update the iPaaS transaction.
      // WAH's request-data page polls and will see the QUARANTINED status.
      console.log(`[System (Internal)] Decline forwarded for WAH's outbound request ${request_id}`);
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
        console.error('[External Microservice (ADAPT)] Forward error:', err);
        return NextResponse.json({ success: true, message: 'Decline recorded (webhook forward failed)' });
      }
    }
  } catch (error) {
    console.error('[System (Internal)] Error:', error);
    return NextResponse.json({ success: false, message: 'Internal server error' }, { status: 500 });
  }
}
```

### `<<component>> Request API [API]`
**File:** [`apps/adapt-ipaas/src/app/api/request/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/request/route.ts) (156 lines)

Handles bidirectional data requests. Determines direction, creates a PENDING transaction, and forwards to WAH or Portal.

```ts
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

    console.log(`[External Microservice (ADAPT)] ${requesting_org} requesting data from ${isWAHRequesting ? target_org : 'WAH'} (PhilHealth: ${philhealth_no || 'N/A'}, Name: ${patient_name || 'N/A'})`);

    // --- Log in iPaaS transaction table as PENDING ---
    const { data: txRecord } = await supabaseAdmin
      .from('adapt_transaction_logs')
      .insert({
        source_system: sourceSystem,
        destination_system: destSystem,
        source_format: srcFormat,
        destination_format: dstFormat,
        raw_payload: { request_id, philhealth_no, patient_name, direction: isWAHRequesting ? 'wah_to_ihomis' : 'ihomis_to_wah' },
        status: 'PENDING',
      })
      .select()
      .single();

    console.log(`[System (Internal)] Created transaction ${txRecord?.id} as PENDING`);

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
        console.error('[External Microservice (ADAPT)] Failed to forward to Portal:', err);
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
        console.error('[External Microservice (ADAPT)] Failed to forward to WAH:', err);
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
    console.error('[System (Internal)] Error:', error);
    return NextResponse.json(
      { success: false, message: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

### `<<component>> Metrics Service [Service]`
**File:** [`apps/adapt-ipaas/src/app/api/metrics/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/metrics/route.ts) (68 lines)

Aggregates 9 parallel Supabase count queries for dashboard metrics.

```ts
import { NextResponse } from 'next/server';
import { supabaseAdmin } from '@/lib/supabase';

export const dynamic = 'force-dynamic';

/**
 * GET /api/metrics
 * Returns aggregated dashboard metrics
 * Updated to support dynamic org names and format tracking
 */
export async function GET() {
  try {
    // Fetch all counts in parallel
    const [totalRes, successRes, pendingRes, quarantinedRes, transformingRes, toWahRes, fromWahRes] =
      await Promise.all([
        supabaseAdmin.from('adapt_transaction_logs').select('*', { count: 'exact', head: true }),
        supabaseAdmin.from('adapt_transaction_logs').select('*', { count: 'exact', head: true }).eq('status', 'SUCCESS'),
        supabaseAdmin.from('adapt_transaction_logs').select('*', { count: 'exact', head: true }).eq('status', 'PENDING'),
        supabaseAdmin.from('adapt_transaction_logs').select('*', { count: 'exact', head: true }).eq('status', 'QUARANTINED'),
        supabaseAdmin.from('adapt_transaction_logs').select('*', { count: 'exact', head: true }).eq('status', 'TRANSFORMING'),
        supabaseAdmin.from('adapt_transaction_logs').select('*', { count: 'exact', head: true }).eq('destination_system', 'WAH'),
        supabaseAdmin.from('adapt_transaction_logs').select('*', { count: 'exact', head: true }).eq('source_system', 'WAH'),
      ]);

    const total = totalRes.count || 0;
    const success = successRes.count || 0;
    const pending = pendingRes.count || 0;
    const quarantined = quarantinedRes.count || 0;
    const transforming = transformingRes.count || 0;
    const toWah = toWahRes.count || 0;
    const fromWah = fromWahRes.count || 0;

    const successRate = total > 0 ? Number(((success / total) * 100).toFixed(1)) : 0;

    // Fetch format-specific stats
    const [hl7v2Res, fhirRes] = await Promise.all([
      supabaseAdmin.from('adapt_transaction_logs').select('*', { count: 'exact', head: true }).eq('source_format', 'HL7V2'),
      supabaseAdmin.from('adapt_transaction_logs').select('*', { count: 'exact', head: true }).eq('source_format', 'FHIR_R4'),
    ]);

    return NextResponse.json({
      success: true,
      metrics: {
        total_records: total,
        success_count: success,
        pending_count: pending,
        quarantined_count: quarantined,
        transforming_count: transforming,
        success_rate: successRate,
        // Direction stats
        ihomis_to_wah: toWah,
        wah_to_ihomis: fromWah,
        // Legacy aliases
        ihomis_to_wah: toWah,
        wah_to_ihomis: fromWah,
        // Format stats
        hl7v2_count: hl7v2Res.count || 0,
        fhir_count: fhirRes.count || 0,
      },
    });
  } catch (error) {
    console.error('[iPaaS Metrics] Error:', error);
    return NextResponse.json(
      { success: false, message: 'Failed to fetch metrics' },
      { status: 500 }
    );
  }
}
```

### `<<component>> AI Transformation Service [Service]`
**Same as Section 1:** [`lib/ai.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/lib/ai.ts) — `transformWithAI()` (211 lines)

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
    A[Admin Dashboard] -->|Fetches| B[Metrics Service]
    A -->|Views| C[Transaction Log Service]
    D[Ingest API] -->|Processes via| E[AI Transformation Service]
    F[Decline API] -->|Logs to| C
    G[Request API] -->|Logs to| C
    E <-->|Prompts| H[Local LLM Engine]
    E -->|On Failure| I[Deterministic Syntactic Fallback]
    D -->|Validates via| J[Validation Service]
    B --> K[(Staging & Audit Database)]
    C --> K
    E --> K

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

### `<<component>> Deterministic Syntactic Fallback [Service]`
**Same as Section 2:** [`lib/mapping-calc.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/lib/mapping-calc.ts) (489 lines)

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

### `<<component>> Validation Service [Service]`
**Same as Section 2:** [`lib/validator.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/lib/validator.ts) (131 lines)

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

### `<<component>> Transaction Log Service [Service]`
**File:** [`apps/adapt-ipaas/src/app/api/transactions/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/transactions/route.ts) (57 lines)

```ts
import { NextRequest, NextResponse } from 'next/server';
import { supabaseAdmin } from '@/lib/supabase';

export const dynamic = 'force-dynamic';

/**
 * GET /api/transactions
 * Fetch transaction logs with optional filtering
 * Query params: ?status=SUCCESS&source=iHOMIS&limit=50&offset=0
 */
export async function GET(request: NextRequest) {
  try {
    const { searchParams } = new URL(request.url);
    const status = searchParams.get('status');
    const source = searchParams.get('source');
    const limit = parseInt(searchParams.get('limit') || '50');
    const offset = parseInt(searchParams.get('offset') || '0');

    let query = supabaseAdmin
      .from('adapt_transaction_logs')
      .select('*', { count: 'exact' })
      .order('created_at', { ascending: false })
      .range(offset, offset + limit - 1);

    if (status) {
      query = query.eq('status', status);
    }

    if (source) {
      query = query.eq('source_system', source);
    }

    const { data, error, count } = await query;

    if (error) {
      console.error('[iPaaS Transactions] Query error:', error);
      return NextResponse.json(
        { success: false, message: error.message },
        { status: 500 }
      );
    }

    return NextResponse.json({
      success: true,
      data,
      total: count,
      limit,
      offset,
    });
  } catch (error) {
    console.error('[iPaaS Transactions] Unexpected error:', error);
    return NextResponse.json(
      { success: false, message: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

### `<<component>> Local LLM Engine [Inference Runtime]`
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

### `<<component>> Staging & Audit Database [MongoDB]`
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