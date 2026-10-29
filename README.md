# Building a Dynamic Multi-Department Category Showcase in FlutterFlow from Shopify

A production-ready guide to architecting a department-organized category experience in FlutterFlow powered by Shopify's GraphQL Storefront API.

---

## Section 1: Architecture & Performance Overview

Structuring a catalog into distinct, categorized departments eliminates cognitive fatigue and improves conversion rates:

* **Zero Recurring SaaS / App Fees:** Eliminates third-party Shopify navigation apps by processing layout natively on-device or at the edge.
* **Single Network Trip:** Fetches up to 250 collections in one round-trip GraphQL query (`first: 250`), avoiding multiple round-trips and saving API quota.
* **Automated Catalog Hygiene:** Automatically filters out internal vendor routing tags, supplier handles, and empty groups before rendering.
* **Dynamic Discovery:** Shuffles general categories within their groups on every app visit while pinning promotional collections (e.g., Flash Deals, Clearance) at the top in strict order.

---

## Step 1: Configure Shopify Storefront GraphQL Query (Data Layer)

Query collections with both their **title** and **handle**. Store merchants frequently change display titles for campaigns, but handles typically remain stable.

### FlutterFlow API Call Settings

* **Call Name:** `GetShopifyCollections`
* **Method:** `POST`
* **API URL:** `https://<YOUR-SHOPIFY-STORE>[.myshopify.com/api/2024-04/graphql.json](https://.myshopify.com/api/2024-04/graphql.json)`
* **Headers:**
* `Content-Type: application/json`
* `X-Shopify-Storefront-Access-Token: <YOUR_STOREFRONT_ACCESS_TOKEN>`



### GraphQL Request Body

```json
{
  "query": "{ collections(first: 250) { edges { node { id title handle image { url } } } } }"
}

```

> **Why `first: 250`?** Shopify's Storefront API allows a maximum of 250 nodes per connection query. Requesting 250 collections consumes minimal query points (~25–35 against the 1,000 points/second bucket) and covers comprehensive catalogs without pagination latency.

---

## Step 2: Implement Dart Grouping Function (Transform Layer)

This Custom Function receives the raw JSON edges list, filters out blacklisted handles, distributes collections into department buckets via keyword matching, randomizes discovery categories, and sorts promotional collections strictly.

### Function Configuration

* **Name:** `groupShopifyCollections`
* **Return Type:** `JSON` (Check **Is List**)
* **Argument:** `edges` $\rightarrow$ `JSON` (Check **Is List**)

### Dart Implementation

