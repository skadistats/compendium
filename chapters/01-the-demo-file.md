
# The Demo File

Dota 2 is a highly competitive game with immense depth. Often described as
"Easy to learn, impossible to master.", games happens within just one map.
At the time of writing, 107 heroes are playable in game. And although each
hero has a small number of skills, there are mathematically enormous
combinations of abilities, strategies, and play styles available to players.

Consequently, every game is unique. And yet there are patterns.

This is why having an built-in game spectating is so integral to the
experience. Discerning patterns in a game with such diversity and complexity
is, simply, what one has to do to *not suck*.

So what does this have to do with replay files? Everything. This is why
replays exist: so we can all suck less at Dota 2.

But to get at the *data* in replays is an entirely different undertaking. This
is why it is worth your time to understand the guts of replay files: if the
data are there, then it becomes possible to understand the game analytically,
rather than merely visually.


## Replays (aka "demo files")

Spectators can download demos directly from the game client. These downloads
do not disappear into a vacuum--they are stored on your computer.

Generally speaking, replays are located within the Steam installation in a
directory structure similar to this:

    SteamApps/common/dota 2 beta/dota/replays

Generally, Windows users can find their Steam installation at
`C:\Program Files (x86)\Steam`.

Mac users can find theirs at `/Users/you/Library/Application Support/Steam`.

The names of downloaded replays correspond to the Valve-issued Dota 2 "match
id," which is guaranteed to be unique across all games. It is a simple number,
the set of which increases over time.

Demo files range in size from <1MB up to 200MB+. Generally, "casted" games
with replayable commentary are the largest--the audio is quite sizeable, so
the more audio tracks, the bigger the file. A typical public matchmaking game
of average duration might occupy around 20-40MB, as it has no commentary.

Replays contain vast amounts of information, including but not limited to:

- all player and caster camera perspectives, clicks, UI interactions
- audio commentary
- "all" chat messages (sorry, no team chat!)
- cosmetic hero item info
- server-calculated game statistics such as gold and experience graphs
- the fog of war at any given time in the game
- particle effects
- hero and neutral creep attack animation timings
- item purchases, inventories
- periodic entire game states for the "replay takeover" feature
- game state events (night/day)
- rune and neutral/ancient spawns, etc.

So, nearly everything.

**But--and this is incredibly important--replay files do not describe the
behaviors that we observe as spectators. Rather, they describe how the
properties of in-game entities change over time. The presentation of these
changes comprises the "replay," but understanding the game coherently through
such low-level data is very, very hard.**

What follows in this book is a dive into the minutiae that, taken together,
orchestrate a "replay." It is very important to understand this distinction.
There are several sources of data that need correlation and analysis. This
book exists to remove the confusion and let you get to work.
