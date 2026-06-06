# 🧮 DAX Measures Documentation

**164 measures** organized into 17 display folders, all stored in a single central measure table (`main_Kpis`).

This document has two parts:
1. **Featured DAX patterns** — the techniques worth highlighting (with code).
2. **Complete measure catalog** — every measure by folder (name + description).

---

## Modeling conventions

- **Central measure table:** all measures live in `main_Kpis` and are grouped by display folder (`00_…` → `16_…`).
- **Filter-awareness:** KPIs count from **fact** tables (e.g. `player_Status`, `League_Table`) so they respond to slicers — counting from a dimension would stay constant.
- **Reliable sources only:** player stats come from **FBref** (`player_Status`); unreliable raw Wikipedia squad tables are not used.
- **90+ minutes threshold:** Club Profile player counts require ≥90 minutes for a consistent, meaningful 'squad'.
- **Color-as-measure:** several measures return hex strings consumed by conditional formatting (`fx → Field value`).
- **Text labels:** `… (Label)` measures use `FORMAT` to bypass a visual's Display Units.
- **HTML measures:** `… HTML` measures build markup for the *HTML Content* custom visual.

---

## 1. Featured DAX patterns

### `Total Players`
*Folder: 00_Overview Headlines*

```dax
DISTINCTCOUNT('player_Status'[PlayerKey])
```

> **Why it matters:** Filter-awareness fix: counting from a FACT table (player_Status) instead of the dimension, so the value reacts to Season/Club slicers (a dimension count would stay constant).

### `Clubs Count`
*Folder: 00_Overview Headlines*

```dax
DISTINCTCOUNT(League_Table[ClubKey])
```

> **Why it matters:** Same pattern — count distinct from the fact table so the KPI is season-aware.

### `Total League Goals`
*Folder: 00_Overview Headlines*

```dax
SUM(League_Table[Goals_For])
```

> **Why it matters:** Simple, reliable base measure built on the trustworthy League_Table source.

### `Season Champion`
*Folder: 00_Overview Headlines*

```dax
VAR _Raw = CALCULATE(SELECTEDVALUE(League_Table[Club], "Multiple"), League_Table[Position] = 1)
RETURN TRIM(SUBSTITUTE(SUBSTITUTE(_Raw, "(C)", ""), "(R)", ""))
```

> **Why it matters:** Cleans Wikipedia's '(C)'/'(R)' tags off the champion's name; SELECTEDVALUE returns 'Multiple' when more than one season is in context.

### `Total Attendance`
*Folder: 02_Attendance*

```dax
VAR _HasClubFilter = ISFILTERED(Dim_Club[Club])
RETURN
SUMX(
    VALUES(Seasons[Season]),
    VAR _S = Seasons[Season]
    VAR _PerSeason = CALCULATE(
        SUMX(Attendances, COALESCE(Attendances[Home_Games], 19) * Attendances[Average_Attendance])
    )
    VAR _Estimate =
        IF(NOT _HasClubFilter,
            SWITCH(_S, "2019–20", 8500000, "2020–21", 600000, BLANK()),
            BLANK())
    RETURN COALESCE(_PerSeason, _Estimate)
)
```

> **Why it matters:** Handles missing data gracefully: rebuilds totals from avg attendance × home games (COALESCE defaults to 19 games), and injects COVID-era estimates for 2019-20 / 2020-21 only when no club is filtered.

### `Squad Size`
*Folder: 04_Player Performance*

```dax
CALCULATE(DISTINCTCOUNT('player_Status'[PlayerKey]), 'player_Status'[Min] >= 90)
```

> **Why it matters:** A 90+ minutes threshold removes youth cameos so 'squad' means real contributors. This threshold is reused across all Club Profile counts for consistency.

### `Foreign Players`
*Folder: 12_Club Profile*

```dax
CALCULATE(DISTINCTCOUNT('player_Status'[PlayerKey]),
    'player_Status'[Nation] <> "England", NOT ISBLANK('player_Status'[Nation]), 'player_Status'[Min] >= 90)
```

