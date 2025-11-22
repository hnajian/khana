# Cloud Notes Synchronization

**Added**: December 2024 (commit 83cb7166 - Dec 24, 2024)

## Overview

The Cloud Notes Synchronization feature enables automatic synchronization of annotations and notes across devices via Supabase cloud storage. This allows users to access their highlights, notes, and bookmarks from any device.

## Key Components

**Primary Files:**
- **`src/app/reader/hooks/useNotesSync.ts`** - Hook for automatic notes synchronization
- **`src/hooks/useSync.ts`** - Shared sync utilities
- **`src/pages/api/sync.ts`** - Cloud sync API endpoint
- **`src/utils/supabase.ts`** - Supabase client utilities
- **`src/context/SyncContext.tsx`** - Cloud sync state context

## Architecture

**Sync Flow:**
1. **User authenticates** via OAuth (see `src/app/auth/`)
2. **On book open**, `useNotesSync` hook pulls latest notes from cloud
3. **Local changes tracked** - New/modified annotations detected based on `updatedAt` timestamp
4. **Periodic sync** - Notes automatically synced at regular intervals (SYNC_NOTES_INTERVAL_SEC)
5. **Conflict resolution** - Merges local and remote notes, prioritizing newer timestamps
6. **Soft deletes** - Deleted notes marked with `deletedAt` timestamp, synced across devices

## Notes Sync Hook

**`useNotesSync(bookKey: string)`**

Automatically invoked in `Annotator.tsx:40` for every book:

```typescript
useNotesSync(bookKey);
```

**How it works:**

1. **Initial Pull** (on mount):
   ```typescript
   useEffect(() => {
     if (!user) return;
     syncNotes([], bookHash, 'pull');
   }, []);
   ```
   Downloads all existing notes for the book from cloud.

2. **Change Detection**:
   ```typescript
   const getNewNotes = () => {
     const bookNotes = config.booknotes ?? [];
     const newNotes = bookNotes.filter(
       (note) => lastSyncedAtNotes < note.updatedAt ||
                 lastSyncedAtNotes < (note.deletedAt ?? 0)
     );
     return newNotes;
   };
   ```
   Identifies notes modified since last sync.

3. **Periodic Sync**:
   - Syncs every `SYNC_NOTES_INTERVAL_SEC` seconds (default: configurable in constants)
   - Uses debouncing to avoid excessive sync calls
   - Mode: `'both'` (pushes local changes AND pulls remote changes)

4. **Merge Strategy**:
   ```typescript
   const mergedNotes = [
     ...oldNotes.filter((oldNote) =>
       !newNotes.some((newNote) => newNote.id === oldNote.id)
     ),
     ...newNotes,
   ];
   ```
   Remote notes override local notes with same ID.

## Data Structure

**BookNote Interface** (synchronized to cloud):
```typescript
interface BookNote {
  id: string;                // Unique note ID
  type: 'annotation' | 'excerpt' | 'bookmark';
  cfi: string;               // EPUB CFI or PDF location
  style?: 'highlight' | 'underline' | 'squiggly';
  color?: string;            // Highlight color
  text?: string;             // Selected text
  note?: string;             // User's note
  createdAt: number;         // Creation timestamp
  updatedAt: number;         // Last modification timestamp
  deletedAt?: number;        // Soft delete timestamp
  bookHash: string;          // Book identifier (MD5)
}
```

## Integration Points

**Annotator Actions that Trigger Sync:**
- **Highlight**: Creates annotation → triggers sync
- **Annotate**: Adds note to annotation → triggers sync
- **Copy**: Creates excerpt → triggers sync
- **Delete Highlight**: Soft-deletes annotation (sets `deletedAt`) → triggers sync

**Authentication Required:**
- Notes sync only active when `user` is authenticated (via `useAuth()`)
- Unauthenticated users store notes locally only
- Upon login, local notes can be merged with cloud notes

## Sync Timing

**Constants** (from `src/services/constants.ts`):
```typescript
export const SYNC_NOTES_INTERVAL_SEC = 30;  // Sync every 30 seconds
```

**Debouncing Logic:**
- If notes change within sync interval, sync is delayed
- Prevents excessive API calls when user is actively annotating
- Final sync occurs after user stops making changes

## Cloud Storage

