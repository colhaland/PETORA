# Petora — Technical Architecture Baseline (SvelteKit)

**Sistem Manajemen Terpadu Petshop & Petcare**
**Dokumen Baseline Contract | 14 September 2026**
**Status:** Normative technical contract. Berlaku bersama `docs/00-baseline-governance.md` dan `docs/01-product-baseline.md`.

> **Catatan:** Isi dokumen ini adalah target kontrak teknis; keberadaan dokumentasi tidak berarti implementasi sudah tersedia atau production-ready.

---

## Daftar Isi

1. [Ringkasan Eksekutif](#1-ringkasan-eksekutif)
2. [Arsitektur Sistem End-to-End](#2-arsitektur-sistem-end-to-end)
3. [Database Schema (Supabase/PostgreSQL)](#3-database-schema-supabasepostgresql)
4. [TypeScript Types & Interfaces](#4-typescript-types--interfaces)
5. [Zod Validation Schemas](#5-zod-validation-schemas)
6. [Service Layer Contracts](#6-service-layer-contracts)
7. [Svelte Query Hooks Contracts](#7-svelte-query-hooks-contracts)
8. [Component Props Contracts (Svelte 5 Runes)](#8-component-props-contracts-svelte-5-runes)
9. [Row Level Security (RLS) Policies](#9-row-level-security-rls-policies)
10. [State Management Contracts](#10-state-management-contracts)
11. [Utility Functions Contracts](#11-utility-functions-contracts)
12. [Error Handling Contracts](#12-error-handling-contracts)
13. [File Structure Contracts (SvelteKit)](#13-file-structure-contracts-sveltekit)
14. [Environment Variables](#14-environment-variables)
15. [API Response Envelope](#15-api-response-envelope)
16. [Naming Conventions](#16-naming-conventions)
17. [Migration Strategy](#17-migration-strategy)
18. [Supabase Edge Functions](#18-supabase-edge-functions)
19. [Realtime Subscriptions](#19-realtime-subscriptions)
20. [Storage & File Upload](#20-storage--file-upload)
21. [Operational Deployment Contract](#21-operational-deployment-contract)

---

## 1. Ringkasan Eksekutif

Dokumen ini mendefinisikan kontrak arsitektur teknis baseline untuk seluruh sistem **Petora** — sistem manajemen terpadu Petshop & Petcare berbasis **SvelteKit 2 + Svelte 5 + TypeScript + Supabase** yang di-deploy ke **Vercel** melalui `@sveltejs/adapter-vercel`.

Dokumen ini menjadi acuan tunggal bagi developer dan AI agent untuk:

- Membangun database schema di Supabase
- Menulis TypeScript types/interfaces yang konsisten
- Membuat Zod validation schemas untuk runtime validation
- Implementasi service layer (Supabase client wrapper)
- Membuat Svelte Query hooks untuk data fetching
- Mendefinisikan component props yang type-safe (Svelte 5 `$props()` rune)
- Menulis RLS policies untuk otorisasi level database
- Mengatur state management (Svelte 5 runes: `$state`, `$derived`, `$effect`)
- Implementasi utility functions yang reusable
- Menangani error secara konsisten di seluruh aplikasi
- Mengatur file structure modular mengikuti konvensi SvelteKit
- Mengelola environment variables
- Mendefinisikan API response envelope
- Menetapkan naming conventions
- Merencanakan migration strategy
- Membuat Supabase Edge Functions untuk logika kompleks
- Mengatur realtime subscriptions
- Mengelola file upload ke Supabase Storage

### Prinsip Arsitektur

| Prinsip | Penjelasan |
| --- | --- |
| **Type-safe** | TypeScript strict mode, Zod untuk runtime validation |
| **Contract-first** | Define interfaces & schemas dulu, implementasi mengikuti |
| **Modular** | Setiap modul memiliki kontrak sendiri, minim coupling |
| **Testable** | Kontrak memudahkan mocking & testing |
| **Documented** | JSDoc untuk setiap interface & function |
| **Secure by default** | RLS policies di level database, validasi di setiap layer |
| **Scalable** | Desain siap untuk multi-cabang di masa depan |
| **Maintainable** | Konvensi naming & struktur yang konsisten |
| **Fine-grained Reactivity** | Svelte 5 runes: reaktivitas di level statement, bukan komponen |
| **Lightweight** | Minimal runtime, compile-time optimization, bundle size kecil |
| **Full-stack capable** | SvelteKit menyediakan `+page.server.ts` untuk server-side logic |

---

## 2. Arsitektur Sistem End-to-End

### 2.1 Arsitektur High-Level

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Layer                             │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │          SvelteKit SPA/SSR (Svelte 5 + TypeScript)        │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐  │  │
│  │  │ Staff      │  │ Customer   │  │ Shared Components  │  │  │
│  │  │ Dashboard  │  │ Portal     │  │ (shadcn-svelte /   │  │  │
│  │  │ /app/*     │  │ /portal/*  │  │  bits-ui+Tailwind) │  │  │
│  │  └────────────┘  └────────────┘  └────────────────────┘  │  │
│  │         │                │                │                │  │
│  │         └────────────────┼────────────────┘                │  │
│  │                          ▼                                 │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │   Svelte Query + Supabase JS Client                  │  │  │
│  │  │   (createQuery, createMutation, Caching, Realtime)   │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                          │                                 │  │
│  │  ┌───────────────────────┴─────────────────────────────┐  │  │
│  │  │   State: $state + $derived (Svelte 5 Runes)          │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Supabase Platform                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ PostgreSQL   │  │ Auth         │  │ Storage              │  │
│  │ (Database)   │  │ (Sessions)   │  │ (File Uploads)       │  │
│  │ + RLS        │  │              │  │                      │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Realtime     │  │ Edge         │  │ Functions            │  │
│  │ (WebSocket)  │  │ Functions    │  │ (Business Logic)     │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    External Services                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ WhatsApp     │  │ Payment      │  │ Email                │  │
│  │ Adapter      │  │ Adapter      │  │ Adapter              │  │
│  │ (selected)   │  │ (selected)   │  │ (selected)           │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Layer Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ Layer 1: Presentation (Svelte 5 Components)                      │
│ - UI Components (shadcn-svelte / bits-ui + Tailwind)             │
│ - Feature Components (modular per domain)                        │
│ - Page Components (+page.svelte, routing via SvelteKit)          │
│ - Runes-based reactivity: $state, $derived, $effect              │
│ - Props via $props() rune dengan type inference                  │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 2: Data Fetching (Svelte Query Hooks)                      │
│ - createQuery (GET operations)                                   │
│ - createMutation (POST/PUT/DELETE operations)                    │
│ - Query keys management                                          │
│ - Optimistic updates                                             │
│ - SvelteKit load functions (untuk server-side prefetch)          │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 3: Service Layer (Supabase Client Wrappers)                │
│ - Domain-specific services (CustomerService, ProductService)     │
│ - Business logic orchestration                                   │
│ - Data transformation                                            │
│ - Error handling (AppError dengan ErrorCode)                     │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 4: Validation (Zod Schemas)                                │
│ - Input validation (create/update schemas)                       │
│ - Output validation (response schemas)                           │
│ - Type inference (z.infer<typeof schema>)                        │
└─────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ Layer 5: Database (Supabase PostgreSQL + RLS)                    │
│ - Tables & relationships                                         │
│ - Row Level Security policies                                    │
│ - Indexes for performance                                        │
│ - Triggers for automation                                        │
│ - RPC functions untuk atomic operations                          │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 Data Flow

```
User Action
      │
      ▼
  Component (Svelte 5 — reactive statements via runes)
      │
      ▼
  Svelte Query Hook (createMutation / createQuery)
      │
      ▼
  Service Layer (validate + transform)
      │
      ▼
  Zod Schema (runtime validation)
      │
      ▼
  Supabase Client (RPC / Insert / Update / Delete)
      │
      ▼
  Supabase Edge Function (optional, complex logic)
      │
      ▼
  PostgreSQL (with RLS enforcement)
      │
      ▼
  Response → Svelte Query Cache → Reactive DOM Update
             (hanya node DOM yang terikat $state yang berubah)
```

### 2.4 Teknologi Utama

| Aspek | Teknologi |
| --- | --- |
| Framework | **SvelteKit 2** (meta-framework) |
| UI Runtime | **Svelte 5** (runes mode) |
| Rendering | Compile-time, fine-grained reactivity |
| State | `$state`, `$derived`, `$effect` (runes) |
| Data Fetching | `@tanstack/svelte-query` |
| Routing | SvelteKit file-based (`+page.svelte`, `+layout.svelte`) |
| UI Library | `shadcn-svelte` / `bits-ui` |
| Forms | `@tanstack/svelte-form` atau `formsnap` + `superforms` |
| Icons | `lucide-svelte` |
| Validation | `zod` + `zod-to-json-schema` |
| Database | Supabase PostgreSQL + RLS |
| Auth | Supabase Auth (custom PIN flow via Edge Function) |
| Storage | Supabase Storage |
| Realtime | Supabase Realtime (WebSocket) |
| Styling | Tailwind CSS 3 |
| Deployment | Vercel via `@sveltejs/adapter-vercel` |
| Testing | Vitest (unit) + Playwright (E2E) |
| Package Manager | pnpm |

---

## 3. Database Schema (Supabase/PostgreSQL)

### 3.1 Naming Conventions

| Aspek | Konvensi | Contoh |
| --- | --- | --- |
| Tabel | snake_case, plural | `customers`, `medical_records` |
| Kolom | snake_case | `created_at`, `customer_id` |
| Primary Key | `id UUID DEFAULT gen_random_uuid()` | — |
| Foreign Key | `<table_singular>_id` | `customer_id`, `pet_id` |
| Timestamp | `created_at`, `updated_at`, `deleted_at` | — |
| Enum | snake_case | `appointment_status` |
| Index | `idx_<table>_<column>` | `idx_customers_phone` |
| Unique | `uniq_<table>_<column>` | `uniq_users_username` |
| Function | `fn_<action>_<entity>` | `fn_calculate_loyalty_points` |
| Trigger | `trg_<table>_<action>` | `trg_users_before_insert` |

### 3.2 Complete Schema

> **Migration rule:** blok SQL di bawah adalah logical schema contract, bukan satu file migration yang boleh dijalankan mentah. Implementasi wajib memecahnya menjadi migration berurutan sesuai dependency foreign key (misalnya `customers` sebelum `users`), menambahkan `updated_at` trigger, RLS, dan rollback/recovery plan. Fresh database migration test adalah release gate.

```sql
-- ============================================
-- ENUMS
-- ============================================
CREATE TYPE user_role AS ENUM ('OWNER', 'ADMIN', 'DOKTER', 'KASIR', 'CUSTOMER');
CREATE TYPE appointment_status AS ENUM ('WAITING', 'IN_PROGRESS', 'DONE', 'CANCELLED');
CREATE TYPE medical_record_status AS ENUM ('OPEN', 'CLOSED');
CREATE TYPE room_status AS ENUM ('AVAILABLE', 'RESERVED', 'OCCUPIED', 'MAINTENANCE', 'INACTIVE');
CREATE TYPE room_cleanliness AS ENUM ('CLEAN', 'DIRTY', 'UNDER_CLEANING');
CREATE TYPE pet_hotel_booking_status AS ENUM ('BOOKED', 'CHECKED_IN', 'CHECKED_OUT', 'CANCELLED');
CREATE TYPE grooming_booking_status AS ENUM ('BOOKED', 'IN_PROGRESS', 'DONE', 'CANCELLED');
CREATE TYPE product_status AS ENUM ('ACTIVE', 'ARCHIVED');
CREATE TYPE stock_movement_type AS ENUM ('IN', 'OUT', 'RETURN', 'ADJUSTMENT', 'DAMAGED', 'EXPIRED', 'OPNAME');
CREATE TYPE invoice_type AS ENUM ('POS', 'CLINICAL', 'PET_HOTEL', 'GROOMING', 'MIXED');
CREATE TYPE invoice_status AS ENUM ('UNPAID', 'PARTIAL_PAYMENT', 'PAID', 'CANCELLED');
CREATE TYPE payment_method AS ENUM ('CASH', 'QRIS', 'TRANSFER', 'E_WALLET', 'CREDIT_CARD', 'MIXED');
CREATE TYPE purchase_order_status AS ENUM ('DRAFT', 'SENT', 'PARTIAL_RECEIVED', 'RECEIVED', 'CANCELLED');
CREATE TYPE loyalty_tier_name AS ENUM ('BRONZE', 'SILVER', 'GOLD', 'PLATINUM');
CREATE TYPE loyalty_transaction_type AS ENUM ('EARN', 'REDEEM', 'EXPIRE', 'ADJUST');
CREATE TYPE promotion_type AS ENUM ('PERCENTAGE', 'FIXED', 'BUNDLE', 'HAPPY_HOUR', 'BIRTHDAY');
CREATE TYPE promotion_status AS ENUM ('ACTIVE', 'EXPIRED', 'CANCELLED');
CREATE TYPE expense_status AS ENUM ('PENDING', 'APPROVED', 'REJECTED', 'REVERSED');
CREATE TYPE feedback_rating AS ENUM ('1', '2', '3', '4', '5');
CREATE TYPE pet_hotel_log_type AS ENUM ('FEEDING', 'MEDICINE', 'NOTE');
CREATE TYPE customer_tag AS ENUM ('VIP', 'REGULAR', 'NEW', 'BLACKLIST');

-- ============================================
-- SEQUENCE COUNTERS (untuk number generation)
-- ============================================
CREATE TABLE sequence_counters (
  prefix TEXT NOT NULL,
  date TEXT NOT NULL,
  current_value INTEGER NOT NULL DEFAULT 1,
  PRIMARY KEY (prefix, date)
);

-- ============================================
-- USERS & AUTH
-- ============================================
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username VARCHAR(50) UNIQUE NOT NULL,
  pin_hash TEXT NOT NULL,
  role user_role NOT NULL,
  full_name VARCHAR(100) NOT NULL,
  customer_id UUID REFERENCES customers(id) ON DELETE SET NULL,
  created_by UUID REFERENCES users(id) ON DELETE SET NULL,
  failed_login_attempts INTEGER DEFAULT 0,
  locked_until TIMESTAMPTZ,
  is_active BOOLEAN DEFAULT TRUE,
  last_login_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_customer_id ON users(customer_id);
CREATE INDEX idx_users_created_by ON users(created_by);

-- ============================================
-- CUSTOMERS
-- ============================================
CREATE TABLE customers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  phone VARCHAR(20),
  email VARCHAR(100),
  address TEXT,
  emergency_contact VARCHAR(100),
  photo_url TEXT,
  notes TEXT,
  is_guest BOOLEAN DEFAULT FALSE,
  tags customer_tag[] DEFAULT '{}',
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ
);

CREATE INDEX idx_customers_name ON customers(name);
CREATE INDEX idx_customers_phone ON customers(phone);
CREATE INDEX idx_customers_is_guest ON customers(is_guest);
CREATE INDEX idx_customers_is_active ON customers(is_active);

-- ============================================
-- PETS
-- ============================================
CREATE TABLE pets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  name VARCHAR(100) NOT NULL,
  species VARCHAR(50) NOT NULL,
  breed VARCHAR(50),
  birth_date DATE,
  gender VARCHAR(10),
  photo_url TEXT,
  microchip_number VARCHAR(50),
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ
);

CREATE INDEX idx_pets_customer_id ON pets(customer_id);
CREATE INDEX idx_pets_species ON pets(species);
CREATE INDEX idx_pets_is_active ON pets(is_active);

-- ============================================
-- PET WEIGHT LOGS
-- ============================================
CREATE TABLE pet_weight_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pet_id UUID NOT NULL REFERENCES pets(id) ON DELETE CASCADE,
  weight_kg DECIMAL(5,2) NOT NULL,
  recorded_at DATE NOT NULL DEFAULT CURRENT_DATE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_pet_weight_logs_pet_id ON pet_weight_logs(pet_id);
CREATE INDEX idx_pet_weight_logs_recorded_at ON pet_weight_logs(recorded_at);

-- ============================================
-- PET VACCINES
-- ============================================
CREATE TABLE pet_vaccines (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pet_id UUID NOT NULL REFERENCES pets(id) ON DELETE CASCADE,
  vaccine_name VARCHAR(100) NOT NULL,
  vaccination_date DATE NOT NULL,
  due_date DATE,
  notes TEXT,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_pet_vaccines_pet_id ON pet_vaccines(pet_id);
CREATE INDEX idx_pet_vaccines_due_date ON pet_vaccines(due_date);
CREATE INDEX idx_pet_vaccines_is_active ON pet_vaccines(is_active);

-- ============================================
-- PET DISEASES & ALLERGIES
-- ============================================
CREATE TABLE pet_diseases (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pet_id UUID NOT NULL REFERENCES pets(id) ON DELETE CASCADE,
  disease_name VARCHAR(100) NOT NULL,
  diagnosed_date DATE,
  notes TEXT,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE pet_allergies (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  pet_id UUID NOT NULL REFERENCES pets(id) ON DELETE CASCADE,
  allergen VARCHAR(100) NOT NULL,
  notes TEXT,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- APPOINTMENTS
-- ============================================
CREATE TABLE appointments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID NOT NULL REFERENCES customers(id),
  pet_id UUID NOT NULL REFERENCES pets(id),
  doctor_id UUID REFERENCES users(id),
  appointment_date DATE NOT NULL,
  appointment_time TIME NOT NULL,
  queue_number INTEGER,
  status appointment_status DEFAULT 'WAITING',
  complaint TEXT,
  notes TEXT,
  is_from_portal BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_appointments_customer_id ON appointments(customer_id);
CREATE INDEX idx_appointments_doctor_id ON appointments(doctor_id);
CREATE INDEX idx_appointments_date ON appointments(appointment_date);
CREATE INDEX idx_appointments_status ON appointments(status);

-- ============================================
-- MEDICAL RECORDS
-- ============================================
CREATE TABLE medical_records (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  record_number VARCHAR(20) UNIQUE NOT NULL,
  appointment_id UUID NOT NULL UNIQUE REFERENCES appointments(id),
  doctor_id UUID NOT NULL REFERENCES users(id),
  chief_complaint TEXT,
  history TEXT,
  physical_exam TEXT,
  weight_kg DECIMAL(5,2),
  temperature_c DECIMAL(4,1),
  heart_rate_bpm INTEGER,
  respiratory_rate_bpm INTEGER,
  diagnosis TEXT,
  treatment TEXT,
  prescription TEXT,
  lab_results TEXT,
  additional_notes TEXT,
  attachments TEXT[],
  status medical_record_status DEFAULT 'OPEN',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ
);

CREATE INDEX idx_medical_records_appointment_id ON medical_records(appointment_id);
CREATE INDEX idx_medical_records_doctor_id ON medical_records(doctor_id);
CREATE INDEX idx_medical_records_status ON medical_records(status);

-- ============================================
-- PROCEDURES (Master Data)
-- ============================================
CREATE TABLE procedures (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code VARCHAR(20) UNIQUE NOT NULL,
  name VARCHAR(100) NOT NULL,
  description TEXT,
  price DECIMAL(12,2) NOT NULL,
  category VARCHAR(50),
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ
);

-- ============================================
-- PET HOTEL ROOMS
-- ============================================
CREATE TABLE rooms (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(50) NOT NULL,
  room_number VARCHAR(20),
  room_type VARCHAR(20) NOT NULL,
  price_per_night DECIMAL(12,2) NOT NULL,
  capacity INTEGER DEFAULT 1,
  status room_status DEFAULT 'AVAILABLE',
  cleanliness room_cleanliness DEFAULT 'CLEAN',
  maintenance_status BOOLEAN DEFAULT FALSE,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ
);

-- ============================================
-- PET HOTEL BOOKINGS
-- ============================================
CREATE TABLE pet_hotel_bookings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_number VARCHAR(20) UNIQUE NOT NULL,
  pet_id UUID NOT NULL REFERENCES pets(id),
  customer_id UUID NOT NULL REFERENCES customers(id),
  room_id UUID REFERENCES rooms(id),
  check_in_date DATE NOT NULL,
  check_out_date DATE NOT NULL,
  actual_check_in_at TIMESTAMPTZ,
  actual_check_out_at TIMESTAMPTZ,
  price_per_night DECIMAL(12,2),
  total_price DECIMAL(12,2),
  status pet_hotel_booking_status DEFAULT 'BOOKED',
  special_notes TEXT,
  is_from_portal BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_pet_hotel_bookings_pet_id ON pet_hotel_bookings(pet_id);
CREATE INDEX idx_pet_hotel_bookings_room_id ON pet_hotel_bookings(room_id);
CREATE INDEX idx_pet_hotel_bookings_status ON pet_hotel_bookings(status);

-- ============================================
-- PET HOTEL LOGS
-- ============================================
CREATE TABLE pet_hotel_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id UUID NOT NULL REFERENCES pet_hotel_bookings(id) ON DELETE CASCADE,
  log_type pet_hotel_log_type NOT NULL,
  description TEXT,
  photo_urls TEXT[],
  logged_at TIMESTAMPTZ DEFAULT NOW(),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_pet_hotel_logs_booking_id ON pet_hotel_logs(booking_id);

-- ============================================
-- GROOMING SERVICES
-- ============================================
CREATE TABLE grooming_services (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  description TEXT,
  base_price DECIMAL(12,2) NOT NULL,
  duration_minutes INTEGER,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ
);

-- ============================================
-- GROOMING BOOKINGS
-- ============================================
CREATE TABLE grooming_bookings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_number VARCHAR(20) UNIQUE NOT NULL,
  pet_id UUID NOT NULL REFERENCES pets(id),
  customer_id UUID NOT NULL REFERENCES customers(id),
  groomer_id UUID REFERENCES users(id),
  service_id UUID NOT NULL REFERENCES grooming_services(id),
  appointment_date DATE NOT NULL,
  appointment_time TIME NOT NULL,
  status grooming_booking_status DEFAULT 'BOOKED',
  total_price DECIMAL(12,2),
  notes TEXT,
  is_from_portal BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_grooming_bookings_pet_id ON grooming_bookings(pet_id);
CREATE INDEX idx_grooming_bookings_groomer_id ON grooming_bookings(groomer_id);
CREATE INDEX idx_grooming_bookings_status ON grooming_bookings(status);

-- ============================================
-- GROOMING RECORDS
-- ============================================
CREATE TABLE grooming_records (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  booking_id UUID NOT NULL UNIQUE REFERENCES grooming_bookings(id),
  skin_condition TEXT,
  flea_tick_found BOOLEAN DEFAULT FALSE,
  recommendations TEXT,
  before_photo_url TEXT,
  after_photo_url TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- PRODUCTS & INVENTORY
-- ============================================
CREATE TABLE categories (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  description TEXT,
  parent_id UUID REFERENCES categories(id),
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE suppliers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  contact_person VARCHAR(100),
  phone VARCHAR(20),
  email VARCHAR(100),
  address TEXT,
  notes TEXT,
  lead_time_days INTEGER,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  sku VARCHAR(50) UNIQUE NOT NULL,
  name VARCHAR(200) NOT NULL,
  category_id UUID REFERENCES categories(id),
  supplier_id UUID REFERENCES suppliers(id),
  barcode VARCHAR(100),
  description TEXT,
  purchase_price DECIMAL(12,2) NOT NULL,
  selling_price DECIMAL(12,2) NOT NULL,
  stock_quantity INTEGER DEFAULT 0,
  stock_minimum INTEGER DEFAULT 0,
  stock_maximum INTEGER DEFAULT 0,
  photo_url TEXT,
  expiry_date DATE,
  status product_status DEFAULT 'ACTIVE',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  deleted_at TIMESTAMPTZ
);

CREATE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_products_category_id ON products(category_id);
CREATE INDEX idx_products_status ON products(status);
CREATE INDEX idx_products_stock_quantity ON products(stock_quantity);

CREATE TABLE product_variants (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
  variant_name VARCHAR(100) NOT NULL,
  variant_value VARCHAR(100) NOT NULL,
  price_adjustment DECIMAL(12,2) DEFAULT 0,
  stock_quantity INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE product_bundles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(200) NOT NULL,
  description TEXT,
  bundle_price DECIMAL(12,2) NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE product_bundle_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  bundle_id UUID NOT NULL REFERENCES product_bundles(id) ON DELETE CASCADE,
  product_id UUID NOT NULL REFERENCES products(id),
  quantity INTEGER NOT NULL DEFAULT 1
);

-- ============================================
-- STOCK MOVEMENTS
-- ============================================
CREATE TABLE stock_movements (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  product_id UUID NOT NULL REFERENCES products(id),
  movement_type stock_movement_type NOT NULL,
  quantity INTEGER NOT NULL,
  reference_type VARCHAR(50),
  reference_id UUID,
  notes TEXT,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_stock_movements_product_id ON stock_movements(product_id);
CREATE INDEX idx_stock_movements_type ON stock_movements(movement_type);
CREATE INDEX idx_stock_movements_created_at ON stock_movements(created_at);

-- ============================================
-- PURCHASE ORDERS
-- ============================================
CREATE TABLE purchase_orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  po_number VARCHAR(20) UNIQUE NOT NULL,
  supplier_id UUID NOT NULL REFERENCES suppliers(id),
  order_date DATE NOT NULL,
  expected_arrival_date DATE,
  actual_arrival_date DATE,
  total_amount DECIMAL(12,2),
  status purchase_order_status DEFAULT 'DRAFT',
  notes TEXT,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE purchase_order_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  po_id UUID NOT NULL REFERENCES purchase_orders(id) ON DELETE CASCADE,
  product_id UUID NOT NULL REFERENCES products(id),
  quantity INTEGER NOT NULL,
  unit_price DECIMAL(12,2) NOT NULL,
  received_quantity INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- INVOICES & PAYMENTS
-- ============================================
CREATE TABLE invoices (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invoice_number VARCHAR(20) UNIQUE NOT NULL,
  invoice_type invoice_type NOT NULL,
  customer_id UUID REFERENCES customers(id),
  subtotal DECIMAL(12,2) NOT NULL,
  discount_amount DECIMAL(12,2) DEFAULT 0,
  tax_amount DECIMAL(12,2) DEFAULT 0,
  total_amount DECIMAL(12,2) NOT NULL,
  paid_amount DECIMAL(12,2) DEFAULT 0,
  status invoice_status DEFAULT 'UNPAID',
  promotion_id UUID REFERENCES promotions(id),
  loyalty_points_earned INTEGER DEFAULT 0,
  loyalty_points_redeemed INTEGER DEFAULT 0,
  notes TEXT,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_invoices_customer_id ON invoices(customer_id);
CREATE INDEX idx_invoices_status ON invoices(status);
CREATE INDEX idx_invoices_created_at ON invoices(created_at);

CREATE TABLE invoice_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invoice_id UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
  item_type VARCHAR(50) NOT NULL,
  product_id UUID REFERENCES products(id),
  procedure_id UUID REFERENCES procedures(id),
  pet_hotel_booking_id UUID REFERENCES pet_hotel_bookings(id),
  grooming_booking_id UUID REFERENCES grooming_bookings(id),
  description VARCHAR(200) NOT NULL,
  quantity INTEGER DEFAULT 1,
  unit_price DECIMAL(12,2) NOT NULL,
  total_price DECIMAL(12,2) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE payments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  invoice_id UUID NOT NULL REFERENCES invoices(id),
  payment_method payment_method NOT NULL,
  amount DECIMAL(12,2) NOT NULL,
  reference_number VARCHAR(100),
  notes TEXT,
  created_by UUID NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_payments_invoice_id ON payments(invoice_id);

-- ============================================
-- CASH SHIFTS
-- ============================================
CREATE TABLE cash_shifts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  kasir_id UUID NOT NULL REFERENCES users(id),
  open_time TIMESTAMPTZ NOT NULL,
  close_time TIMESTAMPTZ,
  opening_cash DECIMAL(12,2) NOT NULL,
  closing_cash DECIMAL(12,2),
  expected_cash DECIMAL(12,2),
  difference DECIMAL(12,2),
  notes TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- LOYALTY PROGRAM
-- ============================================
CREATE TABLE loyalty_tiers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tier_name loyalty_tier_name NOT NULL,
  min_points INTEGER NOT NULL,
  min_spending DECIMAL(12,2) NOT NULL,
  point_multiplier DECIMAL(3,2) DEFAULT 1.0,
  benefits JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE loyalty_members (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID NOT NULL REFERENCES customers(id) UNIQUE,
  tier_id UUID REFERENCES loyalty_tiers(id),
  total_points INTEGER DEFAULT 0,
  available_points INTEGER DEFAULT 0,
  total_spending DECIMAL(12,2) DEFAULT 0,
  joined_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE loyalty_transactions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  member_id UUID NOT NULL REFERENCES loyalty_members(id),
  transaction_type loyalty_transaction_type NOT NULL,
  points INTEGER NOT NULL,
  invoice_id UUID REFERENCES invoices(id),
  description TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_loyalty_transactions_member_id ON loyalty_transactions(member_id);

-- ============================================
-- PROMOTIONS
-- ============================================
CREATE TABLE promotions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code VARCHAR(50) UNIQUE,
  name VARCHAR(200) NOT NULL,
  description TEXT,
  promotion_type promotion_type NOT NULL,
  discount_value DECIMAL(12,2) NOT NULL,
  min_purchase DECIMAL(12,2) DEFAULT 0,
  max_usage INTEGER,
  current_usage INTEGER DEFAULT 0,
  start_date DATE NOT NULL,
  end_date DATE NOT NULL,
  applicable_products UUID[],
  status promotion_status DEFAULT 'ACTIVE',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE promotion_usage (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  promotion_id UUID NOT NULL REFERENCES promotions(id),
  invoice_id UUID NOT NULL REFERENCES invoices(id),
  customer_id UUID REFERENCES customers(id),
  discount_applied DECIMAL(12,2) NOT NULL,
  used_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- EXPENSES
-- ============================================
CREATE TABLE expense_categories (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(100) NOT NULL,
  description TEXT,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE expenses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  expense_date DATE NOT NULL,
  category_id UUID NOT NULL REFERENCES expense_categories(id),
  amount DECIMAL(12,2) NOT NULL,
  description TEXT,
  receipt_url TEXT,
  status expense_status DEFAULT 'PENDING',
  is_recurring BOOLEAN DEFAULT FALSE,
  recurring_day INTEGER,
  created_by UUID NOT NULL REFERENCES users(id),
  approved_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_expenses_date ON expenses(expense_date);
CREATE INDEX idx_expenses_category_id ON expenses(category_id);

-- ============================================
-- CUSTOMER FEEDBACK
-- ============================================
CREATE TABLE customer_feedback (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id UUID NOT NULL REFERENCES customers(id),
  invoice_id UUID REFERENCES invoices(id),
  rating feedback_rating NOT NULL,
  comment TEXT,
  nps_score INTEGER,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- SETTINGS
-- ============================================
CREATE TABLE settings (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  key VARCHAR(100) UNIQUE NOT NULL,
  value JSONB NOT NULL,
  updated_by UUID REFERENCES users(id),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- AUDIT LOGS
-- ============================================
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  action VARCHAR(50) NOT NULL,
  entity_type VARCHAR(50) NOT NULL,
  entity_id UUID,
  old_values JSONB,
  new_values JSONB,
  ip_address VARCHAR(50),
  user_agent TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_entity ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at);

-- ============================================
-- NOTIFICATIONS
-- ============================================
CREATE TABLE notifications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id),
  title VARCHAR(200) NOT NULL,
  message TEXT NOT NULL,
  type VARCHAR(50) NOT NULL,
  is_read BOOLEAN DEFAULT FALSE,
  data JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_notifications_user_id ON notifications(user_id);
CREATE INDEX idx_notifications_is_read ON notifications(is_read);

-- ============================================
-- TRIGGERS
-- ============================================
CREATE OR REPLACE FUNCTION fn_update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_users_updated_at BEFORE UPDATE ON users FOR EACH ROW EXECUTE FUNCTION fn_update_updated_at();
CREATE TRIGGER trg_customers_updated_at BEFORE UPDATE ON customers FOR EACH ROW EXECUTE FUNCTION fn_update_updated_at();
CREATE TRIGGER trg_pets_updated_at BEFORE UPDATE ON pets FOR EACH ROW EXECUTE FUNCTION fn_update_updated_at();
CREATE TRIGGER trg_appointments_updated_at BEFORE UPDATE ON appointments FOR EACH ROW EXECUTE FUNCTION fn_update_updated_at();
CREATE TRIGGER trg_medical_records_updated_at BEFORE UPDATE ON medical_records FOR EACH ROW EXECUTE FUNCTION fn_update_updated_at();
CREATE TRIGGER trg_products_updated_at BEFORE UPDATE ON products FOR EACH ROW EXECUTE FUNCTION fn_update_updated_at();
CREATE TRIGGER trg_invoices_updated_at BEFORE UPDATE ON invoices FOR EACH ROW EXECUTE FUNCTION fn_update_updated_at();
CREATE TRIGGER trg_loyalty_members_updated_at BEFORE UPDATE ON loyalty_members FOR EACH ROW EXECUTE FUNCTION fn_update_updated_at();
CREATE TRIGGER trg_promotions_updated_at BEFORE UPDATE ON promotions FOR EACH ROW EXECUTE FUNCTION fn_update_updated_at();
CREATE TRIGGER trg_expenses_updated_at BEFORE UPDATE ON expenses FOR EACH ROW EXECUTE FUNCTION fn_update_updated_at();
CREATE TRIGGER trg_settings_updated_at BEFORE UPDATE ON settings FOR EACH ROW EXECUTE FUNCTION fn_update_updated_at();

-- Prevent createdBy update
CREATE OR REPLACE FUNCTION fn_prevent_created_by_update()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.created_by IS DISTINCT FROM OLD.created_by THEN
    RAISE EXCEPTION 'created_by is immutable';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_invoices_prevent_created_by BEFORE UPDATE ON invoices FOR EACH ROW EXECUTE FUNCTION fn_prevent_created_by_update();
CREATE TRIGGER trg_stock_movements_prevent_created_by BEFORE UPDATE ON stock_movements FOR EACH ROW EXECUTE FUNCTION fn_prevent_created_by_update();

-- ============================================
-- HELPER FUNCTIONS
-- ============================================
CREATE OR REPLACE FUNCTION get_user_role(p_user_id UUID)
RETURNS user_role AS $$
  SELECT role FROM users WHERE id = p_user_id;
$$ LANGUAGE sql SECURITY DEFINER STABLE;

CREATE OR REPLACE FUNCTION get_customer_id(p_user_id UUID)
RETURNS UUID AS $$
  SELECT customer_id FROM users WHERE id = p_user_id;
$$ LANGUAGE sql SECURITY DEFINER STABLE;

-- Generate sequence number (atomic)
CREATE OR REPLACE FUNCTION fn_generate_sequence_number(
  p_prefix TEXT,
  p_date DATE
) RETURNS TEXT
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
  v_sequence INTEGER;
  v_date_str TEXT;
BEGIN
  v_date_str := TO_CHAR(p_date, 'YYYYMMDD');
  INSERT INTO sequence_counters (prefix, date, current_value)
  VALUES (p_prefix, v_date_str, 1)
  ON CONFLICT (prefix, date)
  DO UPDATE SET current_value = sequence_counters.current_value + 1
  RETURNING current_value INTO v_sequence;
  RETURN p_prefix || '-' || v_date_str || '-' || LPAD(v_sequence::TEXT, 4, '0');
END;
$$;
```

---

## 4. TypeScript Types & Interfaces

### 4.1 Base Types

```ts
// src/lib/types/base.ts
export type UUID = string;
export type Timestamp = string; // ISO 8601

export interface BaseEntity {
  id: UUID;
  created_at: Timestamp;
  updated_at: Timestamp;
}

export interface SoftDeletable extends BaseEntity {
  deleted_at: Timestamp | null;
}

export interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}
```

### 4.2 User & Auth Types

```ts
// src/lib/types/user.ts
export type UserRole = 'OWNER' | 'ADMIN' | 'DOKTER' | 'KASIR' | 'CUSTOMER';

export interface User extends BaseEntity {
  username: string;
  pin_hash: string;
  role: UserRole;
  full_name: string;
  customer_id: UUID | null;
  created_by: UUID | null;
  failed_login_attempts: number;
  locked_until: Timestamp | null;
  is_active: boolean;
  last_login_at: Timestamp | null;
}

/** User tanpa pin_hash — untuk response ke client */
export type SafeUser = Omit<User, 'pin_hash'>;

export interface LoginCredentials {
  username: string;
  pin: string;
}

export interface LoginResponse {
  user: SafeUser;
  session_token: string;
}

export interface Session {
  user_id: UUID;
  role: UserRole;
  expires_at: Timestamp;
}

export interface CreateUserInput {
  username: string;
  pin: string;
  role: Exclude<UserRole, 'OWNER'>;
  full_name: string;
  customer_id?: UUID;
}

export interface UpdatePinInput {
  old_pin: string;
  new_pin: string;
}
```

### 4.3 Customer & Pet Types

```ts
// src/lib/types/customer.ts
export type CustomerTag = 'VIP' | 'REGULAR' | 'NEW' | 'BLACKLIST';

export interface Customer extends SoftDeletable {
  name: string;
  phone: string | null;
  email: string | null;
  address: string | null;
  emergency_contact: string | null;
  photo_url: string | null;
  notes: string | null;
  is_guest: boolean;
  tags: CustomerTag[];
  is_active: boolean;
}

export interface CreateCustomerInput {
  name: string;
  phone?: string;
  email?: string;
  address?: string;
  emergency_contact?: string;
  photo_url?: string;
  notes?: string;
  is_guest?: boolean;
  tags?: CustomerTag[];
  create_account?: boolean;
  username?: string;
  pin?: string;
}

export type UpdateCustomerInput = Partial<Omit<CreateCustomerInput, 'create_account' | 'username' | 'pin'>>;
```

```ts
// src/lib/types/pet.ts
export interface Pet extends SoftDeletable {
  customer_id: UUID;
  name: string;
  species: string;
  breed: string | null;
  birth_date: string | null;
  gender: string | null;
  photo_url: string | null;
  microchip_number: string | null;
  is_active: boolean;
}

export interface CreatePetInput {
  customer_id: UUID;
  name: string;
  species: string;
  breed?: string;
  birth_date?: string;
  gender?: string;
  photo_url?: string;
  microchip_number?: string;
}

export type UpdatePetInput = Partial<Omit<CreatePetInput, 'customer_id'>>;

export interface PetWeightLog extends BaseEntity {
  pet_id: UUID;
  weight_kg: number;
  recorded_at: string;
}

export interface PetVaccine extends SoftDeletable {
  pet_id: UUID;
  vaccine_name: string;
  vaccination_date: string;
  due_date: string | null;
  notes: string | null;
  is_active: boolean;
}

export interface PetDisease extends SoftDeletable {
  pet_id: UUID;
  disease_name: string;
  diagnosed_date: string | null;
  notes: string | null;
  is_active: boolean;
}

export interface PetAllergy extends SoftDeletable {
  pet_id: UUID;
  allergen: string;
  notes: string | null;
  is_active: boolean;
}
```

### 4.4 Appointment & Medical Record Types

```ts
// src/lib/types/appointment.ts
export type AppointmentStatus = 'WAITING' | 'IN_PROGRESS' | 'DONE' | 'CANCELLED';

export interface Appointment extends BaseEntity {
  customer_id: UUID;
  pet_id: UUID;
  doctor_id: UUID | null;
  appointment_date: string;
  appointment_time: string;
  queue_number: number | null;
  status: AppointmentStatus;
  complaint: string | null;
  notes: string | null;
  is_from_portal: boolean;
}

export interface CreateAppointmentInput {
  customer_id: UUID;
  pet_id: UUID;
  doctor_id?: UUID;
  appointment_date: string;
  appointment_time: string;
  complaint?: string;
  notes?: string;
  is_from_portal?: boolean;
}

export interface UpdateAppointmentStatusInput {
  status: AppointmentStatus;
}
```

```ts
// src/lib/types/medical-record.ts
export type MedicalRecordStatus = 'OPEN' | 'CLOSED';

export interface MedicalRecord extends SoftDeletable {
  record_number: string;
  appointment_id: UUID;
  doctor_id: UUID;
  chief_complaint: string | null;
  history: string | null;
  physical_exam: string | null;
  weight_kg: number | null;
  temperature_c: number | null;
  heart_rate_bpm: number | null;
  respiratory_rate_bpm: number | null;
  diagnosis: string | null;
  treatment: string | null;
  prescription: string | null;
  lab_results: string | null;
  additional_notes: string | null;
  attachments: string[];
  status: MedicalRecordStatus;
}

export interface CreateMedicalRecordInput {
  appointment_id: UUID;
  chief_complaint?: string;
  history?: string;
  physical_exam?: string;
  weight_kg?: number;
  temperature_c?: number;
  heart_rate_bpm?: number;
  respiratory_rate_bpm?: number;
  diagnosis?: string;
  treatment?: string;
  prescription?: string;
  lab_results?: string;
  additional_notes?: string;
  attachments?: string[];
}

export type UpdateMedicalRecordInput = Partial<CreateMedicalRecordInput>;
```

### 4.5 Pet Hotel Types

```ts
// src/lib/types/pet-hotel.ts
export type RoomStatus = 'AVAILABLE' | 'RESERVED' | 'OCCUPIED' | 'MAINTENANCE' | 'INACTIVE';
export type RoomCleanliness = 'CLEAN' | 'DIRTY' | 'UNDER_CLEANING';
export type PetHotelBookingStatus = 'BOOKED' | 'CHECKED_IN' | 'CHECKED_OUT' | 'CANCELLED';
export type PetHotelLogType = 'FEEDING' | 'MEDICINE' | 'NOTE';

export interface Room extends SoftDeletable {
  name: string;
  room_number: string | null;
  room_type: string;
  price_per_night: number;
  capacity: number;
  status: RoomStatus;
  cleanliness: RoomCleanliness;
  maintenance_status: boolean;
  is_active: boolean;
}

export interface PetHotelBooking extends BaseEntity {
  booking_number: string;
  pet_id: UUID;
  customer_id: UUID;
  room_id: UUID | null;
  check_in_date: string;
  check_out_date: string;
  actual_check_in_at: Timestamp | null;
  actual_check_out_at: Timestamp | null;
  price_per_night: number;
  total_price: number;
  status: PetHotelBookingStatus;
  special_notes: string | null;
  is_from_portal: boolean;
}

export interface PetHotelLog extends BaseEntity {
  booking_id: UUID;
  log_type: PetHotelLogType;
  description: string | null;
  photo_urls: string[];
  logged_at: Timestamp;
}

export interface CreatePetHotelBookingInput {
  pet_id: UUID;
  customer_id: UUID;
  room_id?: UUID;
  check_in_date: string;
  check_out_date: string;
  price_per_night?: number;
  special_notes?: string;
  is_from_portal?: boolean;
}

export interface CreatePetHotelLogInput {
  booking_id: UUID;
  log_type: PetHotelLogType;
  description?: string;
  photo_urls?: string[];
}
```

### 4.6 Grooming Types

```ts
// src/lib/types/grooming.ts
export type GroomingBookingStatus = 'BOOKED' | 'IN_PROGRESS' | 'DONE' | 'CANCELLED';

export interface GroomingService extends SoftDeletable {
  name: string;
  description: string | null;
  base_price: number;
  duration_minutes: number | null;
  is_active: boolean;
}

export interface GroomingBooking extends BaseEntity {
  booking_number: string;
  pet_id: UUID;
  customer_id: UUID;
  groomer_id: UUID | null;
  service_id: UUID;
  appointment_date: string;
  appointment_time: string;
  status: GroomingBookingStatus;
  total_price: number;
  notes: string | null;
  is_from_portal: boolean;
}

export interface GroomingRecord extends BaseEntity {
  booking_id: UUID;
  skin_condition: string | null;
  flea_tick_found: boolean;
  recommendations: string | null;
  before_photo_url: string | null;
  after_photo_url: string | null;
}

export interface CreateGroomingBookingInput {
  pet_id: UUID;
  customer_id: UUID;
  groomer_id?: UUID;
  service_id: UUID;
  appointment_date: string;
  appointment_time: string;
  notes?: string;
  is_from_portal?: boolean;
}

export interface CreateGroomingRecordInput {
  booking_id: UUID;
  skin_condition?: string;
  flea_tick_found?: boolean;
  recommendations?: string;
  before_photo_url?: string;
  after_photo_url?: string;
}
```

### 4.7 Product & Inventory Types

```ts
// src/lib/types/product.ts
export type ProductStatus = 'ACTIVE' | 'ARCHIVED';
export type StockMovementType = 'IN' | 'OUT' | 'RETURN' | 'ADJUSTMENT' | 'DAMAGED' | 'EXPIRED' | 'OPNAME';

export interface Category extends BaseEntity {
  name: string;
  description: string | null;
  parent_id: UUID | null;
  is_active: boolean;
}

export interface Supplier extends BaseEntity {
  name: string;
  contact_person: string | null;
  phone: string | null;
  email: string | null;
  address: string | null;
  notes: string | null;
  lead_time_days: number | null;
  is_active: boolean;
}

export interface Product extends SoftDeletable {
  sku: string;
  name: string;
  category_id: UUID | null;
  supplier_id: UUID | null;
  barcode: string | null;
  description: string | null;
  purchase_price: number;
  selling_price: number;
  stock_quantity: number;
  stock_minimum: number;
  stock_maximum: number;
  photo_url: string | null;
  expiry_date: string | null;
  status: ProductStatus;
}

export interface CreateProductInput {
  sku: string;
  name: string;
  category_id?: UUID;
  supplier_id?: UUID;
  barcode?: string;
  description?: string;
  purchase_price: number;
  selling_price: number;
  stock_quantity?: number;
  stock_minimum?: number;
  stock_maximum?: number;
  photo_url?: string;
  expiry_date?: string;
}

export type UpdateProductInput = Partial<Omit<CreateProductInput, 'sku'>>;

export interface ProductVariant extends BaseEntity {
  product_id: UUID;
  variant_name: string;
  variant_value: string;
  price_adjustment: number;
  stock_quantity: number;
}

export interface ProductBundle extends BaseEntity {
  name: string;
  description: string | null;
  bundle_price: number;
  is_active: boolean;
}

export interface StockMovement extends BaseEntity {
  product_id: UUID;
  movement_type: StockMovementType;
  quantity: number;
  reference_type: string | null;
  reference_id: UUID | null;
  notes: string | null;
  created_by: UUID;
}

export interface CreateStockMovementInput {
  product_id: UUID;
  movement_type: StockMovementType;
  quantity: number;
  reference_type?: string;
  reference_id?: UUID;
  notes?: string;
}
```

### 4.8 Purchase Order Types

```ts
// src/lib/types/purchase-order.ts
export type PurchaseOrderStatus = 'DRAFT' | 'SENT' | 'PARTIAL_RECEIVED' | 'RECEIVED' | 'CANCELLED';

export interface PurchaseOrder extends BaseEntity {
  po_number: string;
  supplier_id: UUID;
  order_date: string;
  expected_arrival_date: string | null;
  actual_arrival_date: string | null;
  total_amount: number;
  status: PurchaseOrderStatus;
  notes: string | null;
  created_by: UUID;
}

export interface PurchaseOrderItem extends BaseEntity {
  po_id: UUID;
  product_id: UUID;
  quantity: number;
  unit_price: number;
  received_quantity: number;
}

export interface CreatePurchaseOrderInput {
  supplier_id: UUID;
  order_date: string;
  expected_arrival_date?: string;
  notes?: string;
  items: Array<{
    product_id: UUID;
    quantity: number;
    unit_price: number;
  }>;
}
```

### 4.9 Invoice & Payment Types

```ts
// src/lib/types/invoice.ts
export type InvoiceType = 'POS' | 'CLINICAL' | 'PET_HOTEL' | 'GROOMING' | 'MIXED';
export type InvoiceStatus = 'UNPAID' | 'PARTIAL_PAYMENT' | 'PAID' | 'CANCELLED';
export type PaymentMethod = 'CASH' | 'QRIS' | 'TRANSFER' | 'E_WALLET' | 'CREDIT_CARD' | 'MIXED';

export interface Invoice extends BaseEntity {
  invoice_number: string;
  invoice_type: InvoiceType;
  customer_id: UUID | null;
  subtotal: number;
  discount_amount: number;
  tax_amount: number;
  total_amount: number;
  paid_amount: number;
  status: InvoiceStatus;
  promotion_id: UUID | null;
  loyalty_points_earned: number;
  loyalty_points_redeemed: number;
  notes: string | null;
  created_by: UUID;
}

export interface InvoiceItem extends BaseEntity {
  invoice_id: UUID;
  item_type: string;
  product_id: UUID | null;
  procedure_id: UUID | null;
  pet_hotel_booking_id: UUID | null;
  grooming_booking_id: UUID | null;
  description: string;
  quantity: number;
  unit_price: number;
  total_price: number;
}

export interface Payment extends BaseEntity {
  invoice_id: UUID;
  payment_method: PaymentMethod;
  amount: number;
  reference_number: string | null;
  notes: string | null;
  created_by: UUID;
}

export interface CreateInvoiceInput {
  invoice_type: InvoiceType;
  customer_id?: UUID;
  items: Array<{
    item_type: string;
    product_id?: UUID;
    procedure_id?: UUID;
    pet_hotel_booking_id?: UUID;
    grooming_booking_id?: UUID;
    description: string;
    quantity?: number;
    unit_price: number;
  }>;
  discount_amount?: number;
  tax_amount?: number;
  promotion_id?: UUID;
  loyalty_points_to_redeem?: number;
  notes?: string;
}

export interface RecordPaymentInput {
  invoice_id: UUID;
  payment_method: PaymentMethod;
  amount: number;
  reference_number?: string;
  notes?: string;
}

export interface CashShift extends BaseEntity {
  kasir_id: UUID;
  open_time: Timestamp;
  close_time: Timestamp | null;
  opening_cash: number;
  closing_cash: number | null;
  expected_cash: number | null;
  difference: number | null;
  notes: string | null;
}
```

### 4.10 Loyalty & Promotion Types

```ts
// src/lib/types/loyalty.ts
export type LoyaltyTierName = 'BRONZE' | 'SILVER' | 'GOLD' | 'PLATINUM';
export type LoyaltyTransactionType = 'EARN' | 'REDEEM' | 'EXPIRE' | 'ADJUST';

export interface LoyaltyTierConfig extends BaseEntity {
  tier_name: LoyaltyTierName;
  min_points: number;
  min_spending: number;
  point_multiplier: number;
  benefits: Record<string, unknown>;
}

export interface LoyaltyMember extends BaseEntity {
  customer_id: UUID;
  tier_id: UUID | null;
  total_points: number;
  available_points: number;
  total_spending: number;
  joined_at: Timestamp;
}

export interface LoyaltyTransaction extends BaseEntity {
  member_id: UUID;
  transaction_type: LoyaltyTransactionType;
  points: number;
  invoice_id: UUID | null;
  description: string | null;
}
```

```ts
// src/lib/types/promotion.ts
export type PromotionType = 'PERCENTAGE' | 'FIXED' | 'BUNDLE' | 'HAPPY_HOUR' | 'BIRTHDAY';
export type PromotionStatus = 'ACTIVE' | 'EXPIRED' | 'CANCELLED';

export interface Promotion extends BaseEntity {
  code: string | null;
  name: string;
  description: string | null;
  promotion_type: PromotionType;
  discount_value: number;
  min_purchase: number;
  max_usage: number | null;
  current_usage: number;
  start_date: string;
  end_date: string;
  applicable_products: UUID[] | null;
  status: PromotionStatus;
}

export interface PromotionUsage extends BaseEntity {
  promotion_id: UUID;
  invoice_id: UUID;
  customer_id: UUID | null;
  discount_applied: number;
  used_at: Timestamp;
}

export interface CreatePromotionInput {
  code?: string;
  name: string;
  description?: string;
  promotion_type: PromotionType;
  discount_value: number;
  min_purchase?: number;
  max_usage?: number;
  start_date: string;
  end_date: string;
  applicable_products?: UUID[];
}
```

### 4.11 Expense & Feedback Types

```ts
// src/lib/types/expense.ts
export type ExpenseStatus = 'PENDING' | 'APPROVED' | 'REJECTED' | 'REVERSED';

export interface ExpenseCategory extends BaseEntity {
  name: string;
  description: string | null;
  is_active: boolean;
}

export interface Expense extends BaseEntity {
  expense_date: string;
  category_id: UUID;
  amount: number;
  description: string | null;
  receipt_url: string | null;
  status: ExpenseStatus;
  is_recurring: boolean;
  recurring_day: number | null;
  created_by: UUID;
  approved_by: UUID | null;
}

export interface CreateExpenseInput {
  expense_date: string;
  category_id: UUID;
  amount: number;
  description?: string;
  receipt_url?: string;
  is_recurring?: boolean;
  recurring_day?: number;
}
```

```ts
// src/lib/types/feedback.ts
export type FeedbackRating = '1' | '2' | '3' | '4' | '5';

export interface CustomerFeedback extends BaseEntity {
  customer_id: UUID;
  invoice_id: UUID | null;
  rating: FeedbackRating;
  comment: string | null;
  nps_score: number | null;
}

export interface CreateFeedbackInput {
  customer_id: UUID;
  invoice_id?: UUID;
  rating: FeedbackRating;
  comment?: string;
  nps_score?: number;
}
```

### 4.12 Audit & Notification Types

```ts
// src/lib/types/audit.ts
export interface AuditLog extends BaseEntity {
  user_id: UUID | null;
  action: string;
  entity_type: string;
  entity_id: UUID | null;
  old_values: Record<string, unknown> | null;
  new_values: Record<string, unknown> | null;
  ip_address: string | null;
  user_agent: string | null;
}
```

```ts
// src/lib/types/notification.ts
export interface Notification extends BaseEntity {
  user_id: UUID | null;
  title: string;
  message: string;
  type: string;
  is_read: boolean;
  data: Record<string, unknown> | null;
}
```

---

## 5. Zod Validation Schemas

### 5.1 Base Schemas

```ts
// src/lib/schemas/base.ts
import { z } from 'zod';

export const uuidSchema = z.string().uuid();
export const timestampSchema = z.string().datetime();
export const dateSchema = z.string().regex(/^\d{4}-\d{2}-\d{2}$/);
export const timeSchema = z.string().regex(/^\d{2}:\d{2}(:\d{2})?$/);
```

### 5.2 User Schemas

```ts
// src/lib/schemas/user.ts
import { z } from 'zod';
import { uuidSchema } from './base';

export const userRoleSchema = z.enum(['OWNER', 'ADMIN', 'DOKTER', 'KASIR', 'CUSTOMER']);

export const loginCredentialsSchema = z.object({
  username: z.string().min(3).max(50).regex(/^[a-z0-9._]+$/),
  pin: z.string().length(6).regex(/^\d+$/),
});

export const createUserSchema = z.object({
  username: z.string().min(3).max(50).regex(/^[a-z0-9._]+$/),
  pin: z.string().length(6).regex(/^\d+$/),
  role: z.enum(['ADMIN', 'DOKTER', 'KASIR', 'CUSTOMER']), // OWNER tidak boleh dibuat via API
  full_name: z.string().min(1).max(100),
  customer_id: uuidSchema.nullable().optional(),
});

export const updatePinSchema = z.object({
  old_pin: z.string().length(6).regex(/^\d+$/),
  new_pin: z.string().length(6).regex(/^\d+$/),
});

export const resetPinSchema = z.object({
  target_user_id: uuidSchema,
  new_pin: z.string().length(6).regex(/^\d+$/),
});

export type LoginCredentialsInput = z.infer<typeof loginCredentialsSchema>;
export type CreateUserInput = z.infer<typeof createUserSchema>;
export type UpdatePinInput = z.infer<typeof updatePinSchema>;
export type ResetPinInput = z.infer<typeof resetPinSchema>;
```

### 5.3 Customer Schemas

```ts
// src/lib/schemas/customer.ts
import { z } from 'zod';
import { uuidSchema } from './base';

export const customerTagSchema = z.enum(['VIP', 'REGULAR', 'NEW', 'BLACKLIST']);

export const createCustomerSchema = z.object({
  name: z.string().min(1).max(100),
  phone: z.string().max(20).optional(),
  email: z.string().email().max(100).optional(),
  address: z.string().optional(),
  emergency_contact: z.string().max(100).optional(),
  photo_url: z.string().url().optional(),
  notes: z.string().optional(),
  is_guest: z.boolean().default(false),
  tags: z.array(customerTagSchema).default([]),
  create_account: z.boolean().default(false),
  username: z.string().min(3).max(50).regex(/^[a-z0-9._]+$/).optional(),
  pin: z.string().length(6).regex(/^\d+$/).optional(),
}).refine(
  (data) => {
    if (data.create_account) {
      return !!data.username && !!data.pin;
    }
    return true;
  },
  { message: 'username dan pin wajib jika create_account = true' }
);

export const updateCustomerSchema = createCustomerSchema.partial().omit({
  create_account: true,
  username: true,
  pin: true,
});

export type CreateCustomerInput = z.infer<typeof createCustomerSchema>;
export type UpdateCustomerInput = z.infer<typeof updateCustomerSchema>;
```

### 5.4 Pet Schemas

```ts
// src/lib/schemas/pet.ts
import { z } from 'zod';
import { uuidSchema, dateSchema } from './base';

export const createPetSchema = z.object({
  customer_id: uuidSchema,
  name: z.string().min(1).max(100),
  species: z.string().min(1).max(50),
  breed: z.string().max(50).optional(),
  birth_date: dateSchema.optional(),
  gender: z.string().max(10).optional(),
  photo_url: z.string().url().optional(),
  microchip_number: z.string().max(50).optional(),
}).refine(
  (data) => {
    if (data.birth_date) {
      return new Date(data.birth_date) <= new Date();
    }
    return true;
  },
  { message: 'birth_date tidak boleh di masa depan' }
);

export const updatePetSchema = createPetSchema.partial().omit({ customer_id: true });

export const createPetWeightLogSchema = z.object({
  pet_id: uuidSchema,
  weight_kg: z.number().positive().max(500),
  recorded_at: dateSchema.optional(),
});

export const createPetVaccineSchema = z.object({
  pet_id: uuidSchema,
  vaccine_name: z.string().min(1).max(100),
  vaccination_date: dateSchema,
  due_date: dateSchema.optional(),
  notes: z.string().optional(),
});

export type CreatePetInput = z.infer<typeof createPetSchema>;
export type UpdatePetInput = z.infer<typeof updatePetSchema>;
```

### 5.5 Appointment Schemas

```ts
// src/lib/schemas/appointment.ts
import { z } from 'zod';
import { uuidSchema, dateSchema, timeSchema } from './base';

export const appointmentStatusSchema = z.enum(['WAITING', 'IN_PROGRESS', 'DONE', 'CANCELLED']);

export const createAppointmentSchema = z.object({
  customer_id: uuidSchema,
  pet_id: uuidSchema,
  doctor_id: uuidSchema.nullable().optional(),
  appointment_date: dateSchema,
  appointment_time: timeSchema,
  complaint: z.string().optional(),
  notes: z.string().optional(),
  is_from_portal: z.boolean().default(false),
});

export const updateAppointmentStatusSchema = z.object({
  status: appointmentStatusSchema,
});

export type CreateAppointmentInput = z.infer<typeof createAppointmentSchema>;
export type UpdateAppointmentStatusInput = z.infer<typeof updateAppointmentStatusSchema>;
```

### 5.6 Medical Record Schemas

```ts
// src/lib/schemas/medical-record.ts
import { z } from 'zod';
import { uuidSchema } from './base';

export const medicalRecordStatusSchema = z.enum(['OPEN', 'CLOSED']);

export const createMedicalRecordSchema = z.object({
  appointment_id: uuidSchema,
  chief_complaint: z.string().optional(),
  history: z.string().optional(),
  physical_exam: z.string().optional(),
  weight_kg: z.number().positive().max(500).optional(),
  temperature_c: z.number().min(30).max(45).optional(),
  heart_rate_bpm: z.number().int().positive().optional(),
  respiratory_rate_bpm: z.number().int().positive().optional(),
  diagnosis: z.string().optional(),
  treatment: z.string().optional(),
  prescription: z.string().optional(),
  lab_results: z.string().optional(),
  additional_notes: z.string().optional(),
  attachments: z.array(z.string().url()).optional(),
});

export const updateMedicalRecordSchema = createMedicalRecordSchema.partial();

export type CreateMedicalRecordInput = z.infer<typeof createMedicalRecordSchema>;
export type UpdateMedicalRecordInput = z.infer<typeof updateMedicalRecordSchema>;
```

### 5.7 POS & Invoice Schemas

```ts
// src/lib/schemas/invoice.ts
import { z } from 'zod';
import { uuidSchema } from './base';

export const invoiceTypeSchema = z.enum(['POS', 'CLINICAL', 'PET_HOTEL', 'GROOMING', 'MIXED']);
export const paymentMethodSchema = z.enum(['CASH', 'QRIS', 'TRANSFER', 'E_WALLET', 'CREDIT_CARD', 'MIXED']);

export const invoiceItemSchema = z.object({
  item_type: z.string().min(1),
  product_id: uuidSchema.nullable().optional(),
  procedure_id: uuidSchema.nullable().optional(),
  pet_hotel_booking_id: uuidSchema.nullable().optional(),
  grooming_booking_id: uuidSchema.nullable().optional(),
  description: z.string().min(1).max(200),
  quantity: z.number().int().positive().default(1),
  unit_price: z.number().nonnegative(),
});

export const createInvoiceSchema = z.object({
  invoice_type: invoiceTypeSchema,
  customer_id: uuidSchema.nullable().optional(),
  items: z.array(invoiceItemSchema).min(1),
  discount_amount: z.number().nonnegative().default(0),
  tax_amount: z.number().nonnegative().default(0),
  promotion_id: uuidSchema.nullable().optional(),
  loyalty_points_to_redeem: z.number().int().nonnegative().default(0),
  notes: z.string().optional(),
});

export const recordPaymentSchema = z.object({
  invoice_id: uuidSchema,
  payment_method: paymentMethodSchema,
  amount: z.number().positive(),
  reference_number: z.string().optional(),
  notes: z.string().optional(),
});

export type CreateInvoiceInput = z.infer<typeof createInvoiceSchema>;
export type RecordPaymentInput = z.infer<typeof recordPaymentSchema>;
```

### 5.8 Product Schemas

```ts
// src/lib/schemas/product.ts
import { z } from 'zod';
import { uuidSchema, dateSchema } from './base';

export const createProductSchema = z.object({
  sku: z.string().min(1).max(50).regex(/^\S+$/), // no spaces
  name: z.string().min(1).max(200),
  category_id: uuidSchema.nullable().optional(),
  supplier_id: uuidSchema.nullable().optional(),
  barcode: z.string().max(100).optional(),
  description: z.string().optional(),
  purchase_price: z.number().nonnegative(),
  selling_price: z.number().nonnegative(),
  stock_quantity: z.number().int().nonnegative().default(0),
  stock_minimum: z.number().int().nonnegative().default(0),
  stock_maximum: z.number().int().nonnegative().default(0),
  photo_url: z.string().url().optional(),
  expiry_date: dateSchema.optional(),
}).refine(
  (data) => {
    if (data.stock_minimum && data.stock_maximum) {
      return data.stock_minimum <= data.stock_maximum;
    }
    return true;
  },
  { message: 'stock_minimum harus <= stock_maximum' }
);

export const updateProductSchema = createProductSchema.partial().omit({ sku: true });

export const createStockMovementSchema = z.object({
  product_id: uuidSchema,
  movement_type: z.enum(['IN', 'OUT', 'RETURN', 'ADJUSTMENT', 'DAMAGED', 'EXPIRED', 'OPNAME']),
  quantity: z.number().int().refine((q) => q !== 0, { message: 'quantity tidak boleh 0' }),
  reference_type: z.string().optional(),
  reference_id: uuidSchema.nullable().optional(),
  notes: z.string().optional(),
});

export type CreateProductInput = z.infer<typeof createProductSchema>;
export type UpdateProductInput = z.infer<typeof updateProductSchema>;
export type CreateStockMovementInput = z.infer<typeof createStockMovementSchema>;
```

### 5.9 Loyalty Schemas

```ts
// src/lib/schemas/loyalty.ts
import { z } from 'zod';
import { uuidSchema } from './base';

export const createLoyaltyTierSchema = z.object({
  tier_name: z.enum(['BRONZE', 'SILVER', 'GOLD', 'PLATINUM']),
  min_points: z.number().int().nonnegative(),
  min_spending: z.number().nonnegative(),
  point_multiplier: z.number().positive(),
  benefits: z.record(z.unknown()),
});

export const redeemPointsSchema = z.object({
  customer_id: uuidSchema,
  points_to_redeem: z.number().int().positive(),
  invoice_id: uuidSchema.nullable().optional(),
});

export type CreateLoyaltyTierInput = z.infer<typeof createLoyaltyTierSchema>;
export type RedeemPointsInput = z.infer<typeof redeemPointsSchema>;
```

### 5.10 Promotion Schemas

```ts
// src/lib/schemas/promotion.ts
import { z } from 'zod';
import { uuidSchema, dateSchema } from './base';

export const promotionTypeSchema = z.enum(['PERCENTAGE', 'FIXED', 'BUNDLE', 'HAPPY_HOUR', 'BIRTHDAY']);

export const createPromotionSchema = z.object({
  code: z.string().max(50).optional(),
  name: z.string().min(1).max(200),
  description: z.string().optional(),
  promotion_type: promotionTypeSchema,
  discount_value: z.number().nonnegative(),
  min_purchase: z.number().nonnegative().default(0),
  max_usage: z.number().int().positive().nullable().optional(),
  start_date: dateSchema,
  end_date: dateSchema,
  applicable_products: z.array(uuidSchema).nullable().optional(),
}).refine(
  (data) => new Date(data.end_date) >= new Date(data.start_date),
  { message: 'end_date harus >= start_date' }
);

export const applyPromoCodeSchema = z.object({
  code: z.string().min(1),
  subtotal: z.number().nonnegative(),
  customer_id: uuidSchema.nullable().optional(),
});

export type CreatePromotionInput = z.infer<typeof createPromotionSchema>;
export type ApplyPromoCodeInput = z.infer<typeof applyPromoCodeSchema>;
```

---

## 6. Service Layer Contracts

### 6.1 Base Service Pattern

```ts
// src/lib/services/base.service.ts
import { supabase } from '$lib/supabase';
import { AppError, ErrorCode } from '$lib/errors';
import { logAudit } from '$lib/audit';
import type { UUID } from '$lib/types/base';

/**
 * Base service pattern yang harus diikuti semua domain service.
 * Setiap method WAJIB:
 * 1. Validasi input via Zod
 * 2. Handle error dengan ErrorCode eksplisit
 * 3. Audit log untuk operasi write
 * 4. Return typed response
 */
export abstract class BaseService {
  protected supabase = supabase;

  protected handleError(error: unknown, context: string): never {
    if (error instanceof AppError) throw error;
    if (error instanceof Error) {
      throw new AppError(
        ErrorCode.INTERNAL_ERROR,
        `${context}: ${error.message}`
      );
    }
    throw new AppError(ErrorCode.INTERNAL_ERROR, context);
  }

  protected async audit(params: {
    userId: UUID;
    action: string;
    entityType: string;
    entityId?: UUID;
    oldValues?: Record<string, unknown>;
    newValues?: Record<string, unknown>;
  }): Promise<void> {
    await logAudit(params);
  }
}
```

### 6.2 Domain Service Contracts

```ts
// src/lib/services/customer.service.ts
export const CustomerService = {
  list: async (params: {
    search?: string;
    isGuest?: boolean;
    isActive?: boolean;
    page?: number;
    limit?: number;
  }) => Promise<PaginatedResponse<Customer>>,

  getById: async (id: UUID) => Promise<Customer>,

  create: async (input: CreateCustomerInput, callerId: UUID) => Promise<Customer>,

  update: async (id: UUID, input: UpdateCustomerInput, callerId: UUID) => Promise<Customer>,

  delete: async (id: UUID, callerId: UUID) => Promise<void>,

  convertGuest: async (id: UUID, data: Partial<Customer>, callerId: UUID) => Promise<Customer>,
};
```

```ts
// src/lib/services/product.service.ts
export const ProductService = {
  list: async (params: {
    search?: string;
    categoryId?: UUID;
    status?: ProductStatus;
    page?: number;
    limit?: number;
  }) => Promise<PaginatedResponse<Product>>,

  getById: async (id: UUID) => Promise<Product>,

  getBySku: async (sku: string) => Promise<Product>,

  getByBarcode: async (barcode: string) => Promise<Product>,

  create: async (input: CreateProductInput, callerId: UUID) => Promise<Product>,

  update: async (id: UUID, input: UpdateProductInput, callerId: UUID) => Promise<Product>,

  archive: async (id: UUID, callerId: UUID) => Promise<void>,

  delete: async (id: UUID, callerId: UUID) => Promise<void>,
};
```

```ts
// src/lib/services/invoice.service.ts
export const InvoiceService = {
  list: async (params: {
    customerId?: UUID;
    status?: InvoiceStatus;
    type?: InvoiceType;
    startDate?: string;
    endDate?: string;
    page?: number;
    limit?: number;
  }) => Promise<PaginatedResponse<Invoice>>,

  getById: async (id: UUID) => Promise<Invoice & { items: InvoiceItem[] }>,

  create: async (input: CreateInvoiceInput, callerId: UUID) => Promise<Invoice>,

  recordPayment: async (input: RecordPaymentInput, callerId: UUID) => Promise<{ payment: Payment; invoice: Invoice }>,

  cancel: async (id: UUID, reason: string, callerId: UUID) => Promise<Invoice>,

  getDailySales: async (date: string, kasirId?: UUID) => Promise<DailySalesReport>,
};
```

```ts
// src/lib/services/inventory.service.ts
export const InventoryService = {
  getStock: async (productId: UUID) => Promise<{ product_id: UUID; stock_quantity: number }>,

  recordMovement: async (input: CreateStockMovementInput, callerId: UUID) => Promise<StockMovement>,

  stockOpname: async (productId: UUID, actualQuantity: number, notes: string, callerId: UUID) => Promise<StockMovement>,

  getLowStock: async () => Promise<Product[]>,

  getMovements: async (productId: UUID, page?: number, limit?: number) => Promise<PaginatedResponse<StockMovement>>,
};
```

```ts
// src/lib/services/appointment.service.ts
export const AppointmentService = {
  list: async (params: {
    date?: string;
    doctorId?: UUID;
    status?: AppointmentStatus;
    customerId?: UUID;
  }) => Promise<Appointment[]>,

  create: async (input: CreateAppointmentInput, callerId: UUID) => Promise<Appointment>,

  updateStatus: async (id: UUID, newStatus: AppointmentStatus, callerId: UUID) => Promise<Appointment>,

  cancel: async (id: UUID, reason: string, callerId: UUID) => Promise<Appointment>,
};
```

```ts
// src/lib/services/auth.service.ts
export const AuthService = {
  login: async (credentials: LoginCredentials) => Promise<LoginResponse>,

  logout: async () => Promise<void>,

  changePin: async (input: UpdatePinInput, userId: UUID) => Promise<void>,

  resetPin: async (input: ResetPinInput, callerId: UUID) => Promise<void>,

  createUser: async (input: CreateUserInput, callerId: UUID) => Promise<SafeUser>,

  deactivateUser: async (userId: UUID, callerId: UUID) => Promise<void>,
};
```

---

## 7. Svelte Query Hooks Contracts

### 7.1 Base Pattern

```ts
// src/lib/hooks/use-query.ts
import { createQuery, createMutation } from '@tanstack/svelte-query';

/**
 * PENTING: Svelte Query Hooks
 * - Gunakan createQuery (bukan useQuery)
 * - Gunakan createMutation (bukan useMutation)
 * - Query key harus reactive (gunakan $derived atau getter)
 * - Return value adalah Svelte store, akses dengan $store atau auto-subscribed
 */
```

### 7.2 Query Hooks per Domain

```ts
// src/lib/hooks/use-customers.ts
import { createQuery, createMutation } from '@tanstack/svelte-query';
import { CustomerService } from '$lib/services/customer.service';
import { queryClient } from '$lib/query-client';
import { getCurrentUserId } from '$lib/auth';

export function useCustomers(params: () => { search?: string; page?: number }) {
  return createQuery({
    get queryKey() { return ['customers', params()] as const; },
    queryFn: () => CustomerService.list(params()),
  });
}

export function useCustomer(id: () => string | undefined) {
  return createQuery({
    get queryKey() { return ['customers', id()] as const; },
    queryFn: () => CustomerService.getById(id()!),
    get enabled() { return !!id(); },
  });
}

export function useCreateCustomer() {
  return createMutation({
    mutationFn: (input: CreateCustomerInput) =>
      CustomerService.create(input, getCurrentUserId()),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['customers'] });
    },
  });
}
```

```ts
// src/lib/hooks/use-products.ts
import { createQuery, createMutation } from '@tanstack/svelte-query';
import { ProductService } from '$lib/services/product.service';
import { queryClient } from '$lib/query-client';
import { getCurrentUserId } from '$lib/auth';

export function useProducts(params: () => { search?: string; status?: ProductStatus }) {
  return createQuery({
    get queryKey() { return ['products', params()] as const; },
    queryFn: () => ProductService.list(params()),
  });
}

export function useCreateProduct() {
  return createMutation({
    mutationFn: (input: CreateProductInput) =>
      ProductService.create(input, getCurrentUserId()),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['products'] });
    },
  });
}
```

```ts
// src/lib/hooks/use-invoices.ts
import { createQuery, createMutation } from '@tanstack/svelte-query';
import { InvoiceService } from '$lib/services/invoice.service';
import { queryClient } from '$lib/query-client';
import { getCurrentUserId } from '$lib/auth';

export function useInvoices(params: () => { status?: InvoiceStatus; page?: number }) {
  return createQuery({
    get queryKey() { return ['invoices', params()] as const; },
    queryFn: () => InvoiceService.list(params()),
  });
}

export function useCreateInvoice() {
  return createMutation({
    mutationFn: (input: CreateInvoiceInput) =>
      InvoiceService.create(input, getCurrentUserId()),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['invoices'] });
      queryClient.invalidateQueries({ queryKey: ['products'] }); // stock berubah
    },
  });
}

export function useRecordPayment() {
  return createMutation({
    mutationFn: (input: RecordPaymentInput) =>
      InvoiceService.recordPayment(input, getCurrentUserId()),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['invoices'] });
    },
  });
}
```

```ts
// src/lib/hooks/use-appointments.ts
import { createQuery, createMutation } from '@tanstack/svelte-query';
import { AppointmentService } from '$lib/services/appointment.service';
import { queryClient } from '$lib/query-client';
import { getCurrentUserId } from '$lib/auth';

export function useAppointments(date: () => string) {
  return createQuery({
    get queryKey() { return ['appointments', date()] as const; },
    queryFn: () => AppointmentService.list({ date: date() }),
  });
}

export function useCreateAppointment() {
  return createMutation({
    mutationFn: (input: CreateAppointmentInput) =>
      AppointmentService.create(input, getCurrentUserId()),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['appointments'] });
    },
  });
}
```

### 7.3 Query Key Convention

```ts
// src/lib/query-keys.ts
import type { UUID } from '$lib/types/base';

export const QueryKeys = {
  // Auth
  currentUser: ['auth', 'current-user'] as const,

  // CRM
  customers: (params?: Record<string, unknown>) => ['customers', params] as const,
  customer: (id: UUID) => ['customers', id] as const,
  pets: (customerId: UUID) => ['pets', customerId] as const,
  pet: (id: UUID) => ['pets', id] as const,

  // Appointments
  appointments: (params?: Record<string, unknown>) => ['appointments', params] as const,
  appointment: (id: UUID) => ['appointments', id] as const,
  medicalRecords: (appointmentId: UUID) => ['medical-records', appointmentId] as const,

  // Pet Hotel
  rooms: ['rooms'] as const,
  petHotelBookings: (params?: Record<string, unknown>) => ['pet-hotel-bookings', params] as const,

  // Grooming
  groomingServices: ['grooming-services'] as const,
  groomingBookings: (params?: Record<string, unknown>) => ['grooming-bookings', params] as const,

  // Products & Inventory
  products: (params?: Record<string, unknown>) => ['products', params] as const,
  product: (id: UUID) => ['products', id] as const,
  stock: (productId: UUID) => ['stock', productId] as const,
  lowStock: ['stock', 'low'] as const,
  stockMovements: (productId: UUID) => ['stock-movements', productId] as const,

  // POS
  invoices: (params?: Record<string, unknown>) => ['invoices', params] as const,
  invoice: (id: UUID) => ['invoices', id] as const,
  dailySales: (date: string) => ['daily-sales', date] as const,

  // Loyalty
  loyaltyMember: (customerId: UUID) => ['loyalty', customerId] as const,
  loyaltyTransactions: (memberId: UUID) => ['loyalty-transactions', memberId] as const,

  // Promotions
  promotions: ['promotions'] as const,

  // Expenses
  expenses: (params?: Record<string, unknown>) => ['expenses', params] as const,

  // Reports
  revenueReport: (params: Record<string, unknown>) => ['reports', 'revenue', params] as const,
  pnlReport: (params: Record<string, unknown>) => ['reports', 'pnl', params] as const,
  inventoryValuation: ['reports', 'inventory-valuation'] as const,

  // Settings
  settings: ['settings'] as const,

  // Notifications
  notifications: (userId: UUID) => ['notifications', userId] as const,
  unreadCount: (userId: UUID) => ['notifications', userId, 'unread'] as const,
} as const;
```

---

## 8. Component Props Contracts (Svelte 5 Runes)

### 8.1 Svelte 5 Props Pattern (CRITICAL)

```ts
/**
 * ⚠️ ATURAN PENTING SVELTE 5 PROPS ⚠️
 *
 * Di Svelte 5, gunakan $props() rune untuk mendeklarasikan props.
 * Props bersifat reactive secara default — tidak perlu accessor function.
 *
 * ✅ BENAR (Svelte 5 pattern):
 *   let { value, label, onClick }: Props = $props();
 *
 * ✅ Untuk optional props dengan default:
 *   let { value = '', disabled = false }: Props = $props();
 *
 * ✅ Untuk bindable props (two-way binding):
 *   let { value = $bindable() }: Props = $props();
 *
 * ✅ Untuk rest props (spread ke element lain):
 *   let { class: className, ...rest }: Props = $props();
 *
 * ❌ SALAH (Svelte 4 legacy pattern):
 *   export let value;  // jangan gunakan di Svelte 5
 */
```

### 8.2 Base Component Props

```ts
// src/lib/components/ui/types.ts
import type { Snippet } from 'svelte';

export interface BaseComponentProps {
  class?: string;
  classList?: Record<string, boolean>;
}

export interface ButtonProps extends BaseComponentProps {
  variant?: 'default' | 'outline' | 'ghost' | 'destructive' | 'secondary';
  size?: 'sm' | 'md' | 'lg' | 'icon';
  disabled?: boolean;
  loading?: boolean;
  type?: 'button' | 'submit' | 'reset';
  onClick?: (e: MouseEvent) => void;
  children: Snippet;
}

export interface InputProps extends BaseComponentProps {
  type?: string;
  placeholder?: string;
  value?: string;
  onInput?: (e: Event) => void;
  onChange?: (e: Event) => void;
  disabled?: boolean;
  error?: string;
  label?: string;
  required?: boolean;
}

export interface SelectProps<T extends string = string> extends BaseComponentProps {
  options: Array<{ value: T; label: string }>;
  value?: T;
  onChange?: (value: T) => void;
  placeholder?: string;
  disabled?: boolean;
  error?: string;
  label?: string;
}

export interface DataTableProps<T> extends BaseComponentProps {
  data: T[];
  columns: ColumnDef<T>[];
  isLoading?: boolean;
  onRowClick?: (row: T) => void;
  pagination?: {
    page: number;
    limit: number;
    total: number;
    onPageChange: (page: number) => void;
  };
}

export interface ModalProps extends BaseComponentProps {
  open: boolean;
  onClose: () => void;
  title: string;
  children: Snippet;
  size?: 'sm' | 'md' | 'lg' | 'xl';
}

export interface StatusBadgeProps extends BaseComponentProps {
  status: string;
  variant?: 'default' | 'success' | 'warning' | 'error' | 'info';
}
```

### 8.3 Contoh Implementasi Komponen (Svelte 5)

```svelte
<!-- src/lib/components/ui/button.svelte -->
<script lang="ts">
  import type { ButtonProps } from './types';
  import { cn } from '$lib/utils';
  import { Loader2 } from 'lucide-svelte';

  let {
    variant = 'default',
    size = 'md',
    disabled = false,
    loading = false,
    type = 'button',
    onClick,
    class: className,
    children
  }: ButtonProps = $props();

  const variantClasses = {
    default: 'bg-primary text-primary-foreground hover:bg-primary/90',
    outline: 'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
    ghost: 'hover:bg-accent hover:text-accent-foreground',
    destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
    secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
  };

  const sizeClasses = {
    sm: 'h-9 px-3 text-sm',
    md: 'h-10 px-4 text-sm',
    lg: 'h-11 px-8 text-base',
    icon: 'h-10 w-10',
  };
</script>

<button
  {type}
  {disabled}
  disabled={disabled || loading}
  onclick={onClick}
  class={cn(
    'inline-flex items-center justify-center rounded-md font-medium transition-colors',
    'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring',
    'disabled:pointer-events-none disabled:opacity-50',
    variantClasses[variant],
    sizeClasses[size],
    className
  )}
>
  {#if loading}
    <Loader2 class="mr-2 h-4 w-4 animate-spin" />
  {/if}
  {@render children()}
</button>
```

### 8.4 Feature Component Props

```ts
// src/lib/components/features/pos/pos-cart.types.ts
import type { UUID } from '$lib/types/base';
import type { Product } from '$lib/types/product';
import type { Appointment, AppointmentStatus } from '$lib/types/appointment';
import type { Room } from '$lib/types/pet-hotel';

export interface CartItem {
  product_id: UUID;
  name: string;
  quantity: number;
  unit_price: number;
  total_price: number;
}

export interface PosCartProps {
  items: CartItem[];
  onAddItem: (product: Product, quantity: number) => void;
  onRemoveItem: (productId: UUID) => void;
  onUpdateQuantity: (productId: UUID, quantity: number) => void;
  onCheckout: () => void;
  discount: number;
  tax: number;
}

export interface AppointmentBoardProps {
  date: string;
  onStatusChange: (id: UUID, status: AppointmentStatus) => void;
  onAppointmentClick: (appointment: Appointment) => void;
}

export interface RoomGridProps {
  rooms: Room[];
  dateRange: { start: string; end: string };
  onRoomClick: (room: Room) => void;
}
```

---

## 9. Row Level Security (RLS) Policies

### 9.1 Helper Functions

```sql
-- Sudah didefinisikan di schema:
-- get_user_role(p_user_id UUID) → user_role
-- get_customer_id(p_user_id UUID) → UUID
```

### 9.2 RLS Policies per Table

```sql
-- ============================================
-- CUSTOMERS
-- ============================================
ALTER TABLE customers ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Owner and Admin can do all" ON customers
  FOR ALL TO authenticated
  USING (get_user_role(auth.uid()) IN ('OWNER', 'ADMIN'))
  WITH CHECK (get_user_role(auth.uid()) IN ('OWNER', 'ADMIN'));

CREATE POLICY "Dokter and Kasir can read" ON customers
  FOR SELECT TO authenticated
  USING (get_user_role(auth.uid()) IN ('DOKTER', 'KASIR'));

CREATE POLICY "Customer can read own" ON customers
  FOR SELECT TO authenticated
  USING (id = get_customer_id(auth.uid()));

-- ============================================
-- PETS
-- ============================================
ALTER TABLE pets ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Staff can do all" ON pets
  FOR ALL TO authenticated
  USING (get_user_role(auth.uid()) IN ('OWNER', 'ADMIN'))
  WITH CHECK (get_user_role(auth.uid()) IN ('OWNER', 'ADMIN'));

CREATE POLICY "Dokter and Kasir can read" ON pets
  FOR SELECT TO authenticated
  USING (get_user_role(auth.uid()) IN ('DOKTER', 'KASIR'));

CREATE POLICY "Customer can read own pets" ON pets
  FOR SELECT TO authenticated
  USING (customer_id = get_customer_id(auth.uid()));

-- ============================================
-- APPOINTMENTS
-- ============================================
ALTER TABLE appointments ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Owner and Admin full access" ON appointments
  FOR ALL TO authenticated
  USING (get_user_role(auth.uid()) IN ('OWNER', 'ADMIN'));

CREATE POLICY "Dokter can read all, write own" ON appointments
  FOR SELECT TO authenticated
  USING (get_user_role(auth.uid()) = 'DOKTER');

CREATE POLICY "Dokter can update own" ON appointments
  FOR UPDATE TO authenticated
  USING (get_user_role(auth.uid()) = 'DOKTER' AND doctor_id = auth.uid());

CREATE POLICY "Kasir can read" ON appointments
  FOR SELECT TO authenticated
  USING (get_user_role(auth.uid()) = 'KASIR');

CREATE POLICY "Customer can manage own" ON appointments
  FOR ALL TO authenticated
  USING (customer_id = get_customer_id(auth.uid()))
  WITH CHECK (customer_id = get_customer_id(auth.uid()));

-- ============================================
-- MEDICAL RECORDS
-- ============================================
ALTER TABLE medical_records ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Owner and Admin full access" ON medical_records
  FOR ALL TO authenticated
  USING (get_user_role(auth.uid()) IN ('OWNER', 'ADMIN'));

CREATE POLICY "Dokter can read all, write own" ON medical_records
  FOR SELECT TO authenticated
  USING (get_user_role(auth.uid()) = 'DOKTER');

CREATE POLICY "Dokter can create/update own" ON medical_records
  FOR INSERT TO authenticated
  WITH CHECK (get_user_role(auth.uid()) = 'DOKTER' AND doctor_id = auth.uid());

CREATE POLICY "Dokter can update own records" ON medical_records
  FOR UPDATE TO authenticated
  USING (get_user_role(auth.uid()) = 'DOKTER' AND doctor_id = auth.uid());

-- ============================================
-- PRODUCTS
-- ============================================
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Owner and Admin can write" ON products
  FOR ALL TO authenticated
  USING (get_user_role(auth.uid()) IN ('OWNER', 'ADMIN'))
  WITH CHECK (get_user_role(auth.uid()) IN ('OWNER', 'ADMIN'));

CREATE POLICY "All authenticated can read active" ON products
  FOR SELECT TO authenticated
  USING (
    status = 'ACTIVE'
    OR get_user_role(auth.uid()) IN ('OWNER', 'ADMIN')
  );

-- ============================================
-- INVOICES
-- ============================================
ALTER TABLE invoices ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Owner and Admin full access" ON invoices
  FOR ALL TO authenticated
  USING (get_user_role(auth.uid()) IN ('OWNER', 'ADMIN'));

CREATE POLICY "Kasir can create and read" ON invoices
  FOR INSERT TO authenticated
  WITH CHECK (get_user_role(auth.uid()) = 'KASIR');

CREATE POLICY "Kasir can read" ON invoices
  FOR SELECT TO authenticated
  USING (get_user_role(auth.uid()) = 'KASIR');

CREATE POLICY "Customer can read own" ON invoices
  FOR SELECT TO authenticated
  USING (customer_id = get_customer_id(auth.uid()));

-- ============================================
-- PAYMENTS
-- ============================================
ALTER TABLE payments ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Owner, Admin, Kasir can manage" ON payments
  FOR ALL TO authenticated
  USING (get_user_role(auth.uid()) IN ('OWNER', 'ADMIN', 'KASIR'))
  WITH CHECK (get_user_role(auth.uid()) IN ('OWNER', 'ADMIN', 'KASIR'));

-- ============================================
-- LOYALTY MEMBERS
-- ============================================
ALTER TABLE loyalty_members ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Owner and Admin full access" ON loyalty_members
  FOR ALL TO authenticated
  USING (get_user_role(auth.uid()) IN ('OWNER', 'ADMIN'));

CREATE POLICY "Customer can read own" ON loyalty_members
  FOR SELECT TO authenticated
  USING (customer_id = get_customer_id(auth.uid()));

-- ============================================
-- AUDIT LOGS (read-only untuk Owner & Admin)
-- ============================================
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Owner and Admin can read" ON audit_logs
  FOR SELECT TO authenticated
  USING (get_user_role(auth.uid()) IN ('OWNER', 'ADMIN'));

-- ============================================
-- NOTIFICATIONS
-- ============================================
ALTER TABLE notifications ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can read own" ON notifications
  FOR SELECT TO authenticated
  USING (user_id = auth.uid() OR user_id IS NULL);

CREATE POLICY "Users can update own" ON notifications
  FOR UPDATE TO authenticated
  USING (user_id = auth.uid());

-- ============================================
-- SETTINGS (Owner only)
-- ============================================
ALTER TABLE settings ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Owner full access" ON settings
  FOR ALL TO authenticated
  USING (get_user_role(auth.uid()) = 'OWNER')
  WITH CHECK (get_user_role(auth.uid()) = 'OWNER');

CREATE POLICY "Admin can read" ON settings
  FOR SELECT TO authenticated
  USING (get_user_role(auth.uid()) = 'ADMIN');
```

---

## 10. State Management Contracts

### 10.1 Svelte 5 Runes (TIDAK pakai Zustand/Redux)

```ts
// src/lib/stores/auth.store.ts
import type { SafeUser, Session } from '$lib/types/user';

interface AuthState {
  user: SafeUser | null;
  session: Session | null;
  isAuthenticated: boolean;
  isLoading: boolean;
}

// Module-level reactive state
let state = $state<AuthState>({
  user: null,
  session: null,
  isAuthenticated: false,
  isLoading: true,
});

export const authStore = {
  get state() { return state; },
  get user() { return state.user; },
  get isAuthenticated() { return state.isAuthenticated; },
  get isLoading() { return state.isLoading; },

  setUser(user: SafeUser, session: Session) {
    state.user = user;
    state.session = session;
    state.isAuthenticated = true;
    state.isLoading = false;
  },

  clearUser() {
    state.user = null;
    state.session = null;
    state.isAuthenticated = false;
    state.isLoading = false;
  },

  setLoading(loading: boolean) {
    state.isLoading = loading;
  },
};
```

### 10.2 POS Cart Store

```ts
// src/lib/stores/cart.store.ts
import type { UUID } from '$lib/types/base';

export interface CartItem {
  product_id: UUID;
  name: string;
  quantity: number;
  unit_price: number;
  total_price: number;
}

interface CartState {
  items: CartItem[];
  discount: number;
  tax: number;
  customerId: UUID | null;
  promotionId: UUID | null;
  loyaltyPointsToRedeem: number;
}

let state = $state<CartState>({
  items: [],
  discount: 0,
  tax: 0,
  customerId: null,
  promotionId: null,
  loyaltyPointsToRedeem: 0,
});

// Derived values (computed automatically)
const subtotal = $derived(state.items.reduce((sum, item) => sum + item.total_price, 0));
const total = $derived(subtotal - state.discount + state.tax);

export const cartStore = {
  get items() { return state.items; },
  get discount() { return state.discount; },
  get tax() { return state.tax; },
  get customerId() { return state.customerId; },
  get subtotal() { return subtotal; },
  get total() { return total; },

  addItem(item: CartItem) {
    const existingIndex = state.items.findIndex(
      (i) => i.product_id === item.product_id
    );
    if (existingIndex >= 0) {
      const newQty = state.items[existingIndex].quantity + item.quantity;
      state.items[existingIndex].quantity = newQty;
      state.items[existingIndex].total_price = state.items[existingIndex].unit_price * newQty;
    } else {
      state.items.push(item);
    }
  },

  removeItem(productId: UUID) {
    state.items = state.items.filter((i) => i.product_id !== productId);
  },

  updateQuantity(productId: UUID, quantity: number) {
    const index = state.items.findIndex((i) => i.product_id === productId);
    if (index >= 0) {
      state.items[index].quantity = quantity;
      state.items[index].total_price = state.items[index].unit_price * quantity;
    }
  },

  setCustomer(customerId: UUID | null) {
    state.customerId = customerId;
  },

  setDiscount(discount: number) {
    state.discount = discount;
  },

  setTax(tax: number) {
    state.tax = tax;
  },

  setPromotion(promotionId: UUID | null) {
    state.promotionId = promotionId;
  },

  setLoyaltyPoints(points: number) {
    state.loyaltyPointsToRedeem = points;
  },

  clearCart() {
    state.items = [];
    state.discount = 0;
    state.tax = 0;
    state.customerId = null;
    state.promotionId = null;
    state.loyaltyPointsToRedeem = 0;
  },
};
```

### 10.3 UI State

```ts
// src/lib/stores/ui.store.ts
let isSidebarOpen = $state(true);
let activeShiftId = $state<string | null>(null);
let globalLoading = $state(false);

export interface Toast {
  id: string;
  type: 'success' | 'error' | 'warning' | 'info';
  title: string;
  message?: string;
}

let toasts = $state<Toast[]>([]);

export const uiStore = {
  get isSidebarOpen() { return isSidebarOpen; },
  get activeShiftId() { return activeShiftId; },
  get globalLoading() { return globalLoading; },
  get toasts() { return toasts; },

  toggleSidebar() {
    isSidebarOpen = !isSidebarOpen;
  },

  setSidebarOpen(open: boolean) {
    isSidebarOpen = open;
  },

  setActiveShiftId(id: string | null) {
    activeShiftId = id;
  },

  setGlobalLoading(loading: boolean) {
    globalLoading = loading;
  },

  addToast(toast: Omit<Toast, 'id'>) {
    const id = crypto.randomUUID();
    toasts.push({ ...toast, id });
    setTimeout(() => {
      toasts = toasts.filter((t) => t.id !== id);
    }, 5000);
  },
};
```

---

## 11. Utility Functions Contracts

### 11.1 Format Utilities

```ts
// src/lib/utils/format.ts
/** Format angka ke Rupiah */
export function formatRupiah(amount: number): string {
  return new Intl.NumberFormat('id-ID', {
    style: 'currency',
    currency: 'IDR',
    minimumFractionDigits: 0,
  }).format(amount);
}

/** Format tanggal ke format Indonesia */
export function formatDate(date: string | Date): string {
  return new Intl.DateTimeFormat('id-ID', {
    day: '2-digit',
    month: 'long',
    year: 'numeric',
  }).format(new Date(date));
}

/** Format waktu */
export function formatTime(time: string): string {
  return time.slice(0, 5); // "08:00"
}

/** Format datetime */
export function formatDateTime(date: string | Date): string {
  return new Intl.DateTimeFormat('id-ID', {
    day: '2-digit',
    month: 'short',
    year: 'numeric',
    hour: '2-digit',
    minute: '2-digit',
  }).format(new Date(date));
}
```

### 11.2 Validation Utilities

```ts
// src/lib/utils/validation.ts
import { z } from 'zod';
import { AppError, ErrorCode } from '$lib/errors';

/** Validasi dan throw AppError jika invalid */
export function validateOrThrow<T>(schema: z.ZodSchema<T>, data: unknown): T {
  const result = schema.safeParse(data);
  if (!result.success) {
    throw new AppError(
      ErrorCode.VALIDATION_ERROR,
      'Validation failed',
      result.error.errors
    );
  }
  return result.data;
}
```

### 11.3 Date Utilities

```ts
// src/lib/utils/date.ts
export function isToday(date: string): boolean {
  const d = new Date(date);
  const now = new Date();
  return d.toDateString() === now.toDateString();
}

export function isPast(date: string): boolean {
  return new Date(date) < new Date();
}

export function isFuture(date: string): boolean {
  return new Date(date) > new Date();
}

export function daysBetween(start: string, end: string): number {
  const s = new Date(start);
  const e = new Date(end);
  return Math.ceil((e.getTime() - s.getTime()) / (1000 * 60 * 60 * 24));
}

export function isWithinOperatingHours(time: string, config: { open: string; close: string }): boolean {
  return time >= config.open && time <= config.close;
}
```

### 11.4 Class Name Utility (Tailwind merge)

```ts
// src/lib/utils/cn.ts
import { type ClassValue, clsx } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

---

## 12. Error Handling Contracts

### 12.1 AppError Class

```ts
// src/lib/errors.ts
export enum ErrorCode {
  // Auth
  INVALID_CREDENTIALS = 'INVALID_CREDENTIALS',
  ACCOUNT_LOCKED = 'ACCOUNT_LOCKED',
  ACCOUNT_INACTIVE = 'ACCOUNT_INACTIVE',
  INVALID_OLD_PIN = 'INVALID_OLD_PIN',

  // Validation
  BAD_REQUEST = 'BAD_REQUEST',
  VALIDATION_ERROR = 'VALIDATION_ERROR',
  INTERNAL_ERROR = 'INTERNAL_ERROR',

  // Authorization
  FORBIDDEN = 'FORBIDDEN',
  NOT_FOUND = 'NOT_FOUND',
  UNAUTHORIZED = 'UNAUTHORIZED',

  // Business
  CONFLICT = 'CONFLICT',
  INVALID_STATE_TRANSITION = 'INVALID_STATE_TRANSITION',
  INSUFFICIENT_STOCK = 'INSUFFICIENT_STOCK',
  INSUFFICIENT_LOYALTY_POINTS = 'INSUFFICIENT_LOYALTY_POINTS',
  PROMOTION_INVALID = 'PROMOTION_INVALID',
  PROMO_NOT_FOUND = 'PROMO_NOT_FOUND',
  SKU_ALREADY_EXISTS = 'SKU_ALREADY_EXISTS',
  BARCODE_ALREADY_EXISTS = 'BARCODE_ALREADY_EXISTS',
  USERNAME_ALREADY_EXISTS = 'USERNAME_ALREADY_EXISTS',
  CANNOT_DELETE_HAS_REFERENCES = 'CANNOT_DELETE_HAS_REFERENCES',
  APPOINTMENT_NOT_IN_PROGRESS = 'APPOINTMENT_NOT_IN_PROGRESS',
  MEDICAL_RECORD_ALREADY_EXISTS = 'MEDICAL_RECORD_ALREADY_EXISTS',
  ROOM_NOT_AVAILABLE = 'ROOM_NOT_AVAILABLE',
  BOOKING_NOT_ACTIVE = 'BOOKING_NOT_ACTIVE',
  GROOMER_NOT_AVAILABLE = 'GROOMER_NOT_AVAILABLE',
  INVOICE_CANCELLED = 'INVOICE_CANCELLED',
  INVOICE_ALREADY_PAID = 'INVOICE_ALREADY_PAID',
  ALREADY_CANCELLED = 'ALREADY_CANCELLED',
  ALREADY_REGISTERED = 'ALREADY_REGISTERED',
  CUSTOMER_NOT_FOUND = 'CUSTOMER_NOT_FOUND',
  APPOINTMENT_NOT_FOUND = 'APPOINTMENT_NOT_FOUND',
  NO_LOYALTY_ACCOUNT = 'NO_LOYALTY_ACCOUNT',
  FEEDBACK_ALREADY_EXISTS = 'FEEDBACK_ALREADY_EXISTS',
  NO_CHANGE_NEEDED = 'NO_CHANGE_NEEDED',
}

export class AppError extends Error {
  constructor(
    public code: ErrorCode,
    message: string,
    public details?: unknown
  ) {
    super(message);
    this.name = 'AppError';
  }

  get httpStatus(): number {
    switch (this.code) {
      case ErrorCode.UNAUTHORIZED:
      case ErrorCode.INVALID_CREDENTIALS:
      case ErrorCode.INVALID_OLD_PIN:
        return 401;
      case ErrorCode.ACCOUNT_LOCKED:
        return 423;
      case ErrorCode.FORBIDDEN:
      case ErrorCode.ACCOUNT_INACTIVE:
        return 403;
      case ErrorCode.NOT_FOUND:
      case ErrorCode.CUSTOMER_NOT_FOUND:
      case ErrorCode.APPOINTMENT_NOT_FOUND:
      case ErrorCode.PROMO_NOT_FOUND:
      case ErrorCode.NO_LOYALTY_ACCOUNT:
        return 404;
      case ErrorCode.CONFLICT:
      case ErrorCode.SKU_ALREADY_EXISTS:
      case ErrorCode.BARCODE_ALREADY_EXISTS:
      case ErrorCode.USERNAME_ALREADY_EXISTS:
      case ErrorCode.MEDICAL_RECORD_ALREADY_EXISTS:
      case ErrorCode.ROOM_NOT_AVAILABLE:
      case ErrorCode.GROOMER_NOT_AVAILABLE:
      case ErrorCode.FEEDBACK_ALREADY_EXISTS:
        return 409;
      case ErrorCode.VALIDATION_ERROR:
      case ErrorCode.BAD_REQUEST:
      case ErrorCode.INVALID_STATE_TRANSITION:
      case ErrorCode.INSUFFICIENT_STOCK:
      case ErrorCode.INSUFFICIENT_LOYALTY_POINTS:
      case ErrorCode.PROMOTION_INVALID:
      case ErrorCode.CANNOT_DELETE_HAS_REFERENCES:
      case ErrorCode.APPOINTMENT_NOT_IN_PROGRESS:
      case ErrorCode.BOOKING_NOT_ACTIVE:
      case ErrorCode.INVOICE_CANCELLED:
      case ErrorCode.INVOICE_ALREADY_PAID:
      case ErrorCode.ALREADY_CANCELLED:
      case ErrorCode.ALREADY_REGISTERED:
      case ErrorCode.NO_CHANGE_NEEDED:
        return 400;
      default:
        return 500;
    }
  }

  toJSON() {
    return {
      success: false,
      error: {
        code: this.code,
        message: this.message,
        details: this.details,
      },
    };
  }
}
```

### 12.2 Global Error Handling (SvelteKit)

```svelte
<!-- src/routes/+error.svelte -->
<script lang="ts">
  import { page } from '$app/stores';
  import { Button } from '$lib/components/ui/button';

  const status = $derived($page.status);
  const message = $derived($page.error?.message || 'Terjadi kesalahan');

  const errorMessages: Record<number, string> = {
    400: 'Permintaan tidak valid',
    401: 'Anda harus login terlebih dahulu',
    403: 'Anda tidak memiliki akses ke halaman ini',
    404: 'Halaman tidak ditemukan',
    500: 'Terjadi kesalahan pada server',
  };
</script>

<div class="min-h-screen flex items-center justify-center bg-background">
  <div class="max-w-md w-full text-center space-y-6 p-8">
    <div class="space-y-2">
      <h1 class="text-6xl font-bold text-foreground">{status}</h1>
      <h2 class="text-2xl font-semibold text-foreground">
        {errorMessages[status] || 'Error'}
      </h2>
      <p class="text-muted-foreground">{message}</p>
    </div>

    <div class="flex gap-3 justify-center">
      <Button variant="outline" onclick={() => history.back()}>
        Kembali
      </Button>
      <Button onclick={() => window.location.href = '/'}>
        Ke Beranda
      </Button>
    </div>
  </div>
</div>
```

```svelte
<!-- src/routes/+layout.svelte (error boundary via snippet) -->
<script lang="ts">
  import { onError } from 'svelte/events';
  import { ToastContainer } from '$lib/components/ui/toast';

  onError((event) => {
    console.error('Unhandled error:', event.error);
    // Kirim ke error tracking service
  });
</script>

<ToastContainer />
<slot />
```

---

## 13. File Structure Contracts (SvelteKit)

```
petora/
├── static/
│   ├── favicon.ico
│   └── robots.txt
├── src/
│   ├── app.html                          # HTML template
│   ├── app.css                           # Global styles (Tailwind)
│   ├── app.d.ts                          # SvelteKit type declarations
│   ├── hooks.client.ts                   # Client-side hooks
│   ├── hooks.server.ts                   # Server-side hooks
│   │
│   ├── routes/                           # SvelteKit file-based routing
│   │   ├── +layout.svelte                # Root layout
│   │   ├── +layout.server.ts             # Root server layout (auth check)
│   │   ├── +page.svelte                  # Landing / redirect
│   │   ├── +error.svelte                 # Global error page
│   │   │
│   │   ├── login/
│   │   │   ├── +page.svelte
│   │   │   └── +page.server.ts           # Form action for login
│   │   │
│   │   ├── app/                          # Staff dashboard
│   │   │   ├── +layout.svelte            # Sidebar + header layout
│   │   │   ├── +layout.server.ts         # Auth guard (staff only)
│   │   │   ├── dashboard/
│   │   │   │   └── +page.svelte
│   │   │   ├── crm/
│   │   │   │   ├── customers/
│   │   │   │   │   ├── +page.svelte
│   │   │   │   │   └── [id]/
│   │   │   │   │       └── +page.svelte
│   │   │   │   └── pets/
│   │   │   │       └── +page.svelte
│   │   │   ├── appointments/
│   │   │   │   ├── +page.svelte
│   │   │   │   └── [id]/
│   │   │   │       └── +page.svelte
│   │   │   ├── pet-hotel/
│   │   │   │   ├── rooms/
│   │   │   │   │   └── +page.svelte
│   │   │   │   └── bookings/
│   │   │   │       └── +page.svelte
│   │   │   ├── grooming/
│   │   │   │   ├── services/
│   │   │   │   │   └── +page.svelte
│   │   │   │   └── bookings/
│   │   │   │       └── +page.svelte
│   │   │   ├── products/
│   │   │   │   ├── +page.svelte
│   │   │   │   └── inventory/
│   │   │   │       └── +page.svelte
│   │   │   ├── pos/
│   │   │   │   ├── +page.svelte
│   │   │   │   └── +page.server.ts
│   │   │   ├── reports/
│   │   │   │   ├── revenue/
│   │   │   │   │   └── +page.svelte
│   │   │   │   ├── pnl/
│   │   │   │   │   └── +page.svelte
│   │   │   │   └── inventory-valuation/
│   │   │   │       └── +page.svelte
│   │   │   ├── engagement/
│   │   │   │   ├── loyalty/
│   │   │   │   │   └── +page.svelte
│   │   │   │   ├── promotions/
│   │   │   │   │   └── +page.svelte
│   │   │   │   └── feedback/
│   │   │   │       └── +page.svelte
│   │   │   ├── keuangan/
│   │   │   │   └── expenses/
│   │   │   │       └── +page.svelte
│   │   │   └── settings/
│   │   │       └── +page.svelte
│   │   │
│   │   └── portal/                       # Customer portal
│   │       ├── +layout.svelte
│   │       ├── +layout.server.ts         # Auth guard (customer only)
│   │       ├── dashboard/
│   │       │   └── +page.svelte
│   │       ├── pets/
│   │       │   └── +page.svelte
│   │       ├── appointments/
│   │       │   └── +page.svelte
│   │       ├── pet-hotel/
│   │       │   └── +page.svelte
│   │       ├── grooming/
│   │       │   └── +page.svelte
│   │       ├── invoices/
│   │       │   └── +page.svelte
│   │       ├── loyalty/
│   │       │   └── +page.svelte
│   │       └── shop/
│   │           └── +page.svelte
│   │
│   ├── lib/                              # $lib alias
│   │   ├── components/
│   │   │   ├── ui/                       # Base components (shadcn-svelte)
│   │   │   │   ├── button.svelte
│   │   │   │   ├── input.svelte
│   │   │   │   ├── select.svelte
│   │   │   │   ├── dialog.svelte
│   │   │   │   ├── table.svelte
│   │   │   │   ├── badge.svelte
│   │   │   │   ├── card.svelte
│   │   │   │   ├── tabs.svelte
│   │   │   │   ├── toast.svelte
│   │   │   │   └── types.ts
│   │   │   │
│   │   │   ├── features/                 # Feature-specific components
│   │   │   │   ├── auth/
│   │   │   │   ├── crm/
│   │   │   │   ├── appointments/
│   │   │   │   ├── pet-hotel/
│   │   │   │   ├── grooming/
│   │   │   │   ├── pos/
│   │   │   │   ├── products/
│   │   │   │   ├── reports/
│   │   │   │   ├── engagement/
│   │   │   │   └── keuangan/
│   │   │   │
│   │   │   ├── layouts/
│   │   │   │   ├── staff-layout.svelte
│   │   │   │   ├── portal-layout.svelte
│   │   │   │   └── sidebar.svelte
│   │   │   │
│   │   │   └── shared/
│   │   │       ├── data-table.svelte
│   │   │       ├── file-upload.svelte
│   │   │       ├── date-time-picker.svelte
│   │   │       ├── search-input.svelte
│   │   │       ├── status-badge.svelte
│   │   │       ├── confirm-dialog.svelte
│   │   │       ├── empty-state.svelte
│   │   │       ├── loading-spinner.svelte
│   │   │       └── error-boundary.svelte
│   │   │
│   │   ├── hooks/                        # Svelte Query hooks
│   │   │   ├── use-auth.ts
│   │   │   ├── use-customers.ts
│   │   │   ├── use-pets.ts
│   │   │   ├── use-appointments.ts
│   │   │   ├── use-medical-records.ts
│   │   │   ├── use-pet-hotel.ts
│   │   │   ├── use-grooming.ts
│   │   │   ├── use-products.ts
│   │   │   ├── use-inventory.ts
│   │   │   ├── use-invoices.ts
│   │   │   ├── use-loyalty.ts
│   │   │   ├── use-promotions.ts
│   │   │   ├── use-expenses.ts
│   │   │   ├── use-reports.ts
│   │   │   ├── use-settings.ts
│   │   │   └── use-notifications.ts
│   │   │
│   │   ├── services/                     # Service layer
│   │   │   ├── auth.service.ts
│   │   │   ├── customer.service.ts
│   │   │   ├── pet.service.ts
│   │   │   ├── appointment.service.ts
│   │   │   ├── medical-record.service.ts
│   │   │   ├── pet-hotel.service.ts
│   │   │   ├── grooming.service.ts
│   │   │   ├── product.service.ts
│   │   │   ├── inventory.service.ts
│   │   │   ├── invoice.service.ts
│   │   │   ├── loyalty.service.ts
│   │   │   ├── promotion.service.ts
│   │   │   ├── expense.service.ts
│   │   │   ├── report.service.ts
│   │   │   ├── settings.service.ts
│   │   │   └── base.service.ts
│   │   │
│   │   ├── stores/                       # Svelte 5 reactive stores
│   │   │   ├── auth.store.ts
│   │   │   ├── cart.store.ts
│   │   │   └── ui.store.ts
│   │   │
│   │   ├── schemas/                      # Zod validation schemas
│   │   │   ├── base.ts
│   │   │   ├── user.ts
│   │   │   ├── customer.ts
│   │   │   ├── pet.ts
│   │   │   ├── appointment.ts
│   │   │   ├── medical-record.ts
│   │   │   ├── invoice.ts
│   │   │   ├── product.ts
│   │   │   ├── loyalty.ts
│   │   │   └── promotion.ts
│   │   │
│   │   ├── types/                        # TypeScript types
│   │   │   ├── base.ts
│   │   │   ├── user.ts
│   │   │   ├── customer.ts
│   │   │   ├── pet.ts
│   │   │   ├── appointment.ts
│   │   │   ├── medical-record.ts
│   │   │   ├── pet-hotel.ts
│   │   │   ├── grooming.ts
│   │   │   ├── product.ts
│   │   │   ├── purchase-order.ts
│   │   │   ├── invoice.ts
│   │   │   ├── loyalty.ts
│   │   │   ├── promotion.ts
│   │   │   ├── expense.ts
│   │   │   ├── feedback.ts
│   │   │   ├── audit.ts
│   │   │   └── notification.ts
│   │   │
│   │   ├── server/                       # Server-only code
│   │   │   ├── supabase-admin.ts         # Service role client
│   │   │   ├── auth.ts                   # Server auth helpers
│   │   │   └── secrets.ts                # Secret accessors
│   │   │
│   │   ├── utils/                        # Pure utility functions
│   │   │   ├── format.ts
│   │   │   ├── validation.ts
│   │   │   ├── date.ts
│   │   │   ├── cn.ts
│   │   │   └── constants.ts
│   │   │
│   │   └── core/                         # Core utilities
│   │       ├── supabase.ts               # Supabase client init (browser)
│   │       ├── errors.ts                 # AppError + ErrorCode
│   │       ├── audit.ts                  # Audit logging
│   │       ├── notifications.ts          # Notification helpers
│   │       ├── realtime.ts               # Realtime subscriptions
│   │       ├── storage.ts                # File upload
│   │       ├── query-client.ts           # Svelte Query client
│   │       ├── query-keys.ts             # Query key constants
│   │       └── auth.ts                   # Client auth helpers
│   │
│   └── params/                           # SvelteKit param matchers
│       └── uuid.ts
│
├── supabase/
│   ├── migrations/                       # Database migrations
│   │   ├── 001_initial_schema.sql
│   │   ├── 002_rls_policies.sql
│   │   ├── 003_functions.sql
│   │   └── 004_seed_data.sql
│   ├── functions/                        # Edge Functions
│   │   ├── auth-login/
│   │   │   └── index.ts
│   │   ├── send-whatsapp/
│   │   │   └── index.ts
│   │   ├── payment-callback/
│   │   │   └── index.ts
│   │   └── scheduled-jobs/
│   │       └── index.ts
│   ├── seed.sql
│   └── config.toml
│
├── tests/
│   ├── unit/
│   │   ├── services/
│   │   ├── schemas/
│   │   └── utils/
│   ├── integration/
│   │   ├── workflows/
│   │   └── rls/
│   └── e2e/
│       ├── auth.spec.ts
│       ├── pos-checkout.spec.ts
│       ├── appointment-flow.spec.ts
│       └── pet-hotel-flow.spec.ts
│
├── .env.example
├── .gitignore
├── package.json
├── svelte.config.js
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.ts
├── postcss.config.js
├── vitest.config.ts
├── playwright.config.ts
├── vercel.json
├── AGENTS.md
├── PRD.md
└── README.md
```

---

## 14. Environment Variables

### 14.1 Frontend (Vercel)

```bash
# .env.example (non-secret names and safe defaults only)
PUBLIC_SUPABASE_URL=https://your-project.supabase.co
PUBLIC_SUPABASE_ANON_KEY=your-anon-key
PUBLIC_ENABLE_WHATSAPP_NOTIFICATIONS=false
PUBLIC_ENABLE_PAYMENT_GATEWAY=false
```

### 14.2 Backend (SvelteKit Server + Supabase Edge Functions)

```bash
# SvelteKit server-side (via Vercel Environment Variables)
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
PAYMENT_PROVIDER=approved-provider-name
PAYMENT_SERVER_SECRET=configure-in-secret-manager
PAYMENT_WEBHOOK_SECRET=configure-in-secret-manager
MESSAGING_PROVIDER=approved-provider-name
MESSAGING_API_SECRET=configure-in-secret-manager
EMAIL_PROVIDER=approved-provider-name
EMAIL_API_SECRET=configure-in-secret-manager
```

### 14.3 Rules

- ❌ DILARANG hardcode credentials di source code
- ❌ DILARANG commit `.env` ke git
- ✅ WAJIB gunakan `.env.example` sebagai template
- ✅ WAJIB set variables di Vercel Dashboard dan Supabase Dashboard
- ✅ Variabel dengan prefix `PUBLIC_` dapat diakses di client (SvelteKit convention)
- ✅ Variabel tanpa prefix `PUBLIC_` hanya dapat diakses di server (`+page.server.ts`, `hooks.server.ts`)

---

## 15. API Response Envelope

### 15.1 Success Response

```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 100,
    "totalPages": 5
  }
}
```

### 15.2 Error Response

```json
{
  "success": false,
  "error": {
    "code": "INSUFFICIENT_STOCK",
    "message": "Stok produk 'Royal Canin 5kg' tidak mencukupi",
    "details": {
      "product_id": "uuid",
      "requested": 5,
      "available": 2
    }
  }
}
```

---

## 16. Naming Conventions

| Aspek | Konvensi | Contoh |
| --- | --- | --- |
| File (component) | kebab-case | `customer-list.svelte` |
| File (service) | kebab-case | `customer.service.ts` |
| File (hook) | kebab-case | `use-customers.ts` |
| File (type) | kebab-case | `customer.ts` |
| File (schema) | kebab-case | `customer.ts` |
| File (route) | kebab-case | `+page.svelte` |
| Component | PascalCase | `CustomerList` |
| Hook | camelCase + `use` prefix | `useCustomers` |
| Service | PascalCase + `Service` suffix | `CustomerService` |
| Function | camelCase | `createCustomer` |
| Constant | UPPER_SNAKE_CASE | `LOW_STOCK_THRESHOLD` |
| Enum | PascalCase (type), UPPER_SNAKE_CASE (value) | `InvoiceStatus.PAID` |
| Type/Interface | PascalCase | `CreateCustomerInput` |
| DB Table | snake_case, plural | `customers` |
| DB Column | snake_case | `created_at` |
| DB Function | `fn_` prefix | `fn_create_invoice` |
| CSS Class | kebab-case (Tailwind) | `bg-blue-500` |
| SvelteKit Route | kebab-case folder | `pet-hotel/` |
| SvelteKit Param | `[param]` atau `[param=matcher]` | `[id]`, `[id=uuid]` |
| Env Variable | UPPER_SNAKE_CASE | `PUBLIC_SUPABASE_URL` |

---

## 17. Migration Strategy

### 17.1 Rules

- ✅ Migration harus idempotent (bisa dijalankan berkali-kali)
- ✅ Migration harus reversible (ada UP dan DOWN)
- ✅ Migration harus di-test di local environment
- ❌ DILARANG migration yang menghapus data tanpa backup
- ❌ DILARANG mengubah tipe data tanpa migration script

### 17.2 Execution Order

```bash
# Local development
npx supabase db reset

# Staging / Production
npx supabase db push
```

### 17.3 Seed Data

```sql
-- supabase/seed.sql
-- Do not seed a static owner credential or PIN.
-- Create the first owner through a one-time, audited bootstrap command in a
-- protected environment. The command must validate input, hash the PIN, and
-- revoke itself after successful bootstrap.

-- Loyalty tiers
INSERT INTO loyalty_tiers (tier_name, min_points, min_spending, point_multiplier, benefits)
VALUES
  ('BRONZE', 0, 0, 1.0, '{"discount": 0}'),
  ('SILVER', 100, 1000000, 1.5, '{"discount": 5}'),
  ('GOLD', 500, 5000000, 2.0, '{"discount": 10}'),
  ('PLATINUM', 1000, 10000000, 3.0, '{"discount": 15}');

-- Default settings
INSERT INTO settings (key, value) VALUES
  ('clinic.name', '"Petora"'),
  ('clinic.operating_hours', '{"open": "08:00", "close": "20:00"}'),
  ('clinic.timezone', '"Asia/Jakarta"'),
  ('security.pin_length', '6'),
  ('security.max_login_attempts', '5'),
  ('security.lockout_duration_minutes', '15'),
  ('loyalty.point_value', '100'),
  ('loyalty.min_transaction_for_points', '10000'),
  ('tax.default_rate', '0'),
  ('tax.enabled', 'false');
```

---

## 18. Supabase Edge Functions

### 18.1 Auth Login

> **Status:** Blocked until `DEC-OPEN-001` is approved. This code block is a contract illustration, not deployable authentication code. Production implementation MUST use one canonical identity/session model, Zod input validation, rate limiting, token revocation, audit transaction, and the approved `auth.uid()` mapping. The custom PIN flow MUST NOT pass a PIN as a Supabase password unless that design is explicitly approved and tested.

```ts
// supabase/functions/auth-login/index.ts
import { serve } from 'https://deno.land/std@0.168.0/http/server.ts';
import { createClient } from 'https://esm.sh/@supabase/supabase-js@2';
import { compare } from 'https://deno.land/x/bcrypt/mod.ts';
import { z } from 'https://esm.sh/zod@3';

const loginCredentialsSchema = z.object({
  username: z.string().min(3).max(50).regex(/^[a-z0-9._]+$/),
  pin: z.string().length(6).regex(/^\d+$/),
});

serve(async (req) => {
  const rawBody: unknown = await req.json();
  const parsed = loginCredentialsSchema.safeParse(rawBody);

  if (!parsed.success) {
    return new Response(JSON.stringify({
      success: false,
      error: { code: 'BAD_REQUEST', message: 'Invalid input' }
    }), { status: 400 });
  }

  const { username, pin } = parsed.data;

  const supabase = createClient(
    Deno.env.get('SUPABASE_URL')!,
    Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!
  );

  // Find user
  const { data: user } = await supabase
    .from('users')
    .select('*')
    .eq('username', username)
    .single();

  if (!user) {
    return new Response(JSON.stringify({
      success: false,
      error: { code: 'INVALID_CREDENTIALS', message: 'Invalid credentials' }
    }), { status: 401 });
  }

  // Check lockout
  if (user.locked_until && new Date(user.locked_until) > new Date()) {
    return new Response(JSON.stringify({
      success: false,
      error: { code: 'ACCOUNT_LOCKED', message: 'Account locked' }
    }), { status: 423 });
  }

  // Check active
  if (!user.is_active) {
    return new Response(JSON.stringify({
      success: false,
      error: { code: 'ACCOUNT_INACTIVE', message: 'Account inactive' }
    }), { status: 403 });
  }

  // Verify PIN
  const valid = await compare(pin, user.pin_hash);
  if (!valid) {
    const attempts = user.failed_login_attempts + 1;
    const updates: Record<string, unknown> = { failed_login_attempts: attempts };
    if (attempts >= 5) {
      updates.locked_until = new Date(Date.now() + 15 * 60 * 1000).toISOString();
    }
    await supabase.from('users').update(updates).eq('id', user.id);
    return new Response(JSON.stringify({
      success: false,
      error: { code: 'INVALID_CREDENTIALS', message: 'Invalid credentials' }
    }), { status: 401 });
  }

  // Success
  await supabase.from('users').update({
    failed_login_attempts: 0,
    locked_until: null,
    last_login_at: new Date().toISOString(),
  }).eq('id', user.id);

  // Audit log
  await supabase.from('audit_logs').insert({
    user_id: user.id,
    action: 'LOGIN',
    entity_type: 'users',
    entity_id: user.id,
  });

  // Generate JWT (implementation depends on DEC-OPEN-001)
  // ...

  const { pin_hash, ...safeUser } = user;
  return new Response(JSON.stringify({
    success: true,
    data: { user: safeUser, session_token: '...' }
  }), { status: 200 });
});
```

### 18.2 Function List

| Function | Trigger | Purpose |
| --- | --- | --- |
| `auth-login` | HTTP | Login dengan PIN |
| `send-whatsapp` | HTTP / Internal | Kirim notifikasi melalui provider adapter yang disetujui |
| `send-email` | HTTP / Internal | Kirim email melalui provider adapter yang disetujui |
| `payment-callback` | HTTP | Handle callback dari provider payment yang disetujui |
| `scheduled-jobs` | Cron | Reminder vaksin, appointment, feedback request |

---

## 19. Realtime Subscriptions

### 19.1 Svelte Integration

```ts
// src/lib/core/realtime.ts
import { supabase } from './supabase';
import type { RealtimeChannel } from '@supabase/supabase-js';

/**
 * Helper untuk subscribe ke Supabase Realtime.
 * Otomatis cleanup saat component unmount.
 */
export function createRealtimeSubscription(
  channelName: string,
  config: {
    event: 'INSERT' | 'UPDATE' | 'DELETE' | '*';
    schema?: string;
    table: string;
    filter?: string;
  },
  callback: (payload: unknown) => void
) {
  let channel: RealtimeChannel | null = null;
  let isConnected = $state(false);

  channel = supabase
    .channel(channelName)
    .on('postgres_changes', config, callback)
    .subscribe((status) => {
      isConnected = status === 'SUBSCRIBED';
    });

  // Return cleanup function dan state
  return {
    get isConnected() { return isConnected; },
    cleanup: () => {
      if (channel) {
        supabase.removeChannel(channel);
      }
    },
  };
}
```

### 19.2 Usage di Component

```svelte
<!-- src/lib/components/features/appointments/appointments-realtime.svelte -->
<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { createRealtimeSubscription } from '$lib/core/realtime';
  import { queryClient } from '$lib/query-client';
  import { QueryKeys } from '$lib/query-keys';

  let today = $state(new Date().toISOString().split('T')[0]);
  let subscription: ReturnType<typeof createRealtimeSubscription> | null = null;

  onMount(() => {
    subscription = createRealtimeSubscription(
      'appointments-today',
      {
        event: '*',
        table: 'appointments',
        filter: `appointment_date=eq.${today}`,
      },
      () => {
        queryClient.invalidateQueries({ queryKey: QueryKeys.appointments({ date: today }) });
      }
    );
  });

  onDestroy(() => {
    subscription?.cleanup();
  });
</script>

{#if subscription?.isConnected}
  <span class="text-xs text-green-600">● Live</span>
{/if}
```

---

## 20. Storage & File Upload

### 20.1 Configuration

```ts
// src/lib/core/storage.ts
import { supabase } from './supabase';
import { AppError, ErrorCode } from './errors';

export type StorageBucket =
  | 'medical-records'
  | 'pet-hotel'
  | 'grooming'
  | 'products'
  | 'customers'
  | 'pets'
  | 'expenses';

const MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf'];
const PUBLIC_BUCKETS: ReadonlySet<StorageBucket> = new Set(['products', 'pets']);

export async function uploadFile(
  bucket: StorageBucket,
  file: File,
  path: string
): Promise<string> {
  // Validate
  if (file.size > MAX_FILE_SIZE) {
    throw new AppError(ErrorCode.BAD_REQUEST, 'File terlalu besar (max 5MB)');
  }
  if (!ALLOWED_TYPES.includes(file.type)) {
    throw new AppError(ErrorCode.BAD_REQUEST, 'Tipe file tidak diizinkan');
  }

  const fileExt = file.name.split('.').pop();
  const fileName = `${path}/${crypto.randomUUID()}.${fileExt}`;

  const { data, error } = await supabase.storage
    .from(bucket)
    .upload(fileName, file, {
      cacheControl: '3600',
      upsert: false,
    });

  if (error) {
    throw new AppError(ErrorCode.INTERNAL_ERROR, `Upload failed: ${error.message}`);
  }

  if (PUBLIC_BUCKETS.has(bucket)) {
    const { data: { publicUrl } } = supabase.storage
      .from(bucket)
      .getPublicUrl(data.path);
    return publicUrl;
  }

  const { data: signed, error: signedUrlError } = await supabase.storage
    .from(bucket)
    .createSignedUrl(data.path, 3600);

  if (signedUrlError || !signed?.signedUrl) {
    throw new AppError(ErrorCode.INTERNAL_ERROR, 'Signed URL generation failed');
  }

  return signed.signedUrl;
}

export async function deleteFile(bucket: StorageBucket, path: string): Promise<void> {
  const { error } = await supabase.storage.from(bucket).remove([path]);
  if (error) {
    throw new AppError(ErrorCode.INTERNAL_ERROR, `Delete failed: ${error.message}`);
  }
}
```

### 20.2 Bucket Configuration

| Bucket | Public | Max Size | Allowed Types |
| --- | --- | --- | --- |
| `medical-records` | No | 5MB | jpg, png, pdf |
| `pet-hotel` | No | 5MB | jpg, png |
| `grooming` | No | 5MB | jpg, png |
| `products` | Yes | 2MB | jpg, png, webp |
| `customers` | No | 2MB | jpg, png |
| `pets` | Yes | 2MB | jpg, png |
| `expenses` | No | 5MB | jpg, png, pdf |

---

## 21. Operational Deployment Contract

### 21.1 Environment Model

| Environment | Data | External providers | Deployment rule |
| --- | --- | --- | --- |
| Local | Disposable local Supabase | Mock/sandbox only | Migration and test development |
| CI | Ephemeral isolated database | Mock/sandbox | Required checks on every PR |
| Staging | Sanitized non-production data | Sandbox | Production-like rehearsal |
| Production | Real customer and financial data | Approved live providers | Manual approval and release record |

Environment values are never copied between environments. Secrets live only in the environment secret manager. `.env.example` documents names and safe defaults, never real values.

### 21.2 Promotion Sequence

1. Build an immutable artifact from a reviewed commit.
2. Run typecheck, lint, unit, integration, E2E, migration, RLS, dependency, secret, and security checks.
3. Apply database migrations to staging and run smoke plus upgrade tests.
4. Verify provider sandbox callbacks, scheduled jobs, storage policies, audit events, and dashboards.
5. Create a release record containing commit, migration list, schema checksum, evidence, owner, rollback version, and open risk waivers.
6. Deploy frontend and backend in compatibility order: additive schema, backend, frontend, cleanup migration only after adoption.
7. Run production smoke tests with non-destructive accounts and verify health, error rate, queue, audit, and payment reconciliation.
8. Announce release and monitor the defined observation window before closing the release.

### 21.3 Health and Observability Contract

Every deployable component MUST expose or emit:

- liveness and readiness checks;
- version, commit, environment, and migration version;
- structured logs with correlation ID and actor ID where allowed;
- metrics for request rate, latency, error rate, queue lag, database errors, RLS denial, payment callback, stock conflict, and notification failure;
- traces for authentication, booking, checkout, payment, webhook, migration, and scheduled jobs;
- alerts with severity, owner, runbook link, and escalation path.

Logs MUST NOT contain PIN, token, secret, full payment credential, or unnecessary medical data.

### 21.4 Rollback and Recovery

- Application rollback MUST be tested independently from database rollback.
- Destructive database changes require expand-migrate-contract sequencing; direct destructive rollback is forbidden without verified backup and recovery procedure.
- Payment, stock, loyalty, invoice, and booking operations use compensating transactions rather than deleting history.
- A failed migration stops promotion and preserves evidence; it is never hidden by manual edits in production.
- Rollback decision is owned by the incident commander and technical owner; financial reconciliation is owned by finance/Owner.

### 21.5 Backup and Disaster Recovery

- Automated database backup, point-in-time recovery, storage backup, and audit retention MUST be configured.
- Restore drills MUST run on a defined cadence and record actual RTO/RPO.
- Recovery MUST verify schema version, RLS, storage access, scheduled jobs, sequence counters, audit continuity, and payment reconciliation.
- Business continuity MUST define read-only mode, service outage message, manual transaction procedure, and reconciliation after recovery.

### 21.6 Release Acceptance

A production deployment is accepted only when:

- [Production Readiness Checklist](05-production-readiness-checklist.md) is complete,
- [Requirement Traceability Matrix](07-requirement-traceability.md) contains evidence for the target release,
- the release record is approved by Product Owner and Technical Owner.

---

## Ringkasan Stack Teknologi

| Aspek | Teknologi |
| --- | --- |
| **Framework** | SvelteKit 2 (meta-framework) |
| **UI Runtime** | Svelte 5 (runes mode: `$state`, `$derived`, `$effect`, `$props`) |
| **Rendering** | Compile-time, fine-grained reactivity |
| **State** | Svelte 5 runes (`$state`, `$derived`) — built-in, proxy-based |
| **Data Fetching** | `@tanstack/svelte-query` |
| **Routing** | SvelteKit file-based (`+page.svelte`, `+layout.svelte`) |
| **UI Library** | `shadcn-svelte` / `bits-ui` |
| **Forms** | `formsnap` / `@tanstack/svelte-form` + `superforms` |
| **Icons** | `lucide-svelte` |
| **Validation** | `zod` + `zod-to-json-schema` |
| **Database** | Supabase PostgreSQL + RLS |
| **Auth** | Supabase Auth (custom PIN flow via Edge Function) |
| **Storage** | Supabase Storage |
| **Realtime** | Supabase Realtime (WebSocket) |
| **Styling** | Tailwind CSS 3 |
| **Deployment** | Vercel via `@sveltejs/adapter-vercel` |
| **Testing** | Vitest (unit) + Playwright (E2E) |
| **Package Manager** | pnpm |

---

> **Dokumen ini merupakan technical baseline Petora berbasis SvelteKit.** Seluruh developer dan AI agent wajib mengikuti kontrak governance, decision register, dan release gate untuk memastikan konsistensi, keamanan, performa, dan maintainability sistem.
