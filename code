# -*- coding: utf-8 -*-
# ============================================================
# JOB APPLICATION AGENT v1.1
# Description-based matching (not title-based)
# Scrape broadly -> Read full JD -> Score by skill cluster -> Filter
# ============================================================

# ------------------------------------------------------------
# CELL 1 - INSTALL DEPENDENCIES
# ------------------------------------------------------------
"""
!pip install playwright streamlit pyngrok python-docx pandas -q
!playwright install chromium
!playwright install-deps chromium
"""

# ------------------------------------------------------------
# CELL 2 - IMPORTS & CONFIG
# ------------------------------------------------------------

import asyncio
import sqlite3
import json
import re
import time
import random
import pandas as pd
from datetime import datetime
from pathlib import Path
from dataclasses import dataclass, field
from playwright.async_api import async_playwright

DB_PATH = "jobs.db"
# For persistence across Colab sessions, use:
# from google.colab import drive
# drive.mount('/content/drive')
# DB_PATH = "/content/drive/MyDrive/job_agent/jobs.db"

Path(DB_PATH).parent.mkdir(parents=True, exist_ok=True)

# -- Your profile ------------------------------------------
CONFIG = {
    "name":          "Shrijanaki Kumar",
    "email":         "shrijanakikumar@gmail.com",     
    "phone":         "+91-9677918830",            
    "location":      "Chennai, India",
    "resume_path":   "Shrijanaki_Kumar_Resume.pdf",  #update
    "experience_years": 5,

    # Broad search terms sent to job boards
    # Intentionally generic -- description scorer does the real filtering
    "search_terms": [
        "AI ML quality",
        "data annotation quality",
        "LLM evaluation",
        "NLU ASR data",
        "prompt engineering operations",
        "AI operations manager",
        "machine learning data quality",
        "AI safety evaluator",
    ],

    # Location filters for the search query
    "search_locations": ["India", "Remote"],

    # Score threshold -- jobs below this are discarded
    # Range: 0-100. Recommended: 30-40 for broad net, 50+ for strict
    "score_threshold": 35,

    # Hard exclusions -- if ANY of these appear in JD, job is skipped
    "hard_exclude": [
        "10+ years", "15+ years",
        "software development engineer",
        "full stack developer",
        "react", "angular", "node.js",
        "devops", "cloud infrastructure",
    ],
}

# -- Skill clusters -- the heart of v1.1 ------------------
# Each cluster has:
#   weight    -> how much this cluster contributes to total score
#   must_have -> if True, job must match at least 1 keyword here to qualify
#   keywords  -> matched against full job description text

SKILL_CLUSTERS = {
    "llm_evaluation": {
        "label": "LLM Evaluation",
        "weight": 35,
        "must_have": True,
        "keywords": [
            "llm evaluation", "llm eval", "language model evaluation",
            "model evaluation", "hallucination detection", "hallucination",
            "response quality", "llm quality", "ai evaluation",
            "model quality", "output evaluation", "generative ai quality",
            "rlhf", "reinforcement learning from human feedback",
            "ai feedback", "human feedback", "model benchmarking",
            "red teaming", "ai red team", "safety evaluation",
            "evaluation pipeline", "eval pipeline", "benchmark",
        ],
    },
    "data_annotation": {
        "label": "Data Annotation / Labeling",
        "weight": 25,
        "must_have": False,
        "keywords": [
            "data annotation", "data labeling", "data labelling",
            "annotation quality", "label quality", "ground truth",
            "training data", "data collection", "human annotation",
            "annotator", "inter-annotator agreement", "iaa",
            "data quality", "quality assurance", "qa operations",
            "content moderation", "trust and safety",
            "crowd sourcing", "crowdsource", "tasq", "scale ai",
        ],
    },
    "nlu_asr_speech": {
        "label": "NLU / ASR / Speech",
        "weight": 20,
        "must_have": False,
        "keywords": [
            "nlu", "natural language understanding",
            "asr", "automatic speech recognition", "speech recognition",
            "conversational ai", "dialogue systems", "intent recognition",
            "entity extraction", "named entity", "text classification",
            "nlp", "natural language processing",
            "voice assistant", "alexa", "siri", "cortana",
            "speech data", "audio annotation", "transcription",
        ],
    },
    "prompt_engineering": {
        "label": "Prompt Engineering",
        "weight": 20,
        "must_have": False,
        "keywords": [
            "prompt engineering", "prompt design", "prompt optimization",
            "prompt writing", "few-shot", "zero-shot", "chain of thought",
            "system prompt", "instruction tuning", "fine-tuning",
            "chatgpt", "gpt-4", "claude", "gemini", "llama",
            "foundation model", "large language model", "generative ai",
            "ai content", "ai operations", "mlops", "llmops",
        ],
    },
}

