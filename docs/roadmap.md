# Roadmap

I want to build Fairway Log in small pieces that I can actually use. The first useful version is simply a dependable way to record a round. Everything else depends on that data being easy to collect and trustworthy.

## 1. Record a round

The first version will be an installable web app for my iPhone.

- Set up courses, tees, hole pars, and yardages
- Keep a reusable list of clubs in my bag
- Start a 9-hole or 18-hole round
- Record one hole at a time with large, simple controls
- Keep working without reception
- Save unfinished rounds and continue later
- Sync safely without creating duplicate holes

**Useful when:** I can finish a real round with my phone in airplane mode, reconnect, and see every hole once in the database.

## 2. Clean and model the data

Raw submissions will be kept so that a pipeline can always be replayed from the beginning.

- Validate scores, putts, penalties, clubs, and shot outcomes
- Keep bad records in a reviewable rejection table
- Process only new or changed submissions during normal runs
- Make reruns safe and repeatable
- Build facts, dimensions, and reporting tables with dbt
- Test metric definitions and table relationships automatically

**Useful when:** a full replay and an incremental run produce the same clean results.

## 3. Build the Tableau view

Only completed rounds and a privacy-safe set of fields will leave the private database.

- Publish cleaned reporting tables to Google Sheets
- Refresh the Tableau Public data automatically
- Build scoring, driving, approach, putting, and club views
- Show sample sizes and the date of the last successful refresh
- Explain GIR, FIR, and other calculated metrics in the dashboard

**Useful when:** someone can open one link and understand both how I play and how the data reached the dashboard.

## 4. Add the science carefully

I do not want to force a model onto a tiny dataset. Descriptive statistics come first.

- Add rolling form and confidence intervals
- Compare performance across shot outcomes and yardage bands
- Wait for at least 360 completed holes before publishing a model
- Compare the model against a simple baseline
- Report weak results honestly instead of presenting noise as insight

**Useful when:** the analysis says something defensible that a chart of averages does not already show.

## Not planned for the first version

- GPS or shot-location tracking
- Social features or multiple golfers
- Weather data
- Handicap-system integration
- A native App Store release
- Live Tableau connections or paid infrastructure

