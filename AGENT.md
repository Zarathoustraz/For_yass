# AGENTS.md — Maison Éclat · Jules Coding Agent Instructions

## Role
Senior Full-Stack Engineer. You write complete, production-ready code. No stubs. No TODOs. No placeholder logic. Every file you touch must compile and run without errors.

## Tech Stack
- **Frontend**: React 19 + Vite 6 + TypeScript 5.8 + TailwindCSS 4 + Shadcn UI (Radix UI based)
- **Backend**: Node.js + Express 4 + TypeScript (tsx runtime)
- **Database**: PostgreSQL + Prisma 6
- **Auth**: JWT (jsonwebtoken) + HttpOnly cookies + RBAC
- **Payments**: Stripe (apiVersion: `2025-01-27.acacia`)
- **Images**: Cloudinary v2
- **Email**: Resend
- **Caching**: ioredis (optional, graceful fallback)
- **Env Validation**: Zod v4

## Absolute Rules

### 1. NEVER assume a file exists — always read it first
Before modifying any file, read its current content. If a file is listed as missing, create it completely from scratch. Do not reference non-existent imports.

### 2. NEVER write stubs or incomplete implementations
Every function must have a real body. Every API route must have real logic. Every React component must render real UI. If you write `// TODO` or `throw new Error("Not implemented")`, that is a failure.

### 3. NEVER use `as any` to silence TypeScript errors
Fix the types properly. The only exception is Express `req` augmentation with `(req as any).user` which is acceptable.

### 4. Shadcn UI components use Radix UI API — NOT Base UI
- `SheetTrigger` uses `asChild` prop, NOT `render` prop
- `DialogTrigger` uses `asChild`, NOT `render`
- All Shadcn trigger components follow the Radix `asChild` pattern

### 5. Shadcn UI components are SOURCE FILES, not npm packages
Components live in `src/components/ui/`. They must be created as `.tsx` files. They are NOT imported from `shadcn` or `@shadcn/ui`. Import them as `../components/ui/button`, etc.

### 6. Express middleware order is sacred
- `express.raw()` for Stripe webhooks MUST be registered BEFORE `express.json()`
- Auth middleware MUST be applied per-route or via router, not globally (to allow public routes)
- Error handler MUST be the last `app.use()` call

### 7. React Router v7 nested routes use relative paths
Inside a `<Routes>` block nested under a wildcard parent (`/admin/*`), child `<Route path>` values must NOT start with `/`. Use `path=""` for the index, `path="products"` not `path="/products"`.

### 8. Cart auto-creation is mandatory
`POST /api/cart/items` must auto-create the cart if it does not exist. Never throw "Cart not found" on an add-item request.

### 9. Prisma model fields come from the schema — always verify
Before writing any Prisma query, re-read `prisma/schema.prisma`. Never invent column names.

### 10. Environment variables
- All env vars are validated in `src/lib/env.ts` using Zod v4
- Optional third-party services (Stripe, Cloudinary, Redis, Resend) must use graceful null-check pattern: initialize to `null`, check before use
- App crashes if `DATABASE_URL` or `JWT_SECRET` are missing

## File Structure
```
/
├── server.ts              # Express + Vite dev server (root entry point)
├── package.json
├── vite.config.ts
├── tsconfig.json
├── prisma/
│   ├── schema.prisma
│   └── seed.ts
└── src/
    ├── main.tsx
    ├── App.tsx
    ├── index.css
    ├── middleware.ts
    ├── hooks/
    │   ├── useAuth.tsx
    │   └── useCart.tsx
    ├── services/
    │   └── api.ts
    ├── lib/
    │   ├── prisma.ts
    │   ├── env.ts
    │   ├── errors.ts
    │   ├── logger.ts
    │   ├── redis.ts
    │   ├── stripe.ts
    │   ├── resend.ts
    │   ├── cloudinary.ts
    │   └── auth/
    │       ├── jwt.ts
    │       └── password.ts
    ├── routes/
    │   └── health.ts
    ├── components/
    │   ├── ui/               ← ALL Shadcn UI source files live here
    │   └── products/
    │       └── ProductCard.tsx
    └── pages/
        ├── Home.tsx
        ├── Products.tsx
        ├── ProductDetail.tsx
        ├── Login.tsx
        ├── Register.tsx
        ├── Cart.tsx
        ├── Orders.tsx
        └── AdminDashboard.tsx
```

## Crash Catalogue (known bugs — do not reproduce)

| # | File | Bug | Fix |
|---|------|-----|-----|
| C1 | `src/components/ui/*` | Entire directory missing — all imports crash | Create all required Shadcn UI components as source files |
| C2 | `src/App.tsx` | `<SheetTrigger render={...}>` — wrong API | Replace with `<SheetTrigger asChild>` |
| C3 | `server.ts` | `express.json()` global intercepts Stripe webhook raw body → signature verification always fails | Register `/api/webhooks/stripe` with `express.raw()` BEFORE the global `express.json()` |
| C4 | `server.ts` | `POST /api/cart/items` throws "Cart not found" for new users | Auto-create cart with `upsert` or `findOrCreate` logic |
| C5 | `src/pages/AdminDashboard.tsx` | Nested `<Route path="/products">` with leading slash — never matches in RR v7 | Change to `<Route path="products">` |
| C6 | `src/hooks/useCart.tsx` | No `removeItem` / `updateQuantity` methods exported | Add both methods |
| C7 | `src/pages/Cart.tsx` | Delete and ±quantity buttons have no onClick handlers | Wire to `removeItem` / `updateQuantity` |
| C8 | `server.ts` | No `DELETE /api/cart/items/:id` or `PATCH /api/cart/items/:id` routes | Add both routes |
| C9 | `src/pages/AdminDashboard.tsx` | Admin sub-pages (categories, orders, users) have sidebar links but no Route + component | Create AdminCategories, AdminOrders, AdminUsers + register routes |
| C10 | `server.ts` | No review API endpoints despite Prisma Review model | Add `GET/POST /api/products/:id/reviews` |

## Quality Bar
- Zero TypeScript compile errors (`npm run lint` must pass)
- Zero runtime crashes on first page load without `.env` configured (graceful degradation)
- All pages render without white-screening
- Cart add/remove/update fully functional
- Admin dashboard navigates to all sub-sections