print("[OK] Config loaded")
print(f"   Search terms:    {len(CONFIG['search_terms'])}")
print(f"   Skill clusters:  {len(SKILL_CLUSTERS)}")
print(f"   Score threshold: {CONFIG['score_threshold']}")


# ------------------------------------------------------------
# CELL 3 - DATABASE SETUP
# ------------------------------------------------------------

def init_db():
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    c.execute("""
        CREATE TABLE IF NOT EXISTS jobs (
            id                  INTEGER PRIMARY KEY AUTOINCREMENT,
            job_id              TEXT UNIQUE,
            title               TEXT,
            company             TEXT,
            location            TEXT,
            source              TEXT,
            url                 TEXT,
            description         TEXT,
            cluster_scores      TEXT,   -- JSON: {cluster: score}
            matched_keywords    TEXT,   -- JSON: {cluster: [keywords]}
            total_score         INTEGER DEFAULT 0,
            status              TEXT DEFAULT 'scraped',
            applied_date        TEXT,
            notes               TEXT,
            scraped_at          TEXT
        )
    """)
    conn.commit()
    conn.close()
    print("[OK] Database ready:", DB_PATH)

init_db()


# ------------------------------------------------------------
# CELL 4 - JOB DATA MODEL
# ------------------------------------------------------------

@dataclass
class Job:
    title:            str
    company:          str
    location:         str
    url:              str
    source:           str
    description:      str = ""
    job_id:           str = ""
    cluster_scores:   dict = field(default_factory=dict)
    matched_keywords: dict = field(default_factory=dict)
    total_score:      int = 0
    status:           str = "scraped"
    applied_date:     str = ""
    notes:            str = ""
    scraped_at:       str = field(default_factory=lambda: datetime.now().isoformat())

    def __post_init__(self):
        if not self.job_id:
            self.job_id = f"{self.source}_{abs(hash(self.url))}"


def save_job(job: Job) -> bool:
    conn = sqlite3.connect(DB_PATH)
    c = conn.cursor()
    try:
        c.execute("""
            INSERT OR IGNORE INTO jobs
            (job_id, title, company, location, source, url, description,
             cluster_scores, matched_keywords, total_score, status, scraped_at)
            VALUES (?,?,?,?,?,?,?,?,?,?,?,?)
        """, (
            job.job_id, job.title, job.company, job.location,
            job.source, job.url, job.description[:3000],  # cap description length
            json.dumps(job.cluster_scores),
            json.dumps(job.matched_keywords),
            job.total_score, job.status, job.scraped_at,
        ))
        conn.commit()
        return c.rowcount > 0
    finally:
        conn.close()


def get_all_jobs() -> pd.DataFrame:
    conn = sqlite3.connect(DB_PATH)
    df = pd.read_sql("SELECT * FROM jobs ORDER BY total_score DESC, scraped_at DESC", conn)
    conn.close()
    return df


def update_job_status(job_id: str, status: str, notes: str = ""):
    conn = sqlite3.connect(DB_PATH)
    applied_date = datetime.now().isoformat() if status == "applied" else ""
    conn.execute(
        "UPDATE jobs SET status=?, notes=?, applied_date=? WHERE job_id=?",
        (status, notes, applied_date, job_id)
    )
    conn.commit()
    conn.close()

print("[OK] Data model ready")


# ------------------------------------------------------------
# CELL 5 - DESCRIPTION SCORER (core of v1.1)
# ------------------------------------------------------------

