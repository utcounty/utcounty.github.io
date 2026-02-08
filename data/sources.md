# **Data Sources**

## Utah VISTA Ballot Areas Precinct Boundaries:
Statewide ballot area polygons were downloaded from Utah’s Open Data Portal:

* **Dataset**: Utah VISTA Ballot Areas
* **URL**: https://opendata.gis.utah.gov/datasets/utah::utah-vista-ballot-areas/about
* **Original file**: VistaBallotAreas_-1176372527926423573.geojson

**Processing**:

* Load the statewide GeoJSON
* Filter to Utah County using `CountyID == 25`:
* Keep only required fields: `['CountyID', 'PrecinctID', 'SubPrecinctID', 'geometry']`
* Reproject for web mapping: `WGS84 (EPSG:4326) `
* Lightly simplify geometry for performance: `tolerance = 0.0001`

**Output**:
data/geo/utah_county_precincts.min.geojson

## 2025-11-04 Utah County Municipal General Election *(JSON data)*
* **Link**: https://electionresults.utah.gov/results/public/utah-county-ut/elections/general11042025/voters (Under: Utah Election Night Reporting Media Export)
* **File Name**: 20251104_utah_county_municipal_general_election.json
* **Original File Name**: export-general11042025.json

## 2025 Utah Municipal General Election: Voter Turnout *(HTML data)*
* **Link**: https://electionresults.utah.gov/results/public/utah-county-ut/elections/general11042025/voters
* **File Name**: utah-county-ut_elections_general11042025_voters_body.html

## 2025-08-12 Utah County Municipal Primary *(JSON data)*
* **Link**: https://electionresults.utah.gov/results/public/utah-county-ut/elections/primary08122025/voters (Under: Utah Election Night Reporting Media Export)
* **File Name**: 20250812_utah_county_municipal_primary_election.json
* **Original File Name**: export-primary08122025.json

## 2025 Utah Municipal Primary: Voter Turnout *(HTML data)*
* **Link**: https://electionresults.utah.gov/results/public/utah-county-ut/elections/primary08122025/voters
* **File Name**: utah-county-ut_elections_primary08122025_voters_body.html

## utah_county_elections.csv

This CSV is the output of **election_parsing.py**:

