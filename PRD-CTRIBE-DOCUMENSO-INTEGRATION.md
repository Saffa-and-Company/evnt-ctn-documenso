# PRD: C-Tribe Documenso Integration - Branding & API Fixes

## Executive Summary

This PRD covers two critical issues with the C-Tribe + Documenso integration:
1. **Signing Portal Branding** - The signing portal shows Documenso branding instead of C-Tribe
2. **API 500 Error** - Document creation fails from C-Tribe app with 500 error

---

## Issue 1: Signing Portal Branding

### Current State
The signing portal at `documenso-app-production-e8aa.up.railway.app/sign/[token]` displays:
- Documenso logo in header
- Green primary color (`#A2E771`) for buttons and accents
- "Documenso" text branding

### Desired State
- C-Tribe Society logo
- Gold primary color (`#FACC15`) matching C-Tribe brand
- No Documenso branding visible to signers

### Technical Analysis

**Current Theming System:**
- CSS variables in `packages/ui/styles/theme.css` define `--primary` color
- Tailwind config extends these with `ctribe` color palette (already added)
- Team branding settings exist but ONLY apply to emails, NOT signing portal

**Files That Control Signing Portal UI:**
| File | Purpose |
|------|---------|
| `apps/web/src/app/(signing)/sign/[token]/form.tsx` | Main signing form |
| `apps/web/src/app/(signing)/sign/[token]/sign-dialog.tsx` | Confirmation dialogs |
| `apps/web/src/app/(signing)/sign/[token]/signing-page-view.tsx` | Page layout |
| `packages/ui/styles/theme.css` | CSS variables for colors |

### Questions to Answer

1. **Scope of branding changes:**
   - [ ] Should branding be hardcoded to C-Tribe (simpler) or team-configurable (more complex)?
   - [ ] If hardcoded, this is a fork-specific change that won't merge upstream

2. **Logo source:**
   - [ ] Where is the C-Tribe logo hosted? (need URL for email `brandingLogo`)
   - [ ] What dimensions should the logo be for the signing portal header?

3. **Color confirmation:**
   - [ ] Primary button: Gold `#FACC15` - correct?
   - [ ] Text on buttons: Dark `#1a1a2e` - correct?
   - [ ] Background: Keep white or use dark theme?

### Implementation Options

**Option A: CSS Variable Override (Recommended)**
- Override `--primary` CSS variable globally
- Simplest approach, affects entire app
- Risk: Changes admin UI too, not just signing portal

**Option B: Signing Portal Theme Wrapper**
- Create a theme provider specific to signing routes
- More targeted, only affects signer experience
- More complex implementation

**Option C: Extend Team Branding to UI**
- Add `brandingPrimaryColor` to `TeamGlobalSettings`
- Apply team colors dynamically in signing portal
- Most flexible, but requires database changes

### Execution Steps (Option A - Simplest)

1. [ ] Update CSS variables in `packages/ui/styles/theme.css`:
   ```css
   --primary: 48 96% 53%;  /* #FACC15 in HSL */
   --primary-foreground: 240 19% 14%;  /* #1a1a2e */
   ```

2. [ ] Replace logo in signing portal header

3. [ ] Test all buttons and interactive elements

4. [ ] Deploy and verify

---

## Issue 2: API 500 Error from C-Tribe

### Current State
When creating a document from `clabs.ctribefestival.com`:
```
Error: Failed to create document: Documenso API error: 500 - {"message":"Something went wrong"}
```

### Root Cause Analysis (Confirmed 2026-01-19)

**Key Finding:** Document IS being created successfully in Documenso (visible with PDF and recipients). The 500 error occurs during `addField` API call due to **response field name mismatch**.

**Error Source:** `packages/lib/errors/app-error.ts` line 219 - The message `"Something went wrong"` is a generic error returned when status=500 to hide internal error details.

#### Confirmed: Documenso API Response Inconsistency

The Documenso API has **inconsistent field naming** for recipient IDs:

| Endpoint | Response Field | Code Location |
|----------|---------------|---------------|
| `createDocument` | `recipientId` | `implementation.ts:363` - explicitly mapped |
| `createRecipient` | `id` | `implementation.ts:837` - Prisma object spread |

**Documenso API Implementation Evidence:**

```javascript
// createDocument response (line 358-371):
recipients: recipients.map((recipient) => ({
  recipientId: recipient.id,  // ← Returns as 'recipientId'
  name: recipient.name,
  ...
})),

// createRecipient response (line 835-840):
return {
  body: {
    ...newRecipient,  // ← Prisma spread gives 'id', not 'recipientId'
    documentId: Number(documentId),
    signingUrl: `...`,
  },
};
```

#### C-Tribe Cloud Function Issue

**File:** `ctribe-event-web/cloud-functions/src/signing-functions.ts` (lines 178-180)

```typescript
const addedRecipient = await documensoClient.addRecipient(documensoDoc.id, recipientData);
addedRecipients.push({
  id: addedRecipient.id,  // ← Expects 'id' but may receive 'recipientId'
  ...recipientData,
});
```