def score_description(description: str, title: str = "") -> tuple[int, dict, dict]:
    """
    Scores a job description against all skill clusters.

    Returns:
        total_score     (int 0-100)
        cluster_scores  {cluster_name: score}
        matched_kws     {cluster_name: [matched keywords]}

    V2 upgrade: replace this with Claude API call that reads
    the full JD and returns structured reasoning + scores.
    """
    text = (title + " " + description).lower()

    # Hard exclusion check
    for excl in CONFIG["hard_exclude"]:
        if excl.lower() in text:
            return 0, {}, {}

    cluster_scores = {}
    matched_kws = {}

    for cluster_id, cluster in SKILL_CLUSTERS.items():
        hits = [kw for kw in cluster["keywords"] if kw in text]
        matched_kws[cluster_id] = hits

        if not hits:
            cluster_scores[cluster_id] = 0
        else:
            # Score = (hits / total keywords) * weight, capped at weight
            raw = (len(hits) / len(cluster["keywords"])) * cluster["weight"] * 3
            cluster_scores[cluster_id] = min(int(raw), cluster["weight"])

    # Must-have check: if a must_have cluster scored 0, disqualify
    for cluster_id, cluster in SKILL_CLUSTERS.items():
        if cluster.get("must_have") and cluster_scores.get(cluster_id, 0) == 0:
            return 0, cluster_scores, matched_kws

    total = min(sum(cluster_scores.values()), 100)
    return total, cluster_scores, matched_kws


def qualifies(job: Job) -> bool:
    return job.total_score >= CONFIG["score_threshold"]


# Quick test
sample_jd = """
We are looking for an LLM Evaluation Specialist to join our AI team.
You will design evaluation pipelines for hallucination detection,
work with annotators on RLHF data, and improve prompt engineering workflows.
Experience with NLU and ASR systems is a plus.
"""
score, clusters, kws = score_description(sample_jd, "LLM Evaluation Specialist")
print(f"[OK] Scorer ready - Sample JD score: {score}/100")
for cid, cscore in clusters.items():
    label = SKILL_CLUSTERS[cid]["label"]
    hits = kws.get(cid, [])
    print(f"   {label}: {cscore} pts - matched: {hits[:3]}")


# ------------------------------------------------------------
# CELL 6 - JD FETCHER (reads full description from job page)
# ------------------------------------------------------------

async def fetch_full_jd(url: str, page) -> str:
    """
    Navigates to a job detail page and extracts the full description text.
    Tries multiple common selectors across job boards.
    """
    try:
        await page.goto(url, timeout=20000)
        await page.wait_for_timeout(2500)

        selectors = [
            # LinkedIn
            ".description__text",
            ".show-more-less-html__markup",
            # Naukri
            ".job-desc",
            ".dang-inner-html",
            # Generic
            "[class*='description']",
            "[class*='job-detail']",
            "[class*='jobDetail']",
            "article",
            "main",
        ]

        for sel in selectors:
            el = await page.query_selector(sel)
            if el:
                text = await el.inner_text()
                if len(text.strip()) > 100:  # meaningful content
                    return text.strip()

        # Fallback: grab all body text
        return await page.inner_text("body")

    except Exception:
        return ""


# ------------------------------------------------------------
# CELL 7 - SCRAPER - LINKEDIN (description-based)
# ------------------------------------------------------------

