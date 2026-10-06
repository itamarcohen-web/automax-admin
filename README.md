# AutoMax Admin

קונסולת הניהול של AutoMax – `https://admin.auto-max.co.il`.
זו שכבת Admin בלבד מעל פרויקט ה‑Supabase הקיים: לא משנה את האפליקציה, את האתר או את מערכת הרישוי, ומשתמשת בפונקציות הקיימות
(`apply_payment`, `grant_talent`, `revoke_talent`, `publish_release`, `publish_legal_document`, `set_registration_status` …).
מבנה, החלטות וההבדלים מול הסכימה: [ARCHITECTURE.md](ARCHITECTURE.md). ביקורת אבטחה: [SECURITY_AUDIT.md](SECURITY_AUDIT.md).

```
דפדפן (Next.js סטטי, מפתח ציבורי בלבד) ── Supabase Auth (+TOTP) ──► Edge Function admin-api ──► public.admin_* (SQL) ──► פונקציות קיימות
                                                                      │ JWT → admin_users → MFA → rate limit → ולידציה → תפקיד
                                                                      └──────────────────────────────────────► admin_audit_log (באותה טרנזקציה)
```

## מה בפנים

| תיקייה | תוכן |
|---|---|
| `src/` | הממשק: Next.js 16 (ייצוא סטטי), TypeScript, Tailwind 4, רכיבים בסגנון shadcn/ui (Radix), Lucide |
| `supabase/migrations/` | 5 מיגרציות (admin_users, audit, rate limit, פעולות, קריאות, אינדקסים) |
| `supabase/functions/admin-api/` | ה‑Edge Function היחידה (`core.ts` = הצינור, `actions.ts` = רשימת הפעולות המותרת) |
| `supabase/manual/` | `add_first_admin.sql`, `verify_admin_security.sql` |
| `supabase/rollback/` | `rollback_admin.sql` |
| `tests/` | `edge/` (Edge Function), `frontend/`, `security/`, `sql/` (Postgres אמיתי), `e2e/mock-supabase.mjs` (שרת דמה) |
| `scripts/` | `postbuild.mjs` (כותרות אבטחה + CSP), `scan-secrets.mjs`, `serve-out.mjs`, `demo.mjs` |

## 1. הרצה מקומית

דרישות: Node.js 20 ומעלה (נבדק על 24). Python רק להרצת בדיקות ה‑SQL.

```bash
npm install
```

**לנסות הכול עם נתוני דמה** (בלי Supabase ובלי לגעת בשום דבר אמיתי):

```bash
npm run demo
```

נפתח `http://localhost:3000`. התחברות: `owner@auto-max.co.il` / `Mock-Passw0rd` (super_admin). עוד חשבונות: `ops@` (admin), `help@` (support),
`mfa@` (קוד TOTP `123456`), `customer@example.com` (חשבון תקין שאינו Admin – נדחה).

**מול הפרויקט האמיתי:** העתיקו `.env.example` ל‑`.env.local`, מלאו `NEXT_PUBLIC_SUPABASE_URL` ו‑`NEXT_PUBLIC_SUPABASE_ANON_KEY` (המפתח **הציבורי** `sb_publishable_…`), ואז:

```bash
npm run dev
```

בפיתוח מקומי הוסיפו `http://localhost:3000` ל‑`ADMIN_ALLOWED_ORIGINS` של ה‑Edge Function (ראו סעיף 3) והסירו אותו בסיום.

בדיקות:

```bash
npm run check      # typecheck + lint + 114 בדיקות vitest + סריקת סודות
npm run test:sql   # 181 בדיקות על PostgreSQL אמיתי (pip install pgserver "psycopg[binary]")
npm run build      # בנייה לפרודקשן ל‑out/
```

## 2. הגדרת Supabase (פעם אחת)

