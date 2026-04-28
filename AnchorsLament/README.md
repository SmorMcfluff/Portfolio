
Anchor's Lament
===

<p align="center">
  <img src="AnchorsGif.gif"/>
</p>

[**Anchor's Lament**](https://store.steampowered.com/app/3831400/Anchors_Lament/) is a grid-based fish-themed Auto battler that I worked on during my time at [**Imperial Playgrounds**](https://imperialplaygrounds.com/).

## 0.0 - Table of Contents
- [1.0 - Back-end Engineering](#back-end-engineering)
  - [1.1 - Rank and In-Game Currency change upon combat end](#rank-and-in-game-currency-change-upon-combat-end)
  - [1.2 - Steam Microtransactions](#steam-microtransactions)
- [2.0 - New Mechanics](#new-mechanics)
  - [2.1 - Haste / Cold](#haste--cold)
- [3.0 - UI Design](#ui-design)


## 1.0 - Back-end Engineering
>Note: All code in this section has been simplified, and fields have had their names changed in order to protect the integrity of the database.

My main responsibility at Anchor's Lament was to maintain and expand the back-end functions of the game, which is hosted via Supabase. This was my first time working in Supabase, and with SQL, PL/pgSQL in general.

### 1.1 - Competitive Rank
<p align="center">
  <img src="Leaderboard.jpg" alt="A screenshot of the leaderboard during Season 1 of Anchor's Lament" style="width:75%; height:75%" />
</p>
<p align="center">
<i>Pictured: A screenshot of the leaderboard during Season 1 of Anchor's Lament</i>
</p>

My first back-end focused assignment was to add a server-authoritative competitive ranking system to the game, making sure to prevent malicious tampering. This system can be summarized by three back-end functions.

### <center>1.1.1 - Fetching player rank on login</center>
### 1.1.1 - Fetching player rank on login

| | |
|---|---|
| We fetch the player's rank on login and store it locally strictly for display purposes, but we never modify this value client-side. In order for this to be diplayed in the first place, we have to make sure that the player *has* a rank to fetch. | <center><img src="RankIcon.jpg" width="500"/></center> |

I made the decision that every player should start at 1000 points, placing them at the very bottom of "Deckhand" rank, our equivalent to a Silver tier.

### <center>1.1.2 - Updating rank on combat end</center>
Due to the asynchronous nature of the game, where you meet stored ghosts of other players rather than facing them directly, we decided against a dynamic Elo-like system and instead stuck to a fixed "get X points if you win, lose Y points if you lose" model. This simplified the implementation to:

1. Load the profile of the authenticated user (making sure there are no missing fields or rows, and failing early if there are).
2. Fetch the rank and currency delta from a server-side settings table.
3. Update the server-side player profile.
    - Apply the rank change (with a minimum floor).
    - Conditionally apply currency amount on wins.
4. Return the updated player state to the game client, for display purposes.

All this is executed in a single function call, ensuring atomicity — either all of it occurs, or none of it does.

When working in an online environment, it is very important to consider what should happen server-side and what happens client-side. In a competitive game such as this, the amount of coins and rank you gain upon winning should *never* be stored or handled on the client, which is why reward values are fetched from a server-side settings table at execution time.

<details><summary>Report Combat Result – PL/pgSQL code and commentary</summary>

```sql
DECLARE
  v_rank_delta INT := 0;
  v_new_rank INT;
  v_currency_gain INT;
  v_minimum_rank INT := 100;
BEGIN
    -- Load authenticated player profile, validate required fields
    -- Fetch rank delta and currency gain from server-side settings

    UPDATE player_profile
    SET
        current_rank = GREATEST(current_rank + v_rank_delta, v_minimum_rank),
        wins = wins + CASE WHEN result = 'WIN' THEN 1 ELSE 0 END,
        losses = losses + CASE WHEN result = 'LOSS' THEN 1 ELSE 0 END,
        currency_balance = currency_balance + CASE WHEN result = 'WIN' THEN v_currency_gain ELSE 0 END
    WHERE player_id = auth.uid()
    RETURNING current_rank INTO v_new_rank;

    RETURN jsonb_build_object(
        'rank', v_new_rank,
        'currency_balance',
        (
            SELECT COALESCE(currency_balance, 0)
            FROM player_profile
            WHERE player_id = auth.uid()
        )
    );
END;
```
</details>
<br>
Not represented in the above code, "player_profile" actually represents two separate player tables:

- Public info (rank, wins/losses)
- Private info (currency)
The function assumes that the client reports each combat result exactly once. If called multiple times, they could lose or gain more points than they should. This could be improved by having a table keyed on (player_id UUID, combat_result_id UUID), making the function idempotent.
<hr>

### <center>Steam Microtransactions</center>


## New Mechanics
### Haste / Cold
<p>

</p>

## UI Design
<p>

</p>