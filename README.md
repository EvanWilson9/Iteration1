# Iteration 1

## API

All football data comes from ESPN's public site API (`site.api.espn.com`, no API key required). Each request is asynchronous: you call a `Fetch` function, and the result arrives later through an event you bind to.
All of these can be called from Blueprint or C++.

### College football: games on a date

`UESPNFootballSubsystem` (game instance subsystem)

```cpp
void FetchGamesForDate(const FString& DateString = "");   // "YYYY-MM-DD", empty = today (UTC)
FOnGamesParsed OnGamesParsed;
```

Calls `https://site.api.espn.com/apis/site/v2/sports/football/college-football/scoreboard?dates={YYYYMMDD}&groups={id}&limit=200`.
Pass the date with dashes; the dashes are stripped before the request.

Source: ESPN site API, college football scoreboard endpoint.
- `dates`: the day to query.
- `groups`: ESPN conference or division ID, used to filter the slate.
- `limit=200`: raises the cap on the number of games returned.

Returns the list of games on that date. Per game:
- `EventId`, the ESPN event ID used to fetch plays
- Start time and status (pregame, in progress, final), including period and clock
- Home and away teams (name, abbreviation, ranking, record) and score
- Venue and broadcast info

### College football: one game's plays

`UESPNFootballSubsystem`

```cpp
void FetchGameSummary(int32 EventId);
FOnFootballGameReady OnFootballGameReady;   // (int32 EventId, const FFootballGameData& Game)
```

Calls `https://site.api.espn.com/apis/site/v2/sports/football/college-football/summary?event={eventId}`.

Source: ESPN site API, game summary endpoint.

Returns the drive-by-drive record for one game. Per play:
- Play type (kickoff, punt, field goal, rush, pass, sack, etc.) and a text description
- Quarter and game clock
- Down, distance, and starting yard line
- Yardage gained and the team with possession
- Scoring flag

The summary also includes the boxscore and team stats.

### College football: playing a play on the field

`AFootballFieldView` (actor placed in `L_Main`)

```cpp
void ShowPlay(const FFootballPlay& Play, bool bHomeOffense);
```

No network call: this uses a play already fetched from the summary endpoint above.

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