async def scrape_linkedin(search_term: str, max_jobs: int = 20) -> list[Job]:
    jobs = []
    query = search_term.replace(" ", "%20")
    # Search without role title filter -- broad net
    url = f"https://www.linkedin.com/jobs/search/?keywords={query}&location=India&f_WT=2%2C1&f_TPR=r604800"
    # f_WT=2,1 -> Remote + Hybrid | f_TPR=r604800 -> last 7 days

    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        context = await browser.new_context(
            user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/120 Safari/537.36"
        )
        page = await context.new_page()
        detail_page = await context.new_page()

        try:
            await page.goto(url, timeout=30000)
            await page.wait_for_timeout(3000)

            for _ in range(3):
                await page.evaluate("window.scrollTo(0, document.body.scrollHeight)")
                await page.wait_for_timeout(1200)

            cards = await page.query_selector_all(".job-search-card")
            print(f"  LinkedIn - '{search_term}' -> {len(cards)} listings")

            for card in cards[:max_jobs]:
                try:
                    title_el   = await card.query_selector(".base-search-card__title")
                    company_el = await card.query_selector(".base-search-card__subtitle")
                    location_el= await card.query_selector(".job-search-card__location")
                    link_el    = await card.query_selector("a.base-card__full-link")

                    title    = (await title_el.inner_text()).strip()    if title_el    else ""
                    company  = (await company_el.inner_text()).strip()  if company_el  else ""
                    location = (await location_el.inner_text()).strip() if location_el else ""
                    link     = await link_el.get_attribute("href")      if link_el     else ""

                    if not title or not link:
                        continue

                    # Fetch full JD from detail page
                    jd_text = await fetch_full_jd(link, detail_page)
                    await asyncio.sleep(random.uniform(1, 2))

                    score, cluster_scores, matched_kws = score_description(jd_text, title)

                    job = Job(
                        title=title, company=company, location=location,
                        url=link, source="linkedin", description=jd_text,
                        cluster_scores=cluster_scores,
                        matched_keywords=matched_kws,
                        total_score=score,
                    )
                    jobs.append(job)

                    status = "[OK]" if qualifies(job) else "-"
                    print(f"    {status} [{score:3d}] {title[:50]} @ {company[:25]}")

                except Exception:
                    continue

        except Exception as e:
            print(f"  [!] LinkedIn error: {e}")
        finally:
            await browser.close()

    return jobs


# ------------------------------------------------------------
# CELL 8 - SCRAPER - NAUKRI (description-based)
# ------------------------------------------------------------

async def scrape_naukri(search_term: str, max_jobs: int = 20) -> list[Job]:
    jobs = []
    query = search_term.replace(" ", "-")
    url = f"https://www.naukri.com/{query}-jobs?jobAge=7&experience=4"

    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        context = await browser.new_context(
            user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/120 Safari/537.36"
        )
        page = await context.new_page()
        detail_page = await context.new_page()

        try:
            await page.goto(url, timeout=30000)
            await page.wait_for_timeout(4000)

            cards = await page.query_selector_all("article.jobTuple, .cust-job-tuple")
            print(f"  Naukri - '{search_term}' -> {len(cards)} listings")

            for card in cards[:max_jobs]:
                try:
                    title_el   = await card.query_selector("a.title")
                    company_el = await card.query_selector("a.subTitle")
                    location_el= await card.query_selector(".locWdth")

                    title   = (await title_el.inner_text()).strip()   if title_el   else ""
                    company = (await company_el.inner_text()).strip() if company_el else ""
                    location= (await location_el.inner_text()).strip()if location_el else ""
                    link    = await title_el.get_attribute("href")    if title_el   else ""

                    if not title or not link:
                        continue

                    jd_text = await fetch_full_jd(link, detail_page)
                    await asyncio.sleep(random.uniform(1.5, 3))

                    score, cluster_scores, matched_kws = score_description(jd_text, title)

                    job = Job(
                        title=title, company=company, location=location,
                        url=link, source="naukri", description=jd_text,
                        cluster_scores=cluster_scores,
                        matched_keywords=matched_kws,
                        total_score=score,
                    )
                    jobs.append(job)

                    status = "[OK]" if qualifies(job) else "-"
                    print(f"    {status} [{score:3d}] {title[:50]} @ {company[:25]}")

                except Exception:
                    continue

        except Exception as e:
            print(f"  [!] Naukri error: {e}")
        finally:
            await browser.close()

    return jobs


# ------------------------------------------------------------
# CELL 9 - SCRAPER - COMPANY CAREER PAGES
# ------------------------------------------------------------

