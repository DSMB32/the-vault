# The Vault — setup notes

## Admin access

- Guests never see the admin buttons. Open the site with **`?admin`** at the end of the link
  (for example `https://…/index.html?admin`) to show **Picks per person**, **Pick Winner** and **Reset**.
- The first admin action asks for the PIN. Once it's accepted, that browser stays an admin,
  so you won't be asked again on that device.
- Until you finish the lockdown below, the PIN is `1234` (or whatever you store at `config/adminPin`).

## Locking down the votes (one-time, ~5 minutes)

Right now the database accepts writes from anyone, so someone who opens the browser console
could clear the votes or declare a winner. These steps make the database itself enforce the rules:
only you can pick a winner, reset or change settings, and each person can only change their own votes.

Do them **in this order**. The site keeps working the whole time.

1. **Turn on anonymous sign-in.**
   [Firebase console](https://console.firebase.google.com/) → project **the-vault-c719f** →
   **Authentication** → **Get started** (only shown the first time) → **Sign-in method** tab →
   **Anonymous** → **Enable** → Save.
   Visitors get an invisible ID; they don't have to log in.

   **Check it worked before going on:** open your live site with `?admin` at the end of the link.
   If a message says *"anonymous sign-in isn't working yet"*, stop here: publishing the rules now
   would block everyone from voting. The usual cause is an API key restriction. In
   [Google Cloud Console → APIs & Services → Credentials](https://console.cloud.google.com/apis/credentials),
   open the browser key for this project and make sure your site's address is allowed, and that
   **Identity Toolkit API** is in its list of allowed APIs. Then reload and check again.

2. **Publish the rules.**
   **Realtime Database** → **Rules** tab → replace everything with the contents of
   [`database.rules.json`](database.rules.json) → **Publish**.

3. **Set your new admin PIN.**
   **Realtime Database** → **Data** tab → hover the root → **+** → key `config`, then add a child
   key `adminPin` with your PIN **in quotes**, for example `"4821"` → Add.
   Pick something other than `1234`: that one is visible in the site's old code on GitHub.
   After the rules are published, nobody can read `config`.

4. **Clear old admin grants.** If there's an `admins` node in **Data**, delete it, so every admin
   device has to enter the new PIN once.

### Check the rules (optional but recommended)

In **Realtime Database → Rules**, open the **Rules Playground**:

| Simulate | Location | Auth | Expected |
| --- | --- | --- | --- |
| set `true` | `/sessions/round_x/ballots/abc/movie_1` | Authenticated, UID `abc` | ✅ Allowed |
| set `true` | `/sessions/round_x/ballots/someone_else/movie_1` | Authenticated, UID `abc` | ❌ Denied |
| set `"winner_declared"` | `/sessions/round_x/phase` | Authenticated, UID `abc` | ❌ Denied |
| set `"round_test"` | `/active_session` | Unauthenticated | ❌ Denied |
| read | `/config/adminPin` | Authenticated | ❌ Denied |

## Picks per person

The default is **3 picks per person**. Admins can change it from the 🎟 menu (1–10, or ∞ for
unlimited). If someone already has more picks than the new limit, they can still remove picks;
they just can't add new ones until they're under the limit.

## Watched history

Every time a winner is picked, it's logged under `history/<round>` with the date. Choosing
**Restart Voting** removes that round's entry again. To delete an old entry, remove it in the
Firebase console under **Data → history**.
