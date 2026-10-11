# Community Events: Code Guide for the Live Call

Part 1 explains how the code works. Part 2 lists the changes they're most likely to ask for, with exact steps. Part 3 covers questions they might ask.

---

# Part 1: How the code works

## 1.1 The big picture

```
Browser (React pages, "use client")
   │  api("/events?...")            ← lib/api-client.ts: the only way pages talk to the server
   ▼
API routes  app/api/**/route.ts     ← each wrapped in handler() from lib/http.ts
   │  zod checks the input
   │  requireUser / requireAdmin    ← lib/auth.ts reads the JWT
   ▼
Prisma (lib/prisma.ts)  →  SQLite file dev.db
```

**Key idea:** pages never touch the database. They always go through the JSON API, the same API a mobile app would use.

## 1.2 Folder map

| Folder / file | What's in it |
|---|---|
| `app/api/` | The JSON API (one `route.ts` per endpoint) |
| `app/page.tsx` + `app/events-browser.tsx` | Events list with filters |
| `app/events/[id]/` | Event detail page |
| `app/login`, `app/signup` | Both use `components/auth-form.tsx` |
| `app/favourites`, `app/feed` | Pick categories, My Feed |
| `app/admin/` | Admin list, add-event form, `layout.tsx` (server check) |
| `components/` | Event card, card designer, header, remind button, states |
| `components/ui/` | shadcn components (Button, Select, ...). I don't change these much |
| `hooks/use-api.ts` | Loads data from the API, gives `{ data, error, loading, reload }` |
| `hooks/use-reminders.ts` | Remind me logic (optimistic + rollback) |
| `lib/auth.ts` | bcrypt, JWT, `requireUser`, `requireAdmin` |
| `lib/http.ts` | `handler()` error wrapper, `HttpError`, `readJson` |
| `lib/events.ts` | `visibleEventsWhere()` (the filter) and `requireActiveEvent()` |
| `lib/format.ts` | ₹ price and date formatting |
| `lib/constants.ts` | Categories, card themes, stickers, max stickers |
| `prisma/schema.prisma` | Database tables |
| `prisma/seed.ts` | Admin + 15 sample events |
| `proxy.ts` | First admin check, runs before /admin pages |

---

## 1.3 Flow: loading the events page

1. `app/page.tsx` renders `<EventsBrowser />` inside `<Suspense>`.
   - **Why Suspense:** `EventsBrowser` reads the URL (`useSearchParams`), and Next.js 16 requires that to be inside Suspense.
2. `EventsBrowser` reads `category`, `city`, `q` and `page` from the URL.
3. It calls `useApi("/events?" + params)`.
4. `useApi` calls `api()`, which does `fetch("/api/events?...")`.
5. On the server, `app/api/events/route.ts`:
   - `Query.parse(...)` (zod) checks the params. Empty strings become "no filter", and `page` defaults to 1, `limit` to 9.
   - `visibleEventsWhere({ categories, city, q })` builds the `where`.
   - `findMany` (sorted by `startsAt asc`, with `skip`/`take`) and `count` run together using `Promise.all`.
   - It returns `{ items, page, limit, total, totalPages }`.
6. Back in the browser, `EventsBrowser` shows a skeleton, an error, the empty state, or `<EventGrid>` plus `<Pagination>`.
7. `EventGrid` draws an `EventCard` for each event, with a `RemindButton` passed in as `action`.

**Changing a filter:** `setFilters()` updates the URL with `router.replace`. The URL change means `useApi` gets a new path, so it fetches again.

**Search box:** it waits 400 ms after you stop typing before updating the URL (the `setTimeout` inside `useEffect`).

## 1.4 The filter: `lib/events.ts`

```ts
{
  isActive: true,
  startsAt: { gte: new Date() },          // upcoming only
  ...(categories && { category: { in: categories } }),
  ...(city && { city }),
  ...(q && { OR: [{ title: { contains: q } }, { description: { contains: q } }] }),
}
```

