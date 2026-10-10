# Component Diagram (Low Level) - UI Layer [Package]

| C4 Component | Implementation File | Description |
| :--- | :--- | :--- |
| 🧑‍💼 **WAH4PCE Admin** | *N/A* | External actor interacting via JWT authenticated session. |
| 🗺️ **App Router & Layout** | [`src/app/layout.tsx`](./src/app/layout.tsx) | The root DOM layout and layout preservation wrapper. |
| 🔑 **Auth & Session Manager** | [`src/app/api/auth/login/route.ts`](./src/app/api/auth/login/route.ts) | JWT creation via bcrypt validation. |
| 📊 **Metrics & Health Monitor** | [`src/components/MetricCard.tsx`](./src/components/MetricCard.tsx) | UI components for system health and throughput visualization. |
| 📋 **Transaction Log Viewer** | [`src/app/transactions/page.tsx`](./src/app/transactions/page.tsx) | Complex React table component for audit logs and status filters. |
| 🔍 **Payload Comparison & Unhash** | [`src/app/mapper/page.tsx`](./src/app/mapper/page.tsx) | 870+ line React component for SHA-256 hash/unhash and side-by-side HL7v2/FHIR view. |
| 🌐 **ADAPT API Client** | [`src/components/Sidebar.tsx`](./src/components/Sidebar.tsx) | Frontend client routing and `safeFetch` utilities. |
| 📦 **API Layer [Package]** | [`src/app/api/`](./src/app/api) | The backend targets for the ADAPT API client. |

### `<<person>> WAH4PCE Admin [Admin User]`
Not a code component — this is the external actor. Represented in the system through the JWT session created at login (see Auth & Scope Middleware above).

### `<<component>> App Router & Layout [Next.js App Router]`
**Same as Section 5:** [`layout.tsx`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/layout.tsx)

```tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "ADAPT iPaaS — Integration Platform Dashboard",
  description: "Intelligent Healthcare Data Integration Platform as a Service for the Philippine Local Health Information Exchange (LHIE)",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en">
      <head>
        <link
          href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700;800&display=swap"
          rel="stylesheet"
        />
      </head>
      <body className="antialiased">
        <div className="flex min-h-screen">
          {children}
        </div>
      </body>
    </html>
  );
}

```

### `<<component>> Auth & Session Manager [JWT / Session Context]`
**Same as Section 5:** [`middleware.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/middleware.ts) + [`api/auth/login/route.ts`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/auth/login/route.ts)

```typescript
import { NextResponse, type NextRequest } from 'next/server';
import { jwtVerify } from 'jose';

const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET || 'super-secret-fallback-key-replace-me-in-production');

