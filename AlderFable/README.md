# AlderFable

A 2D platformer developed in under two weeks for the [2025 Stop Killing Games Community Jam](https://itch.io/jam/stop-killing-games-game-jam). 

Heavily inspired by MapleStory, the idea was to tell the story of a dying MMORPG. The story I wanted to tell would turn out to be much too ambitious for me to have time to develop, so the released game is little more than a tutorial with a comedic twist ending. But that doesn't mean there's nothing I'm proud of! 

## NPC Pathfinding
<p align="center"><img src="https://github.com/SmorMcfluff/Portfolio/blob/main/AlderFable/AlderFable%20Pathfinding.gif"/></p>
<p align="center"><i>The NPC finds the closest enemy, taking into account the time it takes to climb a ladder.</i></p>

Every physics tick, we check 

<details>
  <summary>GetNearestEnemy()</summary>
  
```cs
private float GetEnemyDistance(IDamageable damageable)
{
    EnemyController enemy = (damageable as Component).GetComponent<EnemyController>();
    float transformXPos = transform.position.x;
    float enemyXPos = enemy.transform.position.x;

    if (enemy.movement.currentPlatform == movement.currentPlatform)
    {
        return Mathf.Abs(enemyXPos - transformXPos);
    }

    List<Platform> platformPath = FindPlatformPath(movement.currentPlatform, enemy.movement.currentPlatform);
    if (platformPath == null || platformPath.Count < 2)
    {
        return float.MaxValue;
    }

    float totalDistance = 0;
    Vector2 currentPos = transform.position;

    for (int i = 0; i < platformPath.Count - 1; i++)
    {
        Platform currentPlatform = platformPath[i];
        Platform nextPlatform = platformPath[i + 1];

        var ladders = currentPlatform.ladders
            .Where(l => l.topPlatform == nextPlatform || l.bottomPlatform == nextPlatform);

        if (!ladders.Any())
        {
            return float.MaxValue;
        }

        Ladder bestLadder = ladders
            .OrderBy(l => Mathf.Abs(l.transform.position.x - currentPos.x))
            .First();

        float ladderXPos = bestLadder.transform.position.x;
        float ladderHeight = bestLadder.Height;

        totalDistance += Mathf.Abs(currentPos.x - ladderXPos);
        totalDistance += ladderHeight;

        currentPos = new Vector2(ladderXPos, nextPlatform.transform.position.y);
    }

    totalDistance += Mathf.Abs(enemyXPos - currentPos.x);
    return totalDistance;
}
```

</details>