1. **הסכימה הקיימת** חייבת להיות מותקנת (`AutoMax-Server-Setup.sql`). המיגרציות בודקות זאת ועוצרות אם חסר.
2. **מיגרציות** – ב‑Supabase → SQL Editor, להריץ לפי הסדר (כל אחת בטוחה להרצה חוזרת):
   `20261006000100_admin_core.sql` → `…0200_account_controls.sql` → `…0300_admin_actions.sql` → `…0400_admin_reads.sql` → `…0500_admin_indexes.sql`.
   אפשר גם `npx supabase link --project-ref <project-ref>` ואז `npx supabase db push`.
   `<project-ref>` הוא החלק הראשון של כתובת הפרויקט (`https://<project-ref>.supabase.co`).
3. **אימות** – להריץ את `supabase/manual/verify_admin_security.sql`. כל שאילתה מציינת מעליה את התוצאה הנדרשת. אם אחת שונה – לעצור.
4. **Auth** (Dashboard → Authentication):
   * *Sign In / Providers → Multi Factor*: ודאו ש‑TOTP פעיל (Enroll + Verify).
   * *URL Configuration*: ה‑Site URL ו‑Redirect URLs לפי האתר שלכם (ל"איפוס סיסמה" שהקונסולה שולחת).
   * *Attack Protection*: מומלץ להפעיל CAPTCHA (Cloudflare Turnstile – חינם) נגד ניחוש סיסמאות. *Rate Limits*: להשאיר ברירת מחדל או להקשיח.
   * *Password requirements*: מינימום 10 תווים. *SMTP*: שולח משלכם (מייל איפוס הסיסמה יוצא ממנו).
5. **Storage**: ה‑bucket `releases` קיים כבר (נוצר ב‑`schema.sql`). ב‑Free יש מגבלה של 50MB לקובץ.

## 3. משתני סביבה

| משתנה | איפה | משמעות |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | build של הממשק | כתובת הפרויקט – ציבורי |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | build של הממשק | **מפתח ציבורי בלבד** (`sb_publishable_…`). הקוד מסרב לרוץ עם `service_role`/`sb_secret_` |
| `NEXT_PUBLIC_IDLE_TIMEOUT_MINUTES` | build | התנתקות אחרי חוסר פעילות (ברירת מחדל 30) |
| `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` | Edge Function | **מוזרקים אוטומטית** על ידי Supabase. לעולם לא בדפדפן/Git |
| `ADMIN_SERVICE_KEY` | Edge Function (אופציונלי) | מפתח `sb_secret_…` במקום ה‑service_role הישן |
| `ADMIN_ALLOWED_ORIGINS` | Edge Function | מקורות מותרים ל‑CORS, ברירת מחדל `https://admin.auto-max.co.il` |
| `ADMIN_REQUIRE_MFA` | Edge Function | `true` = כל Admin חייב TOTP (מומלץ אחרי שהגדרתם את שלכם) |
| `PASSWORD_RESET_REDIRECT` | Edge Function | לאן מגיע הלינק במייל "איפוס סיסמה" |
| `RELEASE_MAX_BYTES` | Edge Function | מגבלת ההעלאה של התוכנית שלכם (ברירת מחדל 52428800) |

פריסת הפונקציה והגדרת סודות:

```bash
npx supabase login
npx supabase link --project-ref <project-ref>
npx supabase secrets set ADMIN_ALLOWED_ORIGINS=https://admin.auto-max.co.il PASSWORD_RESET_REDIRECT=https://auto-max.co.il/account
npx supabase functions deploy admin-api
```

## 4. יצירת ה‑Admin הראשון

1. ודאו שיש חשבון **מאושר** עם האימייל שלכם (Authentication → Users → Add user עם "Auto Confirm User", או הרשמה רגילה).
2. פתחו את `supabase/manual/add_first_admin.sql`, החליפו את `PUT-THE-ADMIN-EMAIL-HERE` באימייל שלכם והריצו ב‑SQL Editor.
   אין שם UUID מזויף: הסקריפט מסרב לרוץ עד שמחליפים, ומוצא את ה‑user id לבד. (אם אתם מעדיפים UUID:
   `insert into public.admin_users(user_id, role) values ('<ה‑id שלכם>', 'super_admin');`)
