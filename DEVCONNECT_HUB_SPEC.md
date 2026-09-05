# DevConnect Hub — System Design Specification

A collaboration platform for IT students and junior developers: post updates, showcase projects, get code help, and find teammates.

---

## 1. Information Architecture

```
┌────────────────────────────────────────────────────────────────┐
│  Topbar (mobile only) — logo · menu · search · stats toggle     │
├───────────────┬──────────────────────────────────┬──────────────┤
│  Sidebar       │  Main Feed                        │  Right Rail  │
│  (280px)       │  (fluid, max-w-2xl)                │  (320px)     │
│                │                                    │              │
│  Profile card  │  Search + category filter chips    │  Looking for │
│  Nav links     │  Post Composer (tabbed editor)      │  Collaborat- │
│  Trending tags │  Feed cards (infinite list)         │  ors reel    │
│                │    └─ nested comment thread          │  Online now  │
│                │                                    │  Platform    │
│                │                                    │  stats       │
└───────────────┴──────────────────────────────────┴──────────────┘
```

Breakpoint: `lg` (1024px). Below it, the sidebar and right rail become slide-in drawers over a dimmed overlay, triggered from the mobile topbar.

## 2. Post Taxonomy

| Category | Composer tab (styled as a file) | Accent | Required fields |
|---|---|---|---|
| General Discussion | `update.md` | Electric Blue `#58a6ff` | body text |
| Project Showcase | `showcase.json` | Purple `#bc8cff` | body, repo URL, live URL, stack tags |
| Code Help / Bug Squashing | `bug.js` | Amber `#f0883e` | body, language, code snippet |
| Team Finder | `team.yml` | Emerald `#2ea043` | body, roles needed, deadline |

Every post also carries free-form topic tags (`#java`, `#firebase`, …) independent of its category.

## 3. Firebase (Firestore) Integration Architecture

Firestore is the primary target (document model maps cleanly to feed cards + subcollections for comments). Realtime Database notes are included where behavior differs.

### 3.1 Collection layout

```
users/{uid}
posts/{postId}
posts/{postId}/comments/{commentId}
posts/{postId}/votes/{uid}          // one doc per voter, doc ID = uid, prevents double-voting
```

### 3.2 Document schemas

```json
// users/{uid}
{
  "displayName": "Naty Gana",
  "handle": "nategana",
  "avatarInitials": "NG",
  "roles": ["Student", "Full-Stack"],
  "school": "Pamantasan ng Cabuyao",
  "githubUrl": "https://github.com/NateGana",
  "bio": "BSIT student, building small tools that ship.",
  "createdAt": "2026-08-28T09:00:00Z",
  "postCount": 12,
  "followerCount": 34
}
```

```json
// posts/{postId}
{
  "authorId": "uid_123",
  "authorName": "Naty Gana",
  "authorRoles": ["Student", "Full-Stack"],
  "category": "showcase",              // general | showcase | codehelp | team
  "title": "Marsline Bus Company — ticketing dashboard",
  "body": "Built a role-based dashboard for our SIA capstone...",
  "codeSnippet": null,                  // string | null — only for codehelp
  "codeLanguage": null,                 // e.g. "javascript" — only for codehelp
  "repoUrl": "https://github.com/NateGana/marsline-dashboard",
  "liveUrl": "https://marsline.example.dev",
  "rolesNeeded": [],                    // only for team posts, e.g. ["UI Designer","QA"]
  "deadline": null,                     // ISO date, only for team posts
  "tags": ["firebase", "tailwindcss", "sql"],
  "solved": false,                      // only meaningful for codehelp posts
  "upvoteCount": 18,
  "commentCount": 5,
  "createdAt": "2026-08-27T14:20:00Z"
}
```

```json
// posts/{postId}/comments/{commentId}
{
  "authorId": "uid_456",
  "authorName": "Klyde",
  "parentCommentId": null,             // set for one-level-deep replies
  "body": "Try checking your Firestore security rules first.",
  "codeSnippet": null,
  "codeLanguage": null,
  "upvoteCount": 3,
  "createdAt": "2026-08-27T15:02:00Z"
}
```