```python
from pathlib import Path
from bs4 import BeautifulSoup
import re
import pandas as pd
import numpy as np
import json

###########

def parse_election_html(path, election_name):
    soup = BeautifulSoup(Path(path).read_text(encoding="utf-8"), "html.parser")

    table = soup.select_one("table")

    def to_num(s: str):
        s = re.sub(r"[^0-9.-]", "", re.sub(r"[,\u00A0]", "", s.strip()))
        return None if not s or s == "." else (float(s) if "." in s else int(s))

    cols = [th.get_text(" ", strip=True) for th in table.select("thead th")][:4]

    rows = []
    for tr in table.select("tbody tr"):
        precinct = tr.find("th", scope="row").get_text(" ", strip=True)

        tds = tr.find_all("td")[:3]
        nums = [to_num(td.get_text(" ", strip=True)) for td in tds]

        rows.append([precinct, *nums])

    df = pd\
        .DataFrame(
            rows, 
            columns=["_".join(c.lower().split()) for c in cols])\
        .assign(election_name = election_name)\
        .rename(columns={
            "precinct": "precinct_code",
            "ballots_cast": "precinct_ballots_cast",
            "registered_voters": "precinct_registered_voters"})

    return df

def parse_election_json(path, county="Utah County"):

    JSON = json.loads(Path(path).read_text(encoding="utf-8-sig"))

    def as_list(x):
        return x if isinstance(x, list) else []

    rows = []

    for county in as_list(JSON.get("localResults")):
        if not isinstance(county, dict):
            continue

        county_id = county.get("id")
        county_name = county.get("name")

        for contest in as_list(county.get("ballotItems")):
            if not isinstance(contest, dict):
                continue

            contest_id = contest.get("id")
            contest_name = contest.get("name")
            contest_type = contest.get("type")
            contest_vote_for = contest.get("voteFor")
            contest_ballot_order = contest.get("ballotOrder")

            for opt in as_list(contest.get("ballotOptions")):
                if not isinstance(opt, dict):
                    continue

                opt_core = {
                    "election_date": JSON.get("electionDate"),
                    "election_name": JSON.get("electionName"),
                    "county_id": county_id,
                    "county_name": county_name,
                    "contest_id": contest_id,
                    "contest_name": contest_name,
                    "contest_type": contest_type,
                    "contest_vote_for": contest_vote_for,
                    "contest_ballot_order": contest_ballot_order,
                    "choice_id": opt.get("id"),
                    "choice_name": opt.get("name"),
                    "choice_ballot_order": opt.get("ballotOrder"),
                    "choice_votes": opt.get("voteCount"),
                    "choice_party": opt.get("politicalParty") or None,
                }

                for pr in as_list(opt.get("precinctResults")):
                    if not isinstance(pr, dict):
                        continue
                    rows.append({**opt_core, **pr})

    return pd\
        .DataFrame(rows)\
        .pipe(lambda df:
            pd.json_normalize(df.to_dict(orient="records"), sep="."))\
        .rename({
            "name": "precinct_code",
            "voteCount": "precinct_votes"}, axis=1)\
        .filter(items=[
            "election_date",
            "election_name",
            "county_name",
            "contest_name",
            "contest_type",
            "choice_name",
            "precinct_code",
            "precinct_votes"])

def add_turnout_metrics(df):
    return df\
    .assign(
        is_zero_registration_precinct = pd.col("voter_registration").eq(0),
        is_zero_ballots_precinct = pd.col("precinct_ballots_cast").eq(0),
        turnout_rate_precinct = pd.col("voter_turnout") / pd.col("voter_registration"))\
    .assign(
        turnout_rate_pct_precinct = pd.col("turnout_rate_precinct").rank(pct=True, method="average", na_option="keep"))

def add_election_metrics(df):
    return df\
        .assign(

            # --- Contest vote distribution (per contest × precinct) ---
            contest_total_votes=lambda d: d.groupby(KEYS_PC)["precinct_votes"].transform("sum"),

            choice_share=(
                (pd.col("precinct_votes") / pd.col("contest_total_votes"))
                .where(pd.col("contest_total_votes") > 0)),

            choice_rank=lambda d: d.groupby(KEYS_PC)["precinct_votes"].rank(method="dense", ascending=False),

            polarization_fragmentation=lambda d: (
                d.groupby(KEYS_PC)["precinct_votes"].transform(
                    lambda s: (1 - np.square(s / s.sum()).sum()) if s.sum() > 0 else np.nan)),

            # --- Contest engagement (ballots that participated in the contest) ---
            contest_participation_rate=(
                (pd.col("contest_total_votes") / pd.col("precinct_ballots_cast"))
                .where(pd.col("precinct_ballots_cast") > 0)),
            rolloff_count=(
                (pd.col("precinct_ballots_cast") - pd.col("contest_total_votes"))
                .where(pd.col("precinct_ballots_cast") > 0)),
            rolloff_rate=(
                ((pd.col("precinct_ballots_cast") - pd.col("contest_total_votes")) / pd.col("precinct_ballots_cast"))
                .where(pd.col("precinct_ballots_cast") > 0)),

            # --- Outcome summary (winner / runner-up / margins) ---
            winner_votes=lambda d: (
                d.groupby(KEYS_PC)["precinct_votes"].transform("max")
                .where(d["contest_total_votes"] > 0)),
            runnerup_votes=lambda d: (
                d.groupby(KEYS_PC)["precinct_votes"]
                .transform(lambda s: (s.nlargest(2).iloc[1] if len(s) > 1 else 0))
                .where(d["contest_total_votes"] > 0)),

            margin_votes=(
                (pd.col("winner_votes") - pd.col("runnerup_votes"))
                .where(pd.col("contest_total_votes") > 0)),
            margin_pct=(
                (pd.col("margin_votes") / pd.col("contest_total_votes"))
                .where(pd.col("contest_total_votes") > 0)),
            winner_share=(
                (pd.col("winner_votes") / pd.col("contest_total_votes"))
                .where(pd.col("contest_total_votes") > 0)),
            runnerup_share=(
                (pd.col("runnerup_votes") / pd.col("contest_total_votes"))
                .where(pd.col("contest_total_votes") > 0)),

            # --- Human-readable labels (ties become "A | B") ---
            winner_choice_name=lambda d: (
                d["choice_name"]
                .where(d["choice_rank"] == 1)
                .groupby([d[c] for c in KEYS_PC])
                .transform(lambda s: " | ".join(pd.unique(s.dropna())) if s.notna().any() else np.nan)),
            runnerup_choice_name=lambda d: (
                d["choice_name"]
                .where(d["choice_rank"] == 2)
                .groupby([d[c] for c in KEYS_PC])
                .transform(lambda s: " | ".join(pd.unique(s.dropna())) if s.notna().any() else np.nan)))\
        .sort_values(KEYS_PC + ["precinct_votes"], ascending=[True, True, True, True, False])

###########

df_primary_json = parse_election_json(r"20250812_utah_county_municipal_primary_election.json")\
        .loc[pd.col("county_name").str.contains("Utah", case=False, na=False)]

df_general_json = parse_election_json(r"20251104_utah_county_municipal_general_election.json")\
        .loc[pd.col("county_name").str.contains("Utah", case=False, na=False)]

df_primary_totals = parse_election_html(r"utah-county-ut_elections_primary08122025_voters_body.html", "2025 Utah Municipal Primary")\
    .pipe(add_turnout_metrics)

df_general_totals = parse_election_html(r"utah-county-ut_elections_general11042025_voters_body.html", "2025 Utah Municipal General Election")\
    .pipe(add_turnout_metrics)


###########

KEYS_PC = ["election_name", "county_name", "contest_name", "precinct_code"]

df_general = df_general_json\
    .merge(
        df_general_totals,
        how="left",
        on=["election_name", "precinct_code"])\
    .pipe(add_election_metrics)
    
df_primary = df_primary_json\
    .merge(
        df_primary_totals,
        how="left",
        on=["election_name", "precinct_code"])\
    .pipe(add_election_metrics)

###########

utah_county_elections = pd.concat([df_primary, df_general], ignore_index=True)

utah_county_elections.to_csv("utah_county_elections.csv", index=False)
```

