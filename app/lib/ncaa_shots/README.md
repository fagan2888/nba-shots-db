# Supplemental NCAA shots data

This subfolder contains a collection of ad hoc scripts used to process NCAA shots data and merge it with the NBA shots data.

NCAA men's college basketball shot data is available via [Sportradar on Google BigQuery](https://console.cloud.google.com/launcher/details/ncaa-bb-public/ncaa-basketball).

## How to get NCAA shots data

You'll need to set up a Google Cloud account to use BigQuery and Google Cloud Storage. Visit https://bigquery.cloud.google.com/dataset/bigquery-public-data:ncaa_basketball and follow any instructions to set up your account before it will let you write a query.

Generate the NCAA shots data with the following query and save the results to a new destination table:

```sql
SELECT *
FROM [bigquery-public-data:ncaa_basketball.mbb_pbp_sr]
WHERE type = 'fieldgoal'
  AND event_coord_x IS NOT NULL
  AND event_coord_y IS NOT NULL
```

Before you can export the data from BigQuery to a .csv file, you'll need to set up a bucket on Google Cloud Storage. Follow the instructions here: https://cloud.google.com/bigquery/docs/exporting-data

Once the .csv file is available on Google Cloud Storage, download and save it to `app/lib/ncaa_shots/data/ncaa_shots.csv`

## Import NCAA shots data into NBA Shots DB

Once you've downloaded the NCAA shots data to `app/lib/ncaa_shots/data/ncaa_shots.csv`, run from the command line:

`psql nba-shots-db_development -f import_ncaa_shots.sql`

Which will create and populate the `ncaa_shots`, `ncaa_players`, and `players_mapping` tables.

## Merge NCAA and NBA shots

After you've imported the NCAA shots data, you can run from the command line:

`psql nba-shots-db_development -f merge_nba_and_ncaa_shots.sql`

Which will create and populate the `merged_shots` table. This table contains both NBA and NCAA shots, standardized so that their x-y coordinates are in the same units (see below), and with player IDs associated across datasets.

## Notable differences between NBA and NCAA shots data

Shot x-y coordinates mean different things in each dataset, so before merging shots we have to convert to the same units.

NBA shots:

- court is oriented vertically, with the bottom hoop centered at (0, 0)
- x-y values are measured in tenths of a foot, e.g. (-50, 75) means the shot was attempted 5 feet "west" of the hoop and 7.5 feet "north" of the hoop, assuming a bird's eye perspective
- all shots are measured in the half-court relative to the bottom hoop

NCAA shots:

- court is oriented horizontally, with the "southwest" corner of the court at (0, 0)
- x-y values are measured in inches
- shots are recorded as at the left or right basket, unlike the NBA data, where all shots are assumed to be at the bottom basket

In order to make the NCAA data comparable to the NBA data, we have to:

1. adjust all shots at the right basket to their appropriate coordinates if they were shot at the left basket
2. rotate all coordinates 90 degrees counter-clockwise
3. translate the origin by 25 feet to the "west" and 5.25 feet to the "north" so that the formerly-left, now-bottom hoop is centered at (0, 0)

NB 25 feet is half the width of the court, and 5.25 feet is the distance from the baseline to the center of the hoop (4 feet baseline to backboard, 6 inches back iron, 9 inches hoop radius)

## NBA <=> NCAA players mapping

Players are mapped across datasets by name, with the additional condition that the timing has to make sense, e.g. someone's last year in the college dataset has to be before his first year in the NBA dataset. The mapping I ran in March 2018 is available in `players_mapping.csv`, but can be regenerated by running the following command from the project's root directory:

`bin/rails runner app/lib/ncaa_shots/create_players_mapping.rb`

Note that some player names were reviewed manually in cases where there were multiple exact text matches.