> **Why it matters:** Multi-condition CALCULATE filter; same 90+ basis keeps English/Foreign/% internally consistent.

### `Foreign Players %`
*Folder: 12_Club Profile*

```dax
DIVIDE([Foreign Players], [Squad Size])
```

> **Why it matters:** DIVIDE avoids divide-by-zero; reuses base measures so the denominator always matches Squad Size.

### `Player Goals per 90`
*Folder: 10_Player Stats (FBref)*

```dax
DIVIDE(SUM('player_Status'[Goals]), SUM('player_Status'[90s Played]))
```

> **Why it matters:** Correct cross-season rate: sum goals and sum 90s first, THEN divide — never average per-season rates.

### `Points Last Season`
*Folder: 07_Advanced & Trends*

```dax
VAR _CurYear = SELECTEDVALUE(Seasons[Start_Year])
RETURN IF(NOT ISBLANK(_CurYear), CALCULATE([Total Points], FILTER(ALL(Seasons), Seasons[Start_Year] = _CurYear - 1)))
```

> **Why it matters:** Custom season-based time intelligence (the data isn't a contiguous date table, so standard time-intel functions don't apply).

### `Points YoY`
*Folder: 07_Advanced & Trends*

```dax
VAR _Prev = [Points Last Season]
RETURN IF(NOT ISBLANK(_Prev), [Total Points] - _Prev)
```

> **Why it matters:** Builds on the previous measure for a clean year-over-year delta.

### `Club Rank by Points`
*Folder: 07_Advanced & Trends*

```dax
IF(HASONEVALUE(Dim_Club[Club]), RANKX(ALL(Dim_Club[Club]), [Total Points], , DESC, Dense))
```

> **Why it matters:** RANKX with a HASONEVALUE guard so the rank only shows for a single club in context.

### `Title Margin`
*Folder: 13_Season Profile*

```dax
CALCULATE(MAX(League_Table[Points]), League_Table[Position] = 1)
  - CALCULATE(MAX(League_Table[Points]), League_Table[Position] = 2)
```

> **Why it matters:** Two position-filtered CALCULATEs measure how dominant the title win was.

### `Season Top Scorer`
*Folder: 13_Season Profile*

```dax
VAR _Max = MAXX(FILTER(VALUES(Dim_Player[Player]), NOT ISBLANK(Dim_Player[Player])), [Player Goals])
RETURN CALCULATE(SELECTEDVALUE(Dim_Player[Player]),
    FILTER(VALUES(Dim_Player[Player]), [Player Goals] = _Max && NOT ISBLANK(Dim_Player[Player])))
```

> **Why it matters:** Returns the name of the max-goals player; explicitly excludes the blank-player aggregate row that arises from unmatched keys.

### `Season Champion Logo`
*Folder: 13_Season Profile*

```dax
VAR _Champ = [Season Champion]
RETURN LOOKUPVALUE(Sheet1[Column2], Sheet1[Column1],
    SWITCH(_Champ, "West Bromwich Albion", "West Bromwich", "Wolverhampton Wanderers", "Wolves", _Champ))
```

> **Why it matters:** LOOKUPVALUE pulls a logo URL from the logos table; SWITCH reconciles two clubs whose names differ between sources. Tagged as ImageUrl so visuals render the picture.

### `POTY Goals`
*Folder: 14_Player Profile*

```dax
VAR _p = SELECTEDVALUE(Player_Of_Year[Player])
RETURN CALCULATE([Player Goals], Dim_Player[Player] = _p)
```

> **Why it matters:** Bridges a presentation table (Player_Of_Year) to the stats engine: read the selected player, then look up their real FBref goals.

### `Player Career Goals`
*Folder: 14_Player Profile*

```dax
CALCULATE(SUM('player_Status'[Goals]), ALL(Seasons))
```

> **Why it matters:** ALL(Seasons) removes the season filter to give a true career total while still respecting the player filter. (Replaced an earlier version that under-counted because it used the top-scorers-only table.)