COMPANY_PAGES = [
    {
        "name": "Welo Data (Welocalize)",
        "search_url": "https://jobs.welocalize.com/?s=AI+quality+data",
        "card_selector": ".job-listing, .careers-item",
        "title_selector": "h2, h3, .job-title",
        "link_selector": "a",
        "base_url": "https://jobs.welocalize.com",
    },
    {
        "name": "Appen",
        "search_url": "https://appen.com/careers/?search=AI+quality+annotation",
        "card_selector": ".job-item, .careers-listing-item",
        "title_selector": ".job-title, h3",
        "link_selector": "a",
        "base_url": "https://appen.com",
    },
    {
        "name": "Telus International",
        "search_url": "https://jobs.telusinternational.com/en_US/careers/SearchJobs/AI%20data%20quality",
        "card_selector": ".article--result",
        "title_selector": "h2",
        "link_selector": "a",
        "base_url": "https://jobs.telusinternational.com",
    },
    {
        "name": "Accenture India",
        "search_url": "https://www.accenture.com/in-en/careers/jobsearch?jk=AI+ML+quality+operations&lc=India",
        "card_selector": ".jobsearch-result-list-item, .cmp-teaser",
        "title_selector": ".job-title, h3",
        "link_selector": "a",
        "base_url": "https://www.accenture.com",
    },
]

async def scrape_company_page(company: dict) -> list[Job]:
    jobs = []

    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        context = await browser.new_context(
            user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
        )
        page = await context.new_page()
        detail_page = await context.new_page()

        try:
            await page.goto(company["search_url"], timeout=30000)
            await page.wait_for_timeout(4000)

            cards = await page.query_selector_all(company["card_selector"])
            print(f"  {company['name']} -> {len(cards)} listings")

            for card in cards[:12]:
                try:
                    title_el = await card.query_selector(company["title_selector"])
                    link_el  = await card.query_selector(company["link_selector"])

                    title = (await title_el.inner_text()).strip() if title_el else ""
                    href  = await link_el.get_attribute("href")   if link_el  else ""
                    if not title or not href:
                        continue

                    link = href if href.startswith("http") else company["base_url"] + href

                    jd_text = await fetch_full_jd(link, detail_page)
                    await asyncio.sleep(random.uniform(1, 2.5))

                    score, cluster_scores, matched_kws = score_description(jd_text, title)

                    job = Job(
                        title=title, company=company["name"],
                        location="India / Remote", url=link,
                        source="company_page", description=jd_text,
                        cluster_scores=cluster_scores,
                        matched_keywords=matched_kws,
                        total_score=score,
                    )
                    jobs.append(job)

                    status = "[OK]" if qualifies(job) else "-"
                    print(f"    {status} [{score:3d}] {title[:55]}")

                except Exception:
                    continue

        except Exception as e:
            print(f"  [!] {company['name']} error: {e}")
        finally:
            await browser.close()

    return jobs


# ------------------------------------------------------------
# CELL 10 - MAIN SCRAPE RUNNER
# ------------------------------------------------------------

async def run_scraper(
    search_terms: list = None,
    sources: list = None,
    max_per_term: int = 15,
):
    """
    Main entry point. Scrapes broadly, scores by description, saves qualified jobs.

    Args:
        search_terms:  override CONFIG search_terms
        sources:       list of "linkedin", "naukri", "company"
        max_per_term:  max listings to fetch per search term
    """
    if search_terms is None:
        search_terms = CONFIG["search_terms"]
    if sources is None:
        sources = ["linkedin", "naukri", "company"]

    all_jobs = []
    print(f"\n Scrape run - {datetime.now().strftime('%Y-%m-%d %H:%M')}")
    print(f"   Terms: {search_terms}")
    print(f"   Sources: {sources}\n")

    for term in search_terms:
        print(f"\n '{term}'")

        if "linkedin" in sources:
            lj = await scrape_linkedin(term, max_jobs=max_per_term)
            all_jobs.extend(lj)
            await asyncio.sleep(random.uniform(3, 5))

        if "naukri" in sources:
            nj = await scrape_naukri(term, max_jobs=max_per_term)
            all_jobs.extend(nj)
            await asyncio.sleep(random.uniform(3, 5))

    if "company" in sources:
        print(f"\n Company career pages...")
        for company in COMPANY_PAGES:
            cj = await scrape_company_page(company)
            all_jobs.extend(cj)
            await asyncio.sleep(random.uniform(2, 4))

    # Deduplicate by URL
    seen_urls = set()
    unique_jobs = []
    for job in all_jobs:
        if job.url not in seen_urls:
            seen_urls.add(job.url)
            unique_jobs.append(job)

    # Save qualifying jobs
    saved = skipped = 0
    for job in unique_jobs:
        if qualifies(job):
            if save_job(job):
                saved += 1
        else:
            skipped += 1

    print(f"\n{'-'*50}")
    print(f"[OK] Done!")
    print(f"   Scraped (unique):  {len(unique_jobs)}")
    print(f"   Qualified + saved: {saved}")
    print(f"   Filtered out:      {skipped}  (score < {CONFIG['score_threshold']})")
    print(f"   Run get_all_jobs() to view results")

    return unique_jobs


