# System Design: Server-Driven UI (SDUI) for Mobile App

## What is SDUI?

Server-Driven UI means the **server controls what UI to render**, not the client. The mobile app becomes a "renderer" — it receives a JSON payload describing the screen, and renders it dynamically without a new app release.

---

## High-Level Design (HLD)

### Core Idea

```
┌─────────────┐     Screen Request      ┌──────────────────┐
│  Mobile App │ ──────────────────────▶ │   SDUI Backend   │
│  (Renderer) │ ◀────────────────────── │  (Orchestrator)  │
└─────────────┘     UI JSON Payload     └──────────────────┘
                                                 │
                     ┌───────────────────────────┼──────────────────────┐
                     ▼                           ▼                      ▼
            ┌────────────────┐      ┌────────────────────┐   ┌──────────────────┐
            │  Template CMS  │      │   Feature Flags /  │   │  Business Logic  │
            │  (UI Schemas)  │      │   A/B Test Service │   │  Microservices   │
            └────────────────┘      └────────────────────┘   └──────────────────┘
```

### HLD Components

**1. Client (Mobile App)**
- Stateless renderer — reads JSON, renders components
- Component Registry maps `type` → native widget
- Handles user actions via `action` descriptors in JSON
- Caches payloads locally (SQLite / disk cache)

**2. SDUI Orchestrator (BFF — Backend for Frontend)**
- Entry point for all screen requests
- Assembles the final UI JSON from multiple sources
- Applies A/B test variants and feature flags
- Personalizes based on user context

**3. Template Service**
- Stores and versions screen templates (skeleton structure)
- Owned by product/design teams via CMS
- Templates are parameterized — data is injected at runtime

**4. Feature Flag Service**
- Controls which components/screens to show per user segment
- Enables gradual rollouts without app updates

**5. Data Services**
- Business microservices (catalog, cart, recommendations, etc.)
- Orchestrator calls them to hydrate template placeholders

**6. CDN + Cache Layer**
- Caches non-personalized or lightly-personalized screens
- Edge caching for low-latency response globally

---

### HLD Architecture Diagram

```
                         ┌────────────────────────────────────────────────┐
                         │                  Mobile App                    │
                         │  ┌──────────┐  ┌──────────┐  ┌─────────────┐  │
                         │  │  Screen  │  │Component │  │Action/Event │  │
                         │  │  Router  │  │ Registry │  │  Handler    │  │
                         │  └──────────┘  └──────────┘  └─────────────┘  │
                         └───────────────────┬────────────────────────────┘
                                             │ HTTPS + gRPC
                         ┌───────────────────▼────────────────────────────┐
                         │              API Gateway / CDN                  │
                         │         (Auth, Rate Limit, Edge Cache)          │
                         └───────────────────┬────────────────────────────┘
                                             │
                         ┌───────────────────▼────────────────────────────┐
                         │           SDUI Orchestrator (BFF)              │
                         │  ┌──────────────────────────────────────────┐  │
                         │  │  Request Context Builder                 │  │
                         │  │  (user_id, platform, locale, version)    │  │
                         │  └─────────────────┬────────────────────────┘  │
                         │                    │ Fan-out                    │
                         │    ┌───────────────┼───────────────┐           │
                         │    ▼               ▼               ▼           │
                         │ Template      Feature Flag     Data APIs        │
                         │ Service       Service          (parallel)       │
                         │    └───────────────┬───────────────┘           │
                         │                    │ Merge & Render             │
                         │  ┌─────────────────▼────────────────────────┐  │
                         │  │        UI JSON Response Builder          │  │
                         │  └──────────────────────────────────────────┘  │
                         └────────────────────────────────────────────────┘
```

---

## Low-Level Design (LLD)

### 1. UI JSON Payload Schema

This is the contract between server and client.

```json
{
  "screen_id": "home_v3",
  "version": "2.1.0",
  "ttl": 300,
  "layout": {
    "type": "scroll_view",
    "children": [
      {
        "type": "banner_carousel",
        "id": "hero_banner",
        "props": {
          "auto_scroll": true,
          "interval_ms": 3000,
          "items": [
            {
              "image_url": "https://cdn.example.com/banner1.jpg",
              "action": {
                "type": "navigate",
                "destination": "product_detail",
                "params": { "product_id": "P123" }
              }
            }
          ]
        }
      },
      {
        "type": "section_header",
        "id": "rec_header",
        "props": {
          "title": "Recommended for You",
          "subtitle": "Based on your history"
        }
      },
      {
        "type": "product_grid",
        "id": "recommendations",
        "props": {
          "columns": 2,
          "items": [ /* injected by data service */ ]
        }
      }
    ]
  },
  "analytics": {
    "screen_name": "home",
    "experiment_id": "exp_home_v3"
  }
}
```

---

### 2. Component Taxonomy