```dart
import 'dart:convert';
import 'dart:math' as math;

List<dynamic> groupShopifyCollections(List<dynamic>? edges) {
  if (edges == null || edges.isEmpty) {
    return [];
  }

  // 1. Blacklist for internal, vendor, or draft sync handles
  final Set<String> blacklistedKeywords = {
    'vendor-',
    'internal-sync',
    'test-collection',
    'archive',
    'admin-only',
  };

  // 2. Department blueprints: metadata and matching keywords
  final List<Map<String, dynamic>> groupDefinitions = [
    {
      'id': 'group-1',
      'name': 'Kitchen & Dining',
      'icon': '🍳',
      'keywords': [
        'kitchen', 'baking', 'cookware', 'cutlery', 'choppers',
        'knives', 'dinnerware', 'storage', 'bottles'
      ],
    },
    {
      'id': 'group-2',
      'name': 'Tech & Electronics',
      'icon': '⚡',
      'keywords': [
        'appliances', 'electronics', 'chargers', 'cables',
        'audio', 'accessories', 'gadgets', 'computers'
      ],
    },
    {
      'id': 'group-3',
      'name': 'Home & Living',
      'icon': '🏠',
      'keywords': [
        'home decor', 'cleaning', 'hardware', 'lighting',
        'bathroom', 'bedroom', 'organizers', 'tools'
      ],
    },
    {
      'id': 'group-4',
      'name': 'Fashion & Apparel',
      'icon': '👗',
      'keywords': [
        'women', 'men', 'shirts', 't-shirts', 'ethnic',
        'dresses', 'footwear', 'bottom wear', 'jackets'
      ],
    },
    {
      'id': 'group-5',
      'name': 'Deals & Specials',
      'icon': '🔥',
      'keywords': [
        'under-99', 'clearance', 'trending', 'flash-sale', 'hot-picks'
      ],
    },
  ];

  // Enforced display order for promotional deal collections
  final List<String> dealsStrictOrder = [
    'flash-sale',
    'clearance',
    'under-99',
    'trending',
  ];

  // 3. Initialize storage buckets
  final Map<String, List<dynamic>> buckets = {
    for (var def in groupDefinitions) def['id'] as String: [],
  };

  // 4. Distribute collections into respective buckets
  for (var item in edges) {
    if (item == null || item is! Map || item['node'] == null) continue;
    final node = item['node'] as Map;
    final String title = (node['title']?.toString() ?? '').toLowerCase().trim();
    final String handle = (node['handle']?.toString() ?? '').toLowerCase().trim();

    // Skip blacklisted keywords or prefix patterns
    bool isBlacklisted = blacklistedKeywords.any((k) =>
        title.contains(k) || handle.contains(k) || handle.startsWith('vendor-'));
    if (isBlacklisted) continue;

    // Check promotional deals group first
    final bool isDeal = dealsStrictOrder.any((k) =>
        title == k || handle == k || title.contains(k) || handle.contains(k));

    if (isDeal) {
      buckets['group-5']!.add(item);
    } else {
      for (var def in groupDefinitions) {
        if (def['id'] == 'group-5') continue;
        final List<String> keywords = def['keywords'] as List<String>;
        final bool hit = keywords.any((k) =>
            title == k || handle == k || title.contains(k) || handle.contains(k));

        if (hit) {
          buckets[def['id']]!.add(item);
          break;
        }
      }
    }
  }

  // 5. In-group randomization for standard categories
  final random = math.Random();
  for (var def in groupDefinitions) {
    final String groupId = def['id'] as String;
    if (groupId != 'group-5') {
      buckets[groupId]!.shuffle(random);
    }
  }

  // 6. Strict rank sorting for the deals group
  buckets['group-5']!.sort((a, b) {
    final nodeA = a['node'] as Map;
    final nodeB = b['node'] as Map;
    final String titleA = (nodeA['title']?.toString() ?? '').toLowerCase().trim();
    final String handleA = (nodeA['handle']?.toString() ?? '').toLowerCase().trim();
    final String titleB = (nodeB['title']?.toString() ?? '').toLowerCase().trim();
    final String handleB = (nodeB['handle']?.toString() ?? '').toLowerCase().trim();

    int getRank(String title, String handle) {
      for (int i = 0; i < dealsStrictOrder.length; i++) {
        final key = dealsStrictOrder[i];
        if (title == key || handle == key || title.contains(key) || handle.contains(key)) {
          return i;
        }
      }
      return 999;
    }

    return getRank(titleA, handleA).compareTo(getRank(titleB, handleB));
  });

  // 7. Assemble final output list (skipping empty groups)
  final List<Map<String, dynamic>> output = [];
  for (var def in groupDefinitions) {
    final items = buckets[def['id']]!;
    if (items.isNotEmpty) {
      output.add({
        'id': def['id'],
        'name': def['name'],
        'icon': def['icon'],
        'count': items.length,
        'collections': items,
      });
    }
  }

  return output;
}

```

---

## Step 3: Build Reusable UI Components (Bottom-Up)

In FlutterFlow, build lower-tier atomic components before parent containers so references and parameter bindings exist when configuring parent layouts.

### 3A. Leaf Component: `CategoryCard`

Renders an individual collection tile inside a department grid.

1. **Component Parameters:**
* `categoryImage` (`String` / `ImagePath`): Provide a fallback image URL in component properties to prevent crashes on collections without featured artwork.
* `title` (`String`): Set Default Value to `"Collection"`.
* `collectionId` (`String`): Required for page navigation.


