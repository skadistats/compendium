
# The Demo File (aka "a replay")

Dota 2 is a highly competitive game with immense depth. Often described as
"Easy to learn, impossible to master.", games happens within just one map.
At the time of writing, 107 heroes are playable in game. And although each
hero has a small number of skills, there are mathematically enormous
combinations of abilities, strategies, and play styles available to players.

Consequently, every game is unique. And yet there are patterns.

This is why having an built-in game spectating is so integral to the
experience. Discerning patterns in a game with such diversity and complexity
is, simply, what one has to do to *suck less over time*.

But to get at the *data* in a demo file is an entirely different undertaking.
This is why it is worth your time to understand the guts of demo files: with
the data, it is possible to understand Dota 2 trends deeply.

Demo files range in size from <1MB up to 200MB+. Generally, "casted" games
with replayable commentary are the largest--the audio is quite sizeable, so
the more audio tracks, the bigger the file. A typical public matchmaking game
of average duration might occupy around 20-40MB, as it has no audio data.

Demo files contain vast information, including but not limited to:

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

**But--and this is incredibly important--demo files do not describe the
behaviors that we observe as spectators. There is nothing written stating that
"Mirana goes top for a gank at 6 minutes." Rather, we get a stream of changes
to in-game entities over time. These changes comprises a visual game replay,
but understanding games behaviorally through so much low-level data is a
difficult undertaking.**

What follows in this book is a dive into demo file data which, taken together,
orchestrate a visual replay. This book exists to remove confusion and let you
get to work with your parser of choice.


## Obtaining Demo Files

Spectators can download demos directly from the game client. These downloads
do not disappear into a vacuum--they are stored on your computer.

Generally speaking, replays are located within the Steam installation in a
directory structure similar to this:

    SteamApps/common/dota 2 beta/dota/replays

Generally, Windows users can find their Steam installation at

    C:\Program Files (x86)\Steam

Mac users can find theirs at 

    /Users/you/Library/Application Support/Steam

The names of downloaded demos correspond to the Valve-issued Dota 2 **match
id**, which is guaranteed to be unique across all games. Consequently, demo
files have names like `12345678.dem`.


## Protocol Buffers

Valve uses Google's [Protocol Buffers](https://code.google.com/p/protobuf/)
(**protobuf** for short) to store data in demo files. You will need a mature,
full-featured protobuf library to parse demo files on your own.
Implementations exist for Java, Python, Javascript, C, C++, Go, Haskell, and
many other languages.

