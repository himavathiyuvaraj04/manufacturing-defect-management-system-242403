# Defect Management Database Schema (PostgreSQL)

This database stores manufacturing defects, configuration (defect types/rules), production context, RCA, corrective actions, attachment metadata, and audit trail.

## Core tables

### users
- `id` (uuid, PK)
- `email` (text, unique, required)
- `full_name` (text)
- `password_hash` (text)
- `is_active` (boolean)
- `created_at`, `updated_at` (timestamptz)

### roles / user_roles
- `roles(id, name unique, description, created_at)`
- `user_roles(user_id FK users, role_id FK roles)` composite PK

Seeded roles:
- operator, engineer, manager, admin

## Production context

### production_lines
- `id` (uuid, PK)
- `code` (text, unique)
- `name` (text)
- `is_active` (boolean)
- `created_at` (timestamptz)

### shifts
- `id` (uuid, PK)
- `code` (text, unique)
- `name` (text)
- `start_time`, `end_time` (time)
- `created_at` (timestamptz)

Seeded shifts:
- A (06:00-14:00), B (14:00-22:00), C (22:00-06:00)

## Defect configuration

### defect_types
- `id` (uuid, PK)
- `code` (text, unique)
- `name` (text)
- `default_severity` (Critical|Major|Minor)
- `is_active` (boolean)
- `created_at` (timestamptz)

### severity_rules
- `id` (uuid, PK)
- `defect_type_id` (uuid FK defect_types)
- `rule_json` (jsonb) - stores rule logic to compute severity (evaluated in backend)
- `severity` (Critical|Major|Minor)
- `is_active` (boolean)
- `created_at` (timestamptz)

## Defects

### defects
- `id` (uuid, PK)
- `defect_number` (text, unique) - human-readable identifier
- `occurred_at` (timestamptz)
- `part_number` (text)
- `description` (text)
- `defect_type_id` (uuid FK defect_types)
- `production_line_id` (uuid FK production_lines)
- `shift_id` (uuid FK shifts)
- `quantity_affected` (int, >=0)
- `severity` (Critical|Major|Minor)
- `status` (Open|In Review|Closed)
- `reported_by_user_id` (uuid FK users)
- `tags` (text[])
- `extra` (jsonb)
- `created_at`, `updated_at` (timestamptz)

Indexes:
- `idx_defects_occurred_at` on (occurred_at)
- `idx_defects_type` on (defect_type_id)

## Root cause analysis (RCA)

### defect_rca
- `id` (uuid, PK)
- `defect_id` (uuid FK defects, unique) - 1 RCA per defect
- `method` (5-Why|Fishbone)
- `five_whys` (jsonb)
- `fishbone` (jsonb)
- `conclusion` (text)
- `created_by_user_id` (uuid FK users)
- `created_at`, `updated_at` (timestamptz)

## Corrective actions

### corrective_actions
- `id` (uuid, PK)
- `defect_id` (uuid FK defects)
- `title` (text)
- `description` (text)
- `assignee_user_id` (uuid FK users)
- `due_date` (date)
- `status` (Open|In Progress|Done|Cancelled|Overdue)
- `completed_at` (timestamptz)
- `created_by_user_id` (uuid FK users)
- `created_at`, `updated_at` (timestamptz)

Index:
- `idx_corrective_actions_due_status` on (due_date, status)

## Attachments (metadata)

### attachments
Stores metadata only; binary storage is handled by the application layer.
- `id` (uuid, PK)
- `defect_id` (uuid FK defects, nullable)
- `corrective_action_id` (uuid FK corrective_actions, nullable)
- `file_name` (text)
- `mime_type` (text)
- `file_size_bytes` (bigint)
- `storage_path` (text) - where the file is stored
- `uploaded_by_user_id` (uuid FK users)
- `created_at` (timestamptz)

Constraint:
- Exactly one of `defect_id` or `corrective_action_id` must be non-null.

## Audit

### audit_log
Tracks change events across any entity.
- `id` (uuid, PK)
- `entity_type` (text)
- `entity_id` (uuid)
- `action` (text)
- `actor_user_id` (uuid FK users)
- `before_state`, `after_state` (jsonb)
- `ip_address` (inet)
- `user_agent` (text)
- `created_at` (timestamptz)

## Notes
- UUIDs use `gen_random_uuid()` (extension: `pgcrypto`).
- This repository currently does not include a migration runner (e.g., Flyway/Alembic). The schema is applied via PostgreSQL DDL.