### `Sack Rate %`
*Folder: 15_Manager Profile*

```dax
DIVIDE([Total Sackings], [Total Manager Changes])
```

> **Why it matters:** Composes two base measures into a rate; DIVIDE guards the denominator.

### `MOTS Points`
*Folder: 15_Manager Profile*

```dax
VAR _t = SELECTEDVALUE(Premier_League_Manafer_of_season[Team])
VAR _s = SELECTEDVALUE(Premier_League_Manafer_of_season[SeasonKey])
RETURN CALCULATE(SUM(League_Table[Points]),
    FILTER(ALL(League_Table), League_Table[ClubKey] = _t && League_Table[Season] = _s))
```

> **Why it matters:** Cross-table lookup: ties the Manager-of-the-Season's club + season to that club's actual league points.

### `Total Spend`
*Folder: 16_Transfer Profile*

```dax
CALCULATE(SUM(Transfers_In[Fee Value]), Transfers_In[Is Real Transfer] = TRUE())
```

> **Why it matters:** Sums the parsed numeric fee, filtered to real transfers (the Is Real Transfer flag excludes Wikipedia 'Total' summary rows that would double-count).

### `Net Spend`
*Folder: 16_Transfer Profile*

```dax
[Total Spend] - [Total Income]
```

> **Why it matters:** Reveals smart operators (net sellers) vs big buyers — more insightful than gross spend alone.

### `Club Bar Color`
*Folder: 01_Team Performance*

```dax
SWITCH(SELECTEDVALUE(Dim_Club[Club]),
    "Manchester City", "#85B7EB",
    "Manchester United", "#E8743B",
    "Chelsea", "#185FA5",
    "Liverpool", "#1D9E75",
    "Leicester City", "#534AB7",
    "#5DCAA5")
```

> **Why it matters:** A measure returning a hex color, used via conditional formatting (fx → Field value) to brand each champion's bar.

### `Spend Bar Color`
*Folder: 16_Transfer Profile*

```dax
VAR _tbl = ADDCOLUMNS(ALL(Seasons[Start_Year]), "@spend",
    VAR _y = Seasons[Start_Year]
    RETURN CALCULATE([Total Spend], ALL(Seasons), Seasons[Start_Year] = _y))
VAR _curYear = SELECTEDVALUE(Seasons[Start_Year])
VAR _cur = MAXX(FILTER(_tbl, [Start_Year] = _curYear), [@spend])
VAR _cnt = COUNTROWS(FILTER(_tbl, [@spend] > 0))
VAR _rank = COUNTROWS(FILTER(_tbl, [@spend] > _cur)) + 1
RETURN IF(ISBLANK(_curYear) || ISBLANK(_cur), BLANK(),
    SWITCH(TRUE(), _rank = 1, "#FFD700", _rank <= 4, "#E8743B", _rank >= _cnt - 3, "#1D5A55", "#378ADD"))
```

> **Why it matters:** Robust rank-based coloring that works no matter which season field is on the axis — it rebuilds each season's spend with ALL(Seasons) + a re-applied year filter, so the axis filter can't collapse the comparison.

### `Total League Goals (Label)`
*Folder: 00_Overview Headlines*

```dax
FORMAT([Total League Goals], "#,##0")
```

> **Why it matters:** Returns text so a card always shows the full number (e.g. 1,120) regardless of the visual's Display Units setting.

---

## 2. Complete measure catalog

### 00_Overview Headlines  ·  (9 measures)

| Measure | Description |
| --- | --- |
| `Total Seasons` | Number of seasons covered by the data |
| `Total League Goals` | Total league goals scored across all seasons in current context |
| `Total League Goals (Label)` | Goals as text with comma separator (always shows full number regardless of display units) |
| `Total Players` | Distinct players in context (Season/Club aware via player_Status fact) |
| `Clubs Count` | Distinct clubs in context (Season aware via League_Table) |
| `Season Champion` | Name of the champion club for the season in context |
| `Champion Short` | Short code of champion (MU, MCI, CHE, LIV, LEI, ARS) |
| `Champions Display` | Combined 'SeasonShort - ChampionShort' (e.g. '10-11 - MU') |
| `Champions Roll Call HTML` | HTML pill-per-season roll call (for HTML Content visual) |

