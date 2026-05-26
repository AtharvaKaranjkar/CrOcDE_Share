# CrOcDE Share

A lightweight code-sharing platform built with **PHP + MySQL** on XAMPP. Inspired by GitLab, scaled down to a college DBMS project showcasing normalization, transactions, triggers, stored procedures, views, recursive CTEs, window functions, and more.

Users can sign up, create code repositories in multiple languages, invite contributors, propose and review changes through a version-controlled workflow, comment in threaded discussions, like/dislike, follow each other, and explore trending repos. An admin can manage users, view all repositories (including private and soft-deleted), browse the full audit log, and reset the entire database for a fresh demo.

---

## Features

- **Authentication** — signup, login, logout with bcrypt-hashed passwords and session management
- **Repositories** — create with language tag (C++, Python, Java, etc.) and visibility (public/private), max 10,000 chars of code
- **Versioning** — every edit creates a new version; owners edit directly, contributors and viewers submit change requests
- **Change request workflow** — owner reviews proposed changes side-by-side and accepts (via stored procedure) or rejects
- **Diff & compare** — view any two versions side-by-side; full version history visible to owner, latest+previous to contributors, latest only to viewers
- **Contributors** — owners add/remove by username; ownership transfer via stored procedure (old owner becomes contributor)
- **Forks** — viewers pin public repos to their dashboard; auto-hidden when repo flips to private
- **Comments** — threaded replies up to 6 levels deep using a recursive CTE; soft-delete preserves thread structure
- **Reactions** — like/dislike toggle (one reaction per user per repo)
- **Tags** — many-to-many; filterable on explore page
- **Follows** — follow other users to see their activity in a personal feed
- **Notifications** — in-app inbox with unread badge in navbar
- **Trending** — top 3 repos per language ranked using `RANK() OVER PARTITION BY`
- **PDF export** — download any visible version as a styled PDF
- **Admin panel** — manage users (suspend/unsuspend), view all repos and the full audit log with filters, reset the database
- **Audit log** — every meaningful action is recorded

---

## DBMS concepts covered

This project intentionally exercises a broad set of DBMS topics:

- **DDL** — `CREATE DATABASE`, `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`, ENUM types, DEFAULT values, AUTO_INCREMENT
- **DML** — `INSERT`, `UPDATE`, `DELETE`, `SELECT` with `WHERE`, `ORDER BY`, `GROUP BY`, `HAVING`, `LIMIT`, `OFFSET`
- **TCL** — `START TRANSACTION`, `COMMIT`, `ROLLBACK` (used in repo creation, change request acceptance, ownership transfer)
- **Constraints** — PRIMARY KEY, FOREIGN KEY with `ON DELETE CASCADE`, UNIQUE (single + composite), NOT NULL, DEFAULT, CHECK (code length ≤ 10000, admin-not-suspended, no-self-follow), ENUM
- **Keys** — primary, foreign, candidate (username & email), composite primary (junction tables), surrogate vs natural
- **Normalization** — full 3NF/BCNF decomposition across 13 tables
- **Joins** — INNER, LEFT OUTER, SELF JOIN (comments thread, follows)
- **Set operations** — `UNION` and `UNION ALL` in the dashboard query (owned ∪ contributed ∪ forked)
- **Subqueries** — correlated, non-correlated, `EXISTS`/`NOT EXISTS`, `IN`
- **Aggregations** — `COUNT`, `SUM`, `AVG`, `MIN`, `MAX` with `GROUP BY` and `HAVING`
- **Window functions** — `RANK() OVER (PARTITION BY language ORDER BY like_count DESC)` for trending
- **CTEs** — including a **recursive CTE** (`WITH RECURSIVE`) in `v_comment_threads` for nested comment rendering
- **Indexes** — B-tree on FK columns, FULLTEXT on `repositories.name`, composite indexes on common filter combos
- **Views** — `v_comment_threads` (recursive)
- **Stored procedures** — `sp_transfer_ownership`, `sp_accept_change_request`, `sp_reset_database` (each demonstrates transactions, parameter passing, `SIGNAL SQLSTATE` for errors)
- **Triggers** — `tr_version_insert` auto-computes lines_changed
- **Pagination** — `LIMIT`/`OFFSET` on notifications, audit log, admin user/repo lists
- **Soft delete** — `deleted_at` pattern across repositories and comments
- **Audit logging** — comprehensive write-trail in `audit_log`
- **Prepared statements** — every query uses `mysqli` prepared statements (SQL injection protection)

---

## Tech stack

- **XAMPP** (Apache, MySQL/MariaDB, PHP 8.x)
- **Vanilla PHP** (no framework, no Composer needed)
- **`mysqli`** with prepared statements throughout
- **Vanilla HTML/CSS/JS** for the frontend (no React, no build step)
- **FPDF** for PDF export (downloaded separately, see install steps)

