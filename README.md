# Iteration 1

## API

All data comes from public stats feeds. Each request is asynchronous: you call a
`Fetch` function, and the result arrives later through an event you bind to.
All of these can be called from Blueprint or C++.

### Baseball: games on a date

`UAPISubsystem` (game instance subsystem)

```cpp
void FetchGamesForDate(const FString& DateString = "");   // "YYYY-MM-DD", empty = today (UTC)
FOnGamesParsed OnGamesParsed;                              // (const TArray<FGameData>& Games)
```

Calls `https://statsapi.mlb.com/api/v1/schedule?sportId=1&date={date}`.

### Baseball: one game's pitches and plays

`UPitchDataSubsystem` (game instance subsystem)

```cpp
void FetchGameReplayData(int32 GamePK);
FOnGameReplayDataReady OnGameReplayDataReady;   // (int32 GamePK, const FGameReplayData& ReplayData)
FGameReplayData* GetCachedReplayData(int32 GamePK);   // C++ only; null until fetched
```

Calls `https://statsapi.mlb.com/api/v1.1/game/{gamePk}/feed/live`.

### College football: games on a date

`UESPNFootballSubsystem` (game instance subsystem)

```cpp
void FetchGamesForDate(const FString& DateString = "");   // "YYYY-MM-DD", empty = today (UTC)
FOnGamesParsed OnGamesParsed;
```

Calls `https://site.api.espn.com/apis/site/v2/sports/football/college-football/scoreboard?dates={YYYYMMDD}&groups={id}&limit=200`.
Pass the date with dashes; the dashes are stripped before the request.

### College football: one game's plays

`UESPNFootballSubsystem`

```cpp
void FetchGameSummary(int32 EventId);
FOnFootballGameReady OnFootballGameReady;   // (int32 EventId, const FFootballGameData& Game)
```

Calls `https://site.api.espn.com/apis/site/v2/sports/football/college-football/summary?event={eventId}`.

### College football: playing a play on the field

`AFootballFieldView` (actor placed in `L_Main`)

```cpp
void ShowPlay(const FFootballPlay& Play, bool bHomeOffense);
```

Pass one play from `FFootballGameData`. The play's type and text decide what is
shown: kickoff, punt, field goal, run, pass, sack, or the ball placed on its
yard line for anything else. The actor animates the play itself every frame.

### Example (C++)

```cpp
UESPNFootballSubsystem* Feed = GetGameInstance()->GetSubsystem<UESPNFootballSubsystem>();
Feed->OnFootballGameReady.AddDynamic(this, &AMyActor::HandleGame);
Feed->FetchGameSummary(401628374);

void AMyActor::HandleGame(int32 EventId, const FFootballGameData& Game)
{
    FieldView->ShowPlay(Game.Plays[0], /*bHomeOffense*/ true);
}
```