### 01_Team Performance  ·  (14 measures)

| Measure | Description |
| --- | --- |
| `Matches Played` | Total matches played |
| `Wins` | Total wins |
| `Draws` | Total draws |
| `Losses` | Total losses |
| `Goals For` | Goals scored |
| `Goals Against` | Goals conceded |
| `Goal Difference` | Goals For minus Goals Against |
| `Total Points` | Total league points |
| `Win %` | Win percentage |
| `Points per Game` | Average points per game |
| `Best Position` | Best (lowest) final league position in context |
| `Worst Position` | Worst final league position in context |
| `Titles Won` | Number of times the club finished 1st |
| `Club Bar Color` | Dynamic title-race color, one per champion club |

### 02_Attendance  ·  (7 measures)

| Measure | Description |
| --- | --- |
| `Total Attendance` | Total attendance with COVID-era estimates injected (2019-20, 2020-21) |
| `Avg Home Attendance` | Average attendance per home game |
| `Highest Attendance` | Highest single-game attendance |
| `Lowest Attendance` | Lowest single-game attendance |
| `Home Games` | Number of home games |
| `Stadium Capacity` | Stadium seating capacity |
| `Fill Rate %` | Avg attendance as % of capacity |

### 03_Transfers  ·  (6 measures)

| Measure | Description |
| --- | --- |
| `Transfers In Count` | Players signed in |
| `Transfers Out Count` | Players sold/leaving |
| `Loans Out Count` | Players out on loan |
| `Net Player Movement` | In minus Out |
| `Free Transfers In` | Free signings |
| `Released Players` | Released/free exits |

### 04_Player Performance  ·  (10 measures)

| Measure | Description |
| --- | --- |
| `Top Scorer Goals` | Goals by the top scorer (Rank=1) |
| `Top Scorer Name` | Name of top scorer |
| `Top Goalkeeper Clean Sheets` | Clean sheets by top goalkeeper |
| `Top Goalkeeper Name` | Name of top goalkeeper |
| `Hat-Tricks Count` | Total hat-tricks |
| `Average Squad Age` | Average squad age (players with 90+ minutes) |
| `Squad Size` | Distinct players who played 90+ minutes (excludes youth cameos) |
| `Top Scorers Total Goals` | Total goals by all listed top scorers |
| `Club Top Scorer Goals` | Most goals by a player of the selected club (club-level) |
| `Club Top Scorer Name` | Name of the club's own top scorer |

### 05_Manager  ·  (6 measures)

| Measure | Description |
| --- | --- |
| `Managerial Changes Count` | Total managerial changes (season-level) |
| `Managers Sacked` | Sackings |
| `Managers Resigned` | Resignations |
| `Mutual Consent Departures` | Mutual-consent departures |
| `Avg Days to Replacement` | Avg days between manager leaving and replacement |
| `Avg Position at Change` | Avg league position at managerial change |

### 06_Disciplinary  ·  (2 measures)

| Measure | Description |
| --- | --- |
| `Disciplinary Players` | Distinct players in disciplinary record |
| `Disciplinary Records Count` | Total disciplinary records |

### 07_Advanced & Trends  ·  (13 measures)

| Measure | Description |
| --- | --- |
| `Goals per Game` | Goals scored per match |
| `Goals Conceded per Game` | Goals conceded per match |
| `Win-Loss Ratio` | Ratio of wins to losses |
| `Seasons Played` | Number of distinct seasons in context |
| `Avg Points per Season` | Average points per season |
| `Avg Goals per Season` | Average goals scored per season |
| `Points Last Season` | Points in the previous season (season time intelligence) |
| `Points YoY` | Year-over-year change in points vs previous season |
| `Position Change YoY` | Positions gained (+) or lost (-) vs previous season |
| `Club Rank by Points` | Club ranking by total points in context |
| `Club Rank by Goals` | Club ranking by goals scored in context |
| `Best Season Points` | Best single-season points total in context |
| `Relegations` | Seasons finishing in the relegation zone (18th or lower) |