# ------------------------------------------------------------
# CELL 11 - RESULTS VIEWER
# ------------------------------------------------------------

def view_results(min_score: int = None, status: str = None, top_n: int = 20):
    """Quick terminal view of matched jobs"""
    df = get_all_jobs()
    if df.empty:
        print("No jobs yet. Run: await run_scraper()")
        return df

    if min_score is not None:
        df = df[df["total_score"] >= min_score]
    if status:
        df = df[df["status"] == status]

    df = df.head(top_n)
    print(f"\n{'-'*70}")
    print(f"{'SCORE':>5}  {'TITLE':<40} {'COMPANY':<22} SOURCE")
    print(f"{'-'*70}")
    for _, row in df.iterrows():
        clusters = json.loads(row["cluster_scores"]) if row["cluster_scores"] else {}
        bar = " | ".join(f"{SKILL_CLUSTERS[c]['label'].split('/')[0].strip()}:{v}" for c, v in clusters.items() if v > 0)
        print(f"  {row['total_score']:3d}  {row['title'][:40]:<40} {row['company'][:22]:<22} {row['source']}")
        if bar:
            print(f"       -> {bar}")
    print(f"{'-'*70}")
    print(f"Total: {len(df)} jobs")
    return df

# Usage:
# view_results()
# view_results(min_score=50)
# view_results(status="applied")


# ------------------------------------------------------------
# CELL 12 - APPLICATION ASSISTANT
# ------------------------------------------------------------

async def open_and_prefill(job_id: str):
    """Opens job URL with browser, pre-fills what it can, hands over to you"""
    conn = sqlite3.connect(DB_PATH)
    row = pd.read_sql(f"SELECT * FROM jobs WHERE job_id='{job_id}'", conn)
    conn.close()

    if row.empty:
        print(f"[X] Job {job_id} not found")
        return

    job = row.iloc[0]
    clusters = json.loads(job["cluster_scores"]) if job["cluster_scores"] else {}

    print(f"\n Opening: {job['title']} @ {job['company']}")
    print(f"   Score: {job['total_score']}/100")
    print(f"   Clusters: {clusters}")
    print(f"   URL: {job['url']}\n")

    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=False, slow_mo=400)
        context = await browser.new_context()
        page = await context.new_page()

        await page.goto(job["url"], timeout=30000)
        await page.wait_for_timeout(3000)

        fill_map = {
            'input[name="name"]':            CONFIG["name"],
            'input[name="fullName"]':         CONFIG["name"],
            'input[name="email"]':            CONFIG["email"],
            'input[type="email"]':            CONFIG["email"],
            'input[name="phone"]':            CONFIG["phone"],
            'input[name="mobile"]':           CONFIG["phone"],
            'input[name="phoneNumber"]':      CONFIG["phone"],
            'input[name="location"]':         CONFIG["location"],
            'input[name="currentLocation"]':  CONFIG["location"],
            'input[name="experience"]':       str(CONFIG["experience_years"]),
            'input[name="totalExperience"]':  str(CONFIG["experience_years"]),
        }

        filled = []
        for selector, value in fill_map.items():
            try:
                el = await page.query_selector(selector)
                if el:
                    await el.fill(value)
                    filled.append(selector.split('"')[1])
            except Exception:
                pass

        resume = Path(CONFIG["resume_path"])
        if resume.exists():
            try:
                file_input = await page.query_selector('input[type="file"]')
                if file_input:
                    await file_input.set_input_files(str(resume))
                    filled.append("resume")
            except Exception:
                pass

        if filled:
            print(f"   [OK] Pre-filled: {', '.join(filled)}")
        else:
            print(f"   [i] No auto-fill fields detected -- manual apply")

        print("\n Browser is open. Review, complete, and submit.")
        print(f"   Press Enter here when done (or skip)...")
        result = input("   [Enter=applied / s=skip]: ").strip().lower()

        if result != "s":
            update_job_status(job_id, "applied")
            print(f"   [OK] Marked as applied")
        else:
            update_job_status(job_id, "skipped")
            print(f"   Skipped")

        await browser.close()