#### Output column definitions:

**Grain:** Each row represents **one `choice_name` (candidate / YES / NO / etc.) within one `contest_name` within one `precinct_code`** for a given `election_name`.  
Because of that grain, precinct-wide totals (like registration, ballots cast) and contest-wide totals (like `contest_total_votes`) **repeat across multiple rows** (one per choice).

#### Election identifiers

- **`election_date`** *(str)*  
  Date of the election in `YYYY-MM-DD` form (as provided by the source).  
  Example: `2025-11-04`.

- **`election_name`** *(str)*  
  Human-readable election label (as provided by the source).  
  Example: `2025 Utah Municipal General Election`.

- **`county_name`** *(str)*  
  County label (`"Utah County"` here).


#### Contest & choice identifiers

- **`contest_name`** *(str)*  
  The race/measure name as shown in results.  
  Examples: `"Saratoga Springs Mayor"`, `"School Board Lake Mountain 5"`, `"Proposition #8 Eagle Mountain"`.

- **`contest_type`** *(str)*  
  High-level contest category from the source (often `"Local"` in municipal datasets).

- **`choice_name`** *(str)*  
  The selectable option within a contest (candidate name, `"YES"`, `"NO"`, write-in bucket, etc.).

#### Precinct identifiers

- **`precinct_code`** *(str)*  
  Precinct identifier used to join precinct-level results and turnout totals.  
  Example: `"25SR15"`.

#### Raw vote and turnout totals (from source / merge)

- **`precinct_votes`** *(int)*  
  Votes recorded **for this specific `choice_name`** in this precinct for this contest.  
  (This is the numerator for `choice_share`.)

- **`precinct_ballots_cast`** *(int)*  
  Total ballots cast in the precinct for the election (precinct-wide).  
  **Repeats** for all rows belonging to that precinct within the election.

- **`voter_turnout`** *(int)*  
  Turnout count for the precinct (precinct-wide).  
  In many exports this equals `precinct_ballots_cast`; if they differ, treat `voter_turnout` as the official turnout figure.

- **`voter_registration`** *(int)*  
  Registered voters in the precinct (precinct-wide).

#### Data quality flags

- **`is_zero_registration_precinct`** *(bool)*  
  Convenience flag:  
  `voter_registration == 0`  
  Useful to avoid divide-by-zero and to quickly locate placeholder/invalid precinct records.

- **`is_zero_ballots_precinct`** *(bool)*  
  Convenience flag:  
  `precinct_ballots_cast == 0`  
  Useful to identify precincts with no reported ballots or incomplete reporting.

#### Precinct-wide turnout rates (precinct-wide; repeats per row)

- **`turnout_rate_precinct`** *(float)*  
  Turnout fraction in the precinct:  
  `turnout_rate_precinct = voter_turnout / voter_registration`  
  Typically `NaN` (or undefined) if `voter_registration == 0`.

- **`turnout_rate_pct_precinct`** *(float)*  
  Percent form of turnout:  
  `turnout_rate_pct_precinct = 100 * turnout_rate_precinct`


#### Contest vote distribution (per contest × precinct; repeats per choice row)

- **`contest_total_votes`** *(int)*  
  Total votes recorded in this contest in this precinct, across all choices:  
  `contest_total_votes = Σ precinct_votes` over all `choice_name` within the same  
  (`election_name`, `county_name`, `contest_name`, `precinct_code`) group.

  **Interpretation note:** For multi-seat contests (e.g., “vote for up to 2”), `contest_total_votes` can be **greater than** `precinct_ballots_cast`.

