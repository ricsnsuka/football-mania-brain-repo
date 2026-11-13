# Commercial ideas, November 2026

**Status:** ideas only, recorded 2026-11-13. Nothing here is built, scheduled or specced. It does
not lift the hold on [group billing](../backend/plans/GROUP-BILLING-PLAN.md), which stays on hold
until the owner lifts it explicitly.

This is the record of one conversation between the owner and Claude about how the app could earn
money and which features might come next. It separates what the owner decided from what was only
advice, because the advice rests on guesses about a market nobody has measured.

## Where the app stood

- One group uses it: the owner's own.
- The iPhone app is on TestFlight, not the App Store.
- Backend and web were on 4.1.0.

That first line shapes everything below. With one group there is no evidence of what another
organiser would use, let alone pay for.

## What the owner decided

- **No money moves through the app.** The organiser goes on chasing people and updating balances
  by hand. The ledger is where the organiser looks when they forget who owes what.
- **Maybe later:** payments through a third party, straight to the organiser, which on
  confirmation add to the balances in the app. The owner does not want to plan this now.
- **No adverts, ever.** The owner dislikes adverts appearing in front of them and will not put
  them in an app of their own.
- **A subscription would be paid by the organiser for the whole group.** The owner's idea is to
  unlock the number of players (players, not user accounts, at least at first) in levels such as
  20, 50, 75 and 100 at different prices.
- **New features should add to the existing concept, not change it.**

## The pricing idea

The owner's idea is player-count levels. Claude's advice on it, not yet accepted or rejected:

- A cap on its own is a toll, and an organiser can dodge it by not registering fringe players or
  by splitting into two groups. Crossing the cap should also unlock features.
- Start with two levels: free up to about 20 to 25 players, and one paid level above. Add the 50,
  75 and 100 levels only when real groups reach a ceiling. This rests on a guess that few casual
  groups pass 30 players.
- Keep everything a player touches free: answering a plan, team draw, the scoresheet, rankings and
  chat. Players are how the app reaches other groups.
- A subscription bought inside the iPhone app gives Apple 15 to 30 per cent. Selling it on the web
  avoids that. Check Apple's current rules before relying on this.
- Expect modest money: a few euros a month from a minority of groups.

## Features that could sit behind the paid level

All four add to what exists. None is built.

| Idea | What it is | What exists already |
| --- | --- | --- |
| Reliability record | Late withdrawals and no-shows counted per player, shown to the organiser, with an option to give reliable players first call on spots | The late-withdrawal notification; nothing keeps a tally |
| Ledger tools | A "who owes what" summary to paste into WhatsApp, a statement per player, a CSV export, and optionally refusing sign-up above a debt limit | The fee ledger and fee reminders |
| Full history | Free groups see the current season; paid groups keep every season, award and comparison | Seasons, awards, head-to-head |
| Shareable cards (free, not paid) | An image of a match result or a player's season, made to be posted in WhatsApp | Nothing |

The shareable card is free on purpose: without adverts it is the only way people in other groups
see the app. Claude's advice was to build the reliability record first of the paid ones, because
it is the smallest and WhatsApp cannot do it.

## Ideas for players

Raised by the owner on the same day, apart from pricing.

- **Favourite teammates.** Picks made in public cause friction over who was not chosen, and they
  pull against balanced team draw. The safer form is worked out, not chosen: "you win most with
  X". The head-to-head record already holds two players' results together and against.
- **Fantasy league.** Players pick a few others before a match day and score from goals, wins and
  votes. It leaves the concept alone. With about 14 players everyone picks the same three, so it
  needs a budget or a price per player.
- **Fixed teams.** Hold. A team today belongs to one match and is drawn fresh each time
  (`MatchTeam`). Permanent squads need a new record, a table of squads, and an answer for the week
  four players of one squad do not turn up. It is a league under another name, which is the kind
  of change the owner said to avoid.

## Offering it to the people who rent out pitches

The owner asked what would have to change to give pitch owners something like a pitch ranking.
Checked against the backend at 4.1.0:

- **A pitch is not a record.** `location` on a match and on a plan is free text, so "Campo 1" and
  "campo 1" are two places. A venue record that groups pick from a list comes first.
- **Everything belongs to one group.** A venue sits above groups and reads across them. It needs a
  new kind of account for the venue's owner: organiser, manager and player all live inside a group.
- **Groups are private.** A board showing several groups' results needs each group to opt in, and
  a change to the privacy policy and the terms.

Two more limits, inferred and not checked in the code:

- The same person in two groups is two separate players, so ranking people across a venue means
  joining them.
- Skill ratings are relative within a group and cannot be compared across groups. Goals, wins and
  matches played can.

"Pitch ranking" has two meanings of very different size:

- **Players rate the pitches:** small. A venue record and a rating after each match.
- **Groups and scorers ranked at a pitch:** large, for the reasons above.

Why it is worth a conversation before any code: a venue has several groups and a reason to keep
them loyal, so one owner could bring several organisers at once, and may be a better payer than
any organiser. Both points are guesses until someone asks a venue.

## What to do before building any of it

Claude's advice, given because only one group uses the app:

1. Finish the App Store submission. An organiser the owner does not know will not install through
   TestFlight.
2. Remove what makes it look like one group's tool: the web address is `amadora-foot.netlify.app`,
   and creating a group needs a code from the owner.
3. Bring in five to ten organisers by hand, starting with the owner's own players who also play in
   other groups and the organisers at the pitches the owner hires.
4. Watch one number: how many of those groups record a second match.
5. Ask them what they miss. That list decides the paid level, not this one.

The mistake to avoid is a month spent on subscriptions and level limits before a second group has
got through setting up.