**Data Flow Leading to Error:**
```
1. createDocument() → { id: "123", recipients: [] }  (empty - recipients added separately)
2. addRecipient()   → { id?: "5", recipientId?: 5, email: "...", ... }
3. addField()       → POST with recipientId: parseInt("undefined") = NaN ❌
4. Documenso API    → "Recipient not found" → 500 error
```

---

### Recommended Fix (in C-Tribe repo)

#### Option A: Handle Both Field Names (Quick Fix)

**File:** `cloud-functions/src/signing-functions.ts`

```typescript
// Change from:
addedRecipients.push({
  id: addedRecipient.id,
  ...recipientData,
});

// To:
addedRecipients.push({
  id: addedRecipient.id || addedRecipient.recipientId,  // Handle both field names
  ...recipientData,
});
```

#### Option B: Pass Recipients in createDocument (Recommended)

More reliable approach - pass recipients in the initial `createDocument` call instead of calling `addRecipient` separately.

**File:** `cloud-functions/src/documenso-client.ts`

```typescript
// In createDocument(), pass recipients in the initial request:
body: JSON.stringify({
  title: request.title,
  recipients: request.recipients.map((r) => ({
    email: r.email,
    name: r.name,
    role: r.role,
  })),
}),
```

Then the response consistently includes:
```typescript
recipients: [{ recipientId: 5, name: "...", email: "...", signingUrl: "..." }]
```

This eliminates the separate `addRecipient` calls and ensures consistent `recipientId` field.

---

### Verification Steps

**Step 1: Add Debug Logging & Deploy**

```typescript
// In documenso-client.ts addRecipient():
const data = await response.json();
logger.info("addRecipient response", { data: JSON.stringify(data) });
return data;

// In documenso-client.ts addField():
logger.info("addField request", {
  documentId,
  recipientId,
  parsedRecipientId: parseInt(recipientId, 10),
});
```

Deploy: `cd cloud-functions && firebase deploy --only functions:createSigningDocument`

**Step 2: Test Manually with curl**

```bash
# Create document WITH recipients in one call
curl -X POST https://documenso-app-production-e8aa.up.railway.app/api/v1/documents \
  -H "Authorization: Bearer $DOCUMENSO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title":"Test","recipients":[{"email":"test@example.com","name":"Test User","role":"SIGNER"}]}'

# Check response - note that recipients have 'recipientId' field
# Then create field using that recipientId
curl -X POST https://documenso-app-production-e8aa.up.railway.app/api/v1/documents/DOCUMENT_ID/fields \
  -H "Authorization: Bearer $DOCUMENSO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"recipientId":RECIPIENT_ID,"type":"SIGNATURE","pageNumber":1,"pageX":10,"pageY":80,"pageWidth":20,"pageHeight":5}'
```

**Step 3: After Fix**

1. Create a new signing document from C-Tribe admin
2. Verify document status is "draft" (not error)
3. Send for signing
4. Verify recipient receives signing email

---

## Priority Order

1. **Fix API 500 Error** (P0 - Blocking) ← **Root cause confirmed, fix in C-Tribe repo**
   - Fix is in `ctribe-event-web` repo, not this Documenso repo

2. **Update Signing Portal Branding** (P1 - Important)
   - Affects user experience but signing still works
   - Fix is in this Documenso repo

---

## Action Items by Repository

### C-Tribe Event Web (`ctribe-event-web`)
- [x] Root cause identified: `recipientId` field name mismatch
- [ ] Apply fix: Handle both `id` and `recipientId` in response
- [ ] OR: Refactor to pass recipients in `createDocument` call
- [ ] Deploy Cloud Functions
- [ ] Test end-to-end flow

### Documenso (`evnt-ctn-documenso`)
- [x] Email branding updated with C-Tribe gold colors
- [x] Update signing portal CSS variables (primary, ring, field-card, card-border-tint → gold)
- [x] Update signing portal components (`no-longer-available.tsx`, `complete/page.tsx`, `signing-field-container.tsx`)
- [x] Remove Documenso promotional text from signing portal
- [ ] Replace logo in signing portal header (pending: need logo URL)
- [ ] Deploy to Railway

---

## Success Criteria

1. **API Integration:**
   - [ ] Documents can be created from C-Tribe app without errors
   - [ ] Fields can be added to documents
   - [ ] Documents can be sent for signing
   - [ ] Signing links work correctly

2. **Branding:**
   - [ ] Signing portal shows C-Tribe logo (pending logo URL)
   - [x] Buttons use gold `#FACC15` color
   - [x] No Documenso branding visible to signers
   - [x] Email templates use C-Tribe branding (already done)

---

## Appendix: Environment Variables Reference

### Documenso (Railway)
```
NEXT_PUBLIC_WEBAPP_URL=https://documenso-app-production-e8aa.up.railway.app
NEXT_PUBLIC_UPLOAD_TRANSPORT=s3
NEXT_PRIVATE_UPLOAD_ENDPOINT=https://7150bc27ec78afaa4bf26c14c27dd242.r2.cloudflarestorage.com
NEXT_PRIVATE_UPLOAD_BUCKET=evnt-ctn-ctribe-documenso
```

### C-Tribe Cloud Functions (Firebase)
```
DOCUMENSO_API_URL=https://documenso-app-production-e8aa.up.railway.app
DOCUMENSO_API_KEY=<configured>
```