### 3.3 Web SDK v9+ integration snippet

```javascript
import { initializeApp } from "firebase/app";
import {
  getFirestore, collection, doc, addDoc, updateDoc,
  increment, serverTimestamp, query, where, orderBy,
  onSnapshot, runTransaction
} from "firebase/firestore";

const app = initializeApp(firebaseConfig);
const db = getFirestore(app);

// --- Create a post ---------------------------------------------------
async function publishPost(postData) {
  return addDoc(collection(db, "posts"), {
    ...postData,
    upvoteCount: 0,
    commentCount: 0,
    createdAt: serverTimestamp(),
  });
}

// --- Add a comment and keep the parent's commentCount in sync -------
async function addComment(postId, commentData) {
  const postRef = doc(db, "posts", postId);
  const commentsRef = collection(postRef, "comments");
  await addDoc(commentsRef, { ...commentData, upvoteCount: 0, createdAt: serverTimestamp() });
  await updateDoc(postRef, { commentCount: increment(1) });
}

// --- Toggle an upvote, guarding against double-votes with a vote doc -
async function toggleUpvote(postId, uid) {
  const postRef = doc(db, "posts", postId);
  const voteRef = doc(db, "posts", postId, "votes", uid);
  await runTransaction(db, async (tx) => {
    const voteSnap = await tx.get(voteRef);
    if (voteSnap.exists()) {
      tx.delete(voteRef);
      tx.update(postRef, { upvoteCount: increment(-1) });
    } else {
      tx.set(voteRef, { votedAt: serverTimestamp() });
      tx.update(postRef, { upvoteCount: increment(1) });
    }
  });
}

// --- Live feed subscription, filtered by category --------------------
function subscribeToFeed(category, onChange) {
  const base = collection(db, "posts");
  const q = category === "all"
    ? query(base, orderBy("createdAt", "desc"))
    : query(base, where("category", "==", category), orderBy("createdAt", "desc"));
  return onSnapshot(q, (snap) => onChange(snap.docs.map(d => ({ id: d.id, ...d.data() }))));
}
```

Notes:
- `runTransaction` prevents double-upvoting and race conditions when many clients vote at once.
- A one-doc-per-voter subcollection (`votes/{uid}`) is cheaper to secure with Firestore rules than an array field, and avoids the 1 MiB document-size ceiling that a giant `upvoterIds` array would eventually hit.
- For Realtime Database instead of Firestore: replace collections with `ref(db, "posts")`, replace `onSnapshot` with `onValue`, and replace the transaction with `runTransaction(ref(db, \`posts/${postId}/upvoteCount\`), ...)`.

## 4. Relational Fallback — Normalized 3NF Schema (MySQL)

For teams shipping a traditional PHP/MySQL backend instead of (or alongside) Firebase.

