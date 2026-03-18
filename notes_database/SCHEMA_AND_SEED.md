# NoteMaster PostgreSQL Schema & Seed (Executed)

This database is managed by `notes_database`.  
Connection command source: `notes_database/db_connection.txt`

## Connection
Run (example from `db_connection.txt`):
- `psql postgresql://appuser:dbuser123@localhost:5000/myapp`

All statements below were executed **one at a time** via:
- `psql ... -c "SINGLE_SQL_STATEMENT;"`

---

## Extensions

1. `CREATE EXTENSION IF NOT EXISTS pgcrypto;`  
   Used for `gen_random_uuid()` defaults.

2. `CREATE EXTENSION IF NOT EXISTS pg_trgm;`  
   Used for trigram GIN indexes to speed up search.

---

## Tables

### users
- Primary key: `id uuid`
- Unique: `email`

```sql
CREATE TABLE IF NOT EXISTS users (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  email text NOT NULL UNIQUE,
  password_hash text NOT NULL,
  display_name text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
```

Index:
```sql
CREATE INDEX IF NOT EXISTS idx_users_created_at ON users(created_at);
```

### notes
- Primary key: `id uuid`
- FK: `user_id -> users(id)` (cascade delete)
- Pin flag: `is_pinned boolean`

```sql
CREATE TABLE IF NOT EXISTS notes (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  title text NOT NULL DEFAULT '',
  content text NOT NULL DEFAULT '',
  is_pinned boolean NOT NULL DEFAULT false,
  created_at timestamptz NOT NULL DEFAULT now(),
  updated_at timestamptz NOT NULL DEFAULT now()
);
```

Indexes:
```sql
CREATE INDEX IF NOT EXISTS idx_notes_user_updated_at ON notes(user_id, updated_at DESC);
CREATE INDEX IF NOT EXISTS idx_notes_user_pinned_updated_at ON notes(user_id, is_pinned, updated_at DESC);
CREATE INDEX IF NOT EXISTS idx_notes_title_trgm ON notes USING gin (title gin_trgm_ops);
CREATE INDEX IF NOT EXISTS idx_notes_content_trgm ON notes USING gin (content gin_trgm_ops);
```

### tags
- Primary key: `id uuid`
- FK: `user_id -> users(id)` (cascade delete)
- Unique per user: `(user_id, name)`

```sql
CREATE TABLE IF NOT EXISTS tags (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now(),
  UNIQUE(user_id, name)
);
```

Index:
```sql
CREATE INDEX IF NOT EXISTS idx_tags_user_name ON tags(user_id, name);
```

### note_tags
- Join table (many-to-many): notes <-> tags
- PK: `(note_id, tag_id)`
- Cascades on delete of either side

```sql
CREATE TABLE IF NOT EXISTS note_tags (
  note_id uuid NOT NULL REFERENCES notes(id) ON DELETE CASCADE,
  tag_id uuid NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
  created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY(note_id, tag_id)
);
```

Index:
```sql
CREATE INDEX IF NOT EXISTS idx_note_tags_tag_id ON note_tags(tag_id);
```

### favorites
- A user favorites a note
- PK: `(user_id, note_id)`
- Cascades on delete of either side

```sql
CREATE TABLE IF NOT EXISTS favorites (
  user_id uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  note_id uuid NOT NULL REFERENCES notes(id) ON DELETE CASCADE,
  created_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY(user_id, note_id)
);
```

Index:
```sql
CREATE INDEX IF NOT EXISTS idx_favorites_note_id ON favorites(note_id);
```

---

## Seed Data (Demo)

### Demo user
```sql
INSERT INTO users (id,email,password_hash,display_name)
VALUES ('00000000-0000-0000-0000-000000000001','demo@notemaster.local','demo-password-hash','Demo User')
ON CONFLICT (email) DO NOTHING;
```

### Tags
```sql
INSERT INTO tags (id,user_id,name)
VALUES ('00000000-0000-0000-0000-000000000101','00000000-0000-0000-0000-000000000001','work')
ON CONFLICT (user_id,name) DO NOTHING;
```

```sql
INSERT INTO tags (id,user_id,name)
VALUES ('00000000-0000-0000-0000-000000000102','00000000-0000-0000-0000-000000000001','personal')
ON CONFLICT (user_id,name) DO NOTHING;
```

```sql
INSERT INTO tags (id,user_id,name)
VALUES ('00000000-0000-0000-0000-000000000103','00000000-0000-0000-0000-000000000001','ideas')
ON CONFLICT (user_id,name) DO NOTHING;
```

### Notes
```sql
INSERT INTO notes (id,user_id,title,content,is_pinned)
VALUES ('00000000-0000-0000-0000-000000001001','00000000-0000-0000-0000-000000000001','Welcome to NoteMaster','This is your first note. You can edit, tag, favorite, and pin notes.',true)
ON CONFLICT (id) DO NOTHING;
```

```sql
INSERT INTO notes (id,user_id,title,content,is_pinned)
VALUES ('00000000-0000-0000-0000-000000001002','00000000-0000-0000-0000-000000000001','Work TODO','- Review PRs\n- Plan sprint\n- Update documentation',false)
ON CONFLICT (id) DO NOTHING;
```

```sql
INSERT INTO notes (id,user_id,title,content,is_pinned)
VALUES ('00000000-0000-0000-0000-000000001003','00000000-0000-0000-0000-000000000001','App Ideas','- Offline mode\n- Markdown editor\n- Shareable links',false)
ON CONFLICT (id) DO NOTHING;
```

### Note tags
```sql
INSERT INTO note_tags (note_id,tag_id)
VALUES ('00000000-0000-0000-0000-000000001002','00000000-0000-0000-0000-000000000101')
ON CONFLICT DO NOTHING;
```

```sql
INSERT INTO note_tags (note_id,tag_id)
VALUES ('00000000-0000-0000-0000-000000001003','00000000-0000-0000-0000-000000000103')
ON CONFLICT DO NOTHING;
```

### Favorites
```sql
INSERT INTO favorites (user_id,note_id)
VALUES ('00000000-0000-0000-0000-000000000001','00000000-0000-0000-0000-000000001001')
ON CONFLICT DO NOTHING;
```

---

## Quick sanity checks (optional)
These were run to verify seed counts:

- `SELECT 'users' AS table, count(*) AS count FROM users;`
- `SELECT 'notes' AS table, count(*) AS count FROM notes;`
- `SELECT 'tags' AS table, count(*) AS count FROM tags;`
- `SELECT 'note_tags' AS table, count(*) AS count FROM note_tags;`
- `SELECT 'favorites' AS table, count(*) AS count FROM favorites;`
