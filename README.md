# ⚽ Premier League Analytics (2010–2025)

An end-to-end data analytics project covering **15 Premier League seasons** — from raw web scraping all the way to an interactive, multi-page Power BI dashboard.

> Built with **Python** (web scraping + cleaning), a **Power BI** star-schema model (25+ tables, 59 relationships, 160+ DAX measures), and **7 themed report pages**.


## 📌 Project Overview

This project turns 15 years of scattered, messy public football data into a single, reliable analytical model and a polished dashboard that answers questions like:

- Who dominated each season — and by how much?
- How has transfer spending exploded over time?
- Which clubs are the smartest operators in the transfer market?
- How brutal is the Premier League's "hire & fire" manager culture?
- Who were the standout players and managers of each season?

---

## 🗺️ The Journey (End-to-End)

### 1. 🕸️ Data Collection — Web Scraping

- Scraped **15 seasons** of Premier League data from **Wikipedia** season pages using **Python** (`requests`, `BeautifulSoup`, `pandas`).
- Extracted dozens of tables per season: league standings, top scorers, hat-tricks, attendances, transfers (in / out), loans, managerial changes, monthly awards, stadiums, and more.

### 2. 🧹 Data Cleaning & Wrangling

- Standardized **club names** across seasons (promotions, relegations, renames).
- Normalized **season keys** to a single format (`2010–11`).
- Parsed messy **transfer fees** from free text (`£89m`, `€85 million`, `£17,000,000`) into clean numeric GBP values — including **EUR → GBP** conversion.
- Removed Wikipedia summary / "Total" rows that polluted aggregations.

### 3. 🔍 Solving the Missing Data Problem

- Wikipedia's player-level stats were **incomplete and unreliable** (scrambled squad / appearance tables).
- Sourced a clean, reliable **player statistics dataset from FBref** (`player_Status`) — minutes, goals, assists, position, cards, and per-90 metrics — to power all player-level analysis.

### 4. 🏗️ Data Modeling — Star Schema

- Designed a **star schema** with 4 dimension (lookup) tables: `Seasons`, `Dim_Club`, `Dim_Player`, `Dim_Date`.
- Connected **20+ fact tables** through **59 single-direction relationships** (Many-to-One).
- Added bridge keys (`SeasonKey`, `ClubKey`, `PlayerKey`) to align sources from different providers.

### 5. 🧮 DAX Measures (160+)

- Built measures across every analytical theme: titles, goals, attendance, squad composition, wages, player stats, manager churn, and transfer economics.
- Robust patterns for filter-awareness, dynamic bar colors, image URLs, and HTML-content visuals.

### 6. 📊 Visualization — 7 Report Pages

A dark, modern, consistent design system across all pages.

| Page | Highlights |
| --- | --- |
| **Overview** | League-wide KPIs, title race, all-time scorers, attendance & wage trends, Champions Roll Call |
| **Club Profile** | Squad composition, nationalities, top earner, club logo, per-club stats |
| **Season Profile** | Champion banner (logo), final standings, top scorers / assists, attack vs defense |
| **Player Profile** | Player of the Season hero card, Top-5 scorers with photos, player explorer |
| **Manager Profile** | Manager of the Season + the "Hire & Fire" analytics (sackings, carousel, sack race) |
| **Transfer Profile** | Spend / income / net, transfer inflation, record signings, biggest spenders, net spend |

---

## 🧰 Tech Stack

| Layer | Tools |
| --- | --- |
| **Scraping** | Python · requests · BeautifulSoup · pandas |
| **Cleaning** | pandas · regex |
| **Data source (players)** | FBref |
| **Modeling** | Power BI (star schema) |
| **Logic** | DAX (160+ measures) · Power Query (M) |
| **Custom visuals** | HTML Content visual + DAX-generated HTML |