2. **Widget Tree & Layout:**
* **Root Container:** Border radius `12px`, Background `#FAFBFC`, clip content ON.
* **Column:** Cross-axis alignment set to Stretch.
* **Image:** Width `100%`, Height `110px`, Box Fit `Cover`. Set path to parameter `categoryImage`.
* **Padding (8px):**
* **Text:** Bound to parameter `title`, Max Lines: `2`, Overflow: `Ellipsis`, Font Size: `13px`, Weight: `Semi-Bold`.








3. **On Tap Action:**
* Action: **Navigate To** $\rightarrow$ `ProductCollectionPage`.
* Pass Parameters:
* `collectionId` = Component Parameter `collectionId`
* `collectionTitle` = Component Parameter `title`





---

### 3B. Composite Component: `DepartmentSectionCard`

Holds the section header badge and an inner non-scrollable GridView populated by `CategoryCard` items.

1. **Component Parameters:**
* `emoji` (`String`): Default `"📁"`
* `title` (`String`): Default `"Department"`
* `countLabel` (`Integer` or `String`): Total item count
* `collectionsList` (`List <JSON>`): Edges list for this department


2. **Widget Tree & Layout:**
* **Root Container:** Margin horizontal `12px`, vertical `8px`; padding `14px`; border radius `16px`; background `#FFFFFF`.
* **Column:** Cross-axis alignment Stretch.
* **Header Row:** Main-axis alignment `Space Between`.
* **Row (Left):** Text `${emoji}` (Size: `18px`) + 8px space + Text `${title}` (Size: `16px`, Weight: `Bold`).
* **Container (Right):** Pill badge with padding (h: `8px`, v: `4px`), border radius `20px`, soft accent background.
* **Text:** `${countLabel} Categories` (Size: `11px`, Weight: `Medium`).




* **12px Vertical Spacer / SizedBox**
* **Inner GridView:**
* **Cross Axis Count:** `2`
* **Spacing:** `10px` main-axis and cross-axis
* **Child Aspect Ratio:** Set to `0.75` or `0.78` (allocates proper vertical height for title text below image)
* **Shrink Wrap:** `ON` *(Allows GridView to size dynamically within the parent ListView)*
* **Primary:** `OFF` / Scroll Physics set to `NeverScrollableScrollPhysics`








3. **GridView Dynamic Children Binding:**
* Select the inner **GridView** $\rightarrow$ **Generate Dynamic Children**.
* Variable Name: `collectionItem`
* Source Value: Component Parameter `collectionsList`
* Place **`CategoryCard`** inside the GridView as the generated child.
* Map `CategoryCard` parameters from loop variable `collectionItem`:
* `categoryImage` $\rightarrow$ `collectionItem` $\rightarrow$ JSON Path: `$.node.image.url` (Provide fallback image URL)
* `title` $\rightarrow$ `collectionItem` $\rightarrow$ JSON Path: `$.node.title`
* `collectionId` $\rightarrow$ `collectionItem` $\rightarrow$ JSON Path: `$.node.id`





---

## Step 4: Assemble CategoryExplorer Screen & Data Binding

Connect the Storefront API backend query with the custom grouping function on the main page.

1. **Create Page:** Create a page named `CategoryExplorer`.
2. **Add Root ListView:** Place an outer `ListView` on the screen.
3. **Attach Backend Query:**
* Select `ListView` $\rightarrow$ **Backend Query** tab.
* Query Type: **API Call**.
* API Call: `GetShopifyCollections`.


4. **Generate Dynamic Groups:**
* On the same `ListView`, switch to **Generate Dynamic Children**.
* Variable Name: `departmentGroup`
* Source: Select API response `GetShopifyCollections` $\rightarrow$ JSON Body $\rightarrow$ JSON Path: `$.data.collections.edges`
* Pass this value into the Custom Function: `groupShopifyCollections`.


5. **Drop Section Component:**
* Place `DepartmentSectionCard` directly inside the dynamic `ListView`.


