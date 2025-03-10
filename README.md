This application will scrape data from the Riot Games API to collect data on Swiftplay and Ranked games for League of Legends. To do this, the program will perform a **breadth-first search** of Swiftplay players, where each node in a graph is a Swiftplay player and each match played between them is an edge. The program will collect relevant data for each player and their neighbors (i.e. players they’ve played swiftplay games with).

This program will be a Python program that uses pandas to save data into three different files:

1. SwiftplayPlayerMatchData.csv - Contains match data for each player in each Swiftplay match, including teammates who are not explicitly enqueued in BFS (one player per match per line)
2. RankedPlayerMatchData.csv - Contains match data for each player in each Ranked match, including teammates who are not explicitly enqueued in BFS (one player per match per line)
3. SwiftplayMatches.csv - Contains aggregate match data for each Swiftplay match (one match per line)
4. RankedMatches.csv - Contains aggregate match data for each Ranked match (one match per line)
5. Players.csv - Contains aggregate data for each player on their Swiftplay/Ranked match performance (one player per line)

**Initiating the Breadth-First Search**:

The Python program will perform a **stratified random sample** of players, randomly sampling from a division (I, II, III, or IV) and tier (DIAMOND, EMERALD, PLATINUM, SILVER, BRONZE, IRON) combination specified by the user at the command line. This allows for the program to be run multiple times with different parameters to collect data on different player populations. The process is described below:

1. Use the /lol/league/v4/entries/{queue}/{tier}/{division} endpoint in LEAGUE-V4 to get a page of summoners of that rank.
    1. Use the NA1 region and choose a random page number between 1 and 17
    2. Example: <https://na1.api.riotgames.com/lol/league/v4/entries/RANKED_SOLO_5x5/DIAMOND/I?page=17&api_key=RGAPI-6d19754c-f0d2-4ecd-8cca-b9c460e0373d>
2. This returns a list where each element is a LeagueEntryDto as a JSON object. For each entry in the list:
    1. Get the “puuid” attribute.
    2. Use the /lol/match/v5/matches/by-puuid/{puuid}/ids in MATCH-V5 to get a list of matches that the player has played in Swiftplay and Ranked. Set “count” to the maximum value (100).
        1. To get Swiftplay matches, set the “queue” parameter to 480, and do not set the “type” parameter. Example: <https://americas.api.riotgames.com/lol/match/v5/matches/by-puuid/\_tMARx_TCchy-D3Na4_CTL95lcGeQeML7c2eYv_xlw93YMOcMoc69cID_jPXJ-j7vfxOnGKPeTGHzA/ids?queue=480&start=0&count=100&api_key=RGAPI-6d19754c-f0d2-4ecd-8cca-b9c460e0373d>
        2. To get Ranked matches, set the “type” parameter to “ranked” and do not set the “queue” parameter. Example: <https://americas.api.riotgames.com/lol/match/v5/matches/by-puuid/\_tMARx_TCchy-D3Na4_CTL95lcGeQeML7c2eYv_xlw93YMOcMoc69cID_jPXJ-j7vfxOnGKPeTGHzA/ids?type=ranked&start=0&count=100&api_key=RGAPI-6d19754c-f0d2-4ecd-8cca-b9c460e0373d>
3. This endpoint returns a list of match IDs. Check that both lists returned by the MATCH-V5 endpoint are not empty (i.e. the player has played both Swiftplay and Ranked games) and that the player PUUID is not in Players.csv already (i.e. the player has not been searched before). If so, they are a valid candidate to start the BFS. Initiate the BFS (using a separate BFS function, for example) and then, once the BFS terminates, “continue” the loop so that the next BFS occurs with a player of a different division/tier.
    1. The BFS will keep track of visited players using a hash set of PUUIDs. Add the player’s PUUID to the set and pass the hash set to the BFS function as well.
4. If the current page of LeagueEntryDtos does not have any valid candidates, choose a new random number from 1 to 17 and a new page from the API.

**The Breadth-First Search**

Ideally a separate function, this function will initialize a queue with the first player’s PUUID and then loop until the queue is empty. The process is as follows:

