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

## Future Enhancements

### Planned Features

1. **Real-time sync**: Use Supabase Realtime for instant sync
2. **Offline mode**: Full offline support with background sync queue
3. **Conflict UI**: Show conflicts to user for manual resolution
4. **Versioning**: Keep history of all changes for rollback
5. **Encryption**: End-to-end encryption for sensitive notes
6. **Selective sync**: Choose which books to sync
7. **Family sharing**: Share books and notes with family members

### Integration Opportunities

1. **Calibre sync**: Import/export from Calibre libraries
2. **Kindle sync**: Sync with Kindle highlights
3. **Goodreads**: Sync reading status with Goodreads
4. **Apple Books**: Import highlights from Apple Books
5. **Export**: Export all notes to Markdown, PDF, or Notion

---

**Last Updated:** Documentation for commits up to def157ca (November 2025)
**Related Documents:** [feature-cross-platform-support.md](./feature-cross-platform-support.md), [feature-library-management.md](./feature-library-management.md)
