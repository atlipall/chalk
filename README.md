# Chalk

<img src="logo.svg" alt="Chalk" height="84">

A darts scorer that runs entirely in the browser. Pick a game, add players, tap where each dart landed.

No build step, no dependencies, no server: the app is all in `index.html`. Beside it are the icons (`favicon.svg`, `favicon-32.png`, `apple-touch-icon.png`) and `logo.svg`; the app runs without them. The only external request is for two Google Fonts (Cabin Sketch and Barlow Semi Condensed); the page works without them.

## Run it

Open `index.html` in a browser, or serve the folder:

```sh
npx serve .
# or
python3 -m http.server
```

Any static host (GitHub Pages, Netlify, Vercel) can serve it as is.

## Games

| Game | Rules implemented |
| --- | --- |
| 501 / 301 / 701 | Double out or straight out, bust handling, three-dart average, checkout suggestions (fewest darts, then easy setup darts and the preferred finishing doubles: D20, D16, D8, …, bull last) |
| Cricket | 15 to 20 and bull, three marks to close, points until everyone has closed |
| Around the Clock | 1 to 20 in order, then the bull; any multiplier counts as a hit |
| Halve-It | Start at 40; targets 20, 16, double 7, 14, 18, triple 10, bull; no hit in a turn halves the score |
| Shanghai | 7 or 20 rounds, round N aims at N for number × multiplier; single, double and triple of the target in one turn wins outright, otherwise highest score |
| Killer | Each player throws for a free number 1 to 20; 3 or 5 lives; hit your own double to become a killer; a killer's hits on other doubles take a life, on their own double cost one; last player standing wins (two or more players) |

## How it works

**State is a dart log.** The app stores only the setup (game, options, players) and the list of darts thrown, each as `{ n, m }` (number 0–20 or 25, multiplier 1–3; a miss is `n: 0`). The scoreboard is rebuilt by replaying that list through the game's rules (`replay()`), so undo is `log.pop()` and a reload resumes mid-game.

**Each game is a rules module** in the `GAMES` object:

