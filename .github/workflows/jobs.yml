"""Saudi Job Radar: free daily job scraper (LinkedIn, Indeed, Google Jobs, Bayt).

Runs on GitHub Actions (free). Only NEW jobs since the last run are reported.
"""
import hashlib
import os
import re
import time
from pathlib import Path

import pandas as pd
import requests
from jobspy import scrape_jobs

# ---------------------------------------------------------------- CONFIG
SEARCH_TERMS = [
    "SAP CPI",
    "SAP integration consultant",
    "SAP BTP",
    "business analyst",
    "IT business analyst",
    "IT support",
    "service desk",
    "ERP consultant",
    "CRM consultant",
]
SITES = ["linkedin", "indeed", "google", "bayt"]
LOCATION = "Saudi Arabia"
RESULTS_PER_SEARCH = 25
HOURS_OLD = 72          # only jobs posted in the last 3 days
MIN_SCORE = 3           # alert threshold

# Title hits are worth 3 points, description hits 1 point (max 4).
TITLE_KEYWORDS = ["sap", "cpi", "integration", "business analyst", "it support",
                  "service desk", "help desk", "technical support", "erp", "crm",
                  "systems analyst", "project coordinator", "btp"]
DESC_KEYWORDS = ["sap cpi", "sap btp", "s/4hana", "iflow", "groovy", "jdbc",
                 "active directory", "power bi", "requirements gathering", "uat",
                 "freshservice", "servicenow", "salesforce", "itil", "pmp"]
EXCLUDE_TITLE = ["intern", "sales executive", "driver", "accountant", "nurse",
                 "teacher", "warehouse", "cashier"]

SA_PLACES = ["saudi", "riyadh", "jeddah", "dammam", "khobar", "dhahran", "makkah",
             "mecca", "medina", "madinah", "neom", "tabuk", "abha", "jubail",
             "yanbu", "kaec", "qassim", "ksa", ", sa"]

SEEN_FILE = Path("seen.csv")
LATEST_FILE = Path("jobs_latest.csv")
# ---------------------------------------------------------------------


def scrape_all() -> pd.DataFrame:
    frames = []
    for term in SEARCH_TERMS:
        for site in SITES:
            try:
                df = scrape_jobs(
                    site_name=[site],
                    search_term=term,
                    google_search_term=f"{term} jobs in {LOCATION} since yesterday",
                    location=LOCATION,
                    country_indeed="saudi arabia",
                    results_wanted=RESULTS_PER_SEARCH,
                    hours_old=HOURS_OLD,
                    description_format="markdown",
                    linkedin_fetch_description=(site == "linkedin"),
                    verbose=0,
                )
                if df is not None and not df.empty:
                    df["query"] = term
                    frames.append(df)
                print(f"[ok]   {site:9} | {term:28} | {0 if df is None else len(df)} jobs")
            except Exception as e:  # one failing source must not kill the run
                print(f"[fail] {site:9} | {term:28} | {str(e)[:80]}")
            time.sleep(3)  # be polite, avoid rate limits
    return pd.concat(frames, ignore_index=True) if frames else pd.DataFrame()


def job_id(row) -> str:
    key = "|".join(str(row.get(c, "")).strip().lower() for c in ("title", "company", "location"))
    return hashlib.md5(key.encode()).hexdigest()[:16]


def in_saudi(loc: str) -> bool:
    loc = f" {str(loc).lower()} "
    return any(p in loc for p in SA_PLACES)


def score(row) -> int:
    title = str(row.get("title", "")).lower()
    desc = str(row.get("description", "")).lower()
    if any(w in title for w in EXCLUDE_TITLE):
        return -1
    s = 3 * sum(k in title for k in TITLE_KEYWORDS)
    s += min(4, sum(k in desc for k in DESC_KEYWORDS))
    return s


def notify(text: str) -> None:
    token, chat = os.getenv("TELEGRAM_BOT_TOKEN"), os.getenv("TELEGRAM_CHAT_ID")
    if not (token and chat):
        print(text)
        return
    for i in range(0, len(text), 3800):  # Telegram limit is 4096 chars
        requests.post(
            f"https://api.telegram.org/bot{token}/sendMessage",
            data={"chat_id": chat, "text": text[i:i + 3800],
                  "disable_web_page_preview": True},
            timeout=30,
        )


def main() -> None:
    raw = scrape_all()
    if raw.empty:
        print("No results from any source.")
        return

    raw = raw[raw["location"].apply(in_saudi) | raw["location"].isna()].copy()
    raw["id"] = raw.apply(job_id, axis=1)
    raw = raw.drop_duplicates("id")
    raw["score"] = raw.apply(score, axis=1)
    raw = raw[raw["score"] >= MIN_SCORE]

    seen = set(pd.read_csv(SEEN_FILE)["id"]) if SEEN_FILE.exists() else set()
    new = raw[~raw["id"].isin(seen)].sort_values("score", ascending=False)

    if new.empty:
        print("Nothing new today.")
        return

    cols = ["score", "title", "company", "location", "site", "date_posted", "job_url"]
    new[cols].to_csv(LATEST_FILE, index=False)
    pd.DataFrame({"id": sorted(seen | set(new["id"]))}).to_csv(SEEN_FILE, index=False)

    lines = [f"🇸🇦 {len(new)} new Saudi jobs\n"]
    for _, r in new.head(25).iterrows():
        lines.append(f"[{r['score']}] {r['title']}\n{r['company']} · {r['location']} · {r['site']}\n{r['job_url']}\n")
    notify("\n".join(lines))


if __name__ == "__main__":
    main()
