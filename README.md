# Chalk

A darts scorer that runs entirely in the browser. Pick a game, add players, tap where each dart landed.

No build step, no dependencies, no server: everything is in `index.html`. The only external request is for two Google Fonts (Cabin Sketch and Barlow Semi Condensed); the page works without them.

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
| `options` | Optional list of `{ key, label, def, choices: [[value, text], …] }`; each becomes a button group on the setup page and is read as `s.setup.opts[key]` |
| `player(opts)` | Starting state for one player |
| `dart(s, p, d)` | Apply one dart; return `true` to end the turn early (a bust); set `s.over` and `s.winners` on a win |
| `startTurn(s)`, `endTurn(s, p)` | Optional hooks |
| `skip(s, p)` | Optional; `true` passes over that player in the turn order (Killer uses it for players who are out) |
| `out(p)` | Optional; `true` fades the player's card |
| `minPlayers` | Optional; Start is disabled below this |
| `roundLabel(s)` | Optional text for the header in place of "Round N"; return an empty string to keep the default (Killer shows "Picking numbers") |
| `live(s)` | Optional; returns `(n, m) => boolean` for whether a dart counts right now (`m` is left out when asking about a whole number). Darts that don't count are faded on the input |
| `hint(s)` | Line shown under the current turn |
| `big(p, s)`, `sub(p, s)` | Main figure and caption on a player card (Cricket draws its own table in `boardView`) |

`throwDart()` owns the turn structure (three darts, next player, round counter), so a new game only needs an entry in `GAMES`.

**Input** is either a tappable SVG dartboard (`dartboardSvg`, drawn with widened double and triple rings so they can be hit with a finger) or a plain button grid. The choice is remembered.

**Keep screen on** uses the Screen Wake Lock API while a game is open (on by default; toggled on the setup page or with "Screen on" in the game header). The browser drops the lock when the tab is hidden, so it is requested again on `visibilitychange`. The option is hidden where the API is missing.

**Storage** is `localStorage` under the key `chalk.v1` (a game saved under the old `oche.v1` key is moved over on load). Nothing leaves the browser.

## Notes

- The `window.claude.hot` lines at the bottom of the script only matter when the page is hosted as a Claude artifact; they are no-ops elsewhere and can be deleted.
- Ideas not done yet: match history and statistics, legs and sets for 501, offline install (manifest and service worker).