3. היכנסו ל‑`https://admin.auto-max.co.il`, ועברו ל‑**Settings → Enable two-factor**.
4. אחרי שה‑MFA פועל: `npx supabase secrets set ADMIN_REQUIRE_MFA=true`.

## 5. פריסה (Cloudflare Pages – חינם, מתאים גם לשימוש מסחרי)

(Vercel Hobby אסור לשימוש מסחרי, ולכן לא נבחר.)

**אפשרות א – העלאה ישירה:**

```bash
# הגדירו את שני משתני ה‑NEXT_PUBLIC_ בטרמינל (או ב‑.env.local), ואז:
npm run build
npx wrangler login
npx wrangler pages deploy out --project-name automax-admin
```

**אפשרות ב – GitHub:** Cloudflare → Workers & Pages → Create → Pages → Connect to Git. Build command: `npm run build`, Output directory: `out`,
משתני סביבה: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `NODE_VERSION=22`.

`npm run build` יוצר ב‑`out/` גם את `_headers` (CSP קפדני, HSTS, nosniff, Referrer-Policy, Permissions-Policy, frame-ancestors) ואת `_redirects`
(שכתוב `/users/<id>` לעמוד המעטפת הסטטי). אחרי פריסה בדקו שהכתובת `/users/<uuid>/` נטענת.

## 6. חיבור `admin.auto-max.co.il` (DNS ב‑Vangus)

1. ב‑Cloudflare Pages → הפרויקט → **Custom domains → Set up a custom domain** → `admin.auto-max.co.il`.
   Cloudflare יציג את היעד, בצורה `automax-admin.pages.dev` (השם תלוי בשם הפרויקט).
2. במסך ניהול ה‑DNS של **Vangus** של הדומיין `auto-max.co.il` הוסיפו רשומה **אחת**:

   | שדה | ערך |
   |---|---|
   | Type | `CNAME` |
   | Name / Host | `admin` (בחלק מהממשקים: `admin.auto-max.co.il`) |
   | Target / Value | `automax-admin.pages.dev` (בדיוק מה ש‑Cloudflare הציג, ללא `https://`) |
   | TTL | אוטומטי / 300 |

