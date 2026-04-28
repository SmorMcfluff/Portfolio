
Anchor's Lament
===

<p align="center">
  <img src="AnchorsGif.gif"/>
</p>

[**Anchor's Lament**](https://store.steampowered.com/app/3831400/Anchors_Lament/) is a grid-based fish-themed Auto battler that I worked on during my time at [**Imperial Playgrounds**](https://imperialplaygrounds.com/).

## 0.0 - Table of Contents
- [1.0 – Backend Engineering](#backend-engineering)
  - [1.1 – Rank and In-Game Currency change upon combat end](#rank-and-in-game-currency-change-upon-combat-end)
  - [1.2 – Steam Microtransactions](#steam-microtransactions)
- [2.0 – New Mechanics](#new-mechanics)
  - [2.1 – Haste / Cold](#haste--cold)
- [3.0 – UI Design](#ui-design)


## 1.0 – Backend Engineering
>Note: All code in this section has been simplified, and fields have had their names changed in order to protect the integrity of the database.

My main responsibility at Anchor's Lament was to maintain and expand the backend functions of the game, which is hosted via Supabase. This was my first time working in Supabase, and with SQL, PL/pgSQL in general.

### 1.1 - Competitive Rank
<p align="center">
  <img src="Leaderboard.jpg" alt="A screenshot of the leaderboard during Season 1 of Anchor's Lament" style="width:75%; height:75%" />
</p>
<p align="center">
<i>Pictured: A screenshot of the leaderboard during Season 1 of Anchor's Lament</i>
</p>

I designed a server-authoritative competitive ranking system for the game, making sure to prevent malicious tampering. This system is mainly comprised of three functions:

### <center>1.1.1 – Fetching player rank on login</center>
We fetch the player's rank on login and store it locally strictly for display purposes, but we never modify this value client-side. In order for this to be displayed in the first place we have to make sure that the player *has* a rank to fetch, and if it doesn't we assume it is a new player and set their rank to a default value we have saved in a server-side settings table.

<details><summary>Fetch player rank if it exists – PL/pgSQL code</summary>

```sql
DECLARE
    v_default_rank INT;
    v_current_points INT;
BEGIN
    SELECT p.current_points
    INTO v_current_points
    FROM player_profiles p
    WHERE p.player_id = auth.uid();

    SELECT s.value::int
    INTO v_default_rank
    FROM settings_table s
    WHERE s.key = 'starting_rank'
    LIMIT 1;

    IF v_default_rank IS NULL THEN
        v_default_rank := 1000;
    END IF;

    IF v_current_points IS NULL OR v_current_points = 0 THEN
        UPDATE player_profiles p
        SET current_points = v_default_rank
        WHERE p.player_id = auth.uid()
        RETURNING current_points INTO v_current_points;
    END IF;

    RETURN v_current_points;
END;
```
</details>
<br>

### <center>1.1.2 – Updating rank on combat end</center>
Due to the asynchronous nature of the game, where you meet stored ghosts of other players rather than facing them directly, we decided against a dynamic Elo-like system and instead stuck to a fixed "get X points if you win, lose Y points if you lose" model. This simplified the implementation to:

1. Load the profile of the authenticated user (making sure there are no missing fields or rows, and failing early if there are).
2. Fetch the rank and currency delta from a server-side settings table.
3. Update the server-side player profile.
    - Apply the rank change (with a minimum floor).
    - Conditionally apply currency amount on wins.
4. Return the updated player state to the game client, for display purposes.

All this is executed in a single Postgres transaction, ensuring atomicity — either all of it occurs, or none of it does.

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

    UPDATE player_profiles p
    SET
        current_points = GREATEST(current_points + v_rank_delta, v_minimum_rank),
        wins = wins + CASE WHEN result = 'WIN' THEN 1 ELSE 0 END,
        losses = losses + CASE WHEN result = 'LOSS' THEN 1 ELSE 0 END,
        currency_balance = currency_balance + CASE WHEN result = 'WIN' THEN v_currency_gain ELSE 0 END
    WHERE p.player_id = auth.uid()
    RETURNING current_points INTO v_new_rank;

    RETURN jsonb_build_object(
        'rank', v_new_rank,
        'currency_balance',
        (
            SELECT COALESCE(currency_balance, 0)
            FROM player_profiles p
            WHERE p.player_id = auth.uid()
        )
    );
END;
-- This function assumes that the client reports each combat result exactly once. 
-- If called multiple times, they could lose or gain more points than they should. 
-- This would be much improved by having a table keyed on (player_id UUID, combat_result_id UUID), ensuring idempotency.
```
</details>
<br>

### <center>1.1.3 - Fetching leaderboard</center>

This is where it all comes together! In this function we return a single page of the leaderboard, as well as information on what position the local player (the one playing the game) has, so this can always be displayed in-game.

1. We sort every player on the database by their current rank points and store them in an array
2. From this array we get yet *another* array, limited to the size of the page (an argument passed in from the client), offset depending on which page we're currently on.
3. 

<details>
<summary>Fetch leaderboard and local player position – PL/pgSQL code</summary>

```sql
BEGIN
    RETURN QUERY
    WITH ranked_players AS (
        SELECT
            p.player_id,
            p.display_name,
            p.current_points::integer AS rank,
            ROW_NUMBER() OVER (
                ORDER BY p.current_points DESC, p.player_id
            )::integer AS leaderboard_position
        FROM player_profiles p
        WHERE p.current_points > 0
    ),
    page AS (
        SELECT
            rp.player_id,
            rp.display_name,
            rp.current_points,
            rp.leaderboard_position,
            (rp.player_id = auth.uid()) AS is_local_player
        FROM ranked_players rp
        ORDER BY rp.leaderboard_position
        LIMIT page_size
        OFFSET GREATEST(page_number, 0) * page_size
    ),
    local_player AS (
        SELECT
            rp.leaderboard_position AS player_position,
            ((rp.leaderboard_position - 1) / page_size)::integer AS player_page
        FROM ranked_players rp
        WHERE rp.player_id = auth.uid()
    )
    SELECT
        p.player_id,
        p.display_name,
        p.rank,
        p.leaderboard_position,
        lp.player_position,
        lp.player_page,
        p.is_local_player
    FROM page p
    CROSS JOIN local_player lp
    ORDER BY p.leaderboard_position;
END;
```
</details>

<hr>

## 1.2 Steam Microtransactions

## 2.0 – Game Mechanics
### 2.1 – Haste / Cold
<p>

</p>

## 3.0 – UI Design
<p>

</p>