# Junction Matchmaking — feature spec

Challenge: **The Junction Community Matchmaker** — help every participant find the right people to build with, even if they arrive alone. Suggest teammates using skills, interests, preferred roles and availability, and explain **why** each match makes sense.
Prizes: €200 for 1st place, Golden Tickets to Junction 2026 for 2nd and 3rd.

Goal: a **new "Matchmaking" page inside the Junction platform** (the dashboard with the Dashboard / Team / Ticket / Challenges / Hackerpack sidebar). Keep it **as simple as possible**.

---

## User flow

1. **Create account**
   - Name, age, email + password
   - **3 preferred roles** (e.g. Frontend, Backend, Designer, ML, Hardware, PM, Pitch)
   - **3 skills** (e.g. React, Python, Figma, PyTorch, Arduino)
   - Links: **LinkedIn**, **GitHub**, plus any other project links (portfolio, Devpost, itch.io, Behance…)
   - ⚠️ The links feed the **compatibility analysis** (see below). That part is built separately — the UI only collects the links and leaves a slot for the result.

2. **"Are you solo or a team?"**
   - **Solo** — I need a team
   - **Team (full)** — we're done, no matching needed
   - **Team (need 1–3 more people)** — we're looking for builders

3. **Solo → browse teams**
   - A scrollable list of teams that still need people
   - Each team card shows: team name, short bio, **roles they already have**, **roles they're looking for**, how many spots are open, **compatibility %** (slot)
   - Click a team → see each member's personal profile (roles, skills, links)
   - Button: **Request to join**

4. **Team (needs people) → browse solo builders**
   - A scrollable list of solo builders
   - Each card shows: name, 3 roles, 3 skills, links, **compatibility %** (slot)
   - Button: **Invite to team**

5. **Accept → chat**
   - The receiver sees incoming requests/invites → **Accept / Decline**
   - On accept, a **chat** opens between the solo and the team (and the solo is added to the team; the team's open spots go down by one, and when it reaches 0 the team becomes "full")

Team (full) users just see their team + a note that matching is off.

---

## Data model (minimal)

```ts
User {
  id, name, age, email,
  roles: [string, string, string],      // preferred roles, ranked
  skills: [string, string, string],
  links: { linkedin?: string, github?: string, other: string[] },
  mode: "solo" | "team_full" | "team_open",
  teamId?: string,
  profileAnalysis?: Analysis             // filled by the analysis feature (friend)
}

Team {
  id, name, bio,
  memberIds: string[],
  lookingFor: string[],                  // roles wanted
  openSpots: 0 | 1 | 2 | 3
}

Request {                                // one type covers both directions
  id, fromUserId, teamId,
  kind: "join_request" | "invite",      // solo→team  |  team→solo
  status: "pending" | "accepted" | "declined",
  createdAt
}

Chat     { id, teamId, soloUserId, createdAt }   // opened on accept
Message  { id, chatId, senderId, text, createdAt }
```

---

## 🔌 Compatibility analysis — slot for the friend's feature

Not built in the UI. The UI calls these and shows the result; until they exist, show "Compatibility: analysing…" / "—".

```ts
// Build a profile from the user's GitHub / LinkedIn / project links
analyzeProfile(user: User): Promise<Analysis>

// Score solo ↔ team (both directions use the same function)
compatibility(soloUserId: string, teamId: string): Promise<{
  score: number,          // 0–100, shown as a % on cards
  reasons: string[]       // short, human "why" lines, e.g.
                          // "Fills your missing Designer role",
                          // "Both shipped React + Python projects",
                          // "Interested in the same health challenge"
}>
```

Notes for the analysis:
- Score teams on **complementary roles** (fills a gap the team has), not just "similar skills" — five frontend devs is a bad team.
- The challenge explicitly rewards **explaining why** a match makes sense → always return `reasons`, built from real signals (repos, languages, past projects), not invented text.
- Lists are sorted by `score` desc.

---

## Pages / components

| Page | Contents |
|---|---|
| `/matchmaking/signup` | Account form + links (+ "analysis will run on these links" note) |
| `/matchmaking/mode` | 3 big choice cards: Solo / Team (full) / Team (need 1–3) — team options also ask team name, bio, roles wanted, open spots |
| `/matchmaking/teams` | Solo view: team cards → team detail with member profiles → Request to join |
| `/matchmaking/builders` | Team view: solo builder cards → Invite |
| `/matchmaking/inbox` | Pending requests / invites → Accept / Decline |
| `/matchmaking/chat/:id` | Simple chat thread |

Add **Matchmaking** to the sidebar under "DURING HACKATHON".

---

## Style — match the Junction platform

Taken from the platform dashboard screenshot + hackjunction.com:

| Token | Value |
|---|---|
| Page / sidebar bg | `#171717` |
| Card bg | `#232323` |
| Inner card / list item | `#2b2b2b`, border `#3c3c3c` |
| Sidebar active item | `#333333` |
| Dividers | `#2e2e2e` |
| Accent green | `#78ce8d` (buttons, pills, headings like "Friday") |
| Highlighted card | bg `#1c211d`, border `#78ce8d` (like "Your Submission") |
| Green rule / left bar | `#3e5944` |
| Text | `#ffffff`, muted `#8e908e` / `#969595`, sidebar labels `#a2a2a2` |
| Buttons | green bg `#78ce8d`, black text, ~6px radius; pills fully rounded |
| Fonts | Headings: **Montserrat** bold (platform) — hackjunction.com uses Inter 800/900 uppercase; body: Titillium Web / Montserrat; small caps labels letter-spaced (`HACKATHON`, `GENERAL`) |

Compatibility % → green pill on each card (e.g. `87% match`).

---

## Out of scope (for now)
- The analysis/scoring itself (friend)
- Real auth / OAuth with LinkedIn & GitHub — just paste URLs
- Notifications, moderation, team-size edge cases beyond "open spots"