3. אם Cloudflare מציג בנוסף רשומת אימות (בדרך כלל `TXT`) – להוסיף אותה **בדיוק** כפי שמוצגת (שם וערך). אני לא יכול לנחש אותה מראש.
4. ממתינים לסטטוס **Active** ולתעודת ה‑HTTPS (דקות עד שעה), ואז: `curl -I https://admin.auto-max.co.il` (צפוי 200 + כותרות האבטחה).
5. אל תשנו את הרשומות הקיימות של האתר (`auto-max.co.il`, `www`, MX וכו').
6. ודאו ש‑`ADMIN_ALLOWED_ORIGINS` כולל `https://admin.auto-max.co.il`.

## 7. הוספת Admin

*super_admin* נכנס ל‑**Settings → Administrators → Add admin**, מזין אימייל של חשבון **מאושר** ובוחר תפקיד:
`support` (קריאה + מכשירים + סטטוס הרשמות), `admin` (פעולות תפעוליות + יומן), `super_admin` (הכול, כולל מחיקות, גרסאות, מסמכים משפטיים וניהול Admins).
או ב‑SQL: `supabase/manual/add_first_admin.sql`. כל שינוי נרשם כ‑`ADMIN_PERMISSION_CHANGE`.

## 8. ביטול Admin

**Settings → Administrators → Change → Deactivated** (נכנס לתוקף בבקשה הבאה, ללא מטמון). או:
`update public.admin_users set is_active = false where user_id = (select id from auth.users where email = '…');`
אי אפשר להוריד/למחוק את ה‑super_admin הפעיל האחרון (טריגר במסד הנתונים).

## 9. עבודה עם ה‑Edge Function

* **הוספת פעולה:** רשומה חדשה ב‑`actions.ts` (תפקיד מינימלי, מחלקת rate limit, `parse` קפדני, `run` שקורא ל‑`public.admin_*`) + פונקציית SQL במיגרציה חדשה.
  בדיקת החוזה (`tests/edge/contract.test.ts`) נכשלת אם שם הפונקציה/הפרמטרים אינם תואמים ל‑SQL.
* **בדיקה בלי Supabase:** `npm test` (הצינור כולו עם תלויות מדומות). בדיקת טיפוסים של נקודת הכניסה: `cd supabase/functions/admin-api && npx deno check index.ts`.
* **לוגים:** Dashboard → Edge Functions → admin-api → Logs. נרשמים רק פעולה, קוד סטטוס ומזהה משתמש – לעולם לא טוקנים, סיסמאות או גוף הבקשה.
* **פריסה:** `npx supabase functions deploy admin-api` (הגדרת `verify_jwt = true` ב‑`supabase/config.toml`).

## 10. מיגרציות

קובץ חדש ב‑`supabase/migrations/<timestamp>_<name>.sql`, אידמפוטנטי, שמסתיים בבלוק ההרשאות (revoke מ‑`anon`/`authenticated`, grant ל‑`service_role`) כמו בקיימות.
בדיקה לפני פרודקשן: `npm run test:sql` מריץ את הסכימה הקיימת + כל המיגרציות על PostgreSQL אמיתי (פעמיים) ובודק הרשאות, RLS, ביקורת והתנהגות.
**חשוב:** הרצה מחודשת של `schema.sql` הישן מחזירה את `check_license` למצבו הקודם (בלי חסימת חשבון מושבת) – להריץ אחריה שוב את `…0200_account_controls.sql`
(`verify_admin_security.sql` שאילתה 7 מראה אם החסימה פעילה).

## 11. Rollback

* **ממשק:** Cloudflare Pages → Deployments → Rollback לפריסה קודמת.
* **Edge Function:** פריסה מחדש של גרסה קודמת מה‑Git (`supabase functions deploy admin-api`), או השבתה: מחיקת הפונקציה מה‑Dashboard (הממשק יפסיק לעבוד, שום דבר אחר לא ייפגע).
* **מסד נתונים:** `supabase/rollback/rollback_admin.sql` מסיר את פונקציות `admin_*`, `admin_users`, `admin_rate_limits` והאינדקסים. הוא **משאיר בכוונה**
  את `admin_audit_log`, `payments_archive` ואת עמודות `profiles.is_disabled` (ראיות והיסטוריה פיננסית). להחזרת `check_license` המקורי – הרצת `schema.sql`.

## 12. הערות אבטחה (תמצית)

* בדפדפן יש רק `URL` ומפתח ציבורי. `service_role` קיים רק כסוד של ה‑Edge Function. `npm run scan:secrets` (וכשל בבדיקות) תופס מפתח סודי, JWT של service_role ומפתח Resend.
* כל בקשה: JWT → `admin_users` (פעיל + תפקיד) → MFA (אם רשום TOTP – חובה `aal2`) → rate limit בצד שרת → ולידציה קפדנית (שדה לא מוכר נדחה) → תפקיד → SQL שבודק שוב הרשאה וכותב ביקורת באותה טרנזקציה.
* רשימת פעולות סגורה; פעולה לא מוכרת = 400 (ולא‑Admin מקבל 403 בלי לדעת אילו פעולות קיימות).
* `admin_audit_log` הוא append‑only גם למנהל המסד (טריגרים) ואין לו מסך עריכה/מחיקה. סיסמאות/טוקנים/מפתחות מנוקים אוטומטית מה‑metadata.
* מחיקת משתמש = `super_admin` + הקלדת `DELETE` + ארכוב התשלומים לפני המחיקה (`payments` נמחק ב‑CASCADE במבנה הקיים). עדיף **Disable**.
* ה‑CSP מאפשר רק סקריפטים מ‑`self` ו‑hash מדויק של שני הסקריפטים הפנימיים של Next בכל עמוד. הדפדפן חוסם כל סקריפט שיוזרק.
* מה שעדיין דורש פעולה שלכם מפורט בסוף [SECURITY_AUDIT.md](SECURITY_AUDIT.md).
"# automax-admin" 
