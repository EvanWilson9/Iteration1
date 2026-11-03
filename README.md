# Iteration 1

## API

All football data comes from ESPN's public site API (`site.api.espn.com`), which needs no API key. Every request is asynchronous: you call a `Fetch` function, and the result arrives later through an event you bind to. All of these can be called from Blueprint or C++.

---

### College football: games on a date

**Class:** `UESPNFootballSubsystem` (a game instance subsystem)

```cpp
void FetchGamesForDate(const FString& DateString = "");
FOnGamesParsed OnGamesParsed;
```

`FetchGamesForDate` starts a request for every game on one day. `DateString` uses the format `YYYY-MM-DD`. If you leave it empty, the subsystem uses today's date in UTC. The dashes are stripped before the request is sent, because ESPN expects `YYYYMMDD`.

`OnGamesParsed` is the delegate that fires once the response has been downloaded and parsed. Bind to it before calling `FetchGamesForDate`, or you will miss the result. *(The original doc did not list this delegate's parameters; add them here.)*

The function calls this endpoint:

```
https://site.api.espn.com/apis/site/v2/sports/football/college-football/scoreboard?dates={YYYYMMDD}&groups={id}&limit=200
```

The query parameters are:

- `dates`: the day to query.
- `groups`: an ESPN conference or division ID used to filter the slate. It is not a parameter of `FetchGamesForDate`, so the subsystem supplies it internally.
- `limit=200`: raises the cap on the number of games returned.

**Returned data**, per game:

- `EventId`, the ESPN event ID, which you pass to `FetchGameSummary` to get plays
- Start time and status (pregame, in progress, final), including period and clock
- Home and away teams (name, abbreviation, ranking, record) and score
- Venue and broadcast info

---

### College football: one game's plays

**Class:** `UESPNFootballSubsystem`

```cpp
void FetchGameSummary(int32 EventId);
FOnFootballGameReady OnFootballGameReady;   // (int32 EventId, const FFootballGameData& Game)
```

`FetchGameSummary` requests the full play-by-play for one game. `EventId` is the value returned per game by `FetchGamesForDate`.

`OnFootballGameReady` fires when the summary has been parsed. It passes back the same `EventId`, so you can match the result to the request, plus an `FFootballGameData` containing the parsed game.

The function calls this endpoint:

```
https://site.api.espn.com/apis/site/v2/sports/football/college-football/summary?event={eventId}
```

**Returned data:** a drive-by-drive record of the game. Per play:

- Play type (kickoff, punt, field goal, rush, pass, sack, etc.) and a text description
- Quarter and game clock
- Down, distance, and starting yard line
- Yardage gained and the team with possession
- Scoring flag

The summary also includes the boxscore and team stats.

---

### College football: playing a play on the field

**Class:** `AFootballFieldView` (an actor placed in `L_Main`)

```cpp
void ShowPlay(const FFootballPlay& Play, bool bHomeOffense);
```

`ShowPlay` animates a single play on the field. It makes no network call. It uses a play you already fetched with `FetchGameSummary`.

- `Play` is one entry from `FFootballGameData`. Its type and text decide what is shown: kickoff, punt, field goal, run, pass, or sack. Any other play type places the ball on its yard line.
- `bHomeOffense` says which team has the ball, so the actor knows which direction to move the play.

The actor animates the play itself every frame, so you only need to call `ShowPlay` once per play.

---

### Example (C++)

**Header (`MyActor.h`)**

```cpp
UFUNCTION()
void HandleGame(int32 EventId, const FFootballGameData& Game);

UPROPERTY(EditAnywhere)
AFootballFieldView* FieldView;
```

`HandleGame` must be a `UFUNCTION()` because `AddDynamic` only binds to reflected functions. `FieldView` is a reference to the `AFootballFieldView` placed in `L_Main`, which you assign in the editor.

**Implementation (`MyActor.cpp`)**

```cpp
UESPNFootballSubsystem* Feed = GetGameInstance()->GetSubsystem<UESPNFootballSubsystem>();
Feed->OnFootballGameReady.AddDynamic(this, &AMyActor::HandleGame);
Feed->FetchGameSummary(401628374);
```

This gets the subsystem from the game instance and subscribes `HandleGame` to the "game ready" event. It then requests the summary for ESPN event `401628374`. Bind first, then fetch, so the result can't arrive before the handler is registered.

```cpp
void AMyActor::HandleGame(int32 EventId, const FFootballGameData& Game)
{
    if (FieldView && Game.Plays.Num() > 0)
    {
        FieldView->ShowPlay(Game.Plays[0], /*bHomeOffense*/ true);
    }
}
```

This is the callback that runs when the data arrives. It checks that `FieldView` is valid and that the game has at least one play. It then shows the first play with the home team on offense. In real use, you would set `bHomeOffense` from the play's possession team instead of hardcoding `true`.