| Member | Purpose |
| --- | --- |
| `title`, `blurb`, `name(opts)` | Text for the setup card and the play header |
| `rules` | HTML for the game's "How to play" dialog, opened from its setup card. Use `<p>`, `<h3>` for sections, `<ul>` and `<b>` for the key phrase, and describe what the code does (including the options), not a house rule it doesn't implement |
| `options` | Optional list of `{ key, label, def, choices: [[value, text], …] }`; each becomes a button group on the setup page and is read as `s.setup.opts[key]` |
| `player(opts)` | Starting state for one player |
| `dart(s, p, d)` | Apply one dart; return `true` to end the turn early (a bust); set `s.over` and `s.winners` on a win |
| `startTurn(s)`, `endTurn(s, p)` | Optional hooks |
| `skip(s, p)` | Optional; `true` passes over that player in the turn order (Killer uses it for players who are out) |
| `out(p)` | Optional; `true` fades the player's card |
| `minPlayers` | Optional; Start is disabled below this |
| `art()` | Optional SVG drawn behind the setup card; see [Card graphics](#card-graphics) |
| `short` | Optional shorter title for the scoreboard's game filter (`"501"`, `"Clock"`) |
| `stats` | Optional scoreboard columns: `{ cols, add(acc, ctx), show(acc) }`. `add` is called once per player per finished game with `ctx = { s, p, i, won, turns }` (the replayed state, the player, their index, whether they won, and their turns) and accumulates into `acc`; `show(acc)` returns one value per column |
| `roundLabel(s)` | Optional text for the header in place of "Round N"; return an empty string to keep the default (Killer shows "Picking numbers") |
| `live(s)` | Optional; returns `(n, m) => boolean` for whether a dart counts right now (`m` is left out when asking about a whole number). Darts that don't count are faded on the input |
| `hint(s)` | Line shown under the current turn |
| `aim(s)` | Where the computer player aims its next dart, as `{ n, m }`; see [Computer player](#computer-player) |
| `big(p, s)`, `sub(p, s)` | Main figure and caption on a player card (Cricket draws its own table in `boardView`) |

`throwDart()` owns the turn structure (three darts, next player, round counter), so a new game only needs an entry in `GAMES`.

**Computer player.** With exactly one person in the game, the setup page offers a computer opponent at Easy, Medium or Hard. It joins the throwing order as `"Computer"` (a reserved name, kept out of the saved players), and `setup.bot` holds its level. It goes away again when a second person joins. On its turn it throws a dart about every 0.75 s and your input is disabled. Each throw takes the game's `aim(s)` target, converts it to a point on a real board in mm (treble or double band, fat outer single, bullseye), and adds Gaussian scatter (`BOT`: 29, 14.5 and 8.5 mm, about a 30, 55 and 85 three-dart average at 501). Only the landed dart goes in the log, so replay stays deterministic. Undo takes back the computer's darts since your last dart and then that dart.

**Card graphics.** Each game card on the setup page has an abstract graphic behind its text. The game's `art()` returns the SVG shapes (no `<svg>` wrapper) for a 300 × 120 box. The card draws it fitted to the card's height and pinned to the right edge, at 42% opacity (70% when selected), and fades it in from the left with a CSS mask so the title stays clean. To draw one for a new game:

- **Hint at how the game works, not what it looks like.** Each existing graphic is a pattern taken from its game's rules: bars stepping down for 501, columns filling in thirds for Cricket, a ring lit part-way round for Around the Clock, a rectangle that keeps halving for Halve-It. Avoid pictures of darts, boards or text.
- **Use the board colours.** Use `ART.red`, `ART.green` and `ART.cream`, plus exactly one `ART.yellow` accent for the thing that matters most (the checkout, the bull, the last life). Show "empty" or "not yet" as cream at about 0.12 to 0.2 opacity.
- **Keep the left third quiet.** It is faded out and sits under the title, so put the shapes from about x = 110 to the right edge. Anything outside the 300 × 120 box is cut off.
- **Keep it flat.** Use solid fills, no gradients, filters or text, and any lines no thicker than about 3. `artRect(x, y, w, h, fill, opacity)` draws a rounded block, and `sector(cx, cy, r1, r2, a1, a2)` returns a path for a ring segment (angles in degrees clockwise from 12 o'clock).
- **Check it at both sizes.** Look at the setup page at desktop width and at about 390px wide (the browser's device mode), with the card both selected and not. The title and blurb should read clearly over it; if not, lower the opacity of the busiest shapes.

For example, five blocks shrinking towards a yellow one:

```js
art: () => [90, 70, 50, 30].map((h, i) => artRect(140 + i * 34, 120 - h, 26, h, i % 2 ? ART.green : ART.red)).join("")
  + artRect(276, 108, 26, 12, ART.yellow),
```

**Input** is either a tappable SVG dartboard (`dartboardSvg`, drawn with widened double and triple rings so they can be hit with a finger) or a plain button grid. The choice is remembered.

**Keep screen on** uses the Screen Wake Lock API while a game is open (on by default; toggled on the setup page or with "Screen on" in the game header). The browser drops the lock when the tab is hidden, so it is requested again on `visibilitychange`. The option is hidden where the API is missing.

**Scoreboard.** Finished games are saved under `chalk.history.v1` (the last 1000), each as its setup, dart log and finish time. A game is recorded when it ends and removed again if the winning dart is undone; games ended with End game are not recorded. The scoreboard replays the saved games through the current rules, so it always matches them. It shows Today (since local midnight) or Lifetime, for all games or one. Players are ranked by wins, then win %; wins, played and win % count only games with two or more people (the computer doesn't count, and has no row), while each game's own columns (`stats`) include solo and computer games too.

**Players** are remembered. `roster` lists everyone ever added; `setup.players` is who is in the current game, in throwing order, by name from the roster. On the setup page, Sit out takes a player out of the game but keeps them saved, a saved name rejoins with one tap, and Delete (tapped twice) forgets them. Adding a name that is already saved, in any letter case, just brings that player back.

**Storage** is `localStorage` under the key `chalk.v1` (a game saved under the old `oche.v1` key is moved over on load). Nothing leaves the browser.

## Notes

- The `window.claude.hot` lines at the bottom of the script only matter when the page is hosted as a Claude artifact; they are no-ops elsewhere and can be deleted.
- Ideas not done yet: match history and statistics, legs and sets for 501, offline install (manifest and service worker).
