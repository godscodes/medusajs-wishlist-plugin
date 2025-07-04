# Medusa v2 Example: Wishlist Plugin

This directory holds the code for the [Wishlist Plugin Guide](https://docs.medusajs.com/resources/plugins/guides/wishlist).

You can either:

- [install and use it as a plugin in the Medusa application](#installation);
- or [copy its source files into an existing Medusa application, without using them as a plugin](#copy-into-existing-medusa-application).

## Prerequisites

- [Node.js v20+](https://nodejs.org/en/download)
- [Git CLI](https://git-scm.com/downloads)
- [PostgreSQL](https://www.postgresql.org/download/)

## Installation

1. In your Medusa application, run the following command to install the wishlist plugin:

```bash
yarn add @godscodes/medusa-wishlist-plugin
```

2. Add the plugin to the `plugins` array in `medusa-config.ts`:

```ts
module.exports = defineConfig({
  // ...
  plugins: [
    {
      resolve: "@godscodes/medusa-wishlist-plugin",
      options: {}
    }
  ]
})
```

3. Add the following `admin` configuration in `medusa-config.ts`:

```ts
module.exports = defineConfig({
  // ...
  admin: {
    vite: () => {
      return {
        optimizeDeps: {
          include: ["qs"],
        },
      };
    },
  },
})

```

4. Run the `db:migrate` command to run migrations and sync links:

```bash
npx medusa db:migrate
```

## Copy into Existing Medusa Application

You can also copy the source files into an existing Medusa application, which will add them not as a plugin, but as standard Medusa customizations.

1. Copy the content of the following directories:

- `src/api/store` and `src/api/middlewares.ts`
- `src/link`
- `src/modules/wishlist`
- `src/workflows`

2. Add the Wishlist Module to `medusa-config.ts`:

```ts
module.exports = defineConfig({
  // ...
  modules: [
    {
      resolve: "./src/modules/wishlist"
    },
  ]
})
```

3. Run the `db:migrate` command to run migrations and sync links:

```bash
npx medusa db:migrate
```

## Test it Out

To test out that the plugin is working, you can go to any product page on the Medusa Admin and see a Wishlist section at the top of the page. You can also try importing the [OpenAPI Spec file](https://res.cloudinary.com/dza7lstvk/raw/upload/v1737459635/OpenApi/Wishlist_Postman_gjk7mn.yml) and using the API routes added by this plugin.

## More Resources

- [Medusa Documentatin](https://docs.medusajs.com)
- [OpenAPI Spec file](https://res.cloudinary.com/dza7lstvk/raw/upload/v1737459635/OpenApi/Wishlist_Postman_gjk7mn.yml): Can be imported into tools like Postman to view and send requests to this project's API routes.



---


# Medusa Wishlist Integration (Next.js Starter)
This guide walks you through integrating a **Wishlist** feature in the [Medusa Next.js Starter](https://docs.medusajs.com/resources/nextjs-starter), using **Medusa v2.8.6** and the `@godscodes/medusa-wishlist-plugin`.

## ✅ Prerequisites
Before starting, ensure you have:
- Medusa backend (`v2.8.6`) installed and running.
- The plugin `@godscodes/medusa-wishlist-plugin` installed and configured.
- Next.js Starter from Medusa: https://docs.medusajs.com/resources/nextjs-starter
- Next.js ≥ 15
- Node.js ≥ 17
- `@tanstack/react-query` installed in your Next.js frontend.

Install React Query:
```bash
npm install @tanstack/react-query
# or
yarn add @tanstack/react-query
```

### 1. Create React Query Client
**File**: `lib/react-query-client.ts`
```ts
import { QueryClient } from "@tanstack/react-query"
export const queryClient = new QueryClient()
```

### 2. Setup Query Provider
**File**: `app/providers.tsx`
```tsx
"use client"
import { QueryClientProvider } from "@tanstack/react-query"
import { queryClient } from "@lib/react-query-client"
export default function Providers({ children }: { children: React.ReactNode }) {
  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
}
```

### 3. Wrap `Providers` in `app/layout.tsx`
```tsx
import Providers from "./providers"
export default function RootLayout({ children }) {
  return (
    <html lang="en">
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  )
}
```

### 4. Implement Wishlist Logic
**File**: `lib/data/wishlist.ts`
```ts
"use server"
import { sdk } from "@lib/config"
import { getAuthHeaders } from "@lib/data/cookies"
import medusaError from "@lib/util/medusa-error"
import { revalidateTag } from "next/cache"

type WishlistItem = {
  id: string
  product_variant_id: string
  wishlist_id: string
  created_at: string
  updated_at: string
  product_variant: {
    id: string
    title: string
    sku: string
    product_id: string
    product: {
      id: string
      images?: { url: string }[]
    }
  }
}

type Wishlist = {
  id: string
  customer_id: string
  sales_channel_id: string
  created_at: string
  updated_at: string
  items: WishlistItem[]
}

type WishlistResponse = { wishlist: Wishlist }

export async function getWishlist(): Promise<Wishlist | null> {
  try {
    const result = await sdk.client.fetch("/store/customers/me/wishlists", {
      method: "GET",
      headers: await getAuthHeaders(),
    })
    if (typeof result === "object" && result !== null && "wishlist" in result) {
      return (result as WishlistResponse).wishlist
    }
    return null
  } catch (err: any) {
    if (err?.response?.status === 404 || err?.message?.toLowerCase().includes("no wishlist")) {
      return null
    }
    medusaError(err)
    return null
  }
}

export async function createWishlist(): Promise<Wishlist> {
  try {
    const res = await sdk.client.fetch("/store/customers/me/wishlists", {
      method: "POST",
      headers: await getAuthHeaders(),
    })
    if (typeof res === "object" && res !== null && "wishlist" in res) {
      return (res as WishlistResponse).wishlist
    }
    throw new Error("Failed to create wishlist")
  } catch (err) {
    medusaError(err)
    throw err
  }
}

export async function addToWishlist({ variantId }: { variantId: string }) {
  if (!variantId) throw new Error("Missing variant ID")
  try {
    let wishlist = await getWishlist()
    if (!wishlist) wishlist = await createWishlist()
    await sdk.client.fetch("/store/customers/me/wishlists/items", {
      method: "POST",
      body: { variant_id: variantId },
      headers: await getAuthHeaders(),
    })
    await revalidateTag("wishlist")
  } catch (err) {
    medusaError(err)
    throw err
  }
}

export async function removeFromWishlist(itemId: string): Promise<void> {
  if (!itemId) throw new Error("Missing wishlist item ID")
  try {
    await sdk.client.fetch(`/store/customers/me/wishlists/items/${itemId}`, {
      method: "DELETE",
      headers: await getAuthHeaders(),
    })
    await revalidateTag("wishlist")
  } catch (err) {
    medusaError(err)
    throw err
  }
}

export async function isVariantInWishlist(variantId: string): Promise<boolean> {
  const wishlist = await getWishlist()
  return wishlist?.items?.some((item) => item.product_variant_id === variantId) ?? false
}
```

### 5. Add UI: Product Page Integration
**File**: `src/modules/products/components/product-actions/index.tsx`
```tsx
import { useMutation, useQueryClient } from "@tanstack/react-query"
import { addToWishlist, isVariantInWishlist } from "@lib/data/wishlist"
const [isAdding, setIsAdding] = useState(false)
const [alreadyInWishlist, setAlreadyInWishlist] = useState(false)
const queryClient = useQueryClient()
const { mutateAsync: addToWishlistMutate, isPending: isWishlistPending } = useMutation({
  mutationKey: ["wishlist-add-item"],
  mutationFn: async ({ variantId }: { variantId: string }) => {
    return await addToWishlist({ variantId })
  },
  onSuccess: async () => {
    await queryClient.invalidateQueries({ queryKey: ["wishlist"] })
  },
})
const handleAddToWishlist = async () => {
  if (!selectedVariant?.id) return
  await addToWishlistMutate({ variantId: selectedVariant.id })
  setAlreadyInWishlist(true)
}
useEffect(() => {
  const checkWishlist = async () => {
    if (selectedVariant?.id) {
      const exists = await isVariantInWishlist(selectedVariant.id)
      setAlreadyInWishlist(exists)
    }
  }
  checkWishlist()
}, [selectedVariant?.id])
<Button
  onClick={handleAddToWishlist}
  disabled={!selectedVariant || !!disabled || alreadyInWishlist}
  isLoading={isWishlistPending}
  variant="primary"
  className="w-full h-10"
>
  {alreadyInWishlist ? "Added to Wishlist" : "Add to Wishlist"}
</Button>
```

### 6. Display Wishlist Page
**File**: `app/[countryCode]/(main)/wishlist/page.tsx`
```tsx
"use client"
import { useQuery } from "@tanstack/react-query"
import { getWishlist, removeFromWishlist } from "@lib/data/wishlist"
import Link from "next/link"
import { useState } from "react"
import { queryClient } from "@lib/react-query-client"
type WishlistItem = { id: string; product_variant: { id: string; title: string; sku: string; product: { id: string; images?: { url: string }[] } } }
type Wishlist = { items: WishlistItem[] }
const Wishlist = () => {
  const { data: wishlist, isLoading } = useQuery<Wishlist | null>({ queryKey: ["wishlist"], queryFn: getWishlist })
  const [removingId, setRemovingId] = useState<string | null>(null)
  const handleRemove = async (itemId: string) => {
    setRemovingId(itemId)
    await removeFromWishlist(itemId)
    await queryClient.invalidateQueries({ queryKey: ["wishlist"] })
    setRemovingId(null)
  }
  if (isLoading) return <div className="text-center">Loading your wishlist...</div>
  if (!wishlist || wishlist.items.length === 0)
    return <div className="text-center mt-24"><h2>No items in wishlist</h2></div>
  return (
    <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-6 mt-10">
      {wishlist.items.map((item) => {
        const product = item.product_variant.product
        const imageUrl = product?.images?.[0]?.url || "/placeholder.jpg"
        return (
          <div key={item.id} className="bg-white p-4 rounded shadow">
            <img src={imageUrl} alt={item.product_variant.title} className="w-full h-48 object-cover" />
            <h3 className="mt-2 font-semibold">{item.product_variant.title}</h3>
            <button
              onClick={() => handleRemove(item.id)}
              disabled={removingId === item.id}
              className="mt-3 text-sm text-red-600 border px-4 py-1 rounded hover:bg-red-100"
            >
              {removingId === item.id ? "Removing..." : "Remove"}
            </button>
          </div>
        )
      })}
    </div>
  )
}
export default Wishlist
```

### 7. Add Wishlist to Menu
**File**: `src/modules/layout/components/side-menu/index.tsx`
```tsx
const SideMenuItems = {
  Home: "/",
  Store: "/store",
  Account: "/account",
  Cart: "/cart",
  Wishlist: "/wishlist",
}
```

### 8. Start the Development Server
```bash
yarn dev
```

---

## ✅ Done!
You've successfully added a full-featured wishlist to your Medusa storefront 🎉

---

## 📌 Notes
- Make sure the Medusa wishlist plugin is enabled in your backend.
- Wishlist will auto-create when a user adds their first item.
- UI updates using `@tanstack/react-query`.

---