1. Add the first PUUID to the queue and ensure that it is marked as visited in the hash set
2. Initialize the number of iterations to 0 and the addToQueue flag to True
3. While the queue is not empty:
    1. Increment the number of iterations. Log the current iteration and queue size in the console.
    2. Pop a PUUID off the queue.
    3. If the queue size is exceeds 1000, set the addToQueue flag to False
    4. **Collect Swiftplay and Ranked Data for the current player:**
        1. Match data for all teammates in a Swiftplay match will be recorded in SwiftplayPlayerMatchData.csv, but only the initially sampled player and explicitly enqueued BFS players will be added to Players.csv.
        2. If the PUUID is already in Players.csv, data has already been collected for this player, so no more data needs to be collected. Add the player’s Swiftplay teamates (only if addToQueue is True) and continue to the next iteration of the while loop.
        3. Use the MATCH-V5 endpoint to gather the player’s list of Swiftplay games, as described previously. Then, for each Swiftplay game:
            1. Then, use the /lol/match/v5/matches/{matchId} endpoint from MATCH-V5 to gather data for each match ID. An example URL would be <https://americas.api.riotgames.com/lol/match/v5/matches/NA1_5234306555?api_key=RGAPI-36f4e841-5557-42d8-b628-c67dbe334574>
            2. All necessary data will be under the “info” key as an InfoDto.
            3. Populating SwiftplayPlayerMatchData.csv: the “participants” key contains a list of ParticipantDtos that will be where the majority of the necessary data can be found. Collect the following data to be placed in the CSV for ALL teammates in the match. Check beforehand if the data already exists in SwiftplayPlayerMatchData.csv (i.e. this match has already been processed in a previous iteration of the BFS):
                1. Match ID
                2. PUUID
                3. teamId
                4. summonerLevel
                5. role
                6. teamPosition
                7. kills
                8. deaths
                9. assists
                10. championId
                11. win (boolean)
                12. teamEarlySurrendered
                13. totalTimeSpentDead
                14. timePlayed
                15. longestTimeSpentLiving
                16. tier

                Use the /lol/league/v4/entries/by-puuid/{encryptedPUUID} endpoint from LEAGUE-V4 with the NA1 region.

                Example: <https://na1.api.riotgames.com/lol/league/v4/entries/by-puuid/fTvvqhF51hJ1-IBvsIKulqpoukRCCOs0eov06kYV6La5gxhlBiME1U0Ut9Uw3rjP0OAnttZuOOFWFA?api_key=RGAPI-36f4e841-5557-42d8-b628-c67dbe334574>

                17. rank

                This is gathered the same way that the player’s tier is

                18. Champion Level

                Use the /lol/champion-mastery/v4/champion-masteries/by-puuid/{encryptedPUUID}/by-champion/{championId} endpoint from CHAMPION-MASTERY-V4 with the NA1 region

                Example: <https://na1.api.riotgames.com/lol/champion-mastery/v4/champion-masteries/by-puuid/fTvvqhF51hJ1-IBvsIKulqpoukRCCOs0eov06kYV6La5gxhlBiME1U0Ut9Uw3rjP0OAnttZuOOFWFA/by-champion/56?api_key=RGAPI-36f4e841-5557-42d8-b628-c67dbe334574>

                19. Champion Points

                This is gathered the same way that the champion level is

            4. Populating SwiftplayMatches.csv: Most of the necessary data will be in the “info” key.
                1.  gameDuration
                2.  endOfGameResult
                3.  gameEndedInSurrender (grab this from one of the participantDtos)
                4.  gameEndedInEarlySurrender (also grab this from one of the participantDtos)
            5.  If the addToQueue flag is true, add any participant PUUIDs to the queue for each Swiftplay (NOT ranked) match
                1.  This should be done as data is getting collected, using the list of PUUIDs under the “metadata” and then “participants” key from the MATCH-V5 endpoint, so that repeat API calls are not needed
                2.  Following the rules of a BFS, add to the queue if a given PUUID does not appear in the hash set of visited players. When adding to the PUUID to the queue, add the PUUID to the hash set as well

      1. Populating Players.csv: This data has all been collected before and just needs to be placed into the file. Remember that only players explicitly enqueued in the BFS will be added to Players.csv:
            1. PUUID
            2. summonerLevel
            3. Tier
            4. Rank
            5. Total Swiftplay kills (sum up all of the kills from all matches)
            6. Total Swiftplay deaths (sum up all of the deaths from all matches)
            7. Total Swiftplay deaths (sum up all of the deaths from all matches)
            8. Swiftplay K/D = total kills / total deaths
            9. Swiftplay A/D = total kills / total assists
            10. KDA - Is (Kills + (Assists / 2)) / Deaths.
            11. KAD - Is (Kills + Assists) / Deaths
            12. Win/Loss ratio
            13. Note for any calculations: If a player has 0 deaths or 0 losses, calculate with the number of deaths/losses as 1
      2. Use the MATCH-V5 endpoint to gather the player’s list of Ranked games, as described previously.
            1. Collect the same data for Ranked games as Swiftplay games, populating the RankedPlayerMatchData.csv and RankedMatches.csv instead.
            2. Additionally, record the player’s total kills, total assists, total deaths, K/D, A/D, KDA, and KAD (for Ranked matches specifically) in Players.csv. However, NEVER add the teammates to the BFS queue.
            3. If a player has no Ranked games played, mark their Ranked statistics in Players.csv with np.nan
      3. Add all newly-gathered data to their respective CSV files. Updating the files frequently ensures that no data is lost if the program terminates unexpectedly. Also save the queue PUUIDs to a new file for the same reason.

**Running the Program**:

The program can be run in two modes:

1. Start new collection from a specific tier/division:

```bash
python main.py --tier <tier> --division <division>
```

2. Resume collection from the last saved state:

```bash
python main.py --resume
```

This will resume the BFS using the saved queue from a previous run. This is useful if the program was interrupted and you want to continue from where it left off.

Valid tiers are: DIAMOND, EMERALD, PLATINUM, GOLD, SILVER, BRONZE, IRON
Valid divisions are: I, II, III, IV

**Error Recovery:**
The program automatically saves the BFS queue state after processing each player. If the program terminates unexpectedly, you can resume the collection process using the `--resume` flag, which will:
1. Load the saved queue state
2. Load the set of already processed players from Players.csv
3. Continue the BFS from where it left off


**Stopping the Program:**
Manually terminate the program by pressing Ctrl+C. This will save the BFS queue state and the set of already processed players to Players.csv, allowing you to resume the collection process later. It is recommended to only stop the program when the program is sleeping due to the rate limit.

**General Notes**:

In order to monitor the execution of the code, the code should include frequent print statements explaining exactly what stop of the program is being executed. There should be logging for what rank and tier is currently being explored in the stratified random sample, what iteration is being executed, and what data is being queried from the API.

Additionally, a crucial element of this program is ensuring that the Riot API limit is not reached. The limit is 20 requests every 1 seconds(s) and 100 requests every 2 minutes. The program should keep track of the timestamps of each request so that the rate limits are obeyed. For example, this could be done by sleeping for 2 minutes after every 100 requests. When the program is sleeping, this behavior should be logged in the console.

In general, the program should retry every failed request (i.e. a non-200 response code) until a valid response (i.e. 200 response code) is given.