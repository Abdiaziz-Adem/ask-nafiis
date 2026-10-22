# Ask Nafiis — hotel integration kit

The customer-facing **Ask Nafiis** chat feature extracted from the hotel project. This is an **integration kit for an existing Next.js App Router application**, not a complete hotel website or independently runnable app.

## Included

- `components/ui/hotel-chat.tsx`: floating chat panel, suggested questions, loading/error states, and new-conversation control.
- `app/api/chat/route.ts`: same-origin API bridge with signed HttpOnly session cookies, request-size bounds, per-process request/concurrency limits and server-only n8n calls.
- `lib/chat/answers.ts` and `quick-answer.ts`: deterministic question routing and fast public database answers with a 30-second cache and four-second read timeout.
- Small UI/server support files, a chat color theme, empty environment template, and the original chatbot verification scripts.

Source: [hotel chatbot commit 94b6bd9](https://github.com/malikyakub/nafiis-hotel/commit/94b6bd947d7e671e0094426d1efb2d377935eafa). The core widget, API and answer modules retain that implementation. To separate it from the rest of the hotel project, the HTTP helper defines its own error class and public fallback contacts are replaced with examples. No hotel account/role code, database records, credentials, booking/payment backend, or blockchain code is included.

## Integrate with your Next.js application

1. Use a compatible Next.js 16 / React 19 App Router application with TypeScript, Tailwind CSS v4 and an `@/*` alias pointing to the application root. The source uses Next.js 16.3.8, React 19.2.8, `@supabase/supabase-js` 2.117.2, `lucide-react` 1.49.0 and `server-only` 0.0.1. Install/pin compatible dependencies in your host application's package manifest and lockfile. TypeScript is needed for the isolated verification script.
2. Copy the `app/api/chat`, `lib/chat`, and widget files to the matching paths. Add the supporting files if your host does not already have them; review/merge existing `lib/http.ts`, branding and shared UI files instead of overwriting them.
3. Import and mount the widget inside your existing root layout's `<body>`:

   ```tsx
   import { HotelChat } from "@/components/ui/hotel-chat";

   // Inside the existing body, alongside the host application's content:
   <HotelChat />
   ```

4. Merge `chat-theme.css` into the host's global Tailwind v4 stylesheet after its Tailwind import, or map the named colors to your existing theme. Ensure Tailwind scans `components/ui`. This theme supplies color tokens, not the full hotel's design system.
5. Copy `.env.example` values into your host's ignored `.env.local`. Set your Supabase public URL/key. Set `SITE_URL` and a random `HOTEL_CHAT_SESSION_SECRET` in production. Keep the webhook URL, session secret and optional Basic Auth value server-only.
6. Replace **all** example fields in `lib/brand.ts`. Review the fixed hotel guidance in `quick-answer.ts` before presenting it to customers. The included guidance describes the original project's booking and payment review steps; adapt it to your application.
7. Update the `/#book` links if your booking form has a different URL. The widget is hidden on `/admin`, `/reception`, `/restaurant`, and `/components` routes by default.

## Public database contract

The read-only anonymous client expects these existing tables and columns:

| Table | Selected columns | Filters |
| --- | --- | --- |
| `room_types` | `name, booking_code, base_price, capacity` | `is_active = true` |
| `food_menu` | `name, category, price` | `is_available = true` |
| `hotel_settings` | `name, address, phone, email` | First row |

This kit does not provision a database. Configure your own schema and RLS so only intended public hotel information is readable by the anonymous role. Do not give the chatbot a service-role/secret key. Questions never become arbitrary SQL, table names or filters. No booking/customer/payment records are queried.

English room, menu and contact questions can bypass AI. Booking/payment/policy guidance uses fixed text. Other wording and languages fall back to n8n. Database failures return an availability message rather than stale prices; actual room availability must be checked in your booking flow.

## Connect the n8n automation

Import [`../workflows/ai-chatbot.json`](../workflows/ai-chatbot.json), connect your Ollama credential and customize the AI Agent system prompt with approved hotel facts. The supplied workflow starts inactive with public chat disabled.

Configure its Chat Trigger for **Embedded Chat**, enable its chat endpoint when ready, set authentication, then publish/activate it. Copy the production **Chat URL** into `HOTEL_CHAT_WEBHOOK_URL`; use a URL reachable from your Next.js server. Match `HOTEL_CHAT_BASIC_AUTH` to the trigger's Basic Auth credential if used. A hosted Next.js server cannot reach n8n through your laptop's `localhost`.

The API sends `{ action: "sendMessage", sessionId, chatInput }` and expects an `output` or `text` string in the response. Routine database answers do not require n8n. The Ask Nafiis workflow must be given approved hotel facts for hotel-specific AI replies.

## Verification

From your host application's root after copying the files and scripts:

```sh
node contracts/scripts/check-chat-cache.mjs
CHAT_TEST_ORIGIN=http://localhost:3000 node contracts/scripts/check-chat.mjs
CHAT_TEST_ORIGIN=http://localhost:3000 node --env-file=.env.local contracts/scripts/check-chat-fast.mjs
```

On PowerShell, set `$env:CHAT_TEST_ORIGIN = 'http://localhost:3000'` before running the corresponding `node` command. The cache script runs isolated tests without network calls; the other scripts require the host server, and the fast-answer script also needs a configured public database with the expected sample room/menu data.

The extracted code was typechecked and the existing cache/routing tests were run. Live database/AI checks must be repeated against your own configured host. Earlier original-project timing results do not guarantee performance on another installation.

## Runtime boundaries

The session and public-data caches are in process. Use a shared gateway/limiter for multiple instances and protect the direct n8n endpoint too. AI calls can take up to the configured timeout and depend on Ollama hardware. New chat rotates the conversation ID; it does not delete n8n execution history. Configure chat retention and access in your own deployment.

This repository publishes source code and a workflow. It does not deploy the hotel chatbot or alter the existing hotel's website.
