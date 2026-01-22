# Research: Raycast Plugin as HTTP API Proxy with URL Actions

**Date:** 2026-01-22
**Status:** Complete

## Problem Statement

Create a Raycast extension that acts as a proxy to the Portal Wholesale tRPC API, allowing users to:
1. Search for brands via the backend API
2. Display search results in a list
3. Perform actions on results like "Open Brand Page", "Open Brand Admin", or "Open Homepage"

## Your Specific API

### Endpoint

```
GET https://portalwholesale.com/trpc/brands.searchByNameOrWebsiteUrl?batch=1&input={...}
```

### Input Format

URL-encoded JSON:
```json
{"0": {"searchQuery": "maria la rosa"}}
```

### Response Format

```typescript
type BrandSearchResponse = [{
  result: {
    data: Array<{
      id: number;
      name: string;
      slug: string;
      headquarters_city: string;
      headquarters_country: string;
      brand_logo: {
        id: number;
        asset_path: string;
        asset_bucket: string;
      };
    }>;
  };
}];
```

### Derived URLs

Based on the slug, you can construct:
- **Brand Page:** `https://portalwholesale.com/brands/${slug}`
- **Brand Admin:** (you'll need to provide this pattern)

## Key Findings

### Technology Stack

Raycast extensions are built with:
- **TypeScript** and **React** (React 19)
- **Node.js 22** (fetch is globally available)
- **@raycast/api** - Core UI components
- **@raycast/utils** - Utility hooks for data fetching

### Making HTTP API Calls

#### Recommended: `useFetch` Hook

```typescript
import { List } from "@raycast/api";
import { useFetch } from "@raycast/utils";

export default function Command() {
  const { isLoading, data } = useFetch<Brand[]>(
    "https://api.example.com/brands?q=search"
  );

  return <List isLoading={isLoading}>...</List>;
}
```

#### For Your tRPC API

Since the tRPC response is wrapped in an array with `result.data`, you'll need to transform it:

```typescript
import { useFetch } from "@raycast/utils";

const { isLoading, data: brands } = useFetch(
  `https://portalwholesale.com/trpc/brands.searchByNameOrWebsiteUrl?batch=1&input=${encodeURIComponent(
    JSON.stringify({ "0": { searchQuery: searchText } })
  )}`,
  {
    execute: searchText.length > 0,
    parseResponse: async (response) => {
      const json = await response.json();
      return json[0]?.result?.data ?? [];
    },
  }
);
```

### Displaying Search Results

Use the `List` component:

```typescript
import { List, ActionPanel, Action } from "@raycast/api";

<List searchBarPlaceholder="Search brands...">
  <List.Item
    title="Brand Name"
    subtitle="Milan, IT"
    actions={<ActionPanel>...</ActionPanel>}
  />
</List>
```

### Opening URLs with Actions

```typescript
<ActionPanel>
  <Action.OpenInBrowser
    title="Open Brand Page"
    url={`https://portalwholesale.com/brands/${brand.slug}`}
  />
  <Action.OpenInBrowser
    title="Open Brand Admin"
    url={`https://portalwholesale.com/admin/brands/${brand.slug}`}
  />
  <Action.OpenInBrowser
    title="Open Homepage"
    url="https://portalwholesale.com"
  />
</ActionPanel>
```

## Complete Implementation Example

```typescript
import { ActionPanel, Action, List, Icon } from "@raycast/api";
import { useFetch } from "@raycast/utils";
import { useState } from "react";

interface Brand {
  id: number;
  name: string;
  slug: string;
  headquarters_city: string;
  headquarters_country: string;
  brand_logo: {
    id: number;
    asset_path: string;
    asset_bucket: string;
  };
}

export default function SearchBrands() {
  const [searchText, setSearchText] = useState("");

  const { isLoading, data: brands } = useFetch(
    `https://portalwholesale.com/trpc/brands.searchByNameOrWebsiteUrl?batch=1&input=${encodeURIComponent(
      JSON.stringify({ "0": { searchQuery: searchText } })
    )}`,
    {
      execute: searchText.length > 0,
      parseResponse: async (response) => {
        const json = await response.json();
        return (json[0]?.result?.data ?? []) as Brand[];
      },
    }
  );

  return (
    <List
      isLoading={isLoading}
      searchBarPlaceholder="Search for a brand..."
      onSearchTextChange={setSearchText}
      throttle
    >
      {(brands || []).map((brand) => (
        <List.Item
          key={brand.id}
          title={brand.name}
          subtitle={`${brand.headquarters_city}, ${brand.headquarters_country}`}
          icon={Icon.Building}
          actions={
            <ActionPanel>
              <Action.OpenInBrowser
                title="Open Brand Page"
                url={`https://portalwholesale.com/brands/${brand.slug}`}
              />
              <Action.OpenInBrowser
                title="Open Brand Admin"
                url={`https://portalwholesale.com/admin/brands/${brand.slug}`}
              />
              <Action.CopyToClipboard
                title="Copy Brand Name"
                content={brand.name}
              />
            </ActionPanel>
          }
        />
      ))}
    </List>
  );
}
```

## Authentication Consideration

Your API requires authentication via cookies (`__Secure-portal-authn.session_token`). Options:

1. **If the API is public** (no auth needed for search): Use as-is
2. **If auth is required**: You may need to:
   - Use Raycast OAuth flow
   - Store a session token in preferences
   - Add a Cookie header to requests

## Codebase Patterns

This is a new/empty project. No existing patterns to follow.

## Recommended Approach

### Project Setup

```bash
npx create-raycast-extension
npm install @raycast/utils
```

### File Structure

```
portal-wholesale-plugin/
├── src/
│   └── search-brands.tsx
├── package.json
└── tsconfig.json
```

### package.json Commands

```json
{
  "commands": [
    {
      "name": "search-brands",
      "title": "Search Brands",
      "description": "Search Portal Wholesale brands",
      "mode": "view"
    }
  ]
}
```

## Sources

- [useFetch | Raycast API](https://developers.raycast.com/utilities/react-hooks/usefetch)
- [Actions | Raycast API](https://developers.raycast.com/api-reference/user-interface/actions)
- [List | Raycast API](https://developers.raycast.com/api-reference/user-interface/list)
- [Getting Started | Raycast API](https://developers.raycast.com/basics/getting-started)
- [GitHub - raycast/extensions](https://github.com/raycast/extensions)
