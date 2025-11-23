# Authentication and Cloud Sync Feature

**Feature Category**: Authentication, Cloud Storage, Data Synchronization
**Added**: December 2024 (commits ed177530, 9be9bc8a, 83cb7166)
**Dependencies**: Supabase, Next.js API Routes

## Overview

The Authentication and Cloud Sync feature enables users to create accounts and synchronize their reading progress, annotations, and library across multiple devices. The system uses Supabase for authentication and data storage, with a custom sync protocol that supports bidirectional synchronization with conflict resolution.

### Key Capabilities

1. **User Authentication**
   - OAuth providers: Google, Apple, GitHub
   - Email/password authentication
   - Platform-specific OAuth flows (native vs web)
   - JWT-based token management
   - Session persistence

2. **Reading Progress Sync**
   - Real-time sync of current reading position
   - Cross-format support (CFI for EPUB, XPointer for compatibility)
   - Koreader integration for cross-app synchronization
   - Automatic conflict resolution (last-writer-wins)

3. **Notes and Annotations Sync**
   - Bidirectional sync of highlights, bookmarks, and notes
   - Soft-delete support (maintains deletion history)
   - Per-note conflict resolution
   - Batch processing for efficiency

4. **Library Sync**
   - Book metadata synchronization
   - Cloud storage for book files
   - Storage quota management
   - Incremental sync (only changed records)

## Architecture

### Component Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Frontend (React)                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ AuthContext  │  │ SyncContext  │  │   Stores     │  │
│  │  - Session   │  │  - State     │  │  - Data      │  │
│  │  - Tokens    │  │  - Hooks     │  │  - Settings  │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────┼──────────────────┼──────────────────┼─────────┘
          │                  │                  │
          ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────┐
│              Next.js API Routes (Serverless)             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │ /api/auth   │  │ /api/sync   │  │ /api/storage│    │
│  │  - OAuth    │  │  - Pull     │  │  - Upload   │    │
│  │  - Session  │  │  - Push     │  │  - Download │    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
└─────────┼──────────────────┼──────────────────┼─────────┘
          │                  │                  │
          ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────┐
│                   Supabase Backend                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  Auth       │  │  Database   │  │  Storage    │    │
│  │  - Users    │  │  - books    │  │  - Files    │    │
│  │  - Sessions │  │  - configs  │  │  - Covers   │    │
│  │  - Tokens   │  │  - notes    │  │             │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────┘
```

## Key Components and Files

### Authentication Components

| File | Purpose | Key Functions |
|------|---------|---------------|
| `src/context/AuthContext.tsx` | Authentication state management | `login()`, `logout()`, session restoration |
| `src/app/auth/page.tsx` | OAuth UI and flow control | Platform-specific OAuth handling |
| `src/helpers/auth.ts` | OAuth callback handler | Token exchange, session creation |
| `src/utils/access.ts` | Token management | `getAccessToken()`, `validateUserAndToken()` |
| `src/utils/supabase.ts` | Supabase client factory | Client initialization with auth |

**AuthContext State:**
```typescript
{
  user: User | null;           // Current authenticated user
  token: string | null;        // Access token (JWT)
  refreshToken: string | null; // Refresh token
  login: () => void;           // Initiate login flow
  logout: () => void;          // Clear session
}
```

**Token Storage:**
- Web: localStorage (`readest_token`, `readest_refresh_token`, `readest_user`)
- Native: Same localStorage keys (persisted via WebView)
- Server: Validated via Supabase JWT verification

### Sync Components

| File | Purpose | Key Functions |
|------|---------|---------------|
| `src/libs/sync.ts` | Sync client implementation | `pullChanges()`, `pushChanges()` |
| `src/context/SyncContext.tsx` | Sync state provider | Sync state management |
| `src/hooks/useSync.ts` | Main sync hook | Orchestrates full library sync |
| `src/app/reader/hooks/useProgressSync.ts` | Progress sync hook | 3-second interval progress sync |
| `src/app/reader/hooks/useNotesSync.ts` | Notes sync hook | 5-second interval notes sync |
| `src/pages/api/sync.ts` | Sync API endpoint | GET (pull), POST (push) |
| `src/utils/transform.ts` | Data transformers | DB ↔ App model conversion |

**Sync Flow:**

```
1. User makes change (e.g., turns page)
   ↓
2. Hook detects change (useProgressSync)
   ↓
3. Debounce timer (3 seconds)
   ↓
4. pushChanges({configs: [updatedConfig]})
   ↓
5. POST /api/sync
   ↓
6. Timestamp comparison (server vs client)
   ↓
7. Update database if client is newer
   ↓
8. Return authoritative record to client
   ↓
9. Update local state with server timestamp
```

### Database Schema

**Tables:**

```sql
-- Books table (library metadata)
CREATE TABLE books (
  id TEXT PRIMARY KEY,
  book_hash TEXT NOT NULL,
  meta_hash TEXT,           -- Version aggregator for progress sync
  user_id TEXT NOT NULL,
  title TEXT,
  authors TEXT[],
  description TEXT,
  cover_url TEXT,
  format TEXT,
  file_size INTEGER,
  updated_at TIMESTAMP,
  deleted_at TIMESTAMP
);

-- Book configs table (reading progress)
CREATE TABLE book_configs (
  id TEXT PRIMARY KEY,
  book_hash TEXT NOT NULL,
  user_id TEXT NOT NULL,
  progress TEXT,            -- JSON: [current, total]
  location TEXT,            -- CFI or XPointer
  view_settings TEXT,       -- JSON serialized ViewSettings
  updated_at TIMESTAMP,
  deleted_at TIMESTAMP
);

-- Book notes table (annotations)
CREATE TABLE book_notes (
  id TEXT PRIMARY KEY,
  book_hash TEXT NOT NULL,
  user_id TEXT NOT NULL,
  type TEXT,               -- 'bookmark' | 'annotation' | 'excerpt'
  cfi TEXT,
  text TEXT,               -- Highlighted text
  note TEXT,               -- User annotation
  style TEXT,              -- 'highlight' | 'underline' | 'squiggly'
  color TEXT,              -- 'red' | 'yellow' | 'green' | 'blue' | 'violet'
  created_at TIMESTAMP,
  updated_at TIMESTAMP,
  deleted_at TIMESTAMP
);
```

**Indexes:**
- `(user_id, book_hash)` on all tables for efficient user+book queries
- `updated_at` for incremental sync queries
- `deleted_at` for filtering soft-deleted records

## Data Flow

### Progress Synchronization

**Location:** `src/app/reader/hooks/useProgressSync.ts`

**Sync Interval:** 3 seconds (SYNC_PROGRESS_INTERVAL_SEC)

**Synced Data:**
- Current reading position (CFI for EPUB, XPointer for others)
- Progress percentage [current, total]
- Page and section information
- Last modified timestamp

**Sync Triggers:**
1. **On book open**: `pullConfig()` fetches latest progress
2. **On location change**: Debounced `pushConfig()` uploads progress
3. **Manual sync**: User can trigger via sync button

**Conflict Resolution:**
- Server compares `updated_at` timestamps
- Latest timestamp wins (last-writer-wins)
- No three-way merge (simplicity over accuracy)

**Format Conversion:**
```typescript
// CFI (EPUB) → XPointer (Koreader compatibility)
const xpointer = convertCFItoXPointer(cfi);