export async function middleware(request: NextRequest) {
  const token = request.cookies.get('auth_token')?.value;
  const isAuthRoute = request.nextUrl.pathname.startsWith('/login') || request.nextUrl.pathname.startsWith('/register');
  let isValid = false;

  if (token) {
    try {
      await jwtVerify(token, JWT_SECRET);
      isValid = true;
    } catch (e) {
      isValid = false;
    }
  }

  if (!isValid && !isAuthRoute) {
    // No valid token, and they're trying to access a protected route (like /)
    const url = request.nextUrl.clone();
    url.pathname = '/login';
    return NextResponse.redirect(url);
  }

  if (isValid && isAuthRoute) {
    // User is already logged in, redirect them away from auth pages to the dashboard
    const url = request.nextUrl.clone();
    url.pathname = '/';
    return NextResponse.redirect(url);
  }

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

```typescript
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

### `<<component>> Metrics & Health Monitor [React View & Hooks]`
**File:** [`apps/adapt-ipaas/src/app/page.tsx`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/page.tsx) — Lines 109–179

The metric cards and health monitoring UI (5 main metrics + direction cards + format breakdown).

```tsx
{/* Main Metrics */}
<div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-5 gap-4 mb-6">
  {[
    { label: 'Total Records', value: metrics?.total_records || 0, color: '#8b5cf6' },
    { label: 'Success Rate', value: `${metrics?.success_rate || 0}%`, color: '#059669' },
    { label: 'Avg Mapping', value: `${avgMapping}%`, color: pctColor(avgMapping) },
    { label: 'Pending', value: (metrics?.pending_count || 0) + (metrics?.transforming_count || 0), color: '#d97706' },
    { label: 'Quarantined', value: metrics?.quarantined_count || 0, color: '#dc2626' },
  ].map(m => (
    <div key={m.label} className="ipaas-card p-5" style={{ borderLeft: `3px solid ${m.color}` }}>
      <p className="text-xs font-medium uppercase tracking-wide">{m.label}</p>
      <p className="text-2xl font-bold">{m.value}</p>
    </div>
  ))}
</div>

{/* Direction + Format Stats */}
<div className="grid grid-cols-1 lg:grid-cols-2 gap-4 mb-6">
  <div className="ipaas-card p-5">
    <p className="text-sm font-medium">Org → WAH</p>
    <p className="text-xs">HL7v2 → FHIR R4</p>
    <p className="text-3xl font-bold" style={{ color: '#2563eb' }}>{metrics?.org_to_wah || 0}</p>
  </div>
  <div className="ipaas-card p-5">
    <p className="text-sm font-medium">WAH → Org</p>
    <p className="text-xs">FHIR R4 → HL7v2</p>
    <p className="text-3xl font-bold" style={{ color: '#8b5cf6' }}>{metrics?.wah_to_org || 0}</p>
  </div>
</div>
```

Also supported by **MetricCard component:**

**File:** [`apps/adapt-ipaas/src/components/MetricCard.tsx`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/components/MetricCard.tsx) (62 lines)

```tsx
'use client';

import { useEffect, useState } from 'react';

interface MetricCardProps {
  title: string;
  value: number;
  suffix?: string;
  variant: 'purple' | 'green' | 'yellow' | 'red';
  icon: React.ReactNode;
}

export default function MetricCard({ title, value, suffix = '', variant, icon }: MetricCardProps) {
  const [displayValue, setDisplayValue] = useState(0);

  useEffect(() => {
    // Animate counter
    const duration = 800;
    const steps = 30;
    const increment = value / steps;
    let current = 0;
    let step = 0;

    const timer = setInterval(() => {
      step++;
      current = Math.min(Math.round(increment * step), value);
      setDisplayValue(current);
      if (step >= steps) clearInterval(timer);
    }, duration / steps);

    return () => clearInterval(timer);
  }, [value]);

  return (
    <div className={`glass-card p-6 metric-gradient-${variant} animate-fade-in`}>
      <div className="flex items-start justify-between mb-4">
        <div className="w-10 h-10 rounded-none flex items-center justify-center"
          style={{
            background: variant === 'purple' ? 'rgba(139, 92, 246, 0.15)' :
                         variant === 'green' ? 'rgba(16, 185, 129, 0.15)' :
                         variant === 'yellow' ? 'rgba(245, 158, 11, 0.15)' :
                         'rgba(239, 68, 68, 0.15)',
            color: variant === 'purple' ? 'var(--color-accent-purple)' :
                   variant === 'green' ? 'var(--color-success)' :
                   variant === 'yellow' ? 'var(--color-warning)' :
                   'var(--color-error)',
          }}
        >
          {icon}
        </div>
      </div>
      <div>
        <p className="text-sm font-medium mb-1" style={{ color: 'var(--color-text-secondary)' }}>
          {title}
        </p>
        <p className="text-3xl font-bold animate-count" style={{ color: 'var(--color-text-primary)' }}>
          {displayValue}{suffix}
        </p>
      </div>
    </div>
  );
}
```

### `<<component>> Transaction Log Viewer [React Table Component]`
**File:** [`apps/adapt-ipaas/src/app/transactions/page.tsx`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/transactions/page.tsx) (226 lines)

Full-page transaction log viewer with status/source filtering, pagination, mapping % bars, and auto-refresh.

```tsx
'use client';
import { useEffect, useState, useMemo } from 'react';
import Sidebar from '@/components/Sidebar';
import { calculateMappingPercentage, calculateSourceFillCount } from '@/lib/mapping-calc';

async function safeFetch(url: string) {
  const res = await fetch(url);
  const text = await res.text();
  try { return JSON.parse(text); } catch { return { success: false, data: [], total: 0 }; }
}

interface Transaction {
  id: string; source_system: string; destination_system: string;
  source_format: string; destination_format: string;
  status: string; error_message: string | null; created_at: string;
  raw_payload?: Record<string, unknown> | null;
  transformed_payload?: Record<string, unknown> | null;
}

export default function TransactionsPage() {
  const [transactions, setTransactions] = useState<Transaction[]>([]);
  const [total, setTotal] = useState(0);
  const [loading, setLoading] = useState(true);
  const [statusFilter, setStatusFilter] = useState('');
  const [sourceFilter, setSourceFilter] = useState('');
  const [page, setPage] = useState(0);
  const limit = 15;

  const fetchTransactions = async () => {
    const params = new URLSearchParams({ limit: String(limit), offset: String(page * limit) });
    if (statusFilter) params.set('status', statusFilter);
    if (sourceFilter) params.set('source', sourceFilter);
    const data = await safeFetch(`/api/transactions?${params.toString()}`);
    if (data.success) { setTransactions(data.data || []); setTotal(data.total || 0); }
    setLoading(false);
  };

  useEffect(() => { setLoading(true); fetchTransactions(); }, [statusFilter, sourceFilter, page]);
  useEffect(() => { const i = setInterval(fetchTransactions, 15000); return () => clearInterval(i); }, [statusFilter, sourceFilter, page]);

  const totalPages = Math.ceil(total / limit);

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

  const pctColor = (pct: number) => {
    if (pct >= 85) return '#059669'; // green
    if (pct >= 60) return '#d97706'; // amber
    return '#dc2626'; // red
  };

  const txMappings = useMemo(() => {
    const map: Record<string, { src: number; dest: number; srcFilled: number; srcTotal: number; destFilled: number; destTotal: number }> = {};
    for (const tx of transactions) {
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
  }, [transactions]);

  return (
    <>
      <Sidebar />
      <main className="flex-1 p-6 overflow-auto">
        <div className="mb-5">
          <h1 className="text-lg font-semibold">Transaction Logs</h1>
          <p className="text-sm" style={{ color: 'var(--color-text-secondary)' }}>Complete audit trail of all data transformations — {total} total records</p>
        </div>

        <div className="flex gap-3 mb-5 flex-wrap">
          <select value={statusFilter} onChange={e => { setStatusFilter(e.target.value); setPage(0); }}
            className="px-3 py-2 rounded-none text-xs outline-none" style={{ background: '#fff', border: '1px solid var(--color-border)', color: 'var(--color-text-primary)' }}>
            <option value="">All Statuses</option>
            <option value="SUCCESS">Success</option>
            <option value="PENDING">Pending</option>
            <option value="TRANSFORMING">Transforming</option>
            <option value="QUARANTINED">Quarantined</option>
          </select>
          <select value={sourceFilter} onChange={e => { setSourceFilter(e.target.value); setPage(0); }}
            className="px-3 py-2 rounded-none text-xs outline-none" style={{ background: '#fff', border: '1px solid var(--color-border)', color: 'var(--color-text-primary)' }}>
            <option value="">All Sources</option>
            <option value="iHOMIS">iHOMIS (DOH)</option>
            <option value="WAH">WAH Hospital</option>
          </select>
          <button onClick={() => { setLoading(true); fetchTransactions(); }} className="ipaas-btn ipaas-btn-secondary text-xs">
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5"><polyline points="23 4 23 10 17 10"/><path d="M20.49 15a9 9 0 11-2.12-9.36L23 10"/></svg>
            Refresh
          </button>
        </div>

        <div className="ipaas-card overflow-hidden mb-5">
          {loading ? (
            <div className="animate-pulse bg-white">
              <div className="h-[42px] bg-gray-50 border-b border-gray-100 flex items-center px-4">
                <div className="w-1/4 h-3 bg-gray-200 rounded-none" />
              </div>
              {[1, 2, 3, 4, 5, 6].map(i => (
                <div key={i} className="flex items-center gap-4 px-4 py-3.5 border-b border-gray-50">
                  <div className="w-24 h-3 bg-gray-200 rounded-none" />
                  <div className="w-32 h-3 bg-gray-200 rounded-none" />
                  <div className="w-20 h-4 bg-gray-200 rounded-none" />
                  <div className="w-16 h-2 bg-gray-200 rounded-none" />
                  <div className="w-16 h-2 bg-gray-200 rounded-none" />
                  <div className="w-20 h-5 bg-gray-200 rounded-none" />
                  <div className="w-24 h-2 bg-gray-200 rounded-none" />
                </div>
              ))}
            </div>
          ) : transactions.length === 0 ? (
            <div className="p-10 text-center">
              <p className="text-sm" style={{ color: 'var(--color-text-muted)' }}>No transactions found matching your filters.</p>
            </div>
          ) : (
            <table className="data-table">
              <thead><tr><th>Transaction ID</th><th>Direction</th><th>Formats</th><th>Sent</th><th>Received</th><th>Status</th><th>Date</th></tr></thead>
              <tbody>
                {transactions.map(tx => {
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
                        <span className="text-[10px] font-bold px-1.5 py-0.5 rounded-none mr-1" style={{ background: srcFmt.bg, color: srcFmt.color }}>{formatLabel(tx.source_format)}</span>
                        <span style={{ color: 'var(--color-text-muted)' }}>→</span>
                        <span className="text-[10px] font-bold px-1.5 py-0.5 rounded-none ml-1" style={{ background: dstFmt.bg, color: dstFmt.color }}>{formatLabel(tx.destination_format)}</span>
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
                      <td>
                        <div className="flex flex-col gap-1">
                          <span className="ipaas-badge self-start" style={{ background: st.bg, color: st.color }}>{tx.status}</span>
                          {tx.error_message && (
                            <span className="text-[10px] max-w-[120px] truncate" style={{ color: '#dc2626' }} title={tx.error_message}>
                              {tx.error_message}
                            </span>
                          )}
                        </div>
                      </td>
                      <td className="text-xs" style={{ color: 'var(--color-text-muted)' }}>{new Date(tx.created_at).toLocaleString()}</td>
                    </tr>
                  );
                })}
              </tbody>
            </table>
          )}
        </div>

        {totalPages > 1 && (
          <div className="flex items-center justify-between">
            <p className="text-xs" style={{ color: 'var(--color-text-muted)' }}>Page {page + 1} of {totalPages}</p>
            <div className="flex gap-2">
              <button onClick={() => setPage(Math.max(0, page - 1))} disabled={page === 0}
                className="ipaas-btn ipaas-btn-secondary text-xs disabled:opacity-30">Previous</button>
              <button onClick={() => setPage(Math.min(totalPages - 1, page + 1))} disabled={page >= totalPages - 1}
                className="ipaas-btn ipaas-btn-secondary text-xs disabled:opacity-30">Next</button>
            </div>
          </div>
        )}
      </main>
    </>
  );
}
```

### `<<component>> Payload Comparison & Unhash [React Component]`
**File:** [`apps/adapt-ipaas/src/app/mapper/page.tsx`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/mapper/page.tsx) (898 lines)

The most complex UI component. Features SHA-256 hashing/unhashing of payloads, HL7v2 pipe-delimited string parsing, FHIR Bundle resource extraction, side-by-side field-level comparison tables, and transaction selection dropdown.

```tsx
'use client';
import { Fragment, Suspense, useEffect, useState, useMemo } from 'react';
import { useSearchParams } from 'next/navigation';
import Sidebar from '@/components/Sidebar';

async function safeFetch(url: string) {
  const res = await fetch(url);
  const text = await res.text();
  try { return JSON.parse(text); } catch { return { success: false, data: [] }; }
}

// SHA-256 hash function (browser-compatible)
async function sha256(text: string): Promise<string> {
  const encoder = new TextEncoder();
  const data = encoder.encode(text);
  const hashBuffer = await crypto.subtle.digest('SHA-256', data);
  const hashArray = Array.from(new Uint8Array(hashBuffer));
  return hashArray.map(b => b.toString(16).padStart(2, '0')).join('');
}

// ─── Extract meaningful data from iHOMIS flat payloads ───
function extractHL7Data(payload: Record<string, unknown>): { category: string; label: string; value: string }[] {
  // Helper: check top-level first, then nested under `vitals`
  const vitalsObj = (payload.vitals && typeof payload.vitals === 'object') ? payload.vitals as Record<string, unknown> : null;
  const g = (k: string) => {
    if (payload[k] != null && String(payload[k]) !== '') return String(payload[k]);
    if (vitalsObj && vitalsObj[k] != null && String(vitalsObj[k]) !== '' && String(vitalsObj[k]) !== '0') return String(vitalsObj[k]);
    return '';
  };
  const rows: { category: string; label: string; value: string }[] = [];
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
  add('Patient', 'Zip Code', g('address_zip'));

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
  add('Diagnosis', 'Type', g('diagnosis_type'));
  add('Diagnosis', 'Clinical Notes', g('clinical_notes'));

  add('Referral', 'Priority', g('priority'));
  add('Referral', 'Reason', g('referral_reason'));
  add('Referral', 'Physician', g('referring_physician'));
  add('Referral', 'Physician License', g('referring_physician_license'));
  add('Referral', 'Facility', g('referring_facility_name'));

  return rows;
}

// ─── Extract meaningful data from WAH FHIR bundles ───
function extractFHIRData(payload: Record<string, unknown>): { category: string; label: string; value: string }[] {
  const rows: { category: string; label: string; value: string }[] = [];
  const add = (cat: string, label: string, val: unknown) => { if (val != null && String(val).trim()) rows.push({ category: cat, label, value: String(val).trim() }); };

  // Navigate FHIR bundle
  const entries = (payload as any)?.entry || [];
  const resources = entries.map((e: any) => e?.resource).filter(Boolean);
  if (resources.length === 0 && (payload as any)?.resourceType) {
    resources.push(payload);
  }

  let chiefComplaintFound = false;

  for (const res of resources) {
    const rt = res?.resourceType;
    if (rt === 'Patient') {
      const name = res.name?.[0] || {};
      const givenArr = name.given || [];
      add('Patient', 'Given Name', givenArr[0]); // First given name only
      add('Patient', 'Middle Name', givenArr.length > 1 ? givenArr.slice(1).join(' ') : null); // Second+ given name = middle
      add('Patient', 'Family Name', name.family);
      add('Patient', 'Suffix', name.suffix?.[0]);
      add('Patient', 'Birth Date', res.birthDate);
      add('Patient', 'Gender', res.gender);
      add('Patient', 'Marital Status', res.maritalStatus?.text || res.maritalStatus?.coding?.[0]?.display || res.maritalStatus?.coding?.[0]?.code);
      // PhilHealth
      for (const id of (res.identifier || [])) {
        if (id.system?.includes('philhealth') || id.type?.coding?.[0]?.code === 'SB') {
          add('Patient', 'PhilHealth No.', id.value);
        }
      }
      // Telecom
      for (const t of (res.telecom || [])) {
        add('Patient', 'Phone', t.value);
      }
      // Address
      const addr = res.address?.[0] || {};
      add('Patient', 'Address Line', (addr.line || []).join(', '));
      add('Patient', 'City', addr.city);
      add('Patient', 'Province/State', addr.state || addr.district);
      add('Patient', 'Postal Code', addr.postalCode);
    }
    if (rt === 'Encounter') {
      add('Encounter', 'Class', res.class?.display || res.class?.code);
      add('Encounter', 'Priority', res.priority?.coding?.[0]?.display || res.priority?.coding?.[0]?.code || res.priority?.text || res.priority);
      // Reason — check multiple paths
      const reason = res.reasonCode?.[0]?.text || res.reasonCode?.[0]?.coding?.[0]?.display || res.reason?.[0]?.concept?.text || res.reason?.[0]?.concept?.coding?.[0]?.display;
      add('Encounter', 'Reason', reason);
      // Chief complaint can also be in Encounter.reasonCode
      if (reason && !chiefComplaintFound) {
        // We'll use this as fallback for chief complaint later
      }
      // Facility — check multiple paths
      add('Encounter', 'Facility', res.serviceProvider?.display || res.serviceProvider?.reference || res.location?.[0]?.location?.display);
      // Physician — check multiple paths
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
      // Chief complaint — check note, category text, or extension
      const complaint = res.note?.[0]?.text || res.category?.[0]?.text;
      if (complaint) {
        add('Diagnosis', 'Chief Complaint', complaint);
        chiefComplaintFound = true;
      }
    }
    if (rt === 'ServiceRequest') {
      add('Referral', 'Reason', res.reasonCode?.[0]?.text || res.reasonCode?.[0]?.coding?.[0]?.display);
      add('Referral', 'Priority', res.priority);
      add('Referral', 'Requester', res.requester?.display);
    }
  }

  // If chief complaint wasn't found in Condition, try Encounter reasonCode
  if (!chiefComplaintFound) {
    for (const res of resources) {
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

// ─── Parse HL7v2 pipe-delimited string into meaningful data ───
function parseHL7v2String(hl7: string): { category: string; label: string; value: string }[] {
  const rows: { category: string; label: string; value: string }[] = [];
  const add = (cat: string, label: string, val: string | undefined) => {
    if (val && val.trim()) rows.push({ category: cat, label, value: val.trim() });
  };

  const segments = hl7.split(/[\r\n\n## Code Flow Diagram\n```mermaid
graph TD
    A[WAH4PCE Admin] -->|Uses| B[App Router & Layout]
    B --> C[Auth & Session Manager]
    B --> D[Metrics & Health Monitor]
    B --> E[Transaction Log Viewer]
    B --> F[Payload Comparison & Unhash]
    D --> G[ADAPT API Client]
    E --> G
    F --> G
    G -->|Fetches| H[API Layer]

    classDef default fill:#f9f9f9,stroke:#333,stroke-width:2px;
```\n\n]+/).filter(Boolean);
  for (const seg of segments) {
    const fields = seg.split('|');
    const segType = fields[0];

    if (segType === 'PID') {
      // PID|1||PhilHealth^^^PhilHealth^SB||LNAME^FNAME^MNAME^^^SUFFIX||DOB|SEX|||STREET^^CITY^PROVINCE^ZIP^PH|||CIVIL_STATUS|||||||||||||||CONTACT
      const phId = fields[3]?.split('^')[0];
      add('Patient', 'PhilHealth No.', phId);
      const nameParts = (fields[5] || '').split('^');
      add('Patient', 'Last Name', nameParts[0]);
      add('Patient', 'First Name', nameParts[1]);
      add('Patient', 'Middle Name', nameParts[2]);
      add('Patient', 'Suffix', nameParts[5]);
      add('Patient', 'Date of Birth', fields[7]);
      add('Patient', 'Sex', fields[8]);
      const addrParts = (fields[11] || '').split('^');
      add('Patient', 'Street', addrParts[0]);
      add('Patient', 'City', addrParts[2]);
      add('Patient', 'Province', addrParts[3]);
      add('Patient', 'Zip Code', addrParts[4]);
      add('Patient', 'Civil Status', fields[16]);
      // Contact is at end of PID
      const contact = fields[fields.length - 1];
      if (contact && /\d/.test(contact)) add('Patient', 'Contact No.', contact);
    }

    if (segType === 'PV1') {
      // PV1|1|O|FACILITY|||||||PHYSICIAN^LICENSE||...|||||||...|||...|||...|||...|||PRIORITY
      add('Referral', 'Physician', fields[9]?.split('^')[0]);
      add('Referral', 'Priority', fields[fields.length - 1]);
    }

    if (segType === 'OBX') {
      // OBX|seq|NM|LOINC^Display^LN||VALUE|UNIT|...
      const display = fields[3]?.split('^')[1] || 'Vital';
      const value = fields[5];
      const unit = fields[6];
      add('Vitals', display, value ? `${value}${unit ? ' ' + unit : ''}` : undefined);
    }

    if (segType === 'DG1') {
      // DG1|1||CODE^DESC^I10|||TYPE||||||||CHIEF_COMPLAINT
      const codeParts = (fields[3] || '').split('^');
      add('Diagnosis', 'ICD-10 Code', codeParts[0]);
      add('Diagnosis', 'Description', codeParts[1]);
      add('Diagnosis', 'Type', fields[6]);
      add('Diagnosis', 'Chief Complaint', fields[fields.length - 1]);
    }

    if (segType === 'RF1') {
      // RF1|PRIORITY|REASON||FACILITY|TIMESTAMP
      add('Referral', 'Reason', fields[2]);
      add('Referral', 'Facility', fields[4]);
    }
  }
  return rows;
}

// ─── Auto-detect and extract data from any payload ───
function extractDataFields(payload: Record<string, unknown>): { category: string; label: string; value: string }[] {
  // FHIR bundle
  if ((payload as any)?.resourceType === 'Bundle' || (payload as any)?.entry) {
    return extractFHIRData(payload);
  }
  // Single FHIR resource
  if ((payload as any)?.resourceType) {
    return extractFHIRData(payload);
  }
  // iHOMIS flat payload (has patient_fname or similar)
  if ((payload as any)?.patient_fname || (payload as any)?.patient_lname || (payload as any)?.philhealth_no || (payload as any)?.bp_systolic) {
    return extractHL7Data(payload);
  }
  // HL7v2 string inside { message: ... }
  if ((payload as any)?.message && typeof (payload as any).message === 'string') {
    return parseHL7v2String((payload as any).message);
  }
  // Raw HL7v2 string directly
  if (typeof payload === 'string' && (payload as string).startsWith('MSH|')) {
    return parseHL7v2String(payload as string);
  }
  // Fallback: try both
  const hl7 = extractHL7Data(payload);
  if (hl7.length > 0) return hl7;
  return extractFHIRData(payload);
}

// ─── Full field templates for each system (mirrors their actual forms) ───
// iHOMIS = 27 fields exactly
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

// WAH = 26 fields exactly
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

// Map extracted labels → template labels (left = iHOMIS label, right = WAH label)
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

// Also map from extractor-produced labels (which may differ from template labels)
const EXTRACTOR_ALIASES: Record<string, string[]> = {
  // WAH template labels ← extractFHIR labels
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
  // iHOMIS template labels ← extractFHIR labels
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
  'Physician License': ['Physician License'],
};

function getDestTemplate(destSystem: string): { category: string; label: string }[] {
  if (destSystem.toLowerCase().includes('wah')) return WAH_TEMPLATE;
  return IHOMIS_TEMPLATE;
}

function findValueForTemplateField(
  templateLabel: string,
  extractedFields: { category: string; label: string; value: string }[],
  isDestWAH: boolean
): string | null {
  // Direct match
  const direct = extractedFields.find(f => f.label === templateLabel);
  if (direct) return direct.value;

  // Check via extractor aliases
  const aliases = EXTRACTOR_ALIASES[templateLabel];
  if (aliases) {
    for (const alias of aliases) {
      const found = extractedFields.find(f => f.label === alias);
      if (found) return found.value;
    }
  }

  // Case-insensitive fallback
  const lower = templateLabel.toLowerCase();
  const ci = extractedFields.find(f => f.label.toLowerCase() === lower);
  if (ci) return ci.value;

  // Check via FIELD_MAP
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

function ComparisonTable({ raw, transformed, source, dest }: { raw: Record<string, unknown> | null, transformed: Record<string, unknown> | null, source: string, dest: string }) {
  const srcExtracted = useMemo(() => raw ? extractDataFields(raw) : [], [raw]);
  const destExtracted = useMemo(() => transformed ? extractDataFields(transformed) : [], [transformed]);
  const srcTemplate = useMemo(() => getDestTemplate(source), [source]);
  const destTemplate = useMemo(() => getDestTemplate(dest), [dest]);
  const isSrcWAH = source.toLowerCase().includes('wah');
  const isDestWAH = dest.toLowerCase().includes('wah');

  // Build left-side rows: ALL source template fields with values from raw payload
  const srcRows = useMemo(() => {
    return srcTemplate.map(tf => {
      const val = findValueForTemplateField(tf.label, srcExtracted, isSrcWAH);
      return { ...tf, value: val || '' };
    });
  }, [srcTemplate, srcExtracted, isSrcWAH]);

  const srcFilledCount = srcRows.filter(r => r.value !== '').length;

  // Build right-side rows: ALL dest template fields, filled or empty
  const destRows = useMemo(() => {
    return destTemplate.map(tf => {
      const fromTransformed = findValueForTemplateField(tf.label, destExtracted, isDestWAH);
      if (fromTransformed) return { ...tf, value: fromTransformed, status: 'Filled' as const };
      return { ...tf, value: '', status: 'Empty' as const };
    });
  }, [destTemplate, destExtracted, isDestWAH]);

  const filledCount = destRows.filter(r => r.status === 'Filled').length;
  const emptyCount = destRows.filter(r => r.status === 'Empty').length;

  return (
    <div className="grid grid-cols-1 lg:grid-cols-2 gap-4">
      {/* ─── LEFT: Source system's data ─── */}
      <div className="ipaas-card overflow-hidden">
        <div className="px-4 py-3 border-b border-[#e5e7eb] bg-[#fafafa]">
          <div className="flex items-center gap-2">
            <div className="w-2.5 h-2.5 rounded-full" style={{ background: 'var(--color-warning)' }} />
            <h3 className="text-xs font-semibold uppercase tracking-wide">{source} — Data Sent</h3>
            <span className="text-[10px] px-1.5 py-0.5 rounded-none bg-gray-100 text-gray-500 ml-auto">
              {srcFilledCount === srcTemplate.length ? `${srcTemplate.length} fields` : `${srcFilledCount} / ${srcTemplate.length} fields`}
            </span>
          </div>
        </div>
        <div className="overflow-auto max-h-[600px]">
          <table className="w-full text-left text-xs" style={{ borderCollapse: 'collapse' }}>
            <thead className="bg-[#f9fafb] sticky top-0" style={{ zIndex: 1 }}>
              <tr>
                <th className="p-3 font-semibold text-gray-600 border-b border-[#e5e7eb]">Field</th>
                <th className="p-3 font-semibold text-gray-600 border-b border-[#e5e7eb]">Value</th>
              </tr>
            </thead>
            <tbody>
              {(() => { let lastCat = ''; return srcRows.map((row, i) => {
                const showCat = row.category !== lastCat;
                lastCat = row.category;
                const hasFill = row.value !== '';
                return (
                  <Fragment key={`src-${i}`}>{showCat && (
                    <tr className="border-t-2 border-t-gray-200">
                      <td colSpan={2} className="px-3 pt-3 pb-1 font-semibold text-gray-700 text-[11px] uppercase tracking-wide">{row.category}</td>
                    </tr>
                  )}
                  <tr className={`hover:bg-gray-50 border-b border-[#e5e7eb] last:border-0 transition-colors ${!hasFill ? 'bg-gray-50/50' : ''}`}>
                    <td className="px-3 py-2" style={{ color: hasFill ? 'var(--color-text-muted)' : '#d1d5db' }}>{row.label}</td>
                    <td className="px-3 py-2 font-mono font-medium" style={{ color: hasFill ? undefined : '#d1d5db' }}>{hasFill ? row.value : '—'}</td>
                  </tr></Fragment>
                );
              }); })()}
            </tbody>
          </table>
        </div>
      </div>

      {/* ─── RIGHT: Destination system's ALL fields ─── */}
      <div className="ipaas-card overflow-hidden">
        <div className="px-4 py-3 border-b border-[#e5e7eb] bg-[#fafafa]">
          <div className="flex items-center gap-2">
            <div className="w-2.5 h-2.5 rounded-full" style={{ background: 'var(--color-success)' }} />
            <h3 className="text-xs font-semibold uppercase tracking-wide">{dest} — Data Received</h3>
            <span className="text-[10px] px-1.5 py-0.5 rounded-none bg-gray-100 text-gray-500 ml-auto">
              {filledCount} filled · {emptyCount} empty
            </span>
          </div>
        </div>
        <div className="overflow-auto max-h-[600px]">
          <table className="w-full text-left text-xs" style={{ borderCollapse: 'collapse' }}>
            <thead className="bg-[#f9fafb] sticky top-0" style={{ zIndex: 1 }}>
              <tr>
                <th className="p-3 font-semibold text-gray-600 border-b border-[#e5e7eb]">Field</th>
                <th className="p-3 font-semibold text-gray-600 border-b border-[#e5e7eb]">Value</th>
                <th className="p-3 font-semibold text-gray-600 border-b border-[#e5e7eb] text-center w-[80px]">Status</th>
              </tr>
            </thead>
            <tbody>
              {(() => { let lastCat = ''; return destRows.map((row, i) => {
                const showCat = row.category !== lastCat;
                lastCat = row.category;
                const isFilled = row.status === 'Filled';
                return (
                  <Fragment key={`dest-${i}`}>{showCat && (
                    <tr className="border-t-2 border-t-gray-200">
                      <td colSpan={3} className="px-3 pt-3 pb-1 font-semibold text-gray-700 text-[11px] uppercase tracking-wide">{row.category}</td>
                    </tr>
                  )}
                  <tr className={`hover:bg-gray-50 border-b border-[#e5e7eb] last:border-0 transition-colors ${!isFilled ? 'bg-red-50/30' : ''}`}>
                    <td className="px-3 py-2" style={{ color: isFilled ? 'var(--color-text-muted)' : '#d1d5db' }}>{row.label}</td>
                    <td className="px-3 py-2 font-mono font-medium" style={{ color: isFilled ? undefined : '#d1d5db' }}>{isFilled ? row.value : '—'}</td>
                    <td className="px-3 py-2 text-center">
                      <span className={`inline-flex items-center px-2 py-0.5 rounded-none text-[10px] font-bold uppercase tracking-wider ${isFilled ? 'bg-[#dcfce7] text-[#166534]' : 'bg-[#fee2e2] text-[#991b1b]'}`}>
                        {row.status}
                      </span>
                    </td>
                  </tr></Fragment>
                );
              }); })()}
            </tbody>
          </table>
        </div>
      </div>
    </div>
  );
}