---

## Setup from scratch

### 1. Install XAMPP

Download from [https://www.apachefriends.org/](https://www.apachefriends.org/) and install. Any version with **PHP 8.0+** works.

### 2. Clone or download this repository

```bash
cd C:\xampp\htdocs
git clone https://github.com/YOUR_USERNAME/mini-gitlab.git
```

Or download the ZIP from GitHub and extract it into `C:\xampp\htdocs\`. The final path should be:

```
C:\xampp\htdocs\mini-gitlab\
```

### 3. Install FPDF (needed for PDF export)

1. Download FPDF from [http://www.fpdf.org/en/dl.php?v=185&f=zip](http://www.fpdf.org/en/dl.php?v=185&f=zip)
2. Extract the zip. Inside you'll find `fpdf.php` and a `font/` folder.
3. Create the folder `C:\xampp\htdocs\mini-gitlab\assets\lib\` if it doesn't already exist.
4. Copy **both** `fpdf.php` and the entire `font/` folder into `assets/lib/`. The final structure should look like:

```
mini-gitlab/
└── assets/
    └── lib/
        ├── fpdf.php
        └── font/
            ├── helvetica.php
            ├── helveticab.php
            ├── courier.php
            └── ... (more font files)
```

### 4. Start XAMPP services

Open the XAMPP Control Panel and click **Start** next to both:
- Apache
- MySQL

Both should show green "Running" labels.

### 5. Create the database

1. Open your browser and go to `http://localhost/phpmyadmin/`.
2. Click the **SQL** tab at the top (make sure no database is selected in the sidebar).
3. Open `sql/01_schema.sql` from this project, copy its entire contents, paste into the SQL tab, click **Go**. You should see "Database changed" or similar success messages — the `mini_gitlab` database is now created with all 13 tables.
4. In the sidebar, click the new `mini_gitlab` database.
5. Click the **SQL** tab again.
6. Open `sql/02_seed.sql`, copy, paste, click **Go**. This inserts the permanent admin user and the default tag set.

### 6. Install the stored procedures, triggers, and views

Still in phpMyAdmin with the `mini_gitlab` database selected, click the **SQL** tab and run each of these files in order. For each file: open it, copy contents, paste into the SQL tab, click **Go**.

1. `sql/03_procedures.sql` — `sp_transfer_ownership`
2. `sql/04_procedures_triggers.sql` — `sp_accept_change_request` and `tr_version_insert`
3. `sql/05_views.sql` — `v_comment_threads` (recursive view)
4. `sql/07_admin_procedures.sql` — `sp_reset_database`

> **Note**: each SQL file uses `DELIMITER //` for stored procedures and triggers. phpMyAdmin handles this natively when you paste the whole file.

### 7. Open the app

Go to `http://localhost/mini-gitlab/` in your browser. The Mini-GitLab landing page should load.

### 8. Log in as admin

Click **Log in** and enter:

| Field    | Value         |
|----------|---------------|
| Username | `admin`       |
| Password | `admin@123`   |

The admin account is permanent and cannot be suspended or deleted (enforced by a CHECK constraint).

---

## First-run walkthrough

1. **Sign up two regular users** (e.g., `alice` and `bob`) so you can test the workflow.
2. As alice, **create a public repository** — click **+ New repository**, give it a name, pick C++ or Python, paste some code, submit.
3. As alice, on the repo's **Settings** page, **add bob as a contributor**.
4. As alice, **add some tags** (Settings → Manage tags).
5. **Log out**, log in as bob.
6. Open the repo via Explore. Click **Propose change**, edit the code, submit.
7. **Log out**, log in as alice. The repo view shows a yellow "1 pending request" badge.
8. Click it → **Review** the proposal → **Accept**. The new version is created.
9. Bob's account gets a notification (visible at the 🔔 icon in the navbar).
10. Explore other features: comments, likes, forks (as a non-contributor), follows, the activity feed.
11. Log in as admin → Admin Panel → explore Users, Repositories, Audit log, and the (scary) Reset database button.

---

## Resetting the database

You have two ways to reset for a fresh demo:

**Option 1 — From the admin panel (recommended):**
1. Log in as admin.
2. Navigate to **Admin Panel** → **Reset database**.
3. Type `RESET` (uppercase) in the confirmation box.
4. Click **Reset database now**.

**Option 2 — From phpMyAdmin:**
1. Open phpMyAdmin → select `mini_gitlab` → SQL tab.
2. Paste the contents of `sql/99_reset.sql` and click Go.

Both methods wipe all non-admin data, re-seed the default tags, and reset AUTO_INCREMENT counters so the next new user becomes `user_id=2` and the next repo becomes `repo_id=1`. The admin account remains untouched.

---

## Project structure

```
mini-gitlab/
├── README.md
├── index.php                    # Home / explore feed + trending
├── dashboard.php                # Signed-in user's dashboard
├── profile.php                  # User profile pages
├── feed.php                     # Activity feed (people you follow)
├── follow.php                   # Follow / unfollow action
├── notifications.php            # Notification inbox with pagination
│
├── config/
│   └── db.php                   # DB connection, BASE_URL auto-detection
│
├── includes/
│   ├── auth.php                 # Session helpers, gates, suspension check
│   ├── helpers.php              # h(), flash(), redirect(), audit_log()
│   ├── header.php               # Site chrome: navbar with bell, banners, flash
│   └── footer.php               # Closing chrome
│
├── assets/
│   ├── css/
│   │   └── style.css            # Crimson/black/white/light-grey theme
│   └── lib/
│       ├── fpdf.php             # PDF library (downloaded separately)
│       └── font/                # FPDF font definitions
│
├── auth/
│   ├── signup.php
│   ├── login.php
│   └── logout.php
│
├── repos/
│   ├── create.php               # New repo form
│   ├── view.php                 # Main repo page: code, comments, reactions
│   ├── edit.php                 # Owner direct edit OR submit change request
│   ├── versions.php             # Version history with role-based visibility
│   ├── view_version.php         # View a specific version
│   ├── compare.php              # Side-by-side compare any two versions
│   ├── change_requests.php      # List pending change requests
│   ├── review_change.php        # Owner accepts/rejects a change request
│   ├── settings.php             # Visibility, contributors, transfer, delete
│   ├── contributors.php         # Add/remove contributor action endpoint
│   ├── transfer.php             # Ownership transfer (calls stored proc)
│   ├── fork.php / unfork.php    # Fork toggle
│   ├── react.php                # Like/dislike toggle
│   ├── tags.php                 # Manage repo tags
│   └── pdf.php                  # PDF export of current version
│
├── comments/
│   ├── post.php                 # Post comment or reply
│   └── delete.php               # Soft-delete own comment (admin: any)
│
├── admin/
│   ├── index.php                # Admin dashboard with site stats
│   ├── users.php                # Users list + suspend toggle + pagination
│   ├── repos.php                # All repos including private/deleted
│   ├── audit_log.php            # Full audit log with filters + pagination
│   └── reset.php                # Database reset (two-step confirm)
│
└── sql/
    ├── 01_schema.sql            # All 13 tables + constraints + indexes
    ├── 02_seed.sql              # Admin user + default tags
    ├── 03_procedures.sql        # sp_transfer_ownership
    ├── 04_procedures_triggers.sql  # sp_accept_change_request, tr_version_insert
    ├── 05_views.sql             # v_comment_threads (recursive)
    ├── 07_admin_procedures.sql  # sp_reset_database
    └── 99_reset.sql             # Standalone reset script
```

---

## Database schema

13 tables, normalized to 3NF/BCNF:

1. `users` — accounts; admin protected by CHECK constraint
2. `repositories` — repos with owner FK, soft-deletable
3. `versions` — every code revision; UNIQUE(repo_id, version_number); 10K char CHECK
4. `repo_contributors` — M:N junction users ↔ repositories
5. `change_requests` — pending/accepted/rejected workflow rows
6. `comments` — self-referencing FK for threading; soft-deletable
7. `reactions` — one like/dislike per (user, repo)
8. `forks` — viewer pins of public repos
9. `tags` — distinct tag names
10. `repo_tags` — M:N junction
11. `follows` — self-referencing M:N on users with no-self-follow CHECK
12. `notifications` — in-app inbox
13. `audit_log` — admin actions and other auditable events

---

## Common issues

**"Database connection failed" on first load**
Make sure MySQL is started in the XAMPP control panel, and that you ran both `01_schema.sql` and `02_seed.sql` in phpMyAdmin.

**"Could not include font definition file" when downloading a PDF**
You only copied `fpdf.php` but not the `font/` folder. Re-do the FPDF install step — both `fpdf.php` AND the `font/` folder need to be in `assets/lib/`.

**404 Not Found when visiting `http://localhost/mini-gitlab/`**
Make sure the folder is named exactly `mini-gitlab` (case-sensitive on some systems) and is directly inside `C:\xampp\htdocs\`, not nested. URLs are case-sensitive in some setups.

**Stored procedure errors when loading SQL files**
phpMyAdmin should handle `DELIMITER //` natively. If you get errors, paste the whole file at once (not line by line) and make sure no database is auto-selected when running `01_schema.sql`.

**Changes to PHP files not showing up**
Hard-refresh the browser (Ctrl+F5). XAMPP doesn't cache PHP, but the browser caches CSS.

---

## Credits

Built as a college DBMS project. Inspired by GitLab. Uses [FPDF](http://www.fpdf.org/) for PDF generation.

---

## License

MIT (or whatever you want — feel free to adapt for your own coursework).