async def batch_apply(limit: int = 5, min_score: int = None):
    """Work through top N jobs one by one"""
    threshold = min_score or CONFIG["score_threshold"]
    conn = sqlite3.connect(DB_PATH)
    df = pd.read_sql(
        f"SELECT * FROM jobs WHERE status='scraped' AND total_score >= {threshold} "
        f"ORDER BY total_score DESC LIMIT {limit}",
        conn
    )
    conn.close()

    if df.empty:
        print("No pending jobs. Run scraper first.")
        return

    print(f"\n {len(df)} jobs queued:\n")
    for _, row in df.iterrows():
        print(f"  [{row['total_score']:3d}] {row['title']} @ {row['company']} ({row['source']})")

    print("\nStarting...\n")
    for _, row in df.iterrows():
        await open_and_prefill(row["job_id"])
        time.sleep(1)


# ------------------------------------------------------------
# CELL 13 - STREAMLIT DASHBOARD
# ------------------------------------------------------------

DASHBOARD_CODE = '''
import streamlit as st
import sqlite3, json, pandas as pd
from datetime import datetime

DB_PATH = "jobs.db"

CLUSTERS = {
    "llm_evaluation":     "LLM Eval",
    "data_annotation":    "Annotation",
    "nlu_asr_speech":     "NLU/ASR",
    "prompt_engineering": "Prompting",
}

st.set_page_config(page_title="Job Agent Tracker", page_icon="", layout="wide")
st.title("Job Application Tracker - v1.1")
st.caption("Description-based skill matching")

conn = sqlite3.connect(DB_PATH)
df = pd.read_sql("SELECT * FROM jobs ORDER BY total_score DESC", conn)
conn.close()

if df.empty:
    st.info("No jobs yet. Run the scraper.")
    st.stop()

# Funnel metrics
c1,c2,c3,c4,c5 = st.columns(5)
c1.metric("Total Found",    len(df))
c2.metric("Qualified",      len(df[df["total_score"] >= 35]))
c3.metric("Applied",        len(df[df["status"] == "applied"]))
c4.metric("Interviewing",   len(df[df["status"] == "interviewing"]))
c5.metric("Offer",          len(df[df["status"] == "offer"]))

st.divider()

# Filters
col_a, col_b, col_c, col_d = st.columns(4)
sources  = col_a.multiselect("Source",  df["source"].unique(),  default=list(df["source"].unique()))
statuses = col_b.multiselect("Status",  df["status"].unique(),  default=["scraped","applied"])
min_sc   = col_c.slider("Min Score", 0, 100, 35)
search   = col_d.text_input("Search title/company")

filtered = df[
    df["source"].isin(sources) &
    df["status"].isin(statuses) &
    (df["total_score"] >= min_sc)
]
if search:
    filtered = filtered[
        filtered["title"].str.contains(search, case=False, na=False) |
        filtered["company"].str.contains(search, case=False, na=False)
    ]

st.markdown(f"**{len(filtered)} jobs**")

def get_cluster(row, cid):
    try:
        return json.loads(row["cluster_scores"]).get(cid, 0)
    except:
        return 0

for cid, label in CLUSTERS.items():
    filtered[label] = filtered.apply(lambda r: get_cluster(r, cid), axis=1)

display = filtered[["title","company","location","source","total_score"] + list(CLUSTERS.values()) + ["status","scraped_at"]]
st.dataframe(display.rename(columns={"total_score":"score","scraped_at":"found"}),
             use_container_width=True, hide_index=True)

st.divider()
st.subheader("Update Status")
cx, cy, cz = st.columns(3)
options    = filtered.apply(lambda r: f"{r[\'title\']} @ {r[\'company\']}", axis=1).tolist()
selected   = cx.selectbox("Job", options)
new_status = cy.selectbox("Status", ["scraped","applied","interviewing","rejected","offer","skipped"])
notes      = cz.text_input("Notes")

if st.button("Update"):
    idx = options.index(selected)
    job_id = filtered.iloc[idx]["job_id"]
    conn2 = sqlite3.connect(DB_PATH)
    conn2.execute("UPDATE jobs SET status=?, notes=?, applied_date=? WHERE job_id=?",
                  (new_status, notes,
                   datetime.now().isoformat() if new_status=="applied" else "",
                   job_id))
    conn2.commit()
    conn2.close()
    st.success(f"Updated to: {new_status}")
    st.rerun()

st.divider()
csv = filtered.to_csv(index=False)
st.download_button("Export CSV", csv, "jobs.csv", "text/csv")
'''