// XPointer → CFI
const cfi = convertXPointerToCFI(xpointer);
```

### Notes Synchronization

**Location:** `src/app/reader/hooks/useNotesSync.ts`

**Sync Interval:** 5 seconds (SYNC_NOTES_INTERVAL_SEC)

**Synced Data:**
- All book notes (highlights, bookmarks, annotations)
- Note metadata (type, color, style)
- Creation and modification timestamps
- Soft delete markers

**Merge Strategy:**
1. Filter local notes newer than `lastSyncedAtNotes`
2. Fetch remote notes since last sync
3. Merge by note ID:
   - If note exists locally and remotely: compare timestamps, keep latest
   - If note only remote: add to local
   - If note only local: push to remote
4. Handle soft deletes (deletedAt timestamp)
5. Update `lastSyncedAtNotes` timestamp

**Hard Delete vs Soft Delete:**
- **Soft Delete**: Sets `deletedAt` timestamp, syncs across devices
- **Hard Delete**: Permanently removes from all devices (user-initiated)

### Library Synchronization

**Location:** `src/hooks/useSync.ts`

**Sync Triggers:**
1. On app startup (if logged in)
2. Manual sync button
3. After importing new books
4. Periodic background sync (5 minutes)

**Full Sync Process:**
```typescript
async function syncLibrary() {
  // 1. Pull remote changes
  const { books, configs, notes } = await syncClient.pullChanges(lastSyncedAt);

  // 2. Merge remote data with local
  mergeBooks(books);
  mergeConfigs(configs);
  mergeNotes(notes);

  // 3. Push local changes
  const localChanges = getLocalChanges(lastSyncedAt);
  await syncClient.pushChanges(localChanges);

  // 4. Update sync timestamp
  setLastSyncedAt(Date.now());
}
```

## AI Agent Modification Guidelines

### Adding a New Synced Field

**Example: Add "last read date" to book configs**

1. **Update TypeScript types** (`src/types/book.ts`):
```typescript
export interface BookConfig {
  // ... existing fields
  lastReadDate?: number;  // Add new field
}
```

2. **Update database schema** (Supabase):
```sql
ALTER TABLE book_configs ADD COLUMN last_read_date TIMESTAMP;
```

3. **Update transform functions** (`src/utils/transform.ts`):
```typescript
export function transformBookConfigToDB(config: BookConfig): DBBookConfig {
  return {
    // ... existing transforms
    last_read_date: config.lastReadDate
      ? new Date(config.lastReadDate).toISOString()
      : null,
  };
}

export function transformBookConfigFromDB(dbConfig: DBBookConfig): BookConfig {
  return {
    // ... existing transforms
    lastReadDate: dbConfig.last_read_date
      ? new Date(dbConfig.last_read_date).getTime()
      : undefined,
  };
}
```

4. **Update sync logic** (`useProgressSync.ts`):
```typescript
// Ensure field is included in pushConfig()
const configToSync = {
  ...currentConfig,
  lastReadDate: Date.now(),  // Set new field
};
await pushConfig(configToSync);
```

5. **Test sync**:
   - Create note on device A
   - Wait for sync (3-5 seconds)
   - Open book on device B
   - Verify field is synced

### Adding OAuth Provider

**Example: Add Microsoft OAuth**

1. **Configure Supabase** (Supabase Dashboard):
   - Add Microsoft provider
   - Set client ID and secret
   - Configure redirect URLs

2. **Update auth page** (`src/app/auth/page.tsx`):
```typescript
const handleMicrosoftLogin = async () => {
  const { error } = await supabase.auth.signInWithOAuth({
    provider: 'azure',
    options: {
      redirectTo: `${window.location.origin}/auth/callback`,
    },
  });
  if (error) console.error('Microsoft login failed:', error);
};

// Add button to UI
<button onClick={handleMicrosoftLogin}>
  Sign in with Microsoft
</button>
```

3. **Test OAuth flow**:
   - Click Microsoft login button
   - Complete OAuth on Microsoft's site
   - Verify redirect to `/auth/callback`
   - Confirm session is created in AuthContext

### Implementing Custom Conflict Resolution

**Example: Three-way merge for notes**

Current implementation uses last-writer-wins. To implement three-way merge:

1. **Store base version** in database:
```sql
ALTER TABLE book_notes ADD COLUMN base_version TEXT;
```

2. **Update merge logic** (`useNotesSync.ts`):
```typescript
function mergeNoteWithConflictResolution(
  local: BookNote,
  remote: BookNote,
  base?: BookNote
) {
  if (!base) return remote.updatedAt > local.updatedAt ? remote : local;

  // Three-way merge
  const merged = { ...base };

  // If local changed from base, keep local change
  if (local.note !== base.note) merged.note = local.note;

  // If remote changed from base, keep remote change (unless local also changed)
  if (remote.note !== base.note && local.note === base.note) {
    merged.note = remote.note;
  }

  // Handle conflicts (both changed): prefer latest
  if (local.note !== base.note && remote.note !== base.note) {
    merged.note = remote.updatedAt > local.updatedAt ? remote.note : local.note;
  }

  return merged;
}
```

3. **Update push/pull** to include base version.

### Adding Sync Status Indicators

**Example: Show sync status in UI**

1. **Create sync status component** (`src/components/SyncStatus.tsx`):
```typescript
export function SyncStatus() {
  const { isSyncing, lastSyncedAt, syncError } = useSync();

  if (syncError) return <div>❌ Sync failed</div>;
  if (isSyncing) return <div>🔄 Syncing...</div>;
  if (lastSyncedAt) {
    const timeAgo = formatTimeAgo(lastSyncedAt);
    return <div>✓ Synced {timeAgo}</div>;
  }
  return <div>Not synced</div>;
}
```

2. **Add to header** (`LibraryHeader.tsx` or `ReaderHeader.tsx`):
```typescript
import { SyncStatus } from '@/components/SyncStatus';