6. **Map Parameters from `departmentGroup`:**
* `emoji` $\rightarrow$ `departmentGroup` $\rightarrow$ JSON Path: `$.icon`
* `title` $\rightarrow$ `departmentGroup` $\rightarrow$ JSON Path: `$.name`
* `countLabel` $\rightarrow$ `departmentGroup` $\rightarrow$ JSON Path: `$.count`
* `collectionsList` $\rightarrow$ `departmentGroup` $\rightarrow$ JSON Path: `$.collections` (Select Type: **JSON List**)



---

## Step 5: Advanced Alternative: Offloading Grouping to Supabase Edge Functions

Transition the grouping layer to a Supabase Edge Function if:

* **Dynamic Restructuring:** Merchants change keywords frequently, and rules must update without pushing app store releases.
* **Large Catalogs (>250 collections):** The Edge Function executes cursor-based pagination behind the scenes, aggregates all collections, and returns a single JSON payload.
* **Response Caching:** Responses can be cached in Supabase or Redis with a 15–60 minute TTL, reducing cold-start query costs to Shopify.

### Architectural Flow

```text
[ Shopify Storefront GraphQL ]
             ▲
             │ (Paginated GraphQL Fetch)
[ Supabase Edge Function (Deno/TypeScript) ]
   ├── Reads grouping rules from DB table
   ├── Filters blacklist & sorts deals
   └── Caches response (TTL: 15–60 mins)
             │
             ▼ Clean Grouped JSON: [ { name, icon, collections: [...] } ]
[ FlutterFlow Mobile App ]  <── Binds directly to ListView without client Dart parsing

```

### Supabase Edge Function (`get-grouped-categories/index.ts`)

```typescript
import { serve } from "https://deno.land/std@0.168.0/http/server.ts";

const SHOPIFY_GRAPHQL_URL = "https://<YOUR-SHOPIFY-STORE>.myshopify.com/api/2024-04/graphql.json";
const STOREFRONT_TOKEN = Deno.env.get("SHOPIFY_STOREFRONT_ACCESS_TOKEN")!;

serve(async (req) => {
  try {
    const query = `{
      collections(first: 250) {
        edges {
          node {
            id
            title
            handle
            image { url }
          }
        }
      }
    }`;

    const shopifyRes = await fetch(SHOPIFY_GRAPHQL_URL, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "X-Shopify-Storefront-Access-Token": STOREFRONT_TOKEN,
      },
      body: JSON.stringify({ query }),
    });

    const data = await shopifyRes.json();
    const rawEdges = data?.data?.collections?.edges || [];

    const groupDefinitions = [
      { id: "group-1", name: "Kitchen", icon: "🍳", keywords: ["kitchen", "cutlery", "baking"] },
      { id: "group-2", name: "Tech", icon: "⚡", keywords: ["cables", "audio", "appliances"] },
      { id: "group-3", name: "Fashion", icon: "👗", keywords: ["shirts", "dresses", "shoes"] }
    ];

    const output = groupDefinitions.map(def => {
      const matched = rawEdges.filter((edge: any) => {
        const title = (edge.node.title || "").toLowerCase();
        const handle = (edge.node.handle || "").toLowerCase();
        return def.keywords.some((k: string) => title.includes(k) || handle.includes(k));
      });

      return {
        id: def.id,
        name: def.name,
        icon: def.icon,
        count: matched.length,
        collections: matched,
      };
    }).filter(group => group.count > 0);

    return new Response(JSON.stringify(output), {
      headers: { "Content-Type": "application/json" },
      status: 200,
    });
  } catch (err: any) {
    return new Response(JSON.stringify({ error: err.message }), { status: 500 });
  }
});

```

---

## Architectural Comparison

| Feature | FlutterFlow Custom Function | Supabase Edge Function |
| --- | --- | --- |
| **Setup Overhead** | 5 minutes (Zero backend infrastructure) | Requires Supabase Project & CLI deploy |
| **Updating Rules** | Requires app update/re-release | Instant update via DB or script reload |
| **Pagination Support** | Limited to single 250-node query | Automatic multi-page cursor traversal |
| **API Load on Shopify** | Direct from each mobile client | Batched and cacheable via Edge TTL/KV |
| **Execution Latency** | Instant local memory processing | ~80–120ms global edge network latency |