with open("dashboard.py", "w", encoding="utf-8") as f:
    f.write(DASHBOARD_CODE)
print("[OK] dashboard.py saved")


# ------------------------------------------------------------
# CELL 14 - LAUNCH DASHBOARD
# ------------------------------------------------------------

def launch_dashboard():
    import subprocess, threading
    def run():
        subprocess.run(["streamlit", "run", "dashboard.py",
                        "--server.port", "8501", "--server.headless", "true"])
    threading.Thread(target=run, daemon=True).start()
    time.sleep(4)
    try:
        from pyngrok import ngrok
        url = ngrok.connect(8501)
        print(f"\nDashboard live at: {url}")
    except Exception:
        print("Dashboard running on port 8501")
        print("Install pyngrok for a public URL: pip install pyngrok")

# launch_dashboard()


# ------------------------------------------------------------
# CELL 15 - QUICK REFERENCE
# ------------------------------------------------------------

print("""
+=================================================================+
|         JOB AGENT v1.1 - DESCRIPTION MATCHER                   |
+=================================================================+
|  HOW IT WORKS:                                                  |
|  1. Searches job boards using broad generic terms               |
|  2. Opens each listing -> reads the FULL job description        |
|  3. Scores description against your 4 skill clusters            |
|  4. Saves only jobs above your threshold (default: 35/100)      |
|                                                                 |
|  SKILL CLUSTERS:                                                |
|  - LLM Evaluation     (weight: 35, MUST match)                 |
|  - Data Annotation    (weight: 25)                              |
|  - NLU / ASR / Speech (weight: 20)                             |
|  - Prompt Engineering (weight: 20)                              |
|                                                                 |
|  RUN SCRAPER:                                                   |
|    await run_scraper()                                          |
|    await run_scraper(sources=["linkedin"])                      |
|    await run_scraper(search_terms=["data annotation AI"])       |
|                                                                 |
|  VIEW RESULTS:                                                  |
|    view_results()                                               |
|    view_results(min_score=50)                                   |
|    get_all_jobs()   <- returns full DataFrame                   |
|                                                                 |
|  APPLY:                                                         |
|    await batch_apply(limit=5)                                   |
|    await open_and_prefill("job_id_here")                        |
|                                                                 |
|  DASHBOARD:                                                     |
|    launch_dashboard()                                           |
|                                                                 |
|  TUNE THE SCORER:                                               |
|    CONFIG["score_threshold"] = 40   <- stricter                 |
|    CONFIG["score_threshold"] = 25   <- broader net              |
|    Add keywords to any SKILL_CLUSTERS block                     |
|                                                                 |
|  -- V2 UPGRADES -------------------------------------------- |
|  [ ] Claude API reads full JD -> structured reasoning score    |
|  [ ] Claude tailors resume summary per matched JD              |
|  [ ] Claude generates cover letter per role                    |
+=================================================================+
""")
