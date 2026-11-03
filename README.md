# Iteration 1

## API

All football data comes from ESPN's free public API (no key needed). Every request is asynchronous: call a `Fetch` function, then receive the result through an event. Works in Blueprint or C++.

---

### Games on a date

```cpp
void FetchGamesForDate(const FString& DateString = "");
FOnGamesParsed OnGamesParsed;
```

`FetchGamesForDate` gets all college football games for one day. Pass `"YYYY-MM-DD"`, or leave it empty for today (UTC). `OnGamesParsed` fires when the games are ready.

Each game includes its `EventId`, start time, status, teams, score, venue, and broadcast info. Use `EventId` to fetch plays.

---

### One game's plays

```cpp
void FetchGameSummary(int32 EventId);
FOnFootballGameReady OnFootballGameReady;   // (int32 EventId, const FFootballGameData& Game)
```

`FetchGameSummary` gets the play-by-play for one game. `OnFootballGameReady` fires when it's ready, giving you the `EventId` and the full game data.

Each play includes its type, description, quarter, clock, down and distance, yards gained, team with the ball, and a scoring flag.

---

### Show a play on the field

```cpp
void ShowPlay(const FFootballPlay& Play, bool bHomeOffense);
```

`ShowPlay` animates one play on `AFootballFieldView`. It makes no network call. Pass a play from the game data, and set `bHomeOffense` to say which team has the ball. Kickoffs, punts, field goals, runs, passes, and sacks have their own animations. Anything else just places the ball on its yard line.

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