```
ComponentType (enum)
├── Layout
│   ├── scroll_view
│   ├── stack_view (vertical / horizontal)
│   ├── grid_view
│   └── z_stack (overlay)
├── Display
│   ├── text
│   ├── image
│   ├── banner_carousel
│   ├── video_player
│   └── lottie_animation
├── Interactive
│   ├── button
│   ├── text_field
│   ├── toggle
│   ├── bottom_sheet_trigger
│   └── tab_bar
├── Composite
│   ├── product_card
│   ├── section_header
│   ├── product_grid
│   └── list_tile
└── Skeleton / Loading State
    └── shimmer_placeholder
```

---

### 3. Action Descriptor Schema

All user interactions are described as actions — no business logic in the app.

```json
{
  "type": "navigate" | "api_call" | "deep_link" | "bottom_sheet" | "track_event" | "share",
  "destination": "cart_screen",
  "params": {},
  "on_success": { /* another action */ },
  "on_failure": { /* fallback action */ },
  "analytics": {
    "event": "add_to_cart_tapped",
    "properties": { "product_id": "P123" }
  }
}
```

---

### 4. Orchestrator — Internal Flow

```
POST /screen?id=home&user_id=U42&platform=android&app_version=4.2.1

┌─────────────────────────────────────────────┐
│            Orchestrator Flow                │
│                                             │
│  1. Parse & validate request context        │
│  2. Fetch template from Template Service    │
│     → GET /templates/home (cached)          │
│  3. Resolve feature flags                   │
│     → GET /flags?user=U42&screen=home       │
│  4. Fan-out data calls (parallel)           │
│     → GET /recommendations?user=U42         │
│     → GET /banners?segment=premium          │
│     → GET /cart/count?user=U42              │
│  5. Merge: inject data into template slots  │
│  6. Apply component overrides from flags    │
│  7. Validate output against JSON schema     │
│  8. Set Cache-Control headers               │
│  9. Return UI JSON                          │
└─────────────────────────────────────────────┘
```

---

### 5. Client-Side Component Registry (Kotlin/Swift pseudocode)

```kotlin
// Component Registry — maps JSON type to Composable/UIView
object ComponentRegistry {
    private val registry = mapOf(
        "text"             to ::TextComponent,
        "image"            to ::ImageComponent,
        "button"           to ::ButtonComponent,
        "product_card"     to ::ProductCardComponent,
        "banner_carousel"  to ::BannerCarouselComponent,
        "scroll_view"      to ::ScrollViewComponent,
        "product_grid"     to ::ProductGridComponent,
        "shimmer_placeholder" to ::ShimmerComponent
    )

    fun render(node: UINode): Composable {
        val renderer = registry[node.type] ?: ::UnknownComponentFallback
        return renderer(node.props, node.children)
    }
}

// Recursive tree renderer
fun renderTree(node: UINode): Composable {
    val component = ComponentRegistry.render(node)
    node.children?.forEach { child -> renderTree(child) }
    return component
}
```

---

### 6. Caching Strategy

| Layer | Mechanism | TTL | Scope |
|---|---|---|---|
| CDN / Edge | Cache-Control headers | 5–60 min | Non-personalized screens |
| App Memory | In-memory LRU | Session | Current session screens |
| App Disk | SQLite / File cache | 5 min | Between app launches |
| Stale-While-Revalidate | Serve stale, fetch fresh | Background | All screens |

```
Client Cache Logic:

  Request screen →  Cache HIT + not expired?  → Render immediately
                          │ No
                          ▼
                    Fetch from server
                          │
                    Show skeleton/shimmer
                          │
                    Response received → Cache + Render
```

---

### 7. Versioning & Backward Compatibility

```
app_version: 4.2.1   →  Orchestrator serves schema v2
app_version: 3.x.x   →  Orchestrator serves schema v1 (legacy)

Component-level versioning:
  "type": "product_card@v2"   // explicit version pinning
  "type": "product_card"      // uses current default
```

**Unknown component handling on client:**
- Log unknown `type` to error tracker
- Render `fallback_component` (empty space or graceful skip)
- Never crash

---

### 8. Real-Time Updates (Optional Layer)

```
WebSocket / SSE Channel:
  Server → Client push:
  {
    "event": "patch_component",
    "target_id": "cart_badge",
    "patch": { "props": { "count": 3 } }
  }
```

Used for: cart count updates, flash sale timers, live stock status.

---

## Key Design Decisions & Trade-offs

| Decision | Choice | Reason |
|---|---|---|
| Payload format | JSON over Protobuf | Debuggability; Protobuf for perf-critical paths |
| Rendering | Declarative (Compose / SwiftUI) | Maps naturally to tree structure |
| Action handling | Descriptor-based | Keeps logic server-side, app stays thin |
| Fallback | Last-good cached screen | Resilience against server failures |
| Experimentation | Server-side flag resolution | No client changes for A/B tests |

---

## What SDUI Enables

- **Zero app-update UI changes** — redesign screens without shipping a new build
- **A/B testing at component level** — test a single button's copy or color server-side
- **Personalization** — different users see different layouts
- **Feature flags** — instant kill-switch for any UI component
- **Faster iteration** — product teams control UI without engineering deploys
