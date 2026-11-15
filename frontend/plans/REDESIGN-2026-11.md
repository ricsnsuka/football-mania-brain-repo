# The 2026-11 redesign: decisions, and what the iPhone app will need

**Opened 2026-11-15.** The web app is being restyled in four steps. The iPhone app keeps its current
look for now and is **rebranded later** (owner, 2026-11-15). This page is the record that rebrand
starts from, so it does not have to be worked out again from screenshots.

Mock-ups of every screen described here:
<https://claude.ai/artifact/9ELuLvS4nzeyBaqy9qtPHx> (private to the owner).

## Why

The owner's words: the app is "plain" and "a bit heavy for what it is". Looking at the screens, both
came from the same cause. It was built like back-office software:

- sortable tables and "Page 1 of 1 · Per page 10" under a three-player list;
- boxes inside boxes: on the plan screen each team sat in a box, in a card, in a section, in a modal;
- seven items in the top menu plus a "Manage" dropdown;
- default blue, grey borders and a neutral font, so nothing said football;
- a home screen that led with "Welcome back" and kept the next match in a small dashed box.

## The direction

The owner chose between three home-screen mock-ups and combined two of them:

- **the look of "Matchday"**: a green-black ground, lime as the accent, tall condensed numerals;
- **the section separation of "Kickabout"**: each section in its own rounded card.

Rules that follow from it:

1. **One card per section, nothing nested.** No borders on cards and no boxes inside them.
   Separation is the card's fill against the page, and the gap between cards.
2. **The next match is the hero of the home screen**, with the person's own answer as the main button.
3. **Lists, not tables**, for anything a group's size makes short. No pagination under a short list.
4. **Big numerals carry the identity**: scores, ratings, balances and stats.
5. **Light and dark both stay.** Only the ground and the cards flip.

## Colour

One brand scale, which is two colours:

| Step | Hex | Role |
|---|---|---|
| `brand-400` | `#B6F23A` | Lime. The accent on a dark ground, and a fill under **dark** text |
| `brand-300` | `#C6F26A` | Lime, lighter: hover on dark |
| `brand-50` to `brand-200` | `#F3FBE6` `#E5F7C8` `#D2F29B` | Pale lime tints: chips and selected rows in light mode |
| `brand-500` | `#37852B` | Pitch green: focus rings, progress fills in light mode |
| `brand-600` | `#2A7221` | Pitch green: buttons under white text, links on white |
| `brand-700` to `brand-950` | `#225E1B` `#1B4B17` `#163D13` `#0C240B` | Hover, and tinted backgrounds in dark mode |

- **Lime is never text on white, and never a fill under white text.** Both fail contrast. In light
  mode links and highlights are the pitch green; lime appears only as a fill with dark text.
- **One green, not two** (owner, 2026-11-15). A win, an active player and a success message are the
  brand colour. There is no separate "success green", and no blue anywhere.
- Losses stay red, the MVP and warnings stay yellow and amber.

Neutrals are tinted slightly green:

| Role | Light | Dark |
|---|---|---|
| Page ground | `#F1F4EF` | `#0B0F0D` |
| Card | `#FFFFFF` | `#151C18` |
| Sunken well, hairline | `#F4F6F2`, `#DDE3DA` | `#1F2823`, `#2B3830` |
| Text, strongest | `#0E1411` | `#F2F5F3` |
| Text, muted | `#616E65` | `#9BA8A0` |

## Type

- **Archivo** for all text.
- **Barlow Condensed** (600, 700, 800) for scores, ratings, balances and headline figures only. It is
  too narrow for running text.
- Small uppercase labels with wide tracking for section names ("NEXT MATCH", "YOUR SEASON").

## The icon and the name

- The brand mark is **black and white** (owner, 2026-11-15): a white ball on `#0B0F0D`. Green and lime
  are the colours inside the app, not the colours of the icon. The web icon changed on 2026-11-15.
  **The iPhone app icon and the App Store listing have not**, and should change together with the
  app's rebrand.
- The app is called **Sidekick**.

## Navigation

Seven destinations become five on a phone, in a bar at the bottom: **Home, Matches, Table, Money,
More**. Matches and match plans merge into "Matches". Players, users, chat and settings sit under
"More". On a desktop there is room for six in a row of pills at the top: Home, Matches, Table, Money,
Chat, Manage.

## Screens

**Home.** In order: the next match, what is waiting on you, your season, the top of the league table,
the last result. On a desktop, two columns: next match, waiting on you and last result on the left;
season and table on the right.

**The next-match card, by stage.** It is the same lime card throughout.

| Stage | Shows | Main button |
|---|---|---|
| Planning, before you answer | "6 of 10 in · 4 more needed", a bar with one segment per place, the names of who is in, the deadline | "I'm in", with "Can't make it" beside it |
| Planning, after you said yes | The same, with you first in the list | A ticked "You're in", with "Change" |
| Squad full, no teams yet | "10 of 10 in", and that teams have not been drawn | For a manager "Draw teams"; for a player only "See squad" |
| Teams drawn | Which team you are on, and against whom | "See teams" |
| Nothing planned | A small dark card | For a manager "Plan a match" |

Names, not initials, for who is in: "is João coming?" is the question people open the app to answer.
On a desktop the card's right half lists who is in during planning, and the two squads once drawn.

**Match, played.** The score in large numerals with the winner in the brand colour, the man of the
match, the **AI report labelled "Written by AI"**, then the scoresheet as one card per team.

**Players.** A list: initials, name, one line of what they have done, and the skill rating on the
right. Filter chips above it. No table, no pagination.

**Plan, choosing teams (manager).** One draw at a time, switched by chips ("Balanced", "Random"),
where every saved draw used to be stacked in full. Teams side by side with no boxes. The balance is
one bar with the two averages at its ends. One main button, which is the next thing that can be done;
"Create match" does not appear while it cannot be used. Team vote and "draw another" are single rows.
A full page on a phone, not a modal.

## The steps on the web

| Step | What | State |
|---|---|---|
| 1 | Colours, type, corners, the icon | Built 2026-11-15, FootMania-Simple-Front `feat/redesign-tokens` |
| 2 | The menu, and the home screen with the next-match card and lime buttons | Not started |
| 3 | The match screen, with the AI preview and report | Not started |
| 4 | The plan screen, then lists in place of tables | Not started |

## What the iPhone app will need

- The colours, type and rules above. The native screens carry their own colours; nothing here reaches
  them through a deploy.
- The icon and the App Store listing, in black and white.
- The same navigation and the same screens, where they exist natively.
- Until then, **web screens opened inside the app look different from the native screens around
  them**. That is accepted for now.
- The AI preview and report still need building in the app, separately from the look
  (`docs/frontend/FRONTEND_ENDPOINT_CHANGES.md` in the backend, 2026-11-15).

## Not decided

- Whether the bottom bar is dark in light mode, as mocked, or white.
- Where reserves and "can't make it" appear on the next-match card.
- The match list, the full league table, money, chat, settings and the sign-in pages have not been
  drawn.