- `...(x && {...})` only adds that condition if the filter was given.
- All the conditions are joined with AND, which is why the filters work together.
- This same function is used by **My Feed** (with the user's categories) and by the **cities** list.

## 1.5 Flow: signup and login

1. `components/auth-form.tsx` handles submit, then calls `api("/auth/login", { method: "POST", json: {...} })`.
2. Server, `app/api/auth/login/route.ts`:
   - zod checks the email and password, and lowercases the email.
   - `prisma.user.findUnique({ where: { email } })`
   - `verifyPassword()` uses bcrypt to compare the password.
   - If either the user or the password is wrong, it returns the same message, "Incorrect email or password". That way nobody can find out which emails have accounts.
   - `signToken()` creates the JWT (HS256, 7 days, containing userId, email and role).
   - It returns `{ token, user }` **and** sets an httpOnly cookie.
3. Back in the browser: `setUser(user)` → reset the form → `router.push(...)`:
   - after signup → `/favourites`
   - an admin → `/admin`
   - otherwise → `?next=` (if present) or `/`

Signup is the same, plus it checks for a duplicate email (409) and hashes the password.

## 1.6 How the server knows who you are: `lib/auth.ts`

```
getSession(req):
  token = "Authorization: Bearer xxx" header   (mobile)
       OR the "token" cookie                    (website)
  → verifyToken(token) → { userId, email, role } or null

requireUser(req)  → no session? throw HttpError(401)
requireAdmin(req) → requireUser, then role !== ADMIN? throw HttpError(403)
```

Every protected route starts with `await requireUser(req)` or `await requireAdmin(req)`.

## 1.7 Error handling: `lib/http.ts`

Every route is wrapped like `export const GET = handler(async (req) => {...})`.

`handler()` catches anything thrown and turns it into JSON:

| Thrown | Response |
|---|---|
| `HttpError(404, "Event not found")` | 404 `{ error: "Event not found" }` |
| `ZodError` (bad input) | 400 with the first validation message |
| anything else | 500 "Something went wrong" (and logs it) |
| Next.js internal signals | re-thrown with `unstable_rethrow`, not caught |

So a route just `throw`s and never builds error responses itself.

## 1.8 Flow: Remind me (`hooks/use-reminders.ts`)

1. On load, if logged in, it fetches `GET /api/me/reminders`, so it knows which events already have a reminder.
2. When the button is clicked, `toggle(eventId)` runs:
   - not logged in → toast, then go to `/login?next=<this page>`
   - already saving this event → ignore (stops double clicks)
   - `wasOn = reminded.has(eventId)`
   - **Step 1:** `setReminded(...!wasOn)`, so the screen changes immediately
   - **Step 2:** `api(... { method: wasOn ? "DELETE" : "PUT" })`
   - **Step 3, if it fails:** `setReminded(...wasOn)` (put it back) and `toast.error(...)`
3. Server, `app/api/events/[id]/reminder/route.ts`:
   - PUT → `requireActiveEvent`, then `upsert` (safe to call twice)
   - DELETE → `deleteMany` (safe to call twice)

## 1.9 Flow: My Feed

- `app/feed/page.tsx` calls `useRequireUser()` (which redirects to login if needed), then `useApi("/feed?page=N")`.
- `app/api/feed/route.ts` loads the user's favourite categories:
  - none → it returns an empty list plus `categories: []`, and the page says "pick favourites"
  - otherwise → `visibleEventsWhere({ categories })`
- Favourites are saved by `PUT /api/me/favourites`, which deletes the old ones and creates the new ones inside one `$transaction`.

## 1.10 Flow: card designer (`components/card-designer.tsx`)

1. `CardDesigner` loads `GET /api/events/:id/card`. If nothing is saved, the API returns the default `{ theme: "sunset", stickers: [] }`.
2. Once loaded, it mounts `<Designer initial={...}>`, which starts its state from the saved design.
3. Clicking a theme or a sticker updates state (so the card changes live) and calls `scheduleSave()`.
4. `scheduleSave` waits 600 ms (`SAVE_DELAY_MS`) and then does `PUT /api/events/:id/card` with the whole design.
5. The server (`app/api/events/[id]/card/route.ts`) checks with zod: a known theme, known stickers, at most `MAX_STICKERS`, no duplicates. Then it upserts. Stickers are stored as a JSON string.

**Animation:**
- **Themes:** all 3 gradients are stacked and only their `opacity` changes (`transition-opacity duration-500`).
- **Stickers:** `animate-in zoom-in-50` when added. When removed, `animate-out` plays for 200 ms, then the sticker is taken out of the list.
- **Sticker positions:** the `SLOTS` array holds the 1st, 2nd and 3rd positions on the card.

## 1.11 Flow: admin

**Protection: three layers**
1. `proxy.ts` runs before any `/admin/*` page. It verifies the cookie's JWT and redirects anyone who isn't the admin.
2. `app/admin/layout.tsx` (server component): `AdminOnly` reads the cookie, verifies it, and calls `redirect()` if that fails.
3. Every `/api/admin/*` route calls `requireAdmin(req)`.

**Toggle active:** in `app/admin/page.tsx`, `AdminEventRow.toggle()` follows the same optimistic pattern as Remind me, using `PATCH /api/admin/events/:id { isActive }`.

**Add event:** `app/admin/new/page.tsx` → `POST /api/admin/events`.
- The price is typed in ₹ and sent as paise: `Math.round(price * 100)`.
- The date picker gives "2026-10-18T18:30", and we add `:00+05:30` so it's treated as IST.
- Server: the zod `NewEvent` schema checks everything (future date, http(s) image URL, ...), then `prisma.event.create`.

## 1.12 Next.js 16 specifics (they may ask "why is this here?")

| Thing | Why |
|---|---|
| `proxy.ts` | In Next 16, "middleware" was renamed to "proxy" |
| `await ctx.params` / `use(params)` | Route params are a Promise in Next 16 |
| `<Suspense>` around URL/cookie readers | Needed because `cacheComponents: true` is on |
| `finally { setSaving(false) }`, `formEl.reset()` | Next 16 keeps visited pages alive and hidden (React `<Activity>`), so old state would still be there when you come back |
| Logout uses `window.location.href = "/"` | A full reload clears every hidden page, so no user data stays in memory |
| `unstable_rethrow(err)` in `handler()` | Lets Next.js's internal signals pass through |

## 1.13 Database (`prisma/schema.prisma`)

- `User`: email is unique; `role` is "USER" or "ADMIN".
- `Event`: has `isActive` instead of being deleted, and `pricePaise` as an integer.
- `FavouriteCategory`, `Reminder` and `CardDesign` have a **composite primary key** `@@id([userId, ...])`, so there's one row per pair.
- Indexes: `@@index([isActive, startsAt])` and `@@index([category, city, startsAt])`.

---

# Part 2: Likely change requests

After any change, check it with:
```bash
npx tsc --noEmit        # type errors
npm run lint            # lint
npm run dev             # try it in the browser
```

## 1. "Add a new category: Art" (very likely)

1. `lib/constants.ts`: add it to the list:
   ```ts
   export const CATEGORIES = ["Music", "Workshop", "Food", "Sports", "Kids", "Art"] as const;
   ```
2. `components/event-card.tsx`: give it a badge colour in `CATEGORY_STYLES`:
   ```ts
   Art: "bg-rose-100 text-rose-700",
   ```
3. `app/favourites/page.tsx`: give it an emoji in `CATEGORY_EMOJI`:
   ```ts
   Art: "🎨",
   ```

That's it. The API validation, the filter dropdown, the admin form and the favourites page all read from `CATEGORIES`. **No database change is needed**, because category is a plain string.

## 2. "Show 6 events per page instead of 9" (also makes pagination visible)

- `app/api/events/route.ts`: in `Query`, change `.default(9)` to `.default(6)` on `limit`.
- `app/api/feed/route.ts`: make the same change.

## 3. "Allow 4 stickers instead of 3"

1. `lib/constants.ts`: `export const MAX_STICKERS = 4;`
2. `components/card-designer.tsx`: add a 4th position to `SLOTS`:
   ```ts
   const SLOTS = [..., "bottom-4 left-1/2 rotate-3"];
   ```

The API limit and the "3/3" counter both use `MAX_STICKERS`, so they update automatically.

## 4. "Add a new sticker" (e.g. 🎸)

1. `lib/constants.ts`: add `"guitar"` to `STICKERS`.
2. `components/card-designer.tsx`: add it to `STICKER_EMOJI`:
   ```ts
   guitar: { emoji: "🎸", label: "Guitar" },
   ```

## 5. "Add a fourth card theme"

1. `lib/constants.ts`: add `"night"` to `CARD_THEMES`.
2. `components/card-designer.tsx`: add it to `THEMES`:
   ```ts
   night: { label: "Night", gradient: "from-slate-700 via-indigo-900 to-black" },
   ```

## 6. "Search should also match the city"

`lib/events.ts`: add the city to the `OR`:
```ts
...(q && { OR: [{ title: { contains: q } }, { description: { contains: q } }, { city: { contains: q } }] }),
```

## 7. "Add a max price filter"

1. `app/api/events/route.ts`, in `Query`:
   ```ts
   maxPrice: z.coerce.number().min(0).optional(),
   ```
   Then pass `maxPrice` into `visibleEventsWhere(...)`.
2. `lib/events.ts`: add `maxPrice?: number` to the filters, and:
   ```ts
   ...(maxPrice !== undefined && { pricePaise: { lte: maxPrice * 100 } }),
   ```
3. `app/events-browser.tsx`: add an input that calls `setFilters({ maxPrice: value })`.

## 8. "Sort by price instead of date"

`app/api/events/route.ts`: `orderBy: { pricePaise: "asc" }`. Or keep date as a tiebreaker:
`orderBy: [{ pricePaise: "asc" }, { startsAt: "asc" }]`

## 9. "Change the date format" (e.g. add the year)

`lib/format.ts`: add `year: "numeric"` to `dateParts`, and use `parts.year` in the returned string:
```ts
return `${parts.weekday}, ${parts.day} ${month} ${parts.year}, ${parts.hour}:${parts.minute} ${parts.dayPeriod.toUpperCase()}`;
```

## 10. "Show free events as ₹0 instead of 'Free'"

`lib/format.ts`, `formatPrice`: remove the `paise === 0 ? "Free" :` part.

## 11. "Add a new field to events" (e.g. venue). The biggest change, so practise this one.

1. `prisma/schema.prisma`, in `model Event`: `venue String @default("")`
2. Run `npx prisma migrate dev --name add_venue`. This creates the migration and regenerates the client.
3. `prisma/seed.ts`: add `venue: "..."` to the events (optional).
4. `app/api/admin/events/route.ts`, in `NewEvent`: `venue: z.string().trim().max(120)`
5. `app/admin/new/page.tsx`: add a `<Field label="Venue">` input, and `venue: form.get("venue")` in the JSON.
6. `lib/types.ts`: add `venue: string` to `EventItem`.
7. `app/events/[id]/event-detail.tsx`: show it, e.g. another `<Fact ... />`.

## 12. "Change the minimum password length to 10"

- `app/api/auth/signup/route.ts`: `.min(10, "Password must be at least 10 characters")`
- `components/auth-form.tsx`: `minLength={isLogin ? undefined : 10}`, and update the hint text.

## 13. "Make login tokens last 1 day"

`lib/auth.ts`: `const TOKEN_TTL_SECONDS = 60 * 60 * 24;`. The cookie's `maxAge` uses the same constant.

## 14. "After login, send users with no favourites to the favourites page"

`components/auth-form.tsx`, after `setUser(user)` for a normal login:
```ts
if (isLogin && user.role !== "ADMIN" && !next) {
  const { categories } = await api<{ categories: string[] }>("/me/favourites");
  if (categories.length === 0) return router.push("/favourites");
}
```

## 15. "Don't allow reminders for past events" (on the server)

`lib/events.ts`, `requireActiveEvent`: also select `startsAt`, and throw if it's in the past:
```ts
const event = await prisma.event.findUnique({ where: { id }, select: { id: true, isActive: true, startsAt: true } });
if (!event || !event.isActive) throw new HttpError(404, "Event not found");
if (event.startsAt < new Date()) throw new HttpError(400, "This event has already happened");
```
Careful: this also blocks the card designer for past events, since it uses the same function.

## 16. "Change the search delay / card save delay"

- Search: `app/events-browser.tsx`, the `400` in `setTimeout`.
- Card: `components/card-designer.tsx`, `SAVE_DELAY_MS`.

## 17. "Change the main colour"

`app/globals.css`: change `--primary` (and `--ring`) in `:root`. For example, `#2563eb` for blue.

## 18. "Show how many people set a reminder for an event"

1. `app/api/events/[id]/route.ts`:
   ```ts
   const reminderCount = await prisma.reminder.count({ where: { eventId: id } });
   return Response.json({ event, reminderCount });
   ```
2. `app/events/[id]/event-detail.tsx`: change the type to `{ event: EventItem; reminderCount: number }` and show `data.reminderCount`.

---

# Part 3: Questions they might ask

**Why store the token in a cookie and also return it in the body?**
The website uses the httpOnly cookie, which JavaScript can't read, so an XSS attack can't steal it. A mobile app can't easily use cookies, so it takes the token from the body and sends it as `Authorization: Bearer`.

**Why PUT/DELETE for reminders instead of one toggle endpoint?**
If a toggle request is retried, it flips the state back. PUT always means on and DELETE always means off, so repeating either is harmless.

**What happens if two requests to save the card arrive out of order?**
Each save sends the whole design, and `latestRequest` makes sure only the newest request updates the status on screen.

**Why is the price in paise?**
Floats can't store money exactly (`0.1 + 0.2 = 0.30000000000000004`). Whole paise are always exact.

**Why does `/api/events/:id` return 404 for an inactive event instead of 403?**
So it doesn't reveal that the event exists.

**What's `useApi`'s "key" for?**
Each request gets a key (`path#attempt`). If the latest result belongs to an older key, we're still loading. That avoids calling setState inside the effect (a React lint rule), and old responses can't overwrite new ones.

**Why `Promise.all` for `findMany` and `count`?**
The two queries run at the same time instead of one after the other.

**Is the admin check in the UI enough?**
No. Hiding the Admin link is only cosmetic. Security comes from the three server checks: proxy, layout and API.

**What would you improve?**
Tests, actually sending reminder notifications, rate limiting on login, and full-text search instead of `LIKE`.

---

# Tips for the live call

- Before changing anything, **say which file you'll change and why**. That shows you know the code.
- Change `lib/constants.ts` first when you can. A lot of the app reads from it.
- After a change, run `npx tsc --noEmit`. It tells you every place that needs updating.
- If the database schema changes, always run `npx prisma migrate dev --name <name>`.
- If the dev server acts strangely after big changes, restart `npm run dev`.