// Synchronous hash for initial display
function simpleHash(text: string): string {
  let hash = 0;
  for (let i = 0; i < text.length; i++) {
    const chr = text.charCodeAt(i);
    hash = ((hash << 5) - hash) + chr;
    hash |= 0;
  }
  const hex = Math.abs(hash).toString(16).padStart(8, '0');
  return `${hex}${hex}${hex}${hex}${hex}${hex}${hex}${hex}`.slice(0, 64);
}

interface Transaction {
  id: string; source_system: string; destination_system: string;
  status: string; raw_payload: Record<string, unknown>;
  transformed_payload: Record<string, unknown> | null;
  error_message: string | null; created_at: string;
}

// Hashable payload panel component
function HashablePayload({ label, dotColor, payload }: { label: string; dotColor: string; payload: Record<string, unknown> | null }) {
  const [revealed, setRevealed] = useState(false);
  const [hash, setHash] = useState<string>('');

  const jsonStr = payload ? JSON.stringify(payload, null, 2) : '';

  useEffect(() => {
    if (jsonStr) {
      setHash(simpleHash(jsonStr));
      sha256(jsonStr).then(setHash);
    }
  }, [jsonStr]);

  if (!payload) {
    return (
      <div className="ipaas-card p-4">
        <div className="flex items-center gap-2 mb-3">
          <div className="w-2.5 h-2.5 rounded-full" style={{ background: dotColor }} />
          <h3 className="text-xs font-semibold uppercase tracking-wide">{label}</h3>
        </div>
        <div className="p-8 text-center rounded-none" style={{ background: 'var(--color-bg-primary)', border: '1px solid var(--color-border)' }}>
          <p className="text-xs" style={{ color: 'var(--color-text-muted)' }}>Transformation pending or failed</p>
        </div>
      </div>
    );
  }

  return (
    <div className="ipaas-card p-4" style={revealed ? { borderColor: 'rgba(139,92,246,0.3)' } : {}}>
      <div className="flex items-center justify-between mb-3">
        <div className="flex items-center gap-2">
          <div className="w-2.5 h-2.5 rounded-full" style={{ background: dotColor }} />
          <h3 className="text-xs font-semibold uppercase tracking-wide">{label}</h3>
          <span className="text-[10px] font-medium px-1.5 py-0.5 rounded-none" style={{
            background: revealed ? 'rgba(139,92,246,0.08)' : 'rgba(245,158,11,0.08)',
            color: revealed ? '#8b5cf6' : '#f59e0b',
          }}>
            {revealed ? 'REVEALED' : 'SHA-256 HASHED'}
          </span>
        </div>
        <button
          onClick={() => setRevealed(!revealed)}
          className="flex items-center gap-1.5 text-xs font-medium px-3 py-1.5 rounded-none transition-all"
          style={{
            background: revealed ? 'rgba(220,38,38,0.06)' : 'rgba(139,92,246,0.06)',
            color: revealed ? '#dc2626' : '#8b5cf6',
            border: `1px solid ${revealed ? 'rgba(220,38,38,0.15)' : 'rgba(139,92,246,0.15)'}`,
          }}
        >
          {revealed ? (
            <>
              <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
                <path d="M17.94 17.94A10.07 10.07 0 0112 20c-7 0-11-8-11-8a18.45 18.45 0 015.06-5.94" />
                <path d="M9.9 4.24A9.12 9.12 0 0112 4c7 0 11 8 11 8a18.5 18.5 0 01-2.16 3.19" />
                <line x1="1" y1="1" x2="23" y2="23" />
              </svg>
              Re-hash
            </>
          ) : (
            <>
              <svg width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
                <path d="M1 12s4-8 11-8 11 8 11 8-4 8-11 8-11-8-11-8z" />
                <circle cx="12" cy="12" r="3" />
              </svg>
              Unhash
            </>
          )}
        </button>
      </div>
      <pre
        className="p-3 rounded-none text-xs overflow-auto"
        style={{
          background: 'var(--color-bg-primary)',
          border: '1px solid var(--color-border)',
          maxHeight: '600px',
          fontFamily: "'JetBrains Mono', monospace",
          color: revealed ? 'var(--color-text-primary)' : 'var(--color-text-muted)',
          wordBreak: revealed ? 'break-word' : 'break-all',
          whiteSpace: revealed ? 'pre-wrap' : 'nowrap',
        }}
      >
        {revealed ? jsonStr : hash}
      </pre>
    </div>
  );
}