### 08_Summary Cards  ·  (4 measures)

| Measure | Description |
| --- | --- |
| `Performance Tier` | Performance tier based on points per game |
| `Title Win Rate %` | Percentage of seasons won as champion |
| `Season Summary` | One-line season summary for card visuals |
| `Total Goals Involved` | Headline KPI: goals scored + conceded |

### 09_Player Profile  ·  (7 measures)

| Measure | Description |
| --- | --- |
| `Player Career Goals (PL)` | Goals across seasons the player was a league top scorer (Top_Scorers source) |
| `Player Golden Boots` | Seasons finished as league top scorer (Rank=1) |
| `Player Best Season Goals` | Player's best single-season goal tally |
| `Player Top-Scorer Seasons` | Seasons appearing in the league top-scorers list |
| `Player Hat-Tricks` | Total hat-tricks scored by the player |
| `Player Transfers In` | Transfers in involving the player |
| `Player Transfers Out` | Transfers out involving the player |

### 10_Player Stats (FBref)  ·  (13 measures)

| Measure | Description |
| --- | --- |
| `Player Goals` | Player goals from FBref Standard Stats (reliable) |
| `Player Assists` | Player assists from FBref |
| `Player G+A` | Goal contributions (G+A) |
| `Player Minutes` | Minutes played |
| `Player Matches` | Matches played |
| `Player Starts` | Starts |
| `Player Yellow Cards` | Yellow cards |
| `Player Red Cards` | Red cards |
| `Player Penalty Goals` | Goals from penalty kicks |
| `Player Non-Penalty Goals` | Non-penalty goals |
| `Player Goals per 90` | Goals per 90 (proper cross-season aggregation) |
| `Player Assists per 90` | Assists per 90 |
| `Player Goals Bar Color` | Dynamic top-scorer bar color (#1 dark, rest light) |

### 11_Wages  ·  (4 measures)

| Measure | Description |
| --- | --- |
| `Player Weekly Wage` | Player weekly wage in GBP |
| `Player Annual Wage` | Player annual wage in GBP |
| `Squad Weekly Wage Bill` | Total weekly wage bill for the squad |
| `Squad Annual Wage Bill` | Total annual wage bill for the squad |

### 12_Club Profile  ·  (16 measures)

| Measure | Description |
| --- | --- |
| `English Players` | English players who played 90+ minutes |
| `Foreign Players` | Foreign (non-English) players who played 90+ minutes |
| `Foreign Players %` | Percentage of foreign players (consistent 90+ basis) |
| `Nationalities Count` | Distinct nationalities among 90+ minute players |
| `Goalkeepers Count` | Goalkeepers who played 90+ minutes |
| `Defenders Count` | Defenders who played 90+ minutes |
| `Midfielders Count` | Midfielders who played 90+ minutes |
| `Forwards Count` | Forwards who played 90+ minutes |
| `Avg Player Minutes` | Average minutes played per player |
| `Top Wage Earner Name` | Name of the highest-paid player in the squad |
| `Top Wage Earner Wage` | Weekly wage of the highest-paid player |
| `Club Total Goals` | Total goals by all players in the squad |
| `Club Total Assists` | Total assists by all players in the squad |
| `Club Yellow Cards` | Total yellow cards received by the squad |
| `Club Red Cards` | Total red cards received by the squad |
| `Current League Position` | Current league position (one season selected) |

### 13_Season Profile  ·  (12 measures)

| Measure | Description |
| --- | --- |
| `Champion Points` | Points of the champion (1st place) in the season |
| `Goals Per Match` | Average goals scored per match across the season |
| `Title Margin` | Champion points minus runner-up points |
| `Season Top Scorer` | Name of the season's top scorer (excludes blanks) |
| `Season Top Scorer Goals` | Goals by the season's top scorer (excludes blanks) |
| `Season Top Assister` | Name of the season's top assist provider (excludes blanks) |
| `Season Top Assister Assists` | Assists by the season's top assister (excludes blanks) |
| `Best Attack Goals` | Highest goals-for by any club (best attack) |
| `Best Defense GA` | Lowest goals-against by any club (best defense) |
| `Avg Match Attendance` | Average match attendance across the season |
| `Season Champion Logo` | Logo URL of the season champion (ImageUrl measure) |
| `Season Pts Bar Color` | Dynamic Points-by-Club color: gold champion, mint UCL, orange relegation |

### 14_Player Profile  ·  (12 measures)

| Measure | Description |
| --- | --- |
| `POTY Name` | Player of the Year name for the season |
| `POTY Image` | Player of the Year image URL (ImageUrl) |
| `POTY Club` | Club of the Player of the Year |
| `POTY Goals` | Goals by the Player of the Year that season |
| `POTY Assists` | Assists by the Player of the Year that season |
| `POTY Matches (Label)` | Matches by the Player of the Year (text label) |
| `POTY Minutes` | Minutes by the Player of the Year that season |
| `POTY Minutes (Label)` | POTY minutes as text (full number regardless of display units) |
| `Top5 Player Goals` | Goals per player in the Top-5 scorers list |
| `Top5 Player Club` | Club per player in the Top-5 scorers list |
| `Top5 Cards HTML` | HTML cards for Top-5 scorers (photo + name + goals) |
| `Player Career Goals` | True total PL career goals across all seasons (FBref) |

### 15_Manager Profile  ·  (16 measures)

| Measure | Description |
| --- | --- |
| `Total Manager Changes` | Total managerial changes in scope |
| `Total Sackings` | Total managers sacked |
| `Sack Rate %` | Percentage of changes that were sackings |
| `Avg Changes per Season` | Average managerial changes per season |
| `Distinct Managers` | Distinct managers appointed |
| `Manager Appointments` | Times a manager was appointed (group by Incoming_Manager) |
| `Manager Departures` | Times a manager left (group by Outgoing_Manager) |
| `Manager Times Sacked` | Times a manager was sacked (group by Outgoing_Manager) |
| `MOTS Name` | Manager of the Season name |
| `MOTS Image` | Manager of the Season image (ImageUrl) |
| `MOTS Club` | Club of the Manager of the Season |
| `MOTS Points` | League points of the MOTS club that season |
| `MOTS Position` | Final league position of the MOTS club |
| `MOTS Wins` | Wins of the MOTS club that season |
| `MOTS Goals For` | Goals scored by the MOTS club that season |
| `Changes per Season HTML` | HTML overlay column chart (changes + sackings, peak gold) |

### 16_Transfer Profile  ·  (12 measures)

| Measure | Description |
| --- | --- |
| `Total Spend` | Total incoming transfer spend in GBP (real transfers only) |
| `Total Income` | Total outgoing sales income in GBP (real transfers only) |
| `Net Spend` | Spend minus income (positive = net buyer) |
| `Record Signing Fee` | Record single incoming transfer fee |
| `Record Sale Fee` | Record single outgoing sale fee |
| `Signings Count` | Number of incoming transfers (real, any fee type) |
| `Paid Signings Count` | Incoming transfers with a real paid fee |
| `Avg Transfer Fee` | Average paid transfer fee (incoming) |
| `Free Signings Count` | Free transfers / free agent signings |
| `Net Spend Bar Color` | Net-spend color: orange buyer, mint seller |
| `Total Spend (Label)` | Total spend as text in millions (e.g. 1,521M) |
| `Spend Bar Color` | Dynamic spend-by-season color: gold peak, orange top 4, muted lowest 4, blue rest |

---

*Total: 163 measures across 17 folders. Generated from the live Power BI model.*