// In component JSX:
<SyncStatus />
```

## Entry Points for Common Tasks

| Task | Primary Files | Steps |
|------|---------------|-------|
| **Enable authentication** | `AuthContext.tsx`, `auth/page.tsx` | User clicks login → OAuth flow → callback → session created |
| **Sync reading progress** | `useProgressSync.ts`, `api/sync.ts` | Location change → debounce → push to server → update local |
| **Sync annotations** | `useNotesSync.ts`, `notebookStore.ts` | Note created → throttle → merge with remote → sync |
| **Add OAuth provider** | `auth/page.tsx`, Supabase dashboard | Configure provider → add UI button → test flow |
| **Query sync status** | `SyncContext.tsx`, `useSync.ts` | Access context → check `isSyncing`, `lastSyncedAt` |
| **Debug sync issues** | `libs/sync.ts`, browser console | Check network tab → verify API calls → inspect timestamps |
| **Manage storage quota** | `api/storage/*.ts`, `access.ts` | Token contains quota → check before upload → enforce limits |

## Common Issues and Debugging

### Issue: Progress not syncing

**Symptoms:** Reading position not updated on other devices

**Debug steps:**
1. Check if user is logged in (`AuthContext.user !== null`)
2. Verify `useProgressSync` is active (check console logs)
3. Inspect network tab for `/api/sync` POST requests
4. Check timestamps: `config.updatedAt` vs server `updated_at`
5. Verify book hash is consistent across devices

**Common causes:**
- Network connectivity issues
- Token expired (check `access.ts` validation)
- Different book files with different hashes
- Sync disabled in settings

### Issue: Annotations duplicated

**Symptoms:** Same annotation appears multiple times

**Debug steps:**
1. Check note IDs (should be unique UUIDs)
2. Verify merge logic in `useNotesSync.ts`
3. Check for race conditions (multiple syncs in parallel)
4. Inspect `lastSyncedAtNotes` timestamp

**Fix:**
- Ensure note IDs are generated once (not regenerated on sync)
- Add deduplication logic in merge function
- Serialize sync operations (use mutex or queue)

### Issue: OAuth redirect fails

**Symptoms:** OAuth callback doesn't create session

**Debug steps:**
1. Check redirect URL configuration in Supabase
2. Verify `NEXT_PUBLIC_SUPABASE_URL` env variable
3. Inspect callback handler in `helpers/auth.ts`
4. Check browser console for errors

**Common causes:**
- Mismatched redirect URLs
- CORS issues
- Invalid OAuth credentials
- Browser blocking third-party cookies

### Issue: Sync conflicts (lost data)

**Symptoms:** Changes from one device overwrite changes from another

**Root cause:** Last-writer-wins conflict resolution

**Solutions:**
1. Implement three-way merge (see "Custom Conflict Resolution" above)
2. Add conflict detection UI (warn user before overwriting)
3. Store conflict history for manual resolution
4. Increase sync frequency to reduce conflict window

## Dependencies

### External Libraries

- **Supabase** (`@supabase/supabase-js`): Authentication and database
- **Supabase Auth Helpers** (`@supabase/auth-helpers-nextjs`): Next.js integration
- **Next.js**: API routes for serverless functions

### Internal Dependencies

- **Stores**: `bookDataStore`, `settingsStore`, `readerStore`
- **Utils**: `transform.ts`, `access.ts`, `supabase.ts`
- **Types**: `book.ts`, `system.ts`
- **AppService**: Platform-specific storage implementations

## Testing Sync Functionality

### Manual Testing Checklist

1. **Authentication:**
   - [ ] Login with Google OAuth
   - [ ] Login with Apple OAuth
   - [ ] Login with GitHub OAuth
   - [ ] Logout and verify session cleared
   - [ ] Session persistence after app restart

2. **Progress Sync:**
   - [ ] Open book on device A, read to page 10
   - [ ] Wait 5 seconds
   - [ ] Open same book on device B
   - [ ] Verify it opens at page 10

3. **Notes Sync:**
   - [ ] Create highlight on device A
   - [ ] Wait 10 seconds
   - [ ] Open book on device B
   - [ ] Verify highlight appears

4. **Conflict Resolution:**
   - [ ] Disconnect device A from network
   - [ ] Make change on device A (e.g., read to page 20)
   - [ ] Make change on device B (e.g., read to page 30)
   - [ ] Reconnect device A
   - [ ] Verify latest change wins (page 30)

5. **Offline Mode:**
   - [ ] Disconnect from network
   - [ ] Continue reading and annotating
   - [ ] Reconnect to network
   - [ ] Verify changes sync automatically

### Automated Testing

**Example: Test progress sync**

```typescript
// test/sync.test.ts
import { SyncClient } from '@/libs/sync';
import { mockSupabase } from './mocks/supabase';

describe('Progress Sync', () => {
  it('should push progress to server', async () => {
    const client = new SyncClient('test-token');
    const config = {
      id: 'config-1',
      bookHash: 'book-123',
      progress: [10, 100],
      location: 'epubcfi(/6/4[id])',
      updatedAt: Date.now(),
    };

    await client.pushChanges({ configs: [config] });

    expect(mockSupabase.from).toHaveBeenCalledWith('book_configs');
    expect(mockSupabase.upsert).toHaveBeenCalledWith(
      expect.objectContaining({ id: 'config-1' })
    );
  });

  it('should pull remote changes', async () => {
    const client = new SyncClient('test-token');
    const since = Date.now() - 10000;

    const result = await client.pullChanges(since);

    expect(result.configs).toHaveLength(1);
    expect(result.configs[0].bookHash).toBe('book-123');
  });
});
```

## Security Considerations

### Token Security

- **Never log tokens**: Avoid console.log or error messages with tokens
- **Use HTTPS only**: Tokens transmitted over secure connections
- **Validate on server**: Always validate tokens server-side (`validateUserAndToken()`)
- **Refresh tokens**: Implement automatic token refresh before expiration
- **Revoke on logout**: Clear tokens from storage on logout

### Data Privacy

- **User isolation**: Database queries always filter by `user_id`
- **Row-level security**: Supabase RLS policies enforce user data access
- **Encryption**: Data encrypted in transit (HTTPS) and at rest (Supabase)
- **Soft deletes**: Deleted data retained temporarily for sync, then purged

### API Security

- **Rate limiting**: Implement rate limits on `/api/sync` endpoints
- **CORS**: Configure allowed origins for API requests
- **Input validation**: Validate all inputs in API routes
- **SQL injection**: Use parameterized queries (Supabase SDK handles this)

## Performance Optimization

### Sync Performance

**Current:**
- Progress sync: every 3 seconds
- Notes sync: every 5 seconds
- Full library sync: every 5 minutes

**Optimizations:**
1. **Batch operations**: Sync multiple records in single API call
2. **Incremental sync**: Only fetch records changed since last sync
3. **Compression**: Gzip API responses for large payloads
4. **Deduplication**: Filter out unchanged records before pushing
5. **Debouncing**: Prevent excessive sync calls during rapid changes

### Database Performance

**Indexes:**
```sql
-- Essential for fast user queries
CREATE INDEX idx_books_user_hash ON books(user_id, book_hash);
CREATE INDEX idx_configs_user_hash ON book_configs(user_id, book_hash);
CREATE INDEX idx_notes_user_hash ON book_notes(user_id, book_hash);

-- Essential for incremental sync
CREATE INDEX idx_books_updated ON books(updated_at);
CREATE INDEX idx_configs_updated ON book_configs(updated_at);
CREATE INDEX idx_notes_updated ON book_notes(updated_at);
```

**Query optimization:**
- Limit result sets (default 1000 records)
- Use pagination for large libraries
- Filter deleted records early (`WHERE deleted_at IS NULL`)

## Updates (v0.9.32-0.9.43)

### Cloud Backup Status Indicators (v0.9.42, #1173, #1167)

**Overview**: Visual indicators showing cloud backup/sync status for each book on both mobile and desktop platforms.

**Features**:
- Status badge on book covers in library
- Real-time sync status updates
- Visual distinction between synced, syncing, and not synced states
- Works on both mobile and desktop

**Status States**:
1. **Synced** (✓ icon, green): Book and progress fully synced
2. **Syncing** (↻ icon, blue): Sync in progress
3. **Not Synced** (× icon, gray): Book not backed up to cloud
4. **Error** (! icon, red): Sync failed, requires attention

**Implementation** (`src/app/library/components/BookCard.tsx`):
```typescript
const CloudStatus = ({ bookHash }: { bookHash: string }) => {
  const syncStatus = useCloudSync(state => state.bookStatus[bookHash]);

  const icons = {
    synced: <FaCheckCircle className="text-green-500" />,
    syncing: <FaSyncAlt className="text-blue-500 animate-spin" />,
    not_synced: <FaTimesCircle className="text-gray-400" />,
    error: <FaExclamationCircle className="text-red-500" />
  };

  return (
    <div className="cloud-status-badge">
      {icons[syncStatus || 'not_synced']}
    </div>
  );
};
```

**Tooltip Information**:
- Hover shows last sync time
- Click opens sync details dialog
- Shows specific error messages when applicable

### Sign in with Apple on macOS (v0.9.32-0.9.33, #856, #866)

**Native Sign in with Apple** (v0.9.32, #856):
- Implemented native Sign in with Apple support on macOS
- Uses Apple's native authentication framework
- Seamless integration with macOS Keychain
- Better security and user experience

**OAuth Flow Improvement** (v0.9.33, #866):
- Refactored OAuth flow to use ASWebAuthenticationSession on macOS
- More reliable authentication process
- Better handling of OAuth callbacks
- Improved error handling

**Implementation** (`src/services/auth/apple.ts`):
```typescript
// macOS-specific Sign in with Apple
const signInWithApple = async () => {
  if (platform === 'macos') {
    // Use native ASWebAuthenticationSession
    const session = await invoke('apple_sign_in');
    return session;
  } else {
    // Fallback to web OAuth flow
    return supabase.auth.signInWithOAuth({
      provider: 'apple'
    });
  }
};
```

**Benefits**:
- Native macOS integration
- Automatic credential filling from Keychain
- Face ID/Touch ID support for re-authentication
- Follows Apple Human Interface Guidelines

### Books Without Covers Sync Support (v0.9.33, #878)

**Issue**: Books without cover images failed to sync across devices.

**Fix**: Implemented proper handling for books missing cover metadata:
```typescript
// Handle books without covers
const syncBook = async (book: BookMetadata) => {
  const bookData = {
    ...book,
    coverImageUrl: book.coverImageUrl || null,  // Allow null covers
    hasCover: !!book.coverImageUrl              // Track cover availability
  };

  await supabase
    .from('books')
    .upsert(bookData);
};
```

**Improvements**:
- Books sync successfully without covers
- Placeholder covers shown in UI
- Cover can be added later without re-sync
- Sync progress not blocked by missing covers

### Avatar Caching (v0.9.41, #1140)

**Feature**: Cache user avatar images for offline usage.

**Implementation**:
```typescript
// Cache avatar on login
const cacheAvatar = async (avatarUrl: string) => {
  const response = await fetch(avatarUrl);
  const blob = await response.blob();

  // Store in IndexedDB
  await avatarCache.set(userId, blob);
};

// Use cached avatar when offline
const getAvatar = async (userId: string) => {
  if (!navigator.onLine) {
    return await avatarCache.get(userId);
  }
  return avatarUrl;
};
```

**Benefits**:
- Avatar available offline
- Faster loading from cache
- Reduced network requests
- Better offline experience

## Version 0.9.44 - 0.9.63 Updates (def157ca → f5b686ab)

### Subscription Management System (v0.9.62, #1491, #1493, #1494, #1499, #1501, #1505)

**Major Feature**: Premium subscription management for Readest with tiered plans and payment integration.

**Overview**: Users can upgrade to premium plans to unlock additional features, increased storage, and higher usage quotas for translation and other cloud services.

**Subscription Tiers**:
1. **Free Plan**:
   - 500MB cloud storage
   - 500K characters/month DeepL translation
   - Basic cloud sync
   - Ad-supported (optional)

2. **Premium Plan** ($4.99/month or $49.99/year):
   - 5GB cloud storage
   - 2M characters/month DeepL translation
   - Priority sync
   - No ads
   - Early access to new features

3. **Pro Plan** ($9.99/month or $99.99/year):
   - Unlimited cloud storage
   - Unlimited DeepL translation
   - Priority support
   - Advanced features
   - Custom domain for web version

**Architecture**:

```
Frontend                          Backend                     External
┌─────────────────┐              ┌──────────────────┐        ┌────────────┐
│ Subscription UI │─────────────▶│ Subscription API │────────│  Stripe    │
│  - Plan cards   │              │  - Create        │        │  Payment   │
│  - Upgrade CTA  │              │  - Cancel        │        └────────────┘
│  - Status       │              │  - Webhook       │
└─────────────────┘              └──────────────────┘
                                          │
                                          ▼
                                 ┌──────────────────┐
                                 │   Supabase DB    │
                                 │  - subscriptions │
                                 │  - payments      │
                                 │  - usage_stats   │
                                 └──────────────────┘
```

**Implementation** (`src/app/api/subscription/route.ts`):
```typescript
// Subscription API endpoint
export async function POST(request: Request) {
  const { userId, plan } = await request.json();

  // Create Stripe checkout session
  const session = await stripe.checkout.sessions.create({
    customer_email: user.email,
    mode: 'subscription',
    line_items: [{
      price: STRIPE_PRICES[plan],
      quantity: 1,
    }],
    success_url: `${APP_URL}/subscription/success`,
    cancel_url: `${APP_URL}/subscription/cancel`,
  });

  return NextResponse.json({ sessionId: session.id });
}

// Stripe webhook handler
export async function handleWebhook(event: Stripe.Event) {
  switch (event.type) {
    case 'checkout.session.completed':
      await activateSubscription(event.data.object);
      break;
    case 'invoice.payment_succeeded':
      await renewSubscription(event.data.object);
      break;
    case 'customer.subscription.deleted':
      await cancelSubscription(event.data.object);
      break;
  }
}
```

**Database Schema** (`supabase/migrations/subscription.sql`):
```sql
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id UUID REFERENCES auth.users(id),
  plan VARCHAR(20) NOT NULL CHECK (plan IN ('free', 'premium', 'pro')),
  status VARCHAR(20) NOT NULL CHECK (status IN ('active', 'canceled', 'expired')),
  stripe_subscription_id VARCHAR(255),
  current_period_start TIMESTAMP,
  current_period_end TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_subscriptions_user_id ON subscriptions(user_id);
CREATE INDEX idx_subscriptions_status ON subscriptions(status);
```

**Frontend Components**:

1. **Plan Cards** (`src/app/subscription/components/PlanCard.tsx`):
   ```typescript
   function PlanCard({ plan, currentPlan }: PlanCardProps) {
     const features = PLAN_FEATURES[plan];
     const isCurrentPlan = plan === currentPlan;

     return (
       <div className="plan-card">
         <h3>{plan.name}</h3>
         <div className="price">
           ${plan.monthlyPrice}/mo
           {plan.yearlyPrice && (
             <span className="yearly">or ${plan.yearlyPrice}/yr (save 20%)</span>
           )}
         </div>

         <ul className="features">
           {features.map(f => <li key={f}>{f}</li>)}
         </ul>

         {isCurrentPlan ? (
           <button disabled>Current Plan</button>
         ) : (
           <button onClick={() => upgradeToPlan(plan)}>
             {plan.price > currentPlan.price ? 'Upgrade' : 'Downgrade'}
           </button>
         )}
       </div>
     );
   }
   ```

2. **Subscription Status** (`src/app/settings/components/SubscriptionStatus.tsx`):
   - Current plan display
   - Renewal date
   - Usage statistics (storage, translation quota)
   - Manage subscription button (cancel, update payment method)

**Web vs Tauri Implementation**:

**Web** (`src/app/subscription/web/SubscriptionManager.tsx`):
- Stripe Checkout integration
- Redirect to Stripe hosted page
- Return URL handling
- Session restoration after payment

**Tauri** (`src/app/subscription/tauri/SubscriptionManager.tsx`):
- In-app browser for Stripe Checkout
- Deep link handling for return URLs
- Native payment prompts (iOS, Android)
- Platform-specific receipt validation

**Platform-Specific Payment** (`src-tauri/src/payment.rs`):
```rust
#[cfg(target_os = "ios")]
use app_store_connect::InAppPurchase;

#[cfg(target_os = "android")]
use google_play_billing::BillingClient;

#[tauri::command]
async fn purchase_subscription(plan: String) -> Result<Receipt> {
    #[cfg(target_os = "ios")]
    {
        let purchase = InAppPurchase::new();
        let receipt = purchase.buy_product(&plan).await?;
        Ok(receipt)
    }

    #[cfg(target_os = "android")]
    {
        let billing = BillingClient::new();
        let receipt = billing.purchase_subscription(&plan).await?;
        Ok(receipt)
    }

    #[cfg(not(any(target_os = "ios", target_os = "android")))]
    {
        // Web/desktop: Use Stripe
        Err("Use Stripe checkout for this platform".into())
    }
}
```

**Subscription Enforcement**:

```typescript
// Check subscription before allowing feature
const useFeatureGate = (feature: string) => {
  const { subscription } = useAuth();

  const canUse = useMemo(() => {
    const requiredPlan = FEATURE_REQUIREMENTS[feature];
    const currentPlanLevel = PLAN_LEVELS[subscription.plan];
    const requiredPlanLevel = PLAN_LEVELS[requiredPlan];

    return currentPlanLevel >= requiredPlanLevel;
  }, [subscription, feature]);

  return canUse;
};

// Usage in components
function TranslationFeature() {
  const canUseDeepL = useFeatureGate('deepl_translation');

  if (!canUseDeepL) {
    return (
      <UpgradePrompt
        feature="DeepL Translation"
        requiredPlan="premium"
        message="Upgrade to Premium for unlimited DeepL translations"
      />
    );
  }

  return <TranslationUI />;
}
```

**Usage Tracking** (`src/utils/usageTracking.ts`):
```typescript
// Track feature usage against quotas
export class UsageTracker {
  async trackTranslation(userId: string, characters: number) {
    const usage = await getUsageToday(userId);
    const subscription = await getSubscription(userId);
    const quota = QUOTAS[subscription.plan].translation;

    if (usage.translation + characters > quota) {
      throw new QuotaExceededError('Translation quota exceeded for today');
    }

    await incrementUsage(userId, 'translation', characters);
  }

  async trackStorage(userId: string, bytes: number) {
    const usage = await getTotalUsage(userId);
    const subscription = await getSubscription(userId);
    const quota = QUOTAS[subscription.plan].storage;

    if (usage.storage + bytes > quota) {
      throw new QuotaExceededError('Storage quota exceeded');
    }

    await incrementUsage(userId, 'storage', bytes);
  }
}
```

**Files**:
- `src/app/subscription/page.tsx` - Subscription management page
- `src/app/api/subscription/route.ts` - Subscription API endpoints
- `src/app/api/stripe/webhook/route.ts` - Stripe webhook handler
- `src/components/UpgradePrompt.tsx` - Upgrade call-to-action component
- `src/hooks/useSubscription.ts` - Subscription state hook
- `src/utils/usageTracking.ts` - Quota and usage tracking
- `src-tauri/src/payment.rs` - Native payment integration

**CORS Fix** (v0.9.62, #1493):
- Added CORS middleware for subscription API
- Fixed cross-origin issues for Stripe callbacks
- Secure origin validation

**Supabase RLS** (v0.9.62, #1501):
- Row-level security policies for subscription data
- Users can only access their own subscription
- Admin role for support queries

**Settings Menu Integration** (v0.9.62, #1505):
- "Upgrade to Premium" option in settings
- Current plan display in profile
- Quick access to subscription management

**Status Badge** (v0.9.62, #1514):
- Fixed action button styling in plan cards
- Clear "Subscribe" vs "Current Plan" vs "Manage" states
- Loading states during payment processing

### Sync Status Menu (v0.9.52, #1324)

**Feature**: Visual indicator in the reader's view menu showing the current synchronization status.

**Purpose**: Provides users with real-time feedback about cloud sync operations, helping them understand when their progress and annotations are being synchronized.

**Settings Location**: View Menu > Sync Status indicator

**Status States**:

1. **Synced** (✓):
   - All changes uploaded to cloud
   - Local and remote are in sync
   - Green checkmark icon

2. **Syncing** (⟳):
   - Upload/download in progress
   - Animated spinner icon
   - Shows progress percentage (optional)

3. **Pending** (⋯):
   - Changes waiting to be synced
   - Will sync when network available
   - Gray dot icon

4. **Error** (⚠):
   - Sync failed
   - Shows error message on click
   - Red warning icon
   - Retry button available

**Implementation** (`src/app/reader/components/ViewMenu.tsx`):
```typescript
function SyncStatusIndicator() {
  const { syncStatus, lastSyncTime, error } = useProgressSync();

  const getStatusIcon = () => {
    switch (syncStatus) {
      case 'synced':
        return <CheckCircleIcon className="text-success" />;
      case 'syncing':
        return <SpinnerIcon className="animate-spin text-info" />;
      case 'pending':
        return <DotsIcon className="text-gray-400" />;
      case 'error':
        return <WarningIcon className="text-error" />;
    }
  };

  const getStatusText = () => {
    if (syncStatus === 'synced' && lastSyncTime) {
      return `Synced ${formatRelativeTime(lastSyncTime)}`;
    }
    if (syncStatus === 'error') {
      return `Sync failed: ${error}`;
    }
    return syncStatus.charAt(0).toUpperCase() + syncStatus.slice(1);
  };

  return (
    <div className="sync-status-item">
      {getStatusIcon()}
      <span>{getStatusText()}</span>
      {syncStatus === 'error' && (
        <button onClick={() => retrySync()}>Retry</button>
      )}
    </div>
  );
}
```

**Hook Integration** (`src/app/reader/hooks/useProgressSync.ts`):
```typescript
export function useProgressSync() {
  const [syncStatus, setSyncStatus] = useState<SyncStatus>('pending');
  const [lastSyncTime, setLastSyncTime] = useState<Date | null>(null);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const syncInterval = setInterval(async () => {
      try {
        setSyncStatus('syncing');
        await syncProgressToCloud();
        setSyncStatus('synced');
        setLastSyncTime(new Date());
        setError(null);
      } catch (err) {
        setSyncStatus('error');
        setError(err.message);
      }
    }, SYNC_INTERVAL);

    return () => clearInterval(syncInterval);
  }, []);

  return { syncStatus, lastSyncTime, error };
}
```

**Sync Timeout**: Default sync interval changed to 5 seconds (from 10 seconds in earlier versions) for more responsive feedback.

**User Benefits**:
- Immediate feedback on sync operations
- Confidence that changes are saved to cloud
- Clear error indication with retry option
- Visibility into last successful sync time

**Files**:
- `src/app/reader/components/ViewMenu.tsx`
- `src/app/reader/hooks/useProgressSync.ts`
- `src/services/syncService.ts`

---

## Version 0.9.64 - 0.9.67 Updates (f5b686ab → 33b2ba16)

### iOS In-App Purchase (IAP) Integration (v0.9.67, #1673, #1676, #1678)

**Major Feature**: Native In-App Purchase support for upgrading to Readest Premium on iOS.

**Overview**: iOS users can now purchase premium subscriptions directly within the app using Apple's native IAP system, without leaving the app or using external payment processors.

**Implementation Architecture**:

1. **Native Bridge Plugin** (#1673):
   - Tauri plugin for iOS IAP communication
   - Bridges Swift IAP APIs to JavaScript/TypeScript
   - Handles product fetching, purchase flow, and receipt validation

2. **Server-Side API** (#1676):
   - Receipt verification endpoint
   - Apple App Store receipt validation
   - Subscription status management
   - Database updates for premium status

3. **Frontend Integration** (#1678):
   - Premium upgrade UI in settings
   - Purchase flow with native payment sheet
   - Subscription status display
   - Auto-renewal management

**Native Bridge** (`src-tauri/src/plugins/iap.rs`):
```rust
use tauri::plugin::{Builder, TauriPlugin};
use tauri::{Runtime, Window};

#[tauri::command]
async fn fetch_products(product_ids: Vec<String>) -> Result<Vec<Product>, String> {
    // Call iOS StoreKit to fetch products
    ios::fetch_products(product_ids).await
}

#[tauri::command]
async fn purchase_product(product_id: String) -> Result<PurchaseResult, String> {
    // Initiate purchase flow
    ios::purchase(product_id).await
}

#[tauri::command]
async fn restore_purchases() -> Result<Vec<Purchase>, String> {
    // Restore previous purchases
    ios::restore_purchases().await
}

pub fn init<R: Runtime>() -> TauriPlugin<R> {
    Builder::new("iap")
        .invoke_handler(tauri::generate_handler![
            fetch_products,
            purchase_product,
            restore_purchases
        ])
        .build()
}
```

**iOS Native Implementation** (`src-tauri/ios/IAPPlugin.swift`):
```swift
import StoreKit

class IAPPlugin: NSObject, SKProductsRequestDelegate, SKPaymentTransactionObserver {
    private var productRequest: SKProductsRequest?
    private var products: [SKProduct] = []

    func fetchProducts(productIds: [String], completion: @escaping ([SKProduct]?, Error?) -> Void) {
        let request = SKProductsRequest(productIdentifiers: Set(productIds))
        request.delegate = self
        productRequest = request
        request.start()
    }

    func purchase(product: SKProduct) {
        let payment = SKPayment(product: product)
        SKPaymentQueue.default().add(payment)
    }

    func productsRequest(_ request: SKProductsRequest, didReceive response: SKProductsResponse) {
        self.products = response.products
        // Notify Tauri bridge of products
    }

    func paymentQueue(_ queue: SKPaymentQueue, updatedTransactions transactions: [SKPaymentTransaction]) {
        for transaction in transactions {
            switch transaction.transactionState {
            case .purchased:
                // Notify success and validate receipt
                validateReceipt(transaction: transaction)
            case .failed:
                // Notify failure
                SKPaymentQueue.default().finishTransaction(transaction)
            case .restored:
                // Handle restoration
                SKPaymentQueue.default().finishTransaction(transaction)
            default:
                break
            }
        }
    }
}
```

**Server API** (`src/pages/api/iap/verify.ts`):
```typescript
import { NextApiRequest, NextApiResponse } from 'next';
import { verifyAppleReceipt } from '@/utils/apple-receipt';
import { updateUserSubscription } from '@/services/supabase';

export default async function handler(
  req: NextApiRequest,
  res: NextApiResponse
) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  const { receiptData, userId } = req.body;

  try {
    // Verify receipt with Apple
    const verification = await verifyAppleReceipt(receiptData);

    if (!verification.valid) {
      return res.status(400).json({ error: 'Invalid receipt' });
    }

    // Extract subscription info
    const { productId, expiresDate, transactionId } = verification;

    // Update user subscription in database
    await updateUserSubscription(userId, {
      platform: 'ios',
      productId,
      expiresDate,
      transactionId,
      status: 'active'
    });

    res.status(200).json({
      success: true,
      subscription: {
        productId,
        expiresDate,
        status: 'active'
      }
    });
  } catch (error) {
    console.error('IAP verification error:', error);
    res.status(500).json({ error: 'Verification failed' });
  }
}
```

**Frontend Integration** (`src/components/PremiumUpgrade.tsx`):
```typescript
import { invoke } from '@tauri-apps/api/tauri';

const PremiumUpgrade = () => {
  const [products, setProducts] = useState<Product[]>([]);
  const [purchasing, setPurchasing] = useState(false);

  useEffect(() => {
    // Fetch available products
    invoke<Product[]>('fetch_products', {
      productIds: ['premium_monthly', 'premium_yearly']
    }).then(setProducts);
  }, []);

  const handlePurchase = async (productId: string) => {
    setPurchasing(true);
    try {
      const result = await invoke<PurchaseResult>('purchase_product', {
        productId
      });

      // Verify with backend
      await fetch('/api/iap/verify', {
        method: 'POST',
        body: JSON.stringify({
          receiptData: result.receiptData,
          userId: user.id
        })
      });

      toast.success('Premium activated!');
      refreshUserStatus();
    } catch (error) {
      toast.error('Purchase failed: ' + error.message);
    } finally {
      setPurchasing(false);
    }
  };

  return (
    <div className="premium-upgrade">
      <h2>Upgrade to Premium</h2>
      {products.map(product => (
        <div key={product.id} className="product-card">
          <h3>{product.title}</h3>
          <p>{product.description}</p>
          <span className="price">{product.price}</span>
          <button
            onClick={() => handlePurchase(product.id)}
            disabled={purchasing}
          >
            {purchasing ? 'Processing...' : 'Subscribe'}
          </button>
        </div>
      ))}
    </div>
  );
};
```

**Sandbox Environment** (#1679):
- TestFlight builds use Apple's sandbox environment
- Separate product IDs for testing
- No real charges during testing
- Receipt verification against sandbox URL

**Receipt Validation**:
- Server-side verification with Apple's verifyReceipt API
- Production URL: `https://buy.itunes.apple.com/verifyReceipt`
- Sandbox URL: `https://sandbox.itunes.apple.com/verifyReceipt`
- Auto-fallback from production to sandbox for testing

**Files**:
- `src-tauri/src/plugins/iap.rs` - Rust IAP plugin
- `src-tauri/ios/IAPPlugin.swift` - iOS native IAP
- `src/pages/api/iap/verify.ts` - Receipt verification API
- `src/pages/api/iap/webhook.ts` - Apple server notifications
- `src/components/PremiumUpgrade.tsx` - Premium upgrade UI
- `src/services/iapService.ts` - IAP service abstraction

### Increased Cloud Sync Storage for Premium Users (v0.9.67, #1696)

**Feature**: Premium users receive significantly increased cloud storage quota.

**Storage Quotas**:

| Plan | Storage Quota | Books Limit | Notes/Highlights |
|------|---------------|-------------|------------------|
| **Free** | 100 MB | ~100 books | Unlimited |
| **Premium** | 1 GB | ~1,000 books | Unlimited |
| **Pro** | 10 GB | ~10,000 books | Unlimited |

**Implementation** (`src/services/quotaManager.ts`):
```typescript
const STORAGE_QUOTAS = {
  free: 100 * 1024 * 1024,      // 100 MB
  premium: 1024 * 1024 * 1024,  // 1 GB
  pro: 10 * 1024 * 1024 * 1024  // 10 GB
};

const checkStorageQuota = async (userId: string): Promise<QuotaStatus> => {
  const { data: user } = await supabase
    .from('users')
    .select('subscription_tier')
    .eq('id', userId)
    .single();

  const tier = user?.subscription_tier || 'free';
  const quota = STORAGE_QUOTAS[tier];

  const { data: usage } = await supabase
    .rpc('get_user_storage_usage', { user_id: userId });

  return {
    used: usage,
    total: quota,
    available: quota - usage,
    percentUsed: (usage / quota) * 100
  };
};
```

**Quota Display** (`src/components/StorageQuota.tsx`):
```typescript
const StorageQuota = () => {
  const { used, total, percentUsed } = useStorageQuota();

  return (
    <div className="storage-quota">
      <div className="quota-bar">
        <div
          className="quota-used"
          style={{ width: `${percentUsed}%` }}
        />
      </div>
      <span className="quota-text">
        {formatBytes(used)} / {formatBytes(total)} used
      </span>
      {percentUsed > 80 && (
        <div className="quota-warning">
          Storage almost full. Consider upgrading to Premium.
        </div>
      )}
    </div>
  );
};
```

**Priority Sync**:
- Premium users get faster sync speeds
- Higher priority in sync queue
- Concurrent sync for multiple devices
- Real-time sync (vs. periodic for free users)

**Enhanced Backup**:
- Automatic daily backups
- 30-day backup retention (vs. 7 days for free)
- Point-in-time recovery
- Backup download option

**Files**:
- `src/services/quotaManager.ts` - Quota management
- `src/components/StorageQuota.tsx` - Quota display
- `src/pages/api/storage/usage.ts` - Usage tracking API
- Database: `users` table with `subscription_tier` column

### API Enhancements for IAP

**Ensure Proper String Decoding on Edge Runtimes** (#1680):
- Fixed string encoding issues in Cloudflare Workers
- Proper UTF-8 handling for international characters
- Base64 decoding for receipt data

**Use Node API Endpoint for IAP Verifying** (#1683):
- Switched from Edge runtime to Node runtime
- Better Apple receipt verification library support
- More reliable HTTPS connections to Apple servers
- Improved error handling and logging

**Batch Updating Daily Usage Key in KV** (#1694):
- Optimized daily usage tracking
- Batch updates to reduce API calls
- Cloudflare Workers KV for fast access
- Quota tracking for translation and other services

**Files**:
- `src/pages/api/iap/verify.ts` - Node-based verification
- `src/utils/kv.ts` - KV storage utilities
- `src/services/usageTracker.ts` - Usage tracking

---

## Version 0.9.79 - 0.9.82 Updates (cc3cc58d → e1691661)

### Metadata Hash Book Aggregation (v0.9.80, #2062, #2063)

**Major Feature**: Use metadata hash to aggregate and sync progress across different editions/versions of the same book.

**Problem**: Previously, each unique book file (different formats, editions, or sources) was treated as a separate book, even if they contained the same content. Users had separate reading positions for:
- EPUB vs PDF versions of the same book
- Different editions from different publishers
- Books converted through Calibre with different metadata
- Same book from different sources

**Solution**: Introduced `meta_hash` - a content-based hash that identifies books by their actual content rather than file characteristics.

**Implementation** (`src/utils/bookHash.ts`):
```typescript
// Generate metadata hash from book content
const generateMetaHash = (book: BookMetadata): string => {
  // Use normalized title, author, and content sample
  const normalizedTitle = normalizeString(book.title);
  const normalizedAuthor = normalizeString(book.authors?.join(' ') || '');

  // Hash based on content, not file
  return md5(`${normalizedTitle}:${normalizedAuthor}`);
};

// Normalize strings for consistent hashing
const normalizeString = (str: string): string => {
  return str
    .toLowerCase()
    .replace(/[^\w\s]/g, '') // Remove punctuation
    .replace(/\s+/g, ' ')     // Normalize whitespace
    .trim();
};
```

**Database Schema Update**:
```sql
-- Add meta_hash column to books table
ALTER TABLE books ADD COLUMN meta_hash TEXT;

-- Add index for efficient meta_hash queries
CREATE INDEX idx_books_meta_hash ON books(user_id, meta_hash);

-- Update book_configs to use meta_hash
ALTER TABLE book_configs ADD COLUMN meta_hash TEXT;
CREATE INDEX idx_configs_meta_hash ON book_configs(user_id, meta_hash);
```

**Progress Sync with Aggregation** (v0.9.80, #2062):

**Overview**: Reading progress is now aggregated across all versions of the same book using `meta_hash`.

**Sync Flow**:
```typescript
// When syncing progress
const syncProgress = async (book: BookMetadata, position: Location) => {
  const metaHash = generateMetaHash(book);

  // Find all books with same meta_hash
  const { data: relatedBooks } = await supabase
    .from('books')
    .select('*')
    .eq('user_id', userId)
    .eq('meta_hash', metaHash);

  // Sync progress to all related books
  for (const relatedBook of relatedBooks) {
    await supabase
      .from('book_configs')
      .upsert({
        book_hash: relatedBook.book_hash,
        meta_hash: metaHash,  // ← Key addition
        progress: position.progress,
        location: position.cfi,
        updated_at: new Date().toISOString()
      });
  }
};
```

**User Experience**:
1. User reads EPUB version of book to 50%
2. User switches to PDF version of same book
3. PDF automatically opens at 50% (synced via meta_hash)
4. User's notes and highlights from EPUB also appear in PDF
5. Works across devices and formats seamlessly

**KOReader Integration** (v0.9.80, #2063):

**Overview**: Extended metadata hash to support cross-app synchronization with KOReader.

**KOReader Compatibility**:
- KOReader uses similar content-based hashing
- Readest now compatible with KOReader's sync protocol
- Users can switch between Readest and KOReader while maintaining progress

**Implementation** (`src/services/kosync.ts`):
```typescript
// Extract meta_hash for KOReader compatibility
const extractMetaHashForKOReader = (book: BookMetadata): string => {
  // KOReader uses title + author for book identification
  // Make extraction more robust for Calibre conversions and metadata edits

  // Try multiple sources for title/author
  const title =
    book.title ||
    book.metadata?.title ||
    extractTitleFromFilename(book.filename);

  const author =
    book.authors?.join(' ') ||
    book.metadata?.creator ||
    'Unknown';

  return generateMetaHash({ title, authors: [author] });
};
```

**Robust Meta Hash Extraction** (v0.9.82, #2154):
- More robust extraction for Calibre-converted books
- Handles metadata edits without losing sync
- Fallback to filename-based identification if metadata missing
- Compatible with various ebook management tools

**Benefits**:
- Single reading position across all book versions
- Consolidated annotations and highlights
- Cross-app compatibility (KOReader, Calibre)
- No manual book merging required
- Automatic version detection and aggregation

**Edge Cases Handled**:
- Books with slightly different titles (e.g., "The Lord of the Rings" vs "Lord of the Rings")
- Different author formats (e.g., "J.R.R. Tolkien" vs "Tolkien, J.R.R.")
- Books with missing or incorrect metadata
- Calibre conversions with modified metadata
- Multiple file formats of same book

**Files**:
- `src/utils/bookHash.ts` - Meta hash generation
- `src/services/kosync.ts` - KOReader integration
- `src/hooks/useProgressSync.ts` - Progress sync with aggregation
- Database migrations for `meta_hash` column

### Sync Reliability Improvements (v0.9.80, #2041, #2112, #2115)

**Resolve Invalid Token Issues** (v0.9.80, #2041):
- Fixed issue where invalid/expired tokens were occasionally used with storage API
- Better token validation before making API calls
- Automatic token refresh when approaching expiration
- Improved error handling and user notification

**Implementation**:
```typescript
const ensureValidToken = async (): Promise<string> => {
  const token = getStoredToken();

  // Validate token before use
  if (!token || isTokenExpired(token)) {
    // Attempt to refresh
    const newToken = await refreshAccessToken();

    if (!newToken) {
      // Force re-authentication
      throw new AuthError('Session expired. Please log in again.');
    }

    return newToken;
  }

  return token;
};
```

**Force Full Sync Periodically** (v0.9.80, #2112):
- Automatic full sync after a certain period (7 days by default)
- Prevents drift between local and remote data
- Resolves inconsistencies from failed incremental syncs
- Configurable sync interval

**Handle Incomplete Config Data** (v0.9.80, #2115):
- Gracefully handle incomplete or corrupted sync data
- Validate incoming config data before applying
- Fallback to local data if remote data is invalid
- Better error reporting for debugging

**Files**:
- `src/utils/access.ts` - Token validation
- `src/services/syncService.ts` - Full sync scheduling
- `src/hooks/useSync.ts` - Config validation

---

## Future Enhancements

### Planned Features

1. **Real-time sync**: Use Supabase Realtime for instant sync
2. **Offline mode**: Full offline support with background sync queue
3. **Conflict UI**: Show conflicts to user for manual resolution
4. **Versioning**: Keep history of all changes for rollback
5. **Encryption**: End-to-end encryption for sensitive notes
6. **Selective sync**: Choose which books to sync
7. **Family sharing**: Share books and notes with family members
8. **Team plans**: Shared libraries for organizations

### Integration Opportunities

1. **Calibre sync**: Import/export from Calibre libraries
2. **Kindle sync**: Sync with Kindle highlights
3. **Goodreads**: Sync reading status with Goodreads
4. **Apple Books**: Import highlights from Apple Books
5. **Export**: Export all notes to Markdown, PDF, or Notion

---

## Version 0.9.83 - 0.9.90 Updates (e1691661 → dd5371d2)

### Cloud Storage Expansion (v0.9.89, #2325, #2331)

**Major Feature**: Expand cloud storage with one-time payment or In-App Purchase (IAP).

**Overview**: Users can now purchase additional cloud storage for syncing more books and annotations.

**Storage Tiers**:
- **Free**: 100 MB cloud storage
- **Expanded**: 1 GB cloud storage (one-time payment)
- **Premium**: 10 GB cloud storage (subscription or IAP)

**Purchase Methods**:
1. **Web/Desktop** (#2325): One-time payment via Stripe
2. **iOS** (#2331): In-App Purchase through App Store
3. **Android** (#2331): In-App Purchase through Google Play

**Implementation** (`src/app/settings/components/StorageUpgrade.tsx`):
```typescript
interface StorageTier {
  name: string;
  storage: number;  // in MB
  price: number;    // in USD
  iap_product_id?: string;  // For mobile IAP
}

const STORAGE_TIERS: StorageTier[] = [
  {
    name: 'Free',
    storage: 100,
    price: 0
  },
  {
    name: 'Expanded',
    storage: 1024,  // 1 GB
    price: 4.99,
    iap_product_id: 'com.readest.storage.1gb'
  },
  {
    name: 'Premium',
    storage: 10240,  // 10 GB
    price: 9.99,
    iap_product_id: 'com.readest.storage.10gb'
  }
];

const StorageUpgradeButton = ({ tier }: { tier: StorageTier }) => {
  const [isPurchasing, setIsPurchasing] = useState(false);

  const handlePurchase = async () => {
    setIsPurchasing(true);

    try {
      if (isNativePlatform()) {
        // Use In-App Purchase
        await purchaseViaIAP(tier.iap_product_id);
      } else {
        // Use web payment (Stripe)
        await purchaseViaStripe(tier.name, tier.price);
      }

      toast.success('Storage upgraded successfully!');
      await refreshStorageQuota();
    } catch (error) {
      toast.error(`Purchase failed: ${error.message}`);
    } finally {
      setIsPurchasing(false);
    }
  };

  return (
    <div className="storage-tier-card">
      <h3>{tier.name}</h3>
      <p>{tier.storage} MB cloud storage</p>
      <p className="price">${tier.price}</p>

      <button
        onClick={handlePurchase}
        disabled={isPurchasing}
        className="btn btn-primary"
      >
        {isPurchasing ? 'Processing...' : 'Upgrade'}
      </button>
    </div>
  );
};
```

**IAP Implementation** (iOS/Android):
```typescript
// src/services/iapService.ts
class IAPService {
  async purchaseViaIAP(productId: string): Promise<void> {
    // Request purchase through native platform
    const result = await invoke('iap_purchase', { product_id: productId });

    if (result.success) {
      // Verify purchase with backend
      await this.verifyPurchase(result.receipt);

      // Update user's storage quota
      await this.updateStorageQuota(productId);
    } else {
      throw new Error(result.error || 'Purchase failed');
    }
  }

  async verifyPurchase(receipt: string): Promise<void> {
    // Send receipt to backend for verification
    const response = await fetch('/api/iap/verify', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ receipt })
    });

    if (!response.ok) {
      throw new Error('Receipt verification failed');
    }
  }

  async updateStorageQuota(productId: string): Promise<void> {
    // Update user quota in database
    await supabase.from('user_storage').update({
      quota_mb: this.getStorageForProduct(productId),
      updated_at: new Date().toISOString()
    });
  }

  private getStorageForProduct(productId: string): number {
    const tier = STORAGE_TIERS.find(t => t.iap_product_id === productId);
    return tier?.storage || 100;
  }
}
```

**Backend Verification** (Rust - Tauri):
```rust
// src-tauri/src/iap/verify.rs
#[tauri::command]
pub async fn iap_purchase(product_id: String) -> Result<PurchaseResult, String> {
    #[cfg(target_os = "ios")]
    {
        use storekit::*;

        let payment = SKPayment::with_product_identifier(&product_id);
        let queue = SKPaymentQueue::default_queue();

        queue.add_payment(payment).await
            .map(|transaction| PurchaseResult {
                success: true,
                receipt: transaction.transactionReceipt,
                error: None
            })
            .map_err(|e| format!("iOS IAP failed: {}", e))
    }

    #[cfg(target_os = "android")]
    {
        // Android IAP implementation
        android_purchase_product(product_id).await
    }

    #[cfg(not(any(target_os = "ios", target_os = "android")))]
    {
        Err("IAP only available on mobile platforms".to_string())
    }
}
```

**Usage Monitoring**:
```typescript
// Show current storage usage
const StorageUsageDisplay = () => {
  const { used, quota } = useStorageQuota();
  const percentUsed = (used / quota) * 100;

  return (
    <div className="storage-usage">
      <div className="progress-bar">
        <div
          className="progress-fill"
          style={{ width: `${percentUsed}%` }}
        />
      </div>

      <p>
        {formatBytes(used)} / {formatBytes(quota)} used ({percentUsed.toFixed(1)}%)
      </p>

      {percentUsed > 80 && (
        <button onClick={() => navigateToUpgrade()}>
          Upgrade Storage
        </button>
      )}
    </div>
  );
};
```

**Benefits**:
- Sync more books across devices
- Store larger book collections
- Preserve all annotations and notes
- Lifetime access (one-time purchase)
- Flexible payment options (web, iOS, Android)

**Files**:
- `src/app/settings/components/StorageUpgrade.tsx` - Upgrade UI
- `src/services/iapService.ts` - IAP service abstraction
- `src-tauri/src/iap/verify.rs` - Native IAP verification
- `apps/readest-api/iap/verify.ts` - Backend receipt verification

---

**Last Updated:** Documentation for commits up to dd5371d2 (November 2025, v0.9.90)
**Related Documents:** [cross-platform-support](../cross-platform-support/index.md), [library-management](../library-management/index.md)