export default function MapperPage() {
  return (
    <Suspense fallback={
      <><Sidebar /><main className="flex-1 p-6 flex flex-col gap-4 overflow-hidden bg-gray-50">
        <div className="mb-1 h-12 w-1/3 bg-gray-200 animate-pulse rounded-none" />
        <div className="h-10 w-full bg-gray-200 animate-pulse rounded-none" />
        <div className="h-16 w-full bg-gray-200 animate-pulse rounded-none" />
        <div className="grid grid-cols-1 lg:grid-cols-2 gap-4 flex-1">
          <div className="ipaas-card animate-pulse bg-white border-none h-full" />
          <div className="ipaas-card animate-pulse bg-white border-none h-full" />
        </div>
      </main></>
    }>
      <MapperContent />
    </Suspense>
  );
}

function MapperContent() {
  const searchParams = useSearchParams();
  const txId = searchParams.get('id');
  const [tx, setTx] = useState<Transaction | null>(null);
  const [allTx, setAllTx] = useState<Transaction[]>([]);
  const [loading, setLoading] = useState(true);
  const [refreshing, setRefreshing] = useState(false);
  const [viewMode, setViewMode] = useState<'json' | 'table'>('json');

  const fetchData = async (isManualRefresh = false) => {
    if (isManualRefresh) setRefreshing(true);
    try {
      const data = await safeFetch('/api/transactions?limit=50');
      if (data.success) {
        const list: Transaction[] = data.data || [];
        setAllTx(list);
        if (list.length > 0) {
          if (isManualRefresh) {
            const hasNewer = allTx.length > 0 && list[0]?.id !== allTx[0]?.id;
            if (hasNewer) {
              setTx(list[0]);
            } else {
              const current = list.find((t: Transaction) => t.id === tx?.id);
              setTx(current || list[0]);
            }
          } else {
            if (txId) {
              const found = list.find((t: Transaction) => t.id === txId);
              if (found) setTx(found);
              else setTx(list[0]);
            } else {
              setTx(list[0]);
            }
          }
        }
      }
    } finally {
      if (isManualRefresh) setRefreshing(false);
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchData();
  }, [txId]);

  const statusStyle = (s: string) => {
    const m: Record<string, { bg: string; color: string }> = {
      SUCCESS: { bg: 'rgba(5,150,105,0.08)', color: '#059669' },
      PENDING: { bg: 'rgba(217,119,6,0.08)', color: '#d97706' },
      TRANSFORMING: { bg: 'rgba(37,99,235,0.08)', color: '#2563eb' },
      QUARANTINED: { bg: 'rgba(220,38,38,0.08)', color: '#dc2626' },
    };
    return m[s] || m.PENDING;
  };

  const rawLabel = (src: string) => src === 'iHOMIS' ? 'HL7 v2 Payload' : 'FHIR R4 Bundle';
  const transformedLabel = (dest: string) => dest === 'WAH' ? 'PH Core FHIR R4' : 'iHOMIS Format';

  return (
    <>
      <Sidebar />
      <main className="flex-1 p-6 overflow-auto">
        <div className="mb-5">
          <h1 className="text-lg font-semibold">Data Mapper</h1>
          <p className="text-sm" style={{ color: 'var(--color-text-secondary)' }}>Side-by-side view of raw input and AI-transformed output. All payloads are SHA-256 hashed by default.</p>
        </div>

        {loading ? (
          <div className="flex flex-col gap-4 w-full">
            <div className="flex justify-between items-center gap-3">
              <div className="h-9 w-64 bg-gray-200 animate-pulse rounded-none" />
              <div className="h-9 w-40 bg-gray-200 animate-pulse rounded-none" />
            </div>
            <div className="h-16 w-full bg-gray-200 animate-pulse rounded-none ipaas-card border-none" />
            <div className="grid grid-cols-1 lg:grid-cols-2 gap-4">
              <div className="ipaas-card animate-pulse bg-white border-none flex flex-col min-h-[160px]">
                <div className="h-10 border-b border-gray-100 flex items-center px-4"><div className="w-32 h-3 bg-gray-200" /></div>
                <div className="p-4 space-y-2">
                  {[...Array(3)].map((_, i) => <div key={i} className="h-3 bg-gray-100 w-full" />)}
                  <div className="h-3 bg-gray-100 w-2/3" />
                </div>
              </div>
              <div className="ipaas-card animate-pulse bg-white border-none flex flex-col min-h-[160px]">
                <div className="h-10 border-b border-gray-100 flex items-center px-4"><div className="w-40 h-3 bg-gray-200" /></div>
                <div className="p-4 space-y-2">
                  {[...Array(4)].map((_, i) => <div key={i} className="h-3 bg-gray-100 w-full" />)}
                  <div className="h-3 bg-gray-100 w-3/4" />
                </div>
              </div>
            </div>
          </div>
        ) : !tx ? (
          <div className="ipaas-card p-10 text-center">
            <svg width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="var(--color-text-muted)" strokeWidth="1" className="mx-auto mb-3"><polyline points="16 18 22 12 16 6"/><polyline points="8 6 2 12 8 18"/></svg>
            <p className="text-sm mb-3" style={{ color: 'var(--color-text-muted)' }}>No transactions to display. Send data from iHOMIS or WAH first.</p>
            <button
              onClick={() => fetchData(true)}
              disabled={refreshing}
              className="ipaas-btn ipaas-btn-secondary text-xs inline-flex items-center gap-1.5"
            >
              <svg className={refreshing ? 'animate-spin' : ''} width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5">
                <polyline points="23 4 23 10 17 10"/><path d="M20.49 15a9 9 0 11-2.12-9.36L23 10"/>
              </svg>
              Refresh
            </button>
          </div>
        ) : (
          <>
            <div className="mb-4 flex items-center justify-between gap-3 flex-wrap">
              <div className="flex items-center gap-2 flex-1 max-w-lg">
                <select
                  value={tx.id}
                  onChange={e => { const found = allTx.find(t => t.id === e.target.value); if (found) setTx(found); }}
                  className="px-3 py-2 rounded-none text-xs outline-none flex-1"
                  style={{ background: '#fff', border: '1px solid var(--color-border)' }}
                >
                  {allTx.map(t => (
                    <option key={t.id} value={t.id}>
                      {t.id.slice(0, 8)} — {t.source_system} → {t.destination_system} ({t.status})
                    </option>
                  ))}
                </select>
                <button
                  onClick={() => fetchData(true)}
                  disabled={refreshing}
                  className="ipaas-btn ipaas-btn-secondary text-xs flex items-center gap-1.5 shrink-0"
                  title="Refresh transactions"
                >
                  <svg
                    className={refreshing ? 'animate-spin' : ''}
                    width="14"
                    height="14"
                    viewBox="0 0 24 24"
                    fill="none"
                    stroke="currentColor"
                    strokeWidth="1.5"
                  >
                    <polyline points="23 4 23 10 17 10"/>
                    <path d="M20.49 15a9 9 0 11-2.12-9.36L23 10"/>
                  </svg>
                  Refresh
                </button>
              </div>

              <div className="flex bg-[#f3f4f6] rounded-none p-1" style={{ border: '1px solid var(--color-border)' }}>
                <button
                  onClick={() => setViewMode('json')}
                  className={`px-4 py-1.5 text-xs font-medium rounded-none transition-colors ${viewMode === 'json' ? 'bg-white shadow-sm text-blue-600' : 'text-gray-500 hover:text-gray-900'}`}
                >
                  JSON
                </button>
                <button
                  onClick={() => setViewMode('table')}
                  className={`px-4 py-1.5 text-xs font-medium rounded-none transition-colors ${viewMode === 'table' ? 'bg-white shadow-sm text-blue-600' : 'text-gray-500 hover:text-gray-900'}`}
                >
                  Comparison Table
                </button>
              </div>
            </div>

            <div className="ipaas-card p-4 mb-4">
              <div className="flex items-center justify-between flex-wrap gap-3">
                <div className="flex items-center gap-3">
                  <span className="text-xs font-semibold px-2 py-1 rounded-none" style={{
                    background: tx.source_system === 'iHOMIS' ? 'rgba(37,99,235,0.08)' : 'rgba(5,150,105,0.08)',
                    color: tx.source_system === 'iHOMIS' ? '#2563eb' : '#059669',
                  }}>{tx.source_system}</span>
                  <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="var(--color-text-muted)" strokeWidth="1.5"><path d="M5 12h14M12 5l7 7-7 7"/></svg>
                  <span className="text-xs font-semibold px-2 py-1 rounded-none" style={{
                    background: tx.destination_system === 'iHOMIS' ? 'rgba(37,99,235,0.08)' : 'rgba(5,150,105,0.08)',
                    color: tx.destination_system === 'iHOMIS' ? '#2563eb' : '#059669',
                  }}>{tx.destination_system}</span>
                  <span className="ipaas-badge" style={{ ...statusStyle(tx.status) }}>{tx.status}</span>
                </div>
                <p className="text-xs" style={{ color: 'var(--color-text-muted)' }}>{new Date(tx.created_at).toLocaleString()}</p>
              </div>
              {tx.error_message && (
                <div className="mt-3 p-2.5 rounded-none text-xs flex items-center gap-2" style={{ background: 'rgba(220,38,38,0.05)', color: '#dc2626', border: '1px solid rgba(220,38,38,0.15)' }}>
                  <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5"><path d="M10.29 3.86L1.82 18a2 2 0 001.71 3h16.94a2 2 0 001.71-3L13.71 3.86a2 2 0 00-3.42 0z"/><line x1="12" y1="9" x2="12" y2="13"/><line x1="12" y1="17" x2="12.01" y2="17"/></svg>
                  {tx.error_message}
                </div>
              )}
            </div>

            {viewMode === 'json' ? (
              <div className="grid grid-cols-1 lg:grid-cols-2 gap-4">
                <HashablePayload
                  label={`Raw — ${rawLabel(tx.source_system)}`}
                  dotColor="var(--color-warning)"
                  payload={tx.raw_payload}
                />
                <HashablePayload
                  label={`Transformed — ${transformedLabel(tx.destination_system)}`}
                  dotColor="var(--color-success)"
                  payload={tx.transformed_payload}
                />
              </div>
            ) : (
              <ComparisonTable raw={tx.raw_payload} transformed={tx.transformed_payload} source={tx.source_system} dest={tx.destination_system} />
            )}
          </>
        )}
      </main>
    </>
  );
}
```

### `<<component>> ADAPT API Client [Axios / Fetch Service]`
**File:** [`apps/adapt-ipaas/src/app/page.tsx`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/page.tsx) — Lines 6–10 + [`transactions/page.tsx`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/transactions/page.tsx) — Lines 6–10

The shared `safeFetch` utility used by all frontend pages to call the backend API.

```tsx
async function safeFetch(url: string) {
  const res = await fetch(url);
  const text = await res.text();
  try { return JSON.parse(text); } catch { return { success: false }; }
}

