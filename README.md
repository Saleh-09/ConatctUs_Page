This is a pure architecture question — no UI to design. Here's the full senior-engineer critique and refactor plan.

---

## TL;DR Verdict on `mode` prop

**Don't do it.** The `mode` prop pattern is an anti-pattern at scale. It turns every shared component into a growing `switch` statement, couples API logic to UI, and forces you to touch 4+ files every time you add a page. Reject it before it spreads.

---

## The Core Problem

Your current mental model is:

> "I have one component that behaves differently depending on where it's used."

The correct mental model is:

> "I have one **UI shell** that renders whatever **data contract** it's given."

The shift is from *conditional components* to *data-driven composition*.

---

## Recommended Architecture: Feature-Sliced + Config-Driven

### Folder Structure

```
src/
├── pages/
│   ├── PageOne/
│   │   ├── index.tsx              ← composes the page
│   │   └── config.ts              ← page-specific config object
│   ├── PageTwo/
│   │   ├── index.tsx
│   │   └── config.ts
│   └── PageThree/
│       ├── index.tsx
│       └── config.ts
│
├── features/
│   ├── map/
│   │   ├── MapSection.tsx         ← pure UI, accepts props only
│   │   ├── useMapData.ts          ← generic hook interface
│   │   └── types.ts
│   ├── search/
│   │   ├── SearchSection.tsx
│   │   ├── useSearchData.ts
│   │   └── types.ts
│   ├── equipment/
│   │   ├── EquipmentSection.tsx
│   │   ├── useEquipmentData.ts
│   │   └── types.ts
│   └── customerSupport/
│       ├── CustomerSupportSection.tsx
│       ├── useCustomerSupportData.ts
│       └── types.ts
│
├── api/
│   ├── pageOne/
│   │   ├── mapApi.ts
│   │   ├── searchApi.ts
│   │   └── equipmentApi.ts
│   ├── pageTwo/
│   │   └── ...
│   └── shared/
│       └── httpClient.ts          ← axios/fetch wrapper
│
└── hooks/
    └── (cross-feature hooks only)
```

---

## The Config Pattern — The Key Insight

Each page owns a `config.ts` that wires its APIs to the shared UI hooks.

```ts
// src/pages/PageOne/config.ts
import { fetchPageOneLocations } from "@/api/pageOne/mapApi";
import { fetchPageOneSearch } from "@/api/pageOne/searchApi";
import { fetchPageOneEquipment } from "@/api/pageOne/equipmentApi";

export const pageOneConfig = {
  map: {
    fetchData: fetchPageOneLocations,
    visualizationMode: "cluster",
  },
  search: {
    fetchLocations: fetchPageOneSearch.locations,
    fetchPoints: fetchPageOneSearch.points,
    fetchCountries: fetchPageOneSearch.countries,
  },
  equipment: {
    fetchData: fetchPageOneEquipment,
  },
  customerSupport: {
    fetchData: fetchPageOneSupport,
  },
};

export type PageConfig = typeof pageOneConfig; // shared contract
```

---

## Shared Feature Hooks — Accept Fetchers, Not Mode Strings

```ts
// src/features/map/useMapData.ts
import { useQuery } from "@tanstack/react-query";

type MapFetcher = () => Promise<MapDataType>;

export function useMapData(fetchData: MapFetcher) {
  return useQuery({
    queryKey: [fetchData.name],   // stable key per page
    queryFn: fetchData,
  });
}
```

```ts
// src/features/map/MapSection.tsx
interface MapSectionProps {
  fetchData: () => Promise<MapDataType>;
  visualizationMode: string;
}

export function MapSection({ fetchData, visualizationMode }: MapSectionProps) {
  const { data, isLoading } = useMapData(fetchData);
  // render — zero conditionals, zero mode checks
}
```

---

## Page Composition — Simple and Clean

```tsx
// src/pages/PageOne/index.tsx
import { pageOneConfig } from "./config";
import { MapSection } from "@/features/map/MapSection";
import { SearchSection } from "@/features/search/SearchSection";
import { EquipmentSection } from "@/features/equipment/EquipmentSection";
import { CustomerSupportSection } from "@/features/customerSupport/CustomerSupportSection";

export default function PageOne() {
  const { map, search, equipment, customerSupport } = pageOneConfig;

  return (
    <>
      <SearchSection
        fetchLocations={search.fetchLocations}
        fetchPoints={search.fetchPoints}
        fetchCountries={search.fetchCountries}
      />
      <MapSection
        fetchData={map.fetchData}
        visualizationMode={map.visualizationMode}
      />
      <EquipmentSection fetchData={equipment.fetchData} />
      <CustomerSupportSection fetchData={customerSupport.fetchData} />
    </>
  );
}
```

Adding **PageFive** later = create `pages/PageFive/config.ts`, wire new API fetchers, compose. **Zero existing files touched.**

---

## API Layer — Isolated Per Page

```ts
// src/api/pageOne/mapApi.ts
import { httpClient } from "@/api/shared/httpClient";
import type { MapDataType } from "@/features/map/types";

export async function fetchPageOneLocations(): Promise<MapDataType> {
  const res = await httpClient.get("/page-one/locations");
  return res.data;
}
```

Each page's API folder is a black box. The feature components never import from `api/` directly — only the config does.

---

## Decision Table

| Concern | Your current approach | Recommended approach |
|---|---|---|
| Page differentiation | `mode` prop + `if/else` | Config object + injected fetchers |
| API coupling | Inside components | In `api/pageX/` + wired via config |
| Adding a new page | Edit 4+ components | Add 1 config file + 1 page file |
| Shared UI | One component with branches | Pure component, data via props |
| Query logic | Scattered | In `useXData(fetcher)` hooks |
| Testability | Hard (mode-dependent) | Easy (mock the fetcher) |

---

## What to Migrate First

1. **Extract all API calls** out of components into `src/api/pageOne/`
2. **Create `pageOneConfig.ts`** wiring those fetchers
3. **Refactor `MapSection`** to accept `fetchData` prop instead of `mode` — this is the proof of concept
4. **Repeat for Search, Equipment, Support**
5. **Create `PageTwo/config.ts`** using its own API folder — if it composes cleanly, the architecture is working

This scales cleanly to 10, 20, or 50 pages with no architectural debt accumulation.
