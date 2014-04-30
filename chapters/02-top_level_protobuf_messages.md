
# "Top Level" Protobuf Messages

The previous chapter discussed the general format of the demo file. In this
chapter, we take a dive deeper into the substructure of the messages we last
introduced.

Again, the messages found at the "top level"--that is, through a parse of the
demo file's (**metadata**, **message**) sequences--are defined in Dota 2's
protobuf message definitions with the following **kind** values:

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

However, the only **kinds** ever encountered in a demo file are:

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


## Demo File Structure

Demo files' top-level protobuf messages occur in a specific order. Below is an
ordered list of what *should* be in a typical replay. Commas represent
multiple possible message types. An plus indicates 'one or more'; otherwise,
exactly one message is expected. For sections with multiple message
possibilities, assume no particular order for now. (This is not exactly true.)

1. *DEM_FileHeader*
2. DEM_ClassInfo,DEM_SendTables,DEM_StringTables,DEM_SignonPacket+,DEM_Packet+
3. *DEM_SyncTick*
4. DEM_FullPacket,DEM_Packet+,DEM_SaveGame
5. *DEM_Stop*
6. DEM_FileInfo

The bolded message types above function as markers of "phase change"--they
tell the parsing application what part of the replay it is in. The authors
refer to:

- everything between `DEM_FileHeader` and `DEM_SyncTick` as the **prologue**
- everything between `DEM_SyncTick` and `DEM_Stop` as the **match**
- everything between `DEM_Stop` and the end-of-file as the **epilogue**

Most parsers simply handle messages as encountered--there is no runtime notion
of being "in the prologue," for example--but it helps to understand the broad
organization in these terms. The remainder of this chapter will focus on these
sections and the messages belonging to each.
