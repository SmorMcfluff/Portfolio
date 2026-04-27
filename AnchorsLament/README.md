Anchor's Lament
===

<p align=center><img src="../AnchorsLament/AnchorsGif.gif"/></p>

[Anchor's Lament](https://store.steampowered.com/app/3831400/Anchors_Lament/) is a grid-based fish-themed Auto battler that I worked on during my time at [Imperial Playgrounds](https://imperialplaygrounds.com/).


## Backend Engineering
<p>
My main responsibility at Anchor's Lament was to maintain and expand the backend functions of the game, which is hosted via Supabase. This was my first time working with SQL, PL/pgSQL in general, and Supabase specifically.
</p>
  
<p>
Note: All code has been simplified, and fields have had their names changed in order to protect the integrity of the database.
</p>


### Rank and In-Game Currency change upon combat end
<p>
My first back-end focused assignment was to add a server-authoritative competitive ranking system to the game, making sure to prevent malicious tampering.
</p>
<p>
Due to the asynchronous nature of the game, where you meet stored ghosts of other players rather than facing them directly, we decided against a dynamic Elo-like system and instead stuck to a fixed "get X points if you win, lose Y points if you lose" model. This simplified the implementation to:
<ol>
    <li>Load the profile of the authenticated user (making sure there are no missing fields or rows, and resolving these issues if there are)</li>
    <li>Fetch the rank and currency delta from a server-side settings table</li>
    <li>Update the server-side player profile
        <ul>
            <li>Apply the rank change (with a minimum floor)</li>
            <li>Conditionally apply currency amount on wins</li>
        </ul>
    </li>
    <li>Return the updated player state to the game client, for display purposes</li>
</ol>

<p>
All this is executed in a single function call, ensuring atomicity — either all of it occurs, or none of it does.
</p>

<p>
When working in an online environment, it is very important to consider what should happen server-side and what happens client-side. In a competitive game such as this, the amount of coins and rank you gain upon winning should <i>never</i> be stored or handled on the client, which is why reward values are fetched from a server-side settings table at execution time.
</p>
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
<p>
Not represented in this code block, "player_profile" actually represents two separate player tables:
<ul>
    <li>Public info (rank, wins/losses)</li> 
    <li>Private info (currency)</li>
</ul>
</p>
<p>
The function assumes that the client reports each combat result exactly once. If called multiple times, they could lose or gain more points than they should. This could be improved by having a table keyed on (player_id UUID, combat_result_id UUID), making the function idempotent.
</p>
</details>

### Steam Microtransactions


## New Mechanics
### Haste / Cold
<p>

</p>

## UI Design
<p>

</p>