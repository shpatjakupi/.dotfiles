# Frontend Architecture (next-app-template)

## Folder Structure

```
next-app-template/
├── app/                    # Next.js App Router
│   ├── api/                # API proxy routes (server-side, call backend)
│   │   ├── account/        # Account endpoints
│   │   ├── admin/          # Admin CRUD endpoints
│   │   ├── customer/       # Customer order endpoints
│   │   ├── menu/           # Menu endpoints
│   │   ├── payment/        # Payment endpoints
│   │   └── wolt/           # Wolt endpoints
│   ├── config.ts           # API base URL config
│   └── utils/              # Server utilities
├── components/             # React components (by domain)
│   ├── Account/            # Login, profile, registration
│   ├── AdminDashboard/     # Admin order management, settings
│   ├── Cart/               # Shopping cart
│   ├── Checkout/           # Checkout flow
│   ├── Payment/            # Payment UI
│   ├── Menu/               # Menu display
│   └── ...
├── services/               # API client functions (call /app/api/ routes)
│   ├── account/            # Account API calls
│   ├── admin/              # Admin API calls (auth, create, getter, setter, toggle, update)
│   ├── customer/           # Customer API calls (delivery, orders, promos, recommendations)
│   ├── menu/               # Menu API calls (food, beverages, fillings)
│   └── websocket/          # WebSocket connection (SockJS/STOMP)
├── pages/                  # Pages Router (legacy, co-exists with App Router)
│   └── payment/callback.tsx  # Bambora payment callback handler (must stay in Pages Router)
├── context/                # React Context providers (global state)
├── hooks/                  # Custom React hooks
├── interfaces/             # TypeScript type definitions
├── data/                   # Static data
└── utils/                  # Client utilities
```

## API Call Flow

```
Component
  --> services/admin/getter.ts (client function)
    --> /app/api/admin/get-foods (Next.js API route, server-side)
      --> http://ordrupspizza-backend:8080/api/admin/foods (Spring Boot)
```

**Why the proxy layer?** The backend runs as a ClusterIP (internal only). The frontend API routes run server-side inside the cluster and can reach the backend. The browser never talks to the backend directly.

## Key Patterns

**Adding a new frontend feature:**
1. Define TypeScript interface in `interfaces/`
2. Create service function in `services/<domain>/` (calls the API route)
3. Create API route in `app/api/<domain>/` (proxies to backend)
4. Create component in `components/` (uses the service)

**State management**: React Context (`context/`), no Redux/Zustand.

**UI framework**: Mantine UI 7.x. Use Mantine components, hooks, and form library. Custom CSS via CSS Modules.

**WebSocket**: SockJS + STOMP for real-time order updates in admin dashboard. Connection setup in `services/websocket/`.

## Non-Obvious

- **`pages/payment/callback.tsx`** must stay in Pages Router (not App Router). Bambora redirects here with query params. Moving to App Router breaks the payment flow.
- **No API proxy/rewrite in `next.config.mjs`** — all proxying is done manually in API route handlers.
- **`NEXT_PUBLIC_BACKEND_API`** is set to the backend's ClusterIP URL (`http://ordrupspizza-backend:8080`). Only works server-side. Client components cannot use it.
- **Standalone output mode** — `next.config.mjs` has `output: "standalone"` for Docker deployment.
- **CSP headers** configured in `next.config.mjs` — must include Wolt domains and `wss://` for WebSocket.