**Supabase Backend:**
- Notes stored in Supabase PostgreSQL database
- Indexed by `bookHash` and `userId`
- Real-time subscriptions possible (future enhancement)

**API Endpoint** (`src/pages/api/sync.ts`):
- POST `/api/sync` - Sync notes for a specific book
- Request body: `{ bookHash, notes, lastSyncedAt }`
- Response: `{ syncedNotes, lastSyncedAtNotes }`

## Modification Guidelines for AI Agents

### Adjusting Sync Interval

To change sync frequency:

1. **Edit `src/services/constants.ts`**:
   ```typescript
   export const SYNC_NOTES_INTERVAL_SEC = 60;  // Sync every 60 seconds
   ```

2. **For immediate sync on every change**:
   Modify `useNotesSync.ts` to sync in `useEffect` without debouncing.

### Adding Real-time Sync

To implement real-time sync with Supabase subscriptions:

1. **Subscribe to changes** in `useNotesSync.ts`:
   ```typescript
   useEffect(() => {
     const subscription = supabase
       .from('notes')
       .on('INSERT', handleRemoteInsert)
       .on('UPDATE', handleRemoteUpdate)
       .on('DELETE', handleRemoteDelete)
       .subscribe();

     return () => subscription.unsubscribe();
   }, [bookHash]);
   ```

2. **Handle remote changes**:
   ```typescript
   const handleRemoteInsert = (payload) => {
     const newNote = payload.new;
     if (newNote.bookHash === bookHash) {
       // Merge into local notes
       setConfig(bookKey, {
         ...config,
         booknotes: [...config.booknotes, newNote]
       });
     }
   };
   ```

### Sync Conflict Resolution

Current strategy: **Last Write Wins** (remote notes override local with same ID)

**Alternative strategies:**

1. **Manual Conflict Resolution**:
   ```typescript
   if (localNote.updatedAt > remoteNote.updatedAt) {
     // Keep local
   } else if (remoteNote.updatedAt > localNote.updatedAt) {
     // Keep remote
   } else {
     // Show conflict dialog to user
     showConflictDialog(localNote, remoteNote);
   }
   ```

2. **Merge Content**:
   ```typescript
   const mergedNote = {
     ...localNote,
     note: localNote.note + '\n---\n' + remoteNote.note,
     updatedAt: Date.now(),
   };
   ```

### Offline Support

Current implementation handles offline gracefully:
- Notes saved locally when offline
- Sync resumes when connection restored
- Use `useSync()` hook's connection status

**Enhancement: Sync Queue**:
```typescript
// In useNotesSync.ts
const syncQueue = useRef<BookNote[]>([]);

const queueNoteForSync = (note: BookNote) => {
  syncQueue.current.push(note);
  if (navigator.onLine) {
    flushSyncQueue();
  }
};

window.addEventListener('online', flushSyncQueue);
```

## Common Issues

**Issue: Notes not syncing**
- Check user is authenticated (`useAuth()`)
- Verify `SYNC_NOTES_INTERVAL_SEC` constant
- Inspect browser console for API errors
- Check Supabase connection in `src/utils/supabase.ts`

**Issue: Duplicate notes after sync**
- Verify `id` field is unique per note
- Check merge logic in `useNotesSync.ts:64-68`
- Ensure `uniqueId()` generates globally unique IDs

**Issue: Deleted notes reappearing**
- Verify soft delete sets `deletedAt` timestamp
- Check sync includes deleted notes (`getNewNotes()` filters)
- Ensure cloud API respects `deletedAt` field

**Issue: Sync too frequent / too slow**
- Adjust `SYNC_NOTES_INTERVAL_SEC` in constants
- Check debouncing logic in `useNotesSync.ts:36-57`
- Monitor network tab for API call frequency

## Related Files

| File | Purpose |
|------|---------|
| `useNotesSync.ts:40` | Hook invocation in Annotator |
| `useNotesSync.ts:17` | Initial pull on mount |
| `useNotesSync.ts:42` | Periodic sync with debouncing |
| `useNotesSync.ts:60` | Merge remote notes into local |
| `useSync.ts:syncNotes` | Shared sync function |
| `api/sync.ts` | Cloud API endpoint |
| `constants.ts:SYNC_NOTES_INTERVAL_SEC` | Sync interval config |

---

**Related**: [index.md](./index.md) (Main annotation system documentation)