- **`choice_share`** *(float)*  
  Share of the contest vote for this choice in this precinct:  
  `choice_share = precinct_votes / contest_total_votes` (when `contest_total_votes > 0`).

- **`choice_rank`** *(float)*  
  Rank of this choice by `precinct_votes` within the contest/precinct group (dense ranking, descending):  
  `choice_rank = rank_dense_desc(precinct_votes within group)`  
  - `1` = top vote-getter(s)  
  - Ties share the same rank  
  - Next rank increments by 1 (dense)

- **`polarization_fragmentation`** *(float)*  
  A concentration/fragmentation index derived from vote shares (Simpson-style):  
  Let `p_i = precinct_votes_i / contest_total_votes` for each choice in the group.  
  Then:  
  `polarization_fragmentation = 1 - Σ(p_i^2)`  (when `contest_total_votes > 0`)

  Interpretation:
  - Near **0** → highly concentrated (one option dominates)
  - Larger values → more fragmented (votes spread across options)
  - Maximum depends on number of choices (more choices can raise the maximum)

#### Contest engagement (ballots that “touched” the contest)

These are computed using `precinct_ballots_cast` as a denominator.

- **`contest_participation_rate`** *(float)*  
  Rate of ballots that recorded a vote in the contest (proxy):  
  `contest_participation_rate = contest_total_votes / precinct_ballots_cast` (when `precinct_ballots_cast > 0`)

  **Important interpretation note:**  
  This behaves as expected for **single-choice** contests (mayor, proposition yes/no, etc.).  
  For **multi-seat** contests, `contest_total_votes` can exceed `precinct_ballots_cast`, so this can be **> 1** and should be interpreted as “votes per ballot” rather than “participation.”

- **`rolloff_count`** *(float)*  
  Ballots that did *not* record a vote in this contest (proxy):  
  `rolloff_count = precinct_ballots_cast - contest_total_votes` (when `precinct_ballots_cast > 0`)

  **Note:** Can be **negative** in multi-seat contests (because `contest_total_votes` may exceed ballots cast).  
  If you want a “never negative” roll-off for mapping, a common variant is:  
  `rolloff_count_clipped = max(rolloff_count, 0)`.

- **`rolloff_rate`** *(float)*  
  Roll-off fraction:  
  `rolloff_rate = rolloff_count / precinct_ballots_cast` (when `precinct_ballots_cast > 0`)  
  Same caveat: may be negative for multi-seat contests.

#### Outcome summary (winner / runner-up / margins)

These values repeat for every `choice_name` row inside the same contest/precinct group.

- **`winner_votes`** *(float)*  
  Top vote total in the contest/precinct group:  
  `winner_votes = max(precinct_votes within group)` (when `contest_total_votes > 0`)

- **`runnerup_votes`** *(float)*  
  Second-highest vote total in the group:  
  `runnerup_votes = 2nd_largest(precinct_votes within group)` (when `contest_total_votes > 0`)

  Edge case:
  - If the contest has only one reported choice in the group, this may be set to `0` (or could be `NaN` depending on implementation preference).

- **`margin_votes`** *(float)*  
  Vote margin between the top two choices:  
  `margin_votes = winner_votes - runnerup_votes` (when `contest_total_votes > 0`)

- **`margin_pct`** *(float)*  
  Margin as a fraction of contest votes:  
  `margin_pct = margin_votes / contest_total_votes` (when `contest_total_votes > 0`)

- **`winner_share`** *(float)*  
  Winner’s share of the contest vote:  
  `winner_share = winner_votes / contest_total_votes` (when `contest_total_votes > 0`)

- **`runnerup_share`** *(float)*  
  Runner-up’s share of the contest vote:  
  `runnerup_share = runnerup_votes / contest_total_votes` (when `contest_total_votes > 0`)

- **`winner_choice_name`** *(str)*  
  Name(s) of the winning choice(s) in that contest/precinct group.  
  Computed by selecting `choice_name` where `choice_rank == 1`.  
  If there is a tie for first, names may be joined like: `"A | B"`.

- **`runnerup_choice_name`** *(object / str; may contain NaN)*  
  Name(s) of the 2nd-place choice(s) in that contest/precinct group.  
  Computed by selecting `choice_name` where `choice_rank == 2`.  
  Can be `NaN` when there is no runner-up (e.g., uncontested contests or only one reported choice).

**Note**:
For multi-seat contests (like many "City Council" contests), you may observe:
- `contest_total_votes > precinct_ballots_cast`
- therefore `rolloff_count < 0` and `rolloff_rate < 0`

This does **not** necessarily indicate bad data; it usually means the contest allows selecting multiple candidates (so ballots can contribute multiple votes).