In a nutshell, protobuf allows developers to define **messages** using
predefined **primitive types** like `uint64`, `string`, and `bytes`. Coders
compile these definitions into language-specific code which
**[serializes and deserializes](http://en.wikipedia.org/wiki/Serialization)**
messages for transmission over a network or storage in a file.

So part of developing your own parsing library is compiling the most recent
Dota 2 protobuf definitions. Here is an example of one definition:

    message CGameInfo {
        message CDotaGameInfo {
            message CPlayerInfo {
                optional string hero_name = 1;
                optional string player_name = 2;
                optional bool is_fake_client = 3;
                optional uint64 steamid = 4;
                optional int32 game_team = 5;
            }

            message CHeroSelectEvent {
                optional bool is_pick = 1;
                optional uint32 team = 2;
                optional uint32 hero_id = 3;
            }

            optional uint32 match_id = 1;
            optional int32 game_mode = 2;
            optional int32 game_winner = 3;
            repeated CPlayerInfo player_info = 4;
            optional uint32 leagueid = 5;
            repeated CHeroSelectEvent picks_bans = 6;
            optional uint32 radiant_team_id = 7;
            optional uint32 dire_team_id = 8;
            optional string radiant_team_tag = 9;
            optional string dire_team_tag = 10;
            optional uint32 end_time = 11;
        }

        optional CDotaGameInfo dota = 4;
    }

Pretty interesting stuff! This message is embedded in another kind of message
which is the very last one in every replay:

    message CDemoFileInfo {
        optional float playback_time = 1;
        optional int32 playback_ticks = 2;
        optional int32 playback_frames = 3;
        optional CGameInfo game_info = 4;
    }

The excellent [SteamRE](https://github.com/SteamRE) project maintains current
protobuf definitions for several Valve games, including
[Dota 2](https://github.com/SteamRE/SteamKit/tree/master/Resources/Protobufs/dota).


## Demo File Format

Except for the constant 8-byte header (the characters `PBUFDEM\0`) at the
beginning of the file, and the 4-byte
[little-endian](http://en.wikipedia.org/wiki/Endianness) integer immediately
after, demo files are just a series of (**metadata**, **message**) sequences.

Notably, the above-mentioned 4-byte little-endian integer is a byte offset
into the file of the `CDemoFileInfo` message shown above. This allows for
quick access to summary information about the match.

### Metadata

The metadata contains information about the serialized message to come. It has
three pieces of data about the following message, in this order:

1. **kind**: integer indicating message type
2. **tick**: integer indicating **replay time**
3. **size**: integer indicating byte length of the message

In the data stream, these values are each encoded as a protobuf **varint**,
which is protobuf's way of using as few bytes as possible to represent the
integer. All protobuf libraries have either code for reading varints from a
data stream, or code you can lift and apply to do it yourself. For a cython
implementation, see
[smoke](https://github.com/skadistats/smoke)'s `io.util` module.

You can also [read about varints](https://developers.google.com/protocol-buffers/docs/encoding)
if you prefer not to look at code.

#### Kind

Protobuf allows definition files to contain **enum** values. Here is an
important one from Dota 2's set:

    enum EDemoCommands {
        DEM_Error = -1;
        DEM_Stop = 0;
        DEM_FileHeader = 1;
        DEM_FileInfo = 2;
        DEM_SyncTick = 3;
        DEM_SendTables = 4;
        DEM_ClassInfo = 5;
        DEM_StringTables = 6;
        DEM_Packet = 7;
        DEM_SignonPacket = 8;
        DEM_ConsoleCmd = 9;
        DEM_CustomData = 10;
        DEM_CustomDataCallbacks = 11;
        DEM_UserCmd = 12;
        DEM_FullPacket = 13;
        DEM_SaveGame = 14;
        DEM_Max = 15;
        DEM_IsCompressed = 112;
    }

This enum, `EDemoCommands`, is awkwardly named, but contains all possible
values for **kind**. And only some of these ever show up in demo files.
Without further elaboration right now, these are:

- `DEM_Stop`
- `DEM_FileHeader`
- `DEM_FileInfo`
- `DEM_SyncTick`
- `DEM_SendTables`
- `DEM_ClassInfo`
- `DEM_StringTables`
- `DEM_Packet`
- `DEM_SignonPacket`
- `DEM_FullPacket`
- `DEM_Savegame`(?)

There is one additional consideration: `DEM_IsCompressed`. Often, the **kind**
value will seem too high; for example, it will frequently be 119. In binary,
this value is `01111101`. Note the four high bits are `0111`.

If we add four 0's to the end, we end up with `01110000`, or 112 in base 10.

Using a binary OR, we can apply `01110000` to any of the values from 0-15
(that is, to any message), to indicate that it is compressed. Developers can
do the logical opposite, ANDing with `00001111`, to obtain the **kind** itself
for any message, regardless of compression.

For more details on *how* data is compressed, see the "Message" section below.

#### Tick

Source engine games (such as Dota 2) refer to one unit of game time as a
**tick**. The tick is the "atom" of game time. Nothing is communicated
regarding a game without an attached tick.

But, as Einstein realized, time is a malleable thing. ;) As it turns out,
*this* tick value isn't particularly interesting. Why? Because this tick is
a *replay time* tick. Although it parallels game time, this tick's timeframe
is the basis for the progress bar you see when spectating--not the game clock.

For understanding game time, you'll want a different timeframe, discussed in
depth later in the book.

Generally, this value can be ignored.

#### Size

The easiest of the three to understand, this integer is simply the number of
bytes to read next in the stream. Once you have read this many bytes, you can
hand the data to the generated protobuf code corresponding to the above
**kind** for deserialization. Decompression may be required before this step.

### Message

The message is a binary blob of data which must be deserialized using protobuf
code compiled from the Dota 2 protobuf definitions.

Often this data is compressed using Google's
[Snappy](http://code.google.com/p/snappy/) library. Snappy provides a decent
level of data compression in exchange for blazing-fast speed.

See the **kind** description above to learn when decompression is necessary.


## Recap

This is all the information necessary to merely parse a replay file.

The next section will introduce a twist: several of the message types detailed
above embed *an entirely different class* of protobuf messages within them.
Those embedded messages are where the very interesting data live.
