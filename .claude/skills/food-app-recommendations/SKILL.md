---
name: food-app-recommendations
description: >
  Domain knowledge skill for the food ordering app's Direct Item Recommendation System.
  Use this skill when the user wants to modify, debug, extend, or understand the recommendation
  feature in their food app (order-backend + next-app-template). Covers: the data model
  (recommendedFoodIds / recommendedBeverageIds on Food entity), the full backend stack
  (entity, DTO, DAO, service, controller), the frontend stack (interfaces, services, API
  routes, RecommendationModal, EditFoodModal), and the skip-modal logic. Trigger phrases
  include anything about "recommendations", "RecommendationModal", "EditFoodModal recommendations
  section", "recommended foods/beverages", or "anbefalinger".
---

# Food App Recommendations Skill

This skill provides full architectural context for the Direct Item Recommendation System
implemented across `order-backend` (Spring Boot / Hibernate) and `next-app-template` (Next.js).

Read `references/architecture.md` immediately — it contains all file paths, data model details,
API contracts, and key implementation patterns needed to work on this feature.
