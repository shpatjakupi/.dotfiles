# Backend Architecture (order-backend)

## Package Structure

```
src/main/java/com/foodapp/foodapp/
├── rest/           # REST controllers (entry points)
│   ├── AdminRestController      # Admin CRUD operations
│   ├── CustomerRestController   # Customer-facing endpoints
│   ├── MenuRestController       # Menu item management
│   ├── AdminAuthController      # Admin login/JWT
│   ├── BamboraRestController    # Payment callbacks
│   ├── WoltWebhookController    # Wolt delivery webhooks
│   ├── ImageController          # Image upload/serve
│   └── SessionController        # Session management
├── service/        # Business logic (grouped by domain)
│   ├── account/    # Account management
│   ├── admin/      # Admin operations
│   ├── customer/   # Customer order flow
│   ├── menu/       # Menu CRUD
│   └── shopconfig/ # Shop settings
├── dao/            # Data access (JPA queries, grouped like service/)
├── entity/         # JPA entities (see below)
├── dto/            # Request/response objects
│   ├── emails/     # Email template DTOs
│   ├── wolt/       # Wolt API DTOs
│   └── ws/         # WebSocket message DTOs
├── repository/     # Spring Data JPA repositories
├── security/       # JWT filter, auth config
├── config/         # Beans (StrictHttpFirewall, WebSocket, CORS)
├── client/         # External API clients (Bambora, Wolt)
├── mail/           # Email service (Simply.com SMTP)
├── websocket/      # STOMP/SockJS config + handlers
├── exception/      # Custom exceptions + global handler
└── utils/          # Helpers
```

## Entities (24 total)

| Entity | Purpose |
|--------|---------|
| **Food** | Menu food items (name, price, description, image, recommendedFoodIds/BeverageIds) |
| **Beverage** | Drinks menu |
| **Filling** | Fillings/toppings that can be added to food items |
| **Combo** | Combo meal deals |
| **Sides** | Side dishes |
| **Sauce** | Sauces |
| **Product** | Generic product (unused/legacy?) |
| **Item** | Line item in an order (food + quantity + selected fillings) |
| **Order** | Customer order (items, total, payment status, delivery info) |
| **OrderCompletion** | Tracks order completion status |
| **DeliveryOrder** | Delivery-specific order details |
| **Account** | Customer accounts |
| **AccountPromoCode** | Many-to-many: which promo codes a customer has used |
| **Admin** | Admin users (restaurant owners) |
| **Employee** | Restaurant employees |
| **WorkHours** | Employee work schedules |
| **PromoCode** | Discount codes |
| **ShopConfig** | Per-restaurant settings (open hours, delivery radius, etc.) |
| **BlockedIP** | Auto-blocked suspicious IPs |
| **ErrorLog** | Application error tracking |
| **EmailLog** | Sent email tracking |
| **ExternalClientLog** | External API call logging |
| **PasswordResetToken** | Password reset flow |
| **WoltDeliveryEvent** | Wolt webhook event log |

## Key Patterns

**Adding a new endpoint:**
1. Create/update Entity in `entity/`
2. Create DTO in `dto/` (request + response)
3. Add repository method in `repository/` (or use existing)
4. Add DAO method in `dao/` (business queries)
5. Add service method in `service/` (business logic)
6. Add controller endpoint in `rest/` (REST mapping)

**Auth model:**
- **Admin**: JWT-based. Login via `AdminAuthController`, token stored in frontend.
- **Customer**: In-memory Spring Security user. Not a DB entity. Single shared username/password per restaurant, passed as env vars.
- **Security filter chain**: JWT filter runs before Spring Security. Endpoints under `/api/admin/**` require admin JWT. Customer endpoints require basic auth.

## Payment Flow (Bambora)

1. Backend creates payment session → returns checkout URL
2. Customer pays on Bambora hosted page
3. Bambora calls `https://ordrupspizza.dk/payment/callback` with transaction ID
4. Frontend `pages/payment/callback.tsx` receives it server-side
5. Frontend proxies to backend `BamboraRestController.handlePaymentCallback`
6. Backend marks order `isPaymentSuccessful = true`

**If callback fails**: orders exist but show as unpaid in admin dashboard. Check `Is_payment_successful` column.

## Wolt Integration

- Backend receives delivery webhooks from Wolt
- `WoltWebhookController` handles status updates
- Events logged in `WoltDeliveryEvent` entity
