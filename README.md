# Linden Café — Guest Wi-Fi Portal

A captive-portal sign-in page with a Supabase backend.

- **Free access**: guest enters an email and gets 1 hour, once per device per day.
- **Voucher code**: staff print codes; each one gives a set amount of time.
- **Logging**: every attempt (email or code, device MAC, LAN IP, public IP, success, reason) goes into `portal_attempts`.
- **Brute-force protection**: 5 wrong codes from one device → that device is locked out for 10 minutes. 30 wrong codes from the whole venue's public IP → everyone is paused for 10 minutes.

```
portal/index.html                         the page guests see (no build step)
supabase/migrations/001_wifi_portal.sql   tables + database functions
supabase/functions/portal-auth/index.ts   Edge Function the page calls
```

## How it fits together

```
Phone joins Wi-Fi ──► router (openNDS) blocks internet, redirects to portal/index.html?fas=...
                                    │
Guest taps Connect ──► Edge Function portal-auth ──► portal_start_session() in Postgres
                                    │                  (rate-limit, check, log, create session)
                                    ▼
             page sends phone to http://<router>/opennds_auth/?tok=...  ──► internet unlocked
```

If the page is opened **without** a router (no `?fas=` in the URL), it runs in **demo mode**. It still checks and logs everything in Supabase, then shows the "You're connected" screen. Use that to test, or to demo it for class.

---

## 1. Supabase setup

1. Create a project at supabase.com.
2. **SQL Editor** → paste all of `supabase/migrations/001_wifi_portal.sql` → Run.
3. Deploy the Edge Function with the Supabase CLI:
   ```bash
   supabase login
   supabase link --project-ref YOUR-PROJECT-REF
   supabase functions deploy portal-auth --no-verify-jwt
   supabase secrets set FASKEY="a-long-random-secret"     # same value as on the router
   ```
   `--no-verify-jwt` is intentional: guests aren't logged in. The function only does one thing and is rate-limited, and the tables can't be read with any public key.
4. In `portal/index.html`, set:
   ```js
   const FUNCTION_URL = "https://YOUR-PROJECT-REF.supabase.co/functions/v1/portal-auth";
   ```

### Make vouchers

In the SQL editor:

```sql
select * from make_vouchers(20, 180);         -- 20 codes, 3 hours each, valid 30 days
select * from make_vouchers(5, 1440, 1, 7);   -- 5 day-passes, valid 7 days
select * from make_vouchers(1, 60, 50);       -- one code that 50 people can use
```

Codes look like `LND-4F7K-92QA` (no 0/O or 1/I/L, so they're easy to read off a receipt). Change the `LND-` prefix in the function if you like.

### Useful queries

```sql
-- Recent attempts, newest first
select created_at, method, identifier, client_mac, client_ip, request_ip, success, reason
from portal_attempts order by created_at desc limit 50;

-- Devices that got locked out today
select client_mac, count(*) from portal_attempts
where reason = 'rate_limited' and created_at > now() - interval '1 day'
group by 1 order by 2 desc;

-- Who's online right now
select * from wifi_sessions where expires_at > now() order by started_at desc;
```

To change the free time, cooldown or lockout limits, edit the `c_...` constants at the top of `portal_start_session` and run the file again.

---

## 2. Try it in demo mode

Open `portal/index.html` in a browser (double-click it, or `npx serve portal`). You'll see the "Demo mode" note under the form. Try:

- a free sign-in → connected screen with a 1-hour timer
- the same email again → "already used your free hour"
- a code from `make_vouchers` → works once, then "already been used"
- six wrong codes → "Too many wrong codes"

Then check `portal_attempts` in Supabase.

> In demo mode there's no MAC address, so the per-device limits fall back to the email and the public IP.

---

## 3. Hook it up to a real router (openNDS)

openNDS is the captive portal for OpenWrt routers. It sends guests to an outside page, which it calls a **FAS** (Forwarding Authentication Service).

1. **Host `portal/index.html` over plain `http://`.** openNDS uses http for FAS levels 0–2. A small web server on the LAN works, such as a Raspberry Pi or a PC running nginx. So does any http host.
2. **Router config** (`/etc/config/opennds`). Example values; adjust the IP and path:
   ```
   option fasport '80'
   option fasremoteip '192.168.8.10'      # the machine serving index.html
   option faspath '/'                      # path to index.html on that server
   option fas_secure_enabled '1'
   option faskey 'a-long-random-secret'    # same as the Supabase FASKEY secret
   option sessiontimeout '60'              # minutes; matches the free hour
   list walledgarden_fqdn_list 'YOUR-PROJECT-REF.supabase.co'
   ```
   Then `service opennds restart`.
3. **Walled garden.** Guests aren't online yet when they see the page, so the router must let them reach your Supabase domain. That's the `walledgarden_fqdn_list` line. On recent openNDS versions it needs the `dnsmasq-full` package (with ipset) installed.

### How the unlock works

- openNDS opens `index.html?fas=<base64>`. Decoded, it's `clientip=..., clientmac=..., hid=..., gatewayaddress=..., authdir=..., originurl=...`.
- The Edge Function computes `rhid = sha256(hid + faskey)`. The key stays secret on the server, so a guest can't fake it.
- The page then goes to `http://<gatewayaddress>/<authdir>/?tok=<rhid>&redir=<this page>?connected=1...&custom=`. openNDS unlocks the device and sends it back to the connected screen.

**Check against your openNDS version.** The FAS details have changed between releases. Compare against the example script that ships with yours (`/etc/opennds/fas-hid.php` or the docs for your version), especially:
- the field name for the hash ID in the decoded string (`hid` vs `client_hid`; both are handled)
- the parameter names on the auth URL

If they differ, the only places to edit are `parseFas` and the `authUrlBase` line in `index.ts`.

### Known limits

- **Session length is set by the router**, not the voucher. `sessiontimeout` applies to everyone. The database records the right expiry for each voucher, but making the router honour it per guest needs openNDS **BinAuth**, a small script on the router that can set each client's timeout. That's the next step if you need vouchers longer than the free hour.
- For the same reason, "Add more time" is hidden after a real router login. Guests reconnect with a voucher when their time runs out.
- **MAC randomisation.** Phones use a random MAC per network, but it stays the same for a given Wi-Fi name. So "one free hour per device" holds unless someone turns that setting off and on or forgets the network.
- **FAS level 1** sends device details base64-encoded, not encrypted. A determined user could change their reported MAC. Levels 2–3 encrypt it (aes-256-cbc with `faskey`). Worth moving to if this goes beyond a school project.
- **Privacy.** You're storing emails, MACs and IPs. Write a real Privacy Notice, link it where the page says `#privacy`, and delete old rows on a schedule, e.g.:
  ```sql
  delete from portal_attempts where created_at < now() - interval '90 days';
  ```

---

## Customising

- Name, network, address and hours are plain text in `index.html`. Search for `Linden` and `[YOUR`.
- The colour is set by `--accent` / `--accent-dark` at the top of the `<style>` block.
- The page uses system fonts only. Guests have no internet yet, so it can't load Google Fonts.