```sql
CREATE TABLE roles (
  role_id      TINYINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  role_name    VARCHAR(30) NOT NULL UNIQUE          -- 'Student','Junior Dev','Full-Stack', etc.
);

CREATE TABLE users (
  user_id      INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  handle       VARCHAR(30)  NOT NULL UNIQUE,
  display_name VARCHAR(80)  NOT NULL,
  email        VARCHAR(120) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  school       VARCHAR(120),
  github_url   VARCHAR(255),
  bio          VARCHAR(280),
  created_at   DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- many-to-many: a user can hold several role badges
CREATE TABLE user_roles (
  user_id      INT UNSIGNED NOT NULL,
  role_id      TINYINT UNSIGNED NOT NULL,
  PRIMARY KEY (user_id, role_id),
  FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
  FOREIGN KEY (role_id) REFERENCES roles(role_id) ON DELETE CASCADE
);

CREATE TABLE categories (
  category_id   TINYINT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  category_key  VARCHAR(20) NOT NULL UNIQUE,        -- 'general','showcase','codehelp','team'
  category_label VARCHAR(40) NOT NULL
);

CREATE TABLE posts (
  post_id       INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  author_id     INT UNSIGNED NOT NULL,
  category_id   TINYINT UNSIGNED NOT NULL,
  title         VARCHAR(160) NOT NULL,
  body          TEXT NOT NULL,
  code_snippet  MEDIUMTEXT,                         -- nullable, codehelp posts only
  code_language VARCHAR(30),
  repo_url      VARCHAR(255),
  live_url      VARCHAR(255),
  roles_needed  VARCHAR(255),                        -- team posts; comma list or normalize further if needed
  deadline      DATE,
  is_solved     BOOLEAN NOT NULL DEFAULT FALSE,       -- codehelp posts only
  created_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (author_id) REFERENCES users(user_id) ON DELETE CASCADE,
  FOREIGN KEY (category_id) REFERENCES categories(category_id)
);

CREATE TABLE tags (
  tag_id       INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  tag_name     VARCHAR(30) NOT NULL UNIQUE            -- 'firebase','sql','tailwindcss'
);

-- many-to-many: a post can carry several topic tags
CREATE TABLE post_tags (
  post_id      INT UNSIGNED NOT NULL,
  tag_id       INT UNSIGNED NOT NULL,
  PRIMARY KEY (post_id, tag_id),
  FOREIGN KEY (post_id) REFERENCES posts(post_id) ON DELETE CASCADE,
  FOREIGN KEY (tag_id) REFERENCES tags(tag_id) ON DELETE CASCADE
);

CREATE TABLE comments (
  comment_id       INT UNSIGNED PRIMARY KEY AUTO_INCREMENT,
  post_id          INT UNSIGNED NOT NULL,
  author_id        INT UNSIGNED NOT NULL,
  parent_comment_id INT UNSIGNED,                     -- NULL = top-level, else one-level reply
  body             TEXT NOT NULL,
  code_snippet     MEDIUMTEXT,
  code_language    VARCHAR(30),
  created_at       DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (post_id) REFERENCES posts(post_id) ON DELETE CASCADE,
  FOREIGN KEY (author_id) REFERENCES users(user_id) ON DELETE CASCADE,
  FOREIGN KEY (parent_comment_id) REFERENCES comments(comment_id) ON DELETE CASCADE
);

-- polymorphic-free voting: separate junctions for posts and comments
CREATE TABLE post_votes (
  user_id      INT UNSIGNED NOT NULL,
  post_id      INT UNSIGNED NOT NULL,
  voted_at     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (user_id, post_id),                     -- guarantees one vote per user per post
  FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
  FOREIGN KEY (post_id) REFERENCES posts(post_id) ON DELETE CASCADE
);

CREATE TABLE comment_votes (
  user_id      INT UNSIGNED NOT NULL,
  comment_id   INT UNSIGNED NOT NULL,
  voted_at     DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (user_id, comment_id),
  FOREIGN KEY (user_id) REFERENCES users(user_id) ON DELETE CASCADE,
  FOREIGN KEY (comment_id) REFERENCES comments(comment_id) ON DELETE CASCADE
);
```

**Why this is 3NF:**
- Every non-key column depends on the whole primary key, not part of it (junction tables `user_roles`, `post_tags` exist precisely so `roles` and `tags` aren't repeating groups inside `users`/`posts`).
- No transitive dependencies: `category_label` lives only in `categories` and is referenced by `category_id`, not copied onto every post row.
- Vote counts (`upvoteCount`) are *derived* — in a strict 3NF read path they'd be `COUNT(*)` over `post_votes`/`comment_votes` rather than stored columns. The prototype's Firestore model above denormalizes an `upvoteCount` counter for read speed (a standard "counter cache" trade-off); the relational schema keeps the source of truth in the vote tables and a view/materialized count can be added if read load requires it.

---

## 5. Prototype Notes

The accompanying `index.html` is a single-file, dependency-light client (Tailwind CDN + Google Fonts, vanilla JS, no build step) that mocks this architecture against `localStorage` instead of a live Firebase project, per the "single index.html, no build tools" convention. Swapping the `Store` object's `load/save` calls in the JS for the Firestore calls in §3.3 is the only change needed to go live.
