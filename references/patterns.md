## 1. Custom API Client with `createUseFetch` (Official Nuxt 4 Recipe)

The preferred pattern for authenticated external API calls. Source: https://nuxt.com/docs/4.x/guide/recipes/custom-usefetch

```ts
// app/composables/useAPI.ts
export const useAPI = createUseFetch({
  baseURL: useRuntimeConfig().public.apiBase,
  onRequest({ options }) {
    const { session } = useUserSession();
    if (session.value?.token) {
      options.headers.set("Authorization", `Bearer ${session.value.token}`);
    }
  },
  async onResponseError({ response }) {
    if (response.status === 401) {
      await navigateTo("/login");
    }
  },
});

// Usage — typed, auth-handled, no boilerplate:
// const { data: profile } = await useAPI<Profile>('/me')
// const { data: orders } = await useAPI<Order[]>('/orders')
```

---

## 2. Custom `$fetch` Plugin (Lower-level control)

Use when you need the raw instance (e.g. for Repository pattern or non-composable contexts).

```ts
// app/plugins/api.ts
export default defineNuxtPlugin((nuxtApp) => {
  const { session } = useUserSession();

  const api = $fetch.create({
    baseURL: useRuntimeConfig().public.apiBase,
    onRequest({ options }) {
      if (session.value?.token) {
        options.headers.set("Authorization", `Bearer ${session.value.token}`);
      }
    },
    async onResponseError({ response }) {
      if (response.status === 401) {
        await nuxtApp.runWithContext(() => navigateTo("/login"));
      }
    },
  });

  return { provide: { api } };
});

// Usage — always wrap with useAsyncData to avoid SSR double-fetch:
// const { $api } = useNuxtApp()
// const { data } = await useAsyncData('users', () => $api<User[]>('/users'))
```

---

## 3. Prisma Singleton (`server/utils/db.ts`)

```ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const db =
  globalForPrisma.prisma ??
  new PrismaClient({
    log:
      process.env.NODE_ENV === "development"
        ? ["query", "error", "warn"]
        : ["error"],
  });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = db;
```

---

## 4. Typed Nitro Route Skeleton

```ts
// server/api/widgets/[id].get.ts
import { db } from "~/server/utils/db";
import type { Widget } from "@prisma/client";

type WidgetResponse = Pick<Widget, "id" | "name" | "createdAt">;

export default defineEventHandler(
  async (event): Promise<{ data: WidgetResponse }> => {
    const { id } = getRouterParams(event);
    if (!id) throw createError({ statusCode: 400, message: "Missing id" });

    const widget = await db.widget.findUnique({
      where: { id },
      select: { id: true, name: true, createdAt: true }, // precise — never select *
    });

    if (!widget)
      throw createError({ statusCode: 404, message: "Widget not found" });

    return { data: widget };
  },
);
```

---

## 5. Composable Contract Template

```ts
// composables/useFetchUserOrders.ts
import type { MaybeRefOrGetter } from "vue";
import type { Order } from "@prisma/client";

// Pure helpers outside composable function to avoid closure memory leaks
type OrderSummary = Pick<Order, "id" | "status" | "total" | "createdAt">;

export function useFetchUserOrders(userId: MaybeRefOrGetter<string>) {
  // State
  const {
    data: orders,
    status,
    error,
    refresh,
  } = useAsyncData(
    () => `orders-${toValue(userId)}`,
    () => $fetch<OrderSummary[]>(`/api/users/${toValue(userId)}/orders`),
    { watch: [() => toValue(userId)] },
  );

  // onUnmounted cleanup if needed (listeners, timers, etc.)

  return {
    orders: readonly(orders),
    status,
    error: readonly(error),
    refresh,
  };
}
```

---

## 6. Hydration-Safe Patterns

```ts
// 🚩 Wrong — localStorage crashes on server
const theme = localStorage.getItem("theme") || "light";

// ✅ Right — SSR-safe cookie
const theme = useCookie("theme", { default: () => "light" });

// 🚩 Wrong — random/time values differ between server and client
const rand = Math.random();

// ✅ Right — shared via useState
const rand = useState("rand", () => Math.random());

// 🚩 Wrong — browser lib in setup()
SomeBrowserLib.init();

// ✅ Right — deferred to client mount
onMounted(async () => {
  const { default: SomeBrowserLib } = await import("browser-only-lib");
  SomeBrowserLib.init();
});
```

---

## 7. Hybrid Rendering Route Rules

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  routeRules: {
    "/": { prerender: true }, // Generated at build time
    "/products/**": { swr: 3600 }, // Cached, revalidates in background
    "/blog": { isr: 3600 }, // Regenerated on demand after 1hr
    "/admin/**": { ssr: false }, // Client-only SPA
  },
});
```

---

## 8. Nuxt Layer Folder Structure

```
layers/
  shared/
    composables/
    components/
    utils/
    types/
  auth/
    composables/
    components/
    server/api/
    server/utils/
  products/
    composables/
    components/
    server/api/
    server/utils/
nuxt.config.ts   ← root orchestrator only; never contains feature logic
```

---

## 9. Pinia Store (Global State Only)

```ts
// stores/useAuthStore.ts
export const useAuthStore = defineStore("auth", () => {
  const user = ref<User | null>(null);
  const isAuthenticated = computed(() => !!user.value);

  async function login(credentials: LoginCredentials) {
    user.value = await $fetch("/api/auth/login", {
      method: "POST",
      body: credentials,
    });
  }

  function logout() {
    user.value = null;
  }

  return { user: readonly(user), isAuthenticated, login, logout };
});
```