// Usage in Dashboard:
const [metricsData, txData] = await Promise.all([
  safeFetch('/api/metrics'),
  safeFetch('/api/transactions?limit=50'),
]);

// Usage in Transactions page:
const data = await safeFetch(`/api/transactions?${params.toString()}`);

// Usage in Mapper page:
const data = await safeFetch('/api/transactions?limit=50');
```

Also, the **Sidebar navigation component** acts as the routing client:

**File:** [`apps/adapt-ipaas/src/components/Sidebar.tsx`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/components/Sidebar.tsx) (66 lines)

```tsx
'use client';
import Link from 'next/link';
import Image from 'next/image';
import { usePathname } from 'next/navigation';

const navItems = [
  { href: '/', label: 'Dashboard', icon: <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round"><rect x="3" y="3" width="7" height="7"/><rect x="14" y="3" width="7" height="7"/><rect x="3" y="14" width="7" height="7"/><rect x="14" y="14" width="7" height="7"/></svg> },
  { href: '/transactions', label: 'Transaction Logs', icon: <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round"><path d="M14 2H6a2 2 0 00-2 2v16a2 2 0 002 2h12a2 2 0 002-2V8z"/><polyline points="14 2 14 8 20 8"/><line x1="16" y1="13" x2="8" y2="13"/><line x1="16" y1="17" x2="8" y2="17"/></svg> },
  { href: '/mapper', label: 'Data Mapper', icon: <svg width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="1.5" strokeLinecap="round" strokeLinejoin="round"><polyline points="16 18 22 12 16 6"/><polyline points="8 6 2 12 8 18"/><line x1="12" y1="2" x2="12" y2="22"/></svg> },
];

export default function Sidebar() {
  const pathname = usePathname();
  return (
    <aside className="w-[240px] min-h-screen flex flex-col border-r" style={{ background: 'var(--color-bg-sidebar)', borderColor: 'var(--color-border-sidebar)' }}>
      <div className="px-5 py-4 border-b" style={{ borderColor: 'var(--color-border-sidebar)' }}>
        <div className="flex items-center gap-2.5">
          <div className="w-10 h-10 flex items-center justify-center bg-transparent overflow-hidden">
            <Image src="/WAH_logo.png" alt="WAH Logo" width={36} height={36} className="object-contain scale-[1.7]" priority />
          </div>
          <div>
            <h1 className="text-sm font-bold leading-tight text-white">ADAPT iPaaS</h1>
            <p className="text-[11px] leading-tight text-slate-400">Integration Platform</p>
          </div>
        </div>
      </div>
      <nav className="flex-1 p-3 flex flex-col gap-0.5">
        <p className="text-[10px] font-bold uppercase tracking-wider px-3 py-2 text-slate-500">Menu</p>
        {navItems.map(item => {
          const isActive = pathname === item.href || (item.href !== '/' && pathname.startsWith(item.href));
          return <Link key={item.href} href={item.href} className={`ipaas-sidebar-link ${isActive ? 'active' : ''}`}>{item.icon}<span>{item.label}</span></Link>;
        })}
      </nav>
      <div className="px-5 py-3 border-t" style={{ borderColor: 'var(--color-border-sidebar)' }}>
        <p className="text-[10px] font-bold uppercase mb-2 text-slate-500">Systems</p>
        <div className="flex flex-col gap-1.5">
          <div className="flex items-center gap-2">
            <div className="w-1.5 h-1.5 rounded-full" style={{ background: '#10b981' }} />
            <span className="text-[11px] text-slate-400">iHOMIS — :3001</span>
          </div>
          <div className="flex items-center gap-2">
            <div className="w-1.5 h-1.5 rounded-full" style={{ background: '#10b981' }} />
            <span className="text-[11px] text-slate-400">WAH Hospital — :3002</span>
          </div>
        </div>
      </div>
      <div className="mt-auto px-5 py-4 border-t" style={{ borderColor: 'var(--color-border-sidebar)' }}>
        <button
          onClick={async () => {
            const { useRouter } = await import('next/navigation');
            await fetch('/api/auth/logout', { method: 'POST' });
            window.location.href = '/login';
          }}
          className="flex w-full items-center gap-2 px-3 py-2 rounded-none text-[13px] font-medium text-slate-400 hover:bg-slate-800 hover:text-white transition-all"
        >
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2" strokeLinecap="round" strokeLinejoin="round">
            <path d="M9 21H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h4"></path>
            <polyline points="16 17 21 12 16 7"></polyline>
            <line x1="21" y1="12" x2="9" y2="12"></line>
          </svg>
          Log Out
        </button>
      </div>
    </aside>
  );
}
```

### `<<package>> API Layer [Golang HTTP Services]`
**Same as Section 4 — `<<package>> API Layer [Package]`:** [`apps/adapt-ipaas/src/app/api/`](file:///d:/Documents_FromC/WAH4PCE-Merge%20Conflict/apps/adapt-ipaas/src/app/api/) — 6 API route directories serving the dashboard

---