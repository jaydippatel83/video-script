# Community Events Board: Video Script

Target length: about 4:00 (the brief asks for 3 to 4 minutes)

## Before recording

- [ ] Run `npx prisma db seed` so the data is fresh
- [ ] Chrome window 1: DevTools device mode, iPhone / 375px wide
- [ ] Chrome window 2: laptop size
- [ ] VS Code tabs open:
  - `hooks/use-reminders.ts`
  - `proxy.ts`
  - `app/admin/layout.tsx`
  - `lib/auth.ts` (scroll to `requireAdmin`)
  - `prisma/schema.prisma`
- [ ] Editor font zoomed in (Cmd +) so code is readable
- [ ] Skip pagination: only 1 page shows with the sample data

---

## 0:00 – 0:15 · Intro

**Show:** events page on the laptop

> Hi, I'm Jaydip. This is my Community Events Board. It's built with Next.js App Router, SQLite with Prisma, and Tailwind with shadcn. I'll show it on a phone screen first, then on a laptop, and then I'll go through a few choices I made in the code.

---

## 0:15 – 1:50 · Phone

**Show:** DevTools phone size, events page

> This is the events page. It only shows active events that haven't happened yet, and the nearest date comes first.

**Do:** type "night", pick Music, then Coimbatore

> The filters work together. Music, Coimbatore and "night" gives just these two events. The filters are saved in the URL, so if I refresh they stay.

**Do:** refresh, point at a price and a date

> Prices use Indian format and dates are shown like "Tue, 13 Oct, 6:30 PM".

**Do:** Log in → Sign up with a new email

> Now I'll sign up. The API returns a JWT. After signing up, I'm asked to pick my favourite categories.

**Do:** pick Music and Food → Save

> This is My Feed. It only shows upcoming events from Music and Food.

**Do:** tap Remind me on an event

> The Remind me button changes straight away. If I refresh, it's still set, because it was saved.

**Do:** refresh → DevTools → Network → **Offline** → tap Remind me on another event

> Now I'll go offline to make saving fail. The button changes, then goes back, and shows an error message.

**Do:** set Network back to Online → open an event → scroll to "Design your card"

> On the event page there's the card designer. I can switch between three backgrounds, and it fades smoothly. I can add up to three stickers. After three the rest are disabled, and I can tap one on the card to remove it. It saves automatically. If I refresh, my design is still there, and it's saved separately for each event.

**Do:** switch themes → add 3 stickers → remove one → wait for "Saved" → refresh

---

## 1:50 – 2:35 · Laptop

**Show:** laptop window, events grid

> On a laptop the same page shows three columns.

**Do:** log in as admin (`admin@events.test` / `Admin@123`) → Admin page

> This is the admin page. It lists every event, including inactive and past ones. I never delete events, I only make them inactive.

**Do:** switch one event to Inactive → go to Events → show it's gone

> Once it's inactive, visitors can't see it.

**Do:** Add event → fill the form, price `150000` → submit

> The admin can add an event with all the fields. The price shows as one lakh fifty thousand rupees, in Indian format.

---

## 2:35 – 3:50 · Three choices in my code

### Choice 1: Remind me button

**Show:** `hooks/use-reminders.ts`

> For Remind me, I update the screen first and then save. If the save fails, I set the button back and show a message. I use PUT to turn it on and DELETE to turn it off, instead of one toggle. That way, if a request is sent twice it doesn't flip back by mistake.

### Choice 2: Admin protection

**Show:** `proxy.ts` → `app/admin/layout.tsx` → `requireAdmin` in `lib/auth.ts`

> The admin side is protected on the server in three places. The proxy checks the token before any admin page loads. The admin layout checks it again on the server. And every admin API checks the role too. Hiding the Admin link in the UI isn't security on its own; the server is what actually blocks access.

### Choice 3: Database

**Show:** `prisma/schema.prisma`

> For reminders, favourites and card designs, the primary key is the user plus the event. So the database itself makes sure a user can't have two reminders for the same event. I store prices in paise as whole numbers to avoid rounding errors, and I added an index on category, city and date because that's exactly what the filter uses.

---

## 3:50 – 4:00 · Close

> That's it. Setup steps, the API list, and what I'd do next are in the README. Thanks for watching.

---

## Optional extras (only if going to 5 minutes)

**+30s · API works for mobile:** in a terminal, run:

```bash
curl -s -H "Content-Type: application/json" \
  -d '{"email":"admin@events.test","password":"Admin@123"}' \
  http://localhost:3000/api/auth/login
```

> The same API works for a mobile app. It can send this token as a Bearer header instead of a cookie.

**+30s · Empty state:** open My Feed for a user with no favourites

> If the feed is empty, it shows a friendly message instead of a blank page.

---

## Tips

- Speak slowly. Say it your own way rather than reading word for word.
- The code part matters most. If you're running long, cut from the demo, not from the explanations.
- Do one practice run before the real recording.
