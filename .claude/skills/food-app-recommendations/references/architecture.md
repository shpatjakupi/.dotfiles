# Food App Recommendations — Architecture Reference

## Table of Contents
1. [Overview](#overview)
2. [Data Model](#data-model)
3. [Backend Files](#backend-files)
4. [Frontend Files](#frontend-files)
5. [API Contracts](#api-contracts)
6. [Key Implementation Patterns](#key-implementation-patterns)
7. [Database Notes](#database-notes)

---

## Overview

**System**: Direct Item Recommendation System
**Backend**: `C:\Users\shpat\Desktop\projects\order-backend` (Spring Boot / Hibernate / HQL)
**Frontend**: `C:\Users\shpat\Desktop\projects\next-app-template` (Next.js / Mantine v7 / TypeScript)

**Direction**: Each `Food` item stores the IDs of the items it recommends. When a customer
opens checkout, the modal fetches recommendations for each food in the cart and shows them —
or skips entirely if none exist or all recommended items are already in the cart.

---

## Data Model

### Food entity — new fields
```
recommended_food_ids     VARCHAR(255)   -- e.g. "3,7,12"  (comma-separated Food IDs)
recommended_beverage_ids VARCHAR(255)   -- e.g. "1,4"     (comma-separated Beverage ID)
```
Stored as comma-separated ID strings (consistent with other string fields like `foodKey`).

### Removed fields (no longer exist)
- `recommended_option_key` was removed from Food, Beverage, and Filling entities.

---

## Backend Files

### Entities
| File | Key changes |
|---|---|
| `src/main/java/com/foodapp/foodapp/entity/Food.java` | Added `recommendedFoodIds`, `recommendedBeverageIds` fields + getters/setters; removed `recommendedOptionKey` |
| `src/main/java/com/foodapp/foodapp/entity/Beverage.java` | Removed `recommendedOptionKey` field |
| `src/main/java/com/foodapp/foodapp/entity/Filling.java` | Removed `recommendedOptionKey` field |

### DTOs
| File | Key changes |
|---|---|
| `src/main/java/com/foodapp/foodapp/dto/FoodDTO.java` | Added `recommendedFoodIds`, `recommendedBeverageIds`; constructor maps from entity |
| `src/main/java/com/foodapp/foodapp/dto/BeverageDTO.java` | Removed `recommendedOptionKey` |
| `src/main/java/com/foodapp/foodapp/dto/FillingDTO.java` | Removed `recommendedOptionKey` |
| `src/main/java/com/foodapp/foodapp/dto/RecommendationRequestDTO.java` | Contains `List<Integer> cartFoodIds` (replaces old `List<String> cartItems`) |

### DAO layer
| File | Key changes |
|---|---|
| `src/main/java/com/foodapp/foodapp/dao/menu/MenuDAO.java` | Interface: `getFoodsByIds(List<Integer>)`, `getBeveragesByIds(List<Integer>)` |
| `src/main/java/com/foodapp/foodapp/dao/menu/MenuDAOImpl.java` | HQL: `from Food where id in (:ids) and isDeactivated = false` (same pattern for Beverage) |
| `src/main/java/com/foodapp/foodapp/dao/admin/AdminDAO.java` | Interface: `getAllFoodsSimple()`, `getAllBeveragesSimple()` — return `List<Object[]>` |
| `src/main/java/com/foodapp/foodapp/dao/admin/AdminDAOImpl.java` | HQL: `select f.id, f.name, f.Key from Food f where f.isDeactivated = false order by f.Key asc, f.name asc` |

### Service layer
| File | Key changes |
|---|---|
| `src/main/java/com/foodapp/foodapp/service/customer/CustomerService.java` | Interface: `getRecommendations(List<Integer> cartFoodIds)` |
| `src/main/java/com/foodapp/foodapp/service/customer/CustomerServiceImpl.java` | For each cartFoodId: fetch Food, parse comma-separated IDs, collect unique IDs, batch-fetch via DAO, return `RecommendationResponseDTO` (fillings always empty) |
| `src/main/java/com/foodapp/foodapp/service/admin/AdminService.java` | Interface: `getAllFoodsSimple()`, `getAllBeveragesSimple()` |
| `src/main/java/com/foodapp/foodapp/service/admin/AdminServiceImpl.java` | `updateFood()` sets `food.setRecommendedFoodIds(...)` and `food.setRecommendedBeverageIds(...)` from DTO; simple list methods delegate to adminDAO |

### Controller
| File | Key changes |
|---|---|
| `src/main/java/com/foodapp/foodapp/rest/AdminRestController.java` | GET `/admin/getAllFoodsSimple` — returns `List<Map>` with keys `id`, `name`, `foodKey`; GET `/admin/getAllBeveragesSimple` — returns `id`, `name`, `beverageKey` |
| `src/main/java/com/foodapp/foodapp/rest/CustomerRestController.java` | Passes `cartFoodIds` from request body to service |

---

## Frontend Files

### Interfaces
| File | Key changes |
|---|---|
| `interfaces/food.tsx` | Added `recommendedFoodIds?: string`, `recommendedBeverageIds?: string`; removed `recommendedOptionKey` |
| `interfaces/beverage.tsx` | Removed `recommendedOptionKey` |
| `interfaces/filling.tsx` | Removed `recommendedOptionKey` |

### Services
| File | Purpose |
|---|---|
| `services/customer/recommendationService.ts` | `RecommendationRequest` has `cartFoodIds: number[]`; response has `recommendedFood`, `recommendedBeverages` (no fillings, no optionKey) |
| `services/admin/getter/getFoodsSimple.tsx` | GET `/api/admin/get/getAllFoodsSimple` — returns `{ id: number; name: string; foodKey: string }[]` |
| `services/admin/getter/getBeveragesSimple.tsx` | GET `/api/admin/get/getAllBeveragesSimple` — returns `{ id: number; name: string; beverageKey: string }[]` |

### Next.js API Routes
| File | Purpose |
|---|---|
| `app/api/customer/recommendations/route.ts` | Proxies POST to backend; body shape: `{ cartFoodIds: number[] }` |
| `app/api/admin/get/getAllFoodsSimple/route.ts` | Proxies GET to backend with Bearer auth |
| `app/api/admin/get/getAllBeveragesSimple/route.ts` | Proxies GET to backend with Bearer auth |

### Components
| File | Purpose |
|---|---|
| `components/Checkout/RecommendationModal.tsx` | Customer-facing modal; skip logic; grouped display by foodKey/beverageKey |
| `components/AdminFeatures/MenuComponents/EditMenuChild/EditFoodModal.tsx` | Admin modal; MultiSelect pickers for setting recommendedFoodIds / recommendedBeverageIds |

---

## API Contracts

### POST `/api/customer/recommendations`
```json
// Request
{ "cartFoodIds": [1, 5, 12] }

// Response
{
  "recommendedFood": [{ "id": 3, "name": "...", "price": 89.0, "image": "...", ... }],
  "recommendedBeverages": [{ "id": 2, "name": "...", "price": 29.0, ... }]
}
```

### GET `/api/admin/get/getAllFoodsSimple`
```json
[{ "id": 1, "name": "Margherita", "foodKey": "PIZZA" }, ...]
```

### GET `/api/admin/get/getAllBeveragesSimple`
```json
[{ "id": 1, "name": "Cola", "beverageKey": "SODAVAND" }, ...]
```

---

## Key Implementation Patterns

### Skip-modal logic (RecommendationModal.tsx)
- Internal `showModal` state starts `false`
- On `opened=true`: fetch recommendations silently
- Filter out items already in cart (by ID)
- If both `recommendedFood` and `recommendedBeverages` are empty after filtering: call `onProceedToCheckout()` directly — never show modal
- Only set `showModal(true)` when there is content to display
- On close: `setShowModal(false); onClose();`
- `useEffect` on `opened` resets `showModal(false)` in the else branch to handle parent closing

### Grouped display in RecommendationModal
Items are grouped by `foodKey` / `beverageKey` into sections with headers like "Anbefalede Pizza", "Anbefalede Sides". Uses IIFE pattern inside JSX to build groups without pre-computing outside render.

### Mantine v7 MultiSelect grouped format (EditFoodModal)
Mantine v7 requires nested grouped format — NOT flat items with a `group` property:
```typescript
// CORRECT
[{ group: 'Pizza', items: [{ value: '1', label: 'Margherita' }] }]

// WRONG (Mantine v6 style — throws "item.items is undefined")
[{ value: '1', label: 'Margherita', group: 'Pizza' }]
```
State type is `any[]` to accommodate the grouped structure.

### formatKey helper (EditFoodModal)
Converts snake_case enum keys to display labels:
```typescript
const formatKey = (key: string) =>
  key ? key.toLowerCase().split('_').map(w => w.charAt(0).toUpperCase() + w.slice(1)).join(' ') : '';
// "CHICKEN_WRAP" => "Chicken Wrap"
```

### Comma-separated ID parsing
```typescript
const parseIds = (ids?: string): string[] => {
  if (!ids || ids.trim() === '') return [];
  return ids.split(',').map(s => s.trim()).filter(Boolean);
};
```

### Backend: parsing comma-separated IDs in CustomerServiceImpl
```java
String[] ids = food.getRecommendedFoodIds().split(",");
List<Integer> foodIds = Arrays.stream(ids)
    .map(String::trim).filter(s -> !s.isEmpty())
    .map(Integer::parseInt).collect(Collectors.toList());
```

---

## Database Notes

Hibernate is NOT configured with `ddl-auto=update`, so columns must be added manually:

```sql
-- Required migration (add new columns)
ALTER TABLE Food ADD COLUMN recommended_food_ids VARCHAR(255);
ALTER TABLE Food ADD COLUMN recommended_beverage_ids VARCHAR(255);

-- Optional cleanup (remove old columns if still present)
ALTER TABLE Food DROP COLUMN recommended_option_key;
ALTER TABLE Beverage DROP COLUMN recommended_option_key;
ALTER TABLE Filling DROP COLUMN recommended_option_key;
```
