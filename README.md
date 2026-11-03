# Iteration 1

## API

All football data comes from ESPN's free public API (`site.api.espn.com`, no key needed). Every request is asynchronous: call a `Fetch` function, then receive the result through an event. Works in Blueprint or C++.

---

### Games on a date

```cpp
void FetchGamesForDate(const FString& DateString = "");
FOnGamesParsed OnGamesParsed;
```

`FetchGamesForDate` gets all college football games for one day. Pass `"YYYY-MM-DD"`, or leave it empty for today (UTC). The dashes are stripped before the request. `OnGamesParsed` fires when the games are ready.

**Endpoint:**

```
https://site.api.espn.com/apis/site/v2/sports/football/college-football/scoreboard?dates={YYYYMMDD}&groups={id}&limit=200
```

- `dates`: the day to query
- `groups`: ESPN conference or division ID, used to filter the games
- `limit=200`: raises the cap on games returned

**Data fetched, per game:**

- `EventId` (use this to fetch plays)
- Start time and status (pregame, in progress, final), with period and clock
- Home and away teams (name, abbreviation, ranking, record) and score
- Venue and broadcast info

---

### One game's plays

```cpp
void FetchGameSummary(int32 EventId);
FOnFootballGameReady OnFootballGameReady;   // (int32 EventId, const FFootballGameData& Game)
```

`FetchGameSummary` gets the play-by-play for one game. `OnFootballGameReady` fires when it's ready, giving you the `EventId` and the full game data.

**Endpoint:**

```
https://site.api.espn.com/apis/site/v2/sports/football/college-football/summary?event={eventId}
```

**Data fetched:** a drive-by-drive record of the game. Per play:

- Play type (kickoff, punt, field goal, rush, pass, sack, etc.) and text description
- Quarter and game clock
- Down, distance, and starting yard line
- Yards gained and team with possession
- Scoring flag

The summary also includes the boxscore and team stats.

---

### Show a play on the field

```cpp
void ShowPlay(const FFootballPlay& Play, bool bHomeOffense);
```

`ShowPlay` (on `AFootballFieldView`, placed in `L_Main`) animates one play. It makes no network call; it uses a play already fetched above. Set `bHomeOffense` to say which team has the ball. Kickoffs, punts, field goals, runs, passes, and sacks have their own animations. Anything else just places the ball on its yard line.

---

### Example (C++)

```cpp
Feed->OnFootballGameReady.AddDynamic(this, &AMyActor::HandleGame);
Feed->FetchGameSummary(401628374);
```

Listen for the "game ready" event first, then request the game.

```cpp
void AMyActor::HandleGame(int32 EventId, const FFootballGameData& Game)
{
    if (FieldView && Game.Plays.Num() > 0)
        FieldView->ShowPlay(Game.Plays[0], true);
}
```

When the data arrives, this shows the first play with the home team on offense. `HandleGame` must be marked `UFUNCTION()` in your header for the binding to work.
