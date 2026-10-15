Run `--validate` once and it'll tell you what's actually alive on your network rather than trusting me.

```bash
cat > ~/rss-automator/feeds_master.py << 'PY'
#!/usr/bin/env python3
"""
Master RSS catalogue + validator for the Termux rss-automator.

Commands
--------
  python feeds_master.py --list                      # show every entry
  python feeds_master.py --list tech                 # one category
  python feeds_master.py --validate                  # probe all feeds
  python feeds_master.py --validate --workers 12 --json report.json
  python feeds_master.py --emit ~/rss-automator/feeds.txt --min-confidence high
  python feeds_master.py --check https://example.com/feed.xml

Confidence key
--------------
  high    long-lived, widely documented; expected to work
  medium  believed live, but the URL scheme has changed before
  low     unverified; may 404, redirect, or need special headers
  avoid   known withdrawn/unusable; retained for reference only

URLs are drawn from training data and WILL drift. Always validate before
trusting a feed; a 200 response with zero entries is a failure, not a pass.
"""

from __future__ import annotations

import argparse
import concurrent.futures as futures
import json
import sys
from pathlib import Path

try:
    import requests
except ImportError:
    requests = None

try:
    import feedparser
except ImportError:
    feedparser = None

USER_AGENT = "feeds-master/1.0 (+personal RSS reader)"
ACCEPT = (
    "application/rss+xml, application/atom+xml, application/xml;q=0.9, "
    "text/xml;q=0.8, */*;q=0.7"
)
CONFIDENCE_ORDER = {"high": 0, "medium": 1, "low": 2, "avoid": 3}

# ---------------------------------------------------------------------------
# THE CATALOGUE   (name, url, confidence, note)
# ---------------------------------------------------------------------------
FEEDS: dict[str, list[tuple[str, str, str, str]]] = {

    "news-world": [
        ("BBC News (world)", "http://feeds.bbci.co.uk/news/world/rss.xml", "high", "Very stable"),
        ("BBC News (top)", "http://feeds.bbci.co.uk/news/rss.xml", "high", ""),
        ("The Guardian (world)", "https://www.theguardian.com/world/rss", "high", ""),
        ("The Guardian (UK tech)", "https://www.theguardian.com/uk/technology/rss", "high", ""),
        ("NYT (home page)", "https://rss.nytimes.com/services/xml/rss/nyt/HomePage.xml", "high", "Paywall on click-through"),
        ("NYT (world)", "https://rss.nytimes.com/services/xml/rss/nyt/World.xml", "high", ""),
        ("NPR (news)", "https://feeds.npr.org/1001/rss.xml", "high", ""),
        ("Al Jazeera", "https://www.aljazeera.com/xml/rss/all.xml", "high", ""),
        ("Deutsche Welle (EN)", "https://rss.dw.com/rdf/rss-en-all", "high", "RDF format"),
        ("France 24 (EN)", "https://www.france24.com/en/rss", "high", ""),
        ("Sky News (world)", "https://feeds.skynews.com/feeds/rss/world.xml", "medium", ""),
        ("CBC (top stories)", "https://www.cbc.ca/webfeed/rss/rss-topstories", "medium", ""),
        ("ABC News (Australia)", "https://www.abc.net.au/news/feed/51120/rss.xml", "low", ""),
        ("NHK World", "https://www3.nhk.or.jp/nhkworld/en/news/rss/all.xml", "low", ""),
        ("Reuters", "https://feeds.reuters.com/reuters/topNews", "avoid", "Reuters withdrew public RSS circa 2020"),
        ("Associated Press", "https://rss.ap.org/Public/TopStories.xml", "medium", "Has moved repeatedly"),
    ],

    "news-us-politics": [
        ("Politico (politics)", "https://rss.politico.com/politics-news.xml", "high", ""),
        ("The Hill", "https://thehill.com/feed/", "high", ""),
        ("ProPublica", "https://www.propublica.org/feeds/propublica/main", "high", "Investigative"),
        ("Axios", "https://api.axios.com/feed/", "medium", ""),
        ("Semafor", "https://www.semafor.com/rss.xml", "low", ""),
        ("Federal Reserve (all press)", "https://www.federalreserve.gov/feeds/press_all.xml", "high", "Primary source"),
    ],

    "tech": [
        ("Hacker News (front page)", "https://news.ycombinator.com/rss", "high", "Official feed"),
        ("Hacker News (newest)", "https://hnrss.org/newest", "high", "Third-party, excellent"),
        ("Hacker News (Ask HN)", "https://hnrss.org/ask", "high", ""),
        ("Hacker News (Show HN)", "https://hnrss.org/show", "high", ""),
        ("Lobsters", "https://lobste.rs/rss", "high", ""),
        ("Ars Technica", "https://feeds.arstechnica.com/arstechnica/index", "high", ""),
        ("The Verge", "https://www.theverge.com/rss/index.xml", "high", ""),
        ("TechCrunch", "https://techcrunch.com/feed/", "high", ""),
        ("Wired", "https://www.wired.com/feed/rss", "high", ""),
        ("MIT Technology Review", "https://www.technologyreview.com/feed/", "high", ""),
        ("The Register", "https://www.theregister.com/headlines.atom", "high", "Atom"),
        ("Slashdot", "http://rss.slashdot.org/Slashdot/slashdotMain", "medium", ""),
        ("Phoronix", "https://www.phoronix.com/rss.php", "high", "Linux hardware"),
        ("LWN (headlines)", "https://lwn.net/headlines/rss", "high", ""),
        ("Hackaday", "https://hackaday.com/feed/", "high", ""),
        ("Tom's Hardware", "https://www.tomshardware.com/feeds/all", "medium", ""),
        ("ServeTheHome", "https://www.servethehome.com/feed/", "medium", ""),
        ("Jeff Geerling", "https://www.jeffgeerling.com/blog.xml", "medium", ""),
        ("DistroWatch", "https://distrowatch.com/news/dw.xml", "medium", ""),
        ("It's FOSS", "https://itsfoss.com/feed/", "medium", ""),
        ("OMG! Ubuntu", "https://www.omgubuntu.co.uk/feed", "medium", ""),
        ("GamingOnLinux", "https://www.gamingonlinux.com/article_rss.php", "medium", ""),
        ("F-Droid", "https://f-droid.org/feed/", "medium", "Android FOSS releases"),
        ("9to5Google", "https://9to5google.com/feed/", "medium", ""),
        ("9to5Mac", "https://9to5mac.com/feed/", "medium", ""),
        ("MacRumors", "https://feeds.macrumors.com/MacRumors-All", "medium", ""),
        ("Android Police", "https://www.androidpolice.com/feed/", "medium", ""),
        ("XDA Developers", "https://www.xda-developers.com/feed/", "medium", ""),
        ("Liliputing", "https://liliputing.com/feed/", "medium", "Small-form-factor computing"),
        ("AnandTech", "https://www.anandtech.com/rss/", "avoid", "Ceased publishing in 2024"),
    ],

    "security-privacy": [
        ("EFF (updates)", "https://www.eff.org/rss/updates.xml", "high", ""),
        ("Schneier on Security", "https://www.schneier.com/feed/atom/", "high", "Atom"),
        ("Krebs on Security", "https://krebsonsecurity.com/feed/", "high", ""),
        ("Bellingcat", "https://www.bellingcat.com/feed/", "medium", "OSINT"),
        ("The Hacker News", "https://feeds.feedburner.com/TheHackersNews", "medium", "Not affiliated with YC"),
        ("Bruce Schneier (legacy)", "https://www.schneier.com/blog/atom.xml", "low", "Superseded by feed above"),
    ],

    "ai-ml": [
        ("arXiv cs.AI", "http://export.arxiv.org/rss/cs.AI", "high", "Preprints"),
        ("arXiv cs.LG", "http://export.arxiv.org/rss/cs.LG", "high", ""),
        ("arXiv cs.CL", "http://export.arxiv.org/rss/cs.CL", "high", "NLP"),
        ("Hugging Face (blog)", "https://huggingface.co/blog/feed.xml", "medium", ""),
        ("Google (AI blog)", "https://blog.google/technology/ai/rss/", "medium", ""),
        ("OpenAI (news)", "https://openai.com/news/rss.xml", "low", "URL has changed before"),
        ("Google DeepMind (blog)", "https://deepmind.google/blog/rss.xml", "low", ""),
        ("Anthropic (news)", "https://www.anthropic.com/news/rss.xml", "low", ""),
        ("Simon Willison", "https://simonwillison.net/atom/everything/", "high", "Atom; LLM-heavy"),
        ("Import AI", "https://importai.substack.com/feed", "low", ""),
    ],

    "science": [
        ("Nature", "https://www.nature.com/nature.rss", "high", ""),
        ("Phys.org", "https://phys.org/rss-feed/", "high", ""),
        ("Quanta Magazine", "https://www.quantamagazine.org/feed/", "high", ""),
        ("IEEE Spectrum", "https://spectrum.ieee.org/feeds/feed.rss", "high", ""),
        ("The Conversation", "https://theconversation.com/articles.atom", "high", "Atom; academic-authored"),
        ("Science (news)", "https://www.science.org/rss/news_current.xml", "medium", ""),
        ("NASA (breaking news)", "https://www.nasa.gov/rss/dyn/breaking_news.rss", "high", ""),
        ("ESA (news)", "https://www.esa.int/rssfeed/Our_Activities/Space_News", "low", ""),
        ("Nautilus", "https://nautil.us/feed/", "low", ""),
        ("Aeon", "https://aeon.co/feed.rss", "high", "Essays, not news"),
    ],

    "finance-markets": [
        ("MarketWatch (top stories)", "http://feeds.marketwatch.com/marketwatch/topstories/", "high", ""),
        ("CNBC (top news)", "https://search.cnbc.com/rs/search/combinedcms/view.xml?partnerId=wrss01&id=100003114", "medium", "Long query string"),
        ("FT (home)", "https://www.ft.com/rss/home", "medium", "Paywalled"),
        ("WSJ (world news)", "https://feeds.a.dj.com/rss/RSSWorldNews.xml", "medium", "Paywalled"),
        ("The Economist (world this week)", "https://www.economist.com/the-world-this-week/rss.xml", "low", ""),
        ("Bloomberg (markets)", "https://feeds.bloomberg.com/markets/news.rss", "medium", ""),
        ("Seeking Alpha", "https://seekingalpha.com/feed", "medium", ""),
        ("CoinDesk", "https://www.coindesk.com/arc/outboundfeeds/rss/", "medium", "Crypto"),
        ("Marginal Revolution", "https://marginalrevolution.com/feed", "high", "Economics blog"),
        ("Stratechery", "https://stratechery.com/feed/", "high", "Partial paywall"),
        ("Astral Codex Ten", "https://astralcodexten.substack.com/feed", "high", ""),
        ("Yahoo Finance", "https://finance.yahoo.com/news/rssindex", "low", ""),
    ],

    "geopolitics-policy": [
        ("War on the Rocks", "https://warontherocks.com/feed/", "high", ""),
        ("Foreign Affairs", "https://www.foreignaffairs.com/rss.xml", "medium", "Metered"),
        ("The Diplomat", "https://thediplomat.com/feed/", "medium", "Asia-Pacific"),
        ("Breaking Defense", "https://breakingdefense.com/feed/", "medium", ""),
        ("Defense One", "https://www.defenseone.com/rss/all/", "low", ""),
        ("Lawfare", "https://www.lawfaremedia.org/feeds/articles.xml", "low", "Rebranded; path unstable"),
        ("CFR", "https://www.cfr.org/rss.xml", "low", ""),
        ("Carnegie Endowment", "https://carnegieendowment.org/rss/solr/", "low", ""),
        ("Chatham House", "https://www.chathamhouse.org/rss.xml", "low", ""),
        ("Brookings", "https://www.brookings.edu/feed/", "low", ""),
        ("Kyiv Independent", "https://kyivindependent.com/feed/", "low", ""),
        ("Meduza (EN)", "https://meduza.io/rss/en/all", "low", ""),
        ("The Moscow Times", "https://www.themoscowtimes.com/rss/news", "low", ""),
        ("SCMP", "https://www.scmp.com/rss/91/feed", "low", ""),
    ],

    "health-medicine": [
        ("STAT News", "https://www.statnews.com/feed/", "medium", ""),
        ("KFF Health News", "https://kffhealthnews.org/feed/", "medium", ""),
        ("BMJ (current)", "https://www.bmj.com/rss/current.xml", "low", ""),
        ("NEJM (current issue)", "https://www.nejm.org/action/showFeed?type=etoc&feed=rss&jc=nejm", "low", ""),
        ("The Lancet", "https://www.thelancet.com/rssfeed/lancet_current.xml", "low", ""),
    ],

    "climate-energy": [
        ("Carbon Brief", "https://www.carbonbrief.org/feed", "medium", ""),
        ("Inside Climate News", "https://insideclimatenews.org/feed/", "medium", ""),
        ("Electrek", "https://electrek.co/feed/", "medium", ""),
        ("CleanTechnica", "https://cleantechnica.com/feed/", "medium", ""),
        ("Canary Media", "https://www.canarymedia.com/rss.xml", "low", ""),
        ("Utility Dive", "https://www.utilitydive.com/feeds/news/", "low", ""),
    ],

    "longform-culture": [
        ("Longreads", "https://longreads.com/feed/", "high", ""),
        ("Daring Fireball", "https://daringfireball.net/feeds/main", "high", ""),
        ("Pitchfork (news)", "https://pitchfork.com/feed/feed-news/rss", "medium", ""),
        ("The Atlantic", "https://www.theatlantic.com/feed/all/", "low", "Metered"),
        ("New Yorker", "https://www.newyorker.com/feed/everything", "low", "Metered"),
        ("404 Media", "https://www.404media.co/rss/", "low", ""),
    ],

    "dev-blogs": [
        ("GitHub (blog)", "https://github.blog/feed/", "high", ""),
        ("Rust (blog)", "https://blog.rust-lang.org/feed.xml", "high", ""),
        ("Python Insider", "https://pythoninsider.blogspot.com/feeds/posts/default", "medium", ""),
        ("Go (blog)", "https://go.dev/blog/feed.atom", "medium", "Atom"),
        ("Kubernetes (blog)", "https://kubernetes.io/feed.xml", "medium", ""),
        ("PostgreSQL (news)", "https://www.postgresql.org/news.rss", "medium", ""),
        ("Julia Evans", "https://jvns.ca/atom.xml", "medium", "Atom"),
        ("Dan Luu", "https://danluu.com/atom.xml", "medium", "Atom; infrequent"),
        ("Julia Language (blog)", "https://julialang.org/feed.xml", "low", ""),
    ],

    "aggregators-reddit": [
        ("r/programming", "https://www.reddit.com/r/programming/.rss", "high", ""),
        ("r/netsec", "https://www.reddit.com/r/netsec/.rss", "high", ""),
        ("r/linux", "https://www.reddit.com/r/linux/.rss", "high", ""),
        ("r/selfhosted", "https://www.reddit.com/r/selfhosted/.rss", "high", ""),
        ("r/Android", "https://www.reddit.com/r/Android/.rss", "high", ""),
        ("r/termux", "https://www.reddit.com/r/termux/.rss", "high", "Directly relevant to you"),
        ("r/dataisbeautiful", "https://www.reddit.com/r/dataisbeautiful/.rss", "medium", ""),
        ("r/worldnews", "https://www.reddit.com/r/worldnews/.rss", "medium", "Heavy moderation"),
    ],

    "sports": [
        ("BBC Sport", "http://feeds.bbci.co.uk/sport/rss.xml", "high", ""),
        ("ESPN (news)", "https://www.espn.com/espn/rss/news", "medium", ""),
        ("Sky Sports (football)", "https://www.skysports.com/rss/11095", "low", ""),
        ("Formula 1 (latest)", "https://www.formula1.com/en/latest/all.xml", "low", ""),
    ],
}


# ---------------------------------------------------------------------------
# Helpers
# ---------------------------------------------------------------------------
def iter_feeds(categories=None, min_confidence="low"):
    floor = CONFIDENCE_ORDER[min_confidence]
    for category, entries in FEEDS.items():
        if categories and category not in categories:
            continue
        for name, url, confidence, note in entries:
            if CONFIDENCE_ORDER[confidence] <= floor:
                yield category, name, url, confidence, note


def _require_deps():
    missing = []
    if requests is None:
        missing.append("requests")
    if feedparser is None:
        missing.append("feedparser")
    if missing:
        sys.exit(
            "Missing Python modules: " + ", ".join(missing)
            + "\nInstall with:  pip install " + " ".join(missing)
        )


def probe(url, timeout=20):
    """Fetch and parse one feed. Returns a result dict."""
    result = {
        "url": url, "status": None, "entries": 0,
        "ok": False, "detail": "",
    }
    headers = {"User-Agent": USER_AGENT, "Accept": ACCEPT}
    try:
        resp = requests.get(
            url, headers=headers, timeout=(10, timeout), allow_redirects=True
        )
        result["status"] = resp.status_code
        if resp.status_code != 200:
            result["detail"] = f"HTTP {resp.status_code}"
            return result

        parsed = feedparser.parse(resp.content)
        count = len(parsed.entries)
        result["entries"] = count

        if count == 0:
            result["detail"] = "HTTP 200 but zero entries (likely not a feed)"
            return result

        if parsed.bozo:
            exc = type(parsed.get("bozo_exception")).__name__
            result["detail"] = f"malformed XML ({exc}) but {count} entries recovered"
        result["ok"] = True

    except Exception as exc:  # noqa: BLE001 - report anything
        result["detail"] = f"{type(exc).__name__}: {exc}"
    return result


# ---------------------------------------------------------------------------
# Commands
# ---------------------------------------------------------------------------
def cmd_list(args):
    categories = args.categories or None
    current = None
    for category, name, url, confidence, note in iter_feeds(
        categories, args.min_confidence
    ):
        if category != current:
            print(f"\n=== {category} ===")
            current = category
        flag = "" if confidence == "high" else f"  [{confidence}]"
        print(f"  {name}{flag}\n    {url}")
        if note and args.verbose:
            print(f"    note: {note}")
    print()


def cmd_check(args):
    _require_deps()
    res = probe(args.url, args.timeout)
    verdict = "OK" if res["ok"] else "FAIL"
    print(f"{verdict}  {args.url}")
    print(f"  status={res['status']} entries={res['entries']}")
    if res["detail"]:
        print(f"  detail: {res['detail']}")
    return 0 if res["ok"] else 1


def cmd_validate(args):
    _require_deps()
    targets = list(iter_feeds(args.categories or None, args.min_confidence))
    print(f"Probing {len(targets)} feeds with {args.workers} workers...\n")

    results = []
    with futures.ThreadPoolExecutor(max_workers=args.workers) as pool:
        jobs = {
            pool.submit(probe, url, args.timeout): (category, name, url, conf, note)
            for category, name, url, conf, note in targets
        }
        for i, job in enumerate(futures.as_completed(jobs), 1):
            category, name, url, conf, note = jobs[job]
            res = job.result()
            res.update(
                {"category": category, "name": name, "confidence": conf}
            )
            results.append(res)
            mark = "ok  " if res["ok"] else "FAIL"
            print(
                f"[{i:>3}/{len(targets)}] {mark} {res['entries']:>4} entries  "
                f"{name}",
                flush=True,
            )

    good = [r for r in results if r["ok"]]
    bad = [r for r in results if not r["ok"]]

    print("\n" + "=" * 60)
    print(f"WORKING: {len(good)}/{len(results)}")
    print("=" * 60)
    for r in sorted(good, key=lambda x: -x["entries"])[:40]:
        print(f"  {r['entries']:>4}  {r['name']}  ({r['confidence']})")

    if bad:
        print(f"\nFAILED: {len(bad)}")
        for r in bad:
            print(f"  {r['name']}  [{r['confidence']}]  {r['detail']}")

    if args.json:
        out = Path(args.json).expanduser()
        out.write_text(json.dumps(results, indent=2), encoding="utf-8")
        print(f"\nFull report written to {out}")

    return 0


def cmd_emit(args):
    _require_deps() if args.validate_first else None
    targets = list(iter_feeds(args.categories or None, args.min_confidence))

    if args.validate_first:
        print(f"Validating {len(targets)} feeds before writing...")
        keep = []
        with futures.ThreadPoolExecutor(max_workers=args.workers) as pool:
            jobs = {pool.submit(probe, url, args.timeout): (cat, nm, url, cf, nt)
                    for cat, nm, url, cf, nt in targets}
            for job in futures.as_completed(jobs):
                cat, nm, url, cf, nt = jobs[job]
                if job.result()["ok"]:
                    keep.append((cat, nm, url, cf, nt))
        targets = sorted(keep, key=lambda t: (t[0], t[1]))
        print(f"{len(targets)} feeds survived validation.")

    out = Path(args.emit).expanduser()
    out.parent.mkdir(parents=True, exist_ok=True)

    lines = [
        "# Generated by feeds_master.py",
        "# One URL per line. Lines starting with # are ignored.",
        "# The automator re-reads this file every poll cycle.",
        "",
    ]
    current = None
    for category, name, url, confidence, note in targets:
        if category != current:
            lines.append(f"\n# --- {category} ---")
            current = category
        suffix = "" if confidence == "high" else f"  ({confidence})"
        lines.append(f"# {name}{suffix}")
        lines.append(url)

    out.write_text("\n".join(lines) + "\n", encoding="utf-8")
    print(f"Wrote {len(targets)} feeds to {out}")
    return 0


# ---------------------------------------------------------------------------
# Entry point
# ---------------------------------------------------------------------------
def main():
    parser = argparse.ArgumentParser(
        description="Master RSS catalogue and validator.",
        formatter_class=argparse.RawDescriptionHelpFormatter,
    )
    sub = parser.add_subparsers(dest="command")

    def add_common(p):
        p.add_argument("--min-confidence", default="low",
                       choices=list(CONFIDENCE_ORDER),
                       help="lowest confidence tier to include (default: low)")
        p.add_argument("--timeout", type=int, default=20,
                       help="per-request read timeout in seconds")
        p.add_argument("--workers", type=int, default=8,
                       help="parallel probes (default: 8)")

    p_list = sub.add_parser("list", help="print the catalogue")
    p_list.add_argument("categories", nargs="*")
    p_list.add_argument("--min-confidence", default="low",
                        choices=list(CONFIDENCE_ORDER))
    p_list.add_argument("-v", "--verbose", action="store_true")
    p_list.set_defaults(func=cmd_list)

    p_val = sub.add_parser("validate", help="probe every feed")
    add_common(p_val)
    p_val.add_argument("categories", nargs="*")
    p_val.add_argument("--json", help="write full results to this path")
    p_val.set_defaults(func=cmd_validate)

    p_emit = sub.add_parser("emit", help="write a feeds.txt for the automator")
    add_common(p_emit)
    p_emit.add_argument("categories", nargs="*")
    p_emit.add_argument("--validate-first", action="store_true",
                        help="drop feeds that fail probing")
    p_emit.set_defaults(func=cmd_emit)

    p_check = sub.add_parser("check", help="probe a single URL")
    p_check.add_argument("url")
    p_check.add_argument("--timeout", type=int, default=20)
    p_check.set_defaults(func=cmd_check)

    # Allow legacy --flag style as well as subcommands.
    argv = sys.argv[1:]
    if argv and argv[0].startswith("--"):
        argv = ["list"] + argv
    args = parser.parse_args(argv)

    if not getattr(args, "func", None):
        parser.print_help()
        return 1
    return args.func(args) or 0


if __name__ == "__main__":
    raise SystemExit(main())
PY

chmod 700 ~/rss-automator/feeds_master.py
```

Then the workflow:

```bash
# See what's in the catalogue
python ~/rss-automator/feeds_master.py list

# Probe everything — this is the step that replaces my guesses with facts
python ~/rss-automator/feeds_master.py validate --workers 12 --json ~/rss-automator/validate-report.json

# Write only the feeds that actually answered, high-confidence tier first
python ~/rss-automator/feeds_master.py emit ~/rss-automator/feeds.txt --validate-first --workers 12

# Confirm the automator agrees
python ~/rss-automator/rss_automator.py --once
sqlite3 ~/rss-automator/state/rss.sqlite3 "SELECT feed_url, COUNT(*) FROM entries GROUP BY 1;"
```

Three things worth knowing about how this behaves:

**Zero entries counts as failure.** A 200 with an empty entry list usually means you fetched an HTML page, a paywall interstitial, or a consent gate — not a feed. Treating that as success is how you end up with silent feeds that never produce anything, so `probe()` rejects it.

**`--validate-first` is the only honest way to build `feeds.txt`.** Roughly speaking, I'd expect most `high` entries to survive, a chunk of `medium` to have moved, and a fair number of `low` to be dead. The `avoid` tier is there so you don't rediscover Reuters' withdrawal yourself.

**Reddit feeds will likely need attention.** Reddit has been progressively restricting unauthenticated RSS and blocking generic user agents. If `r/termux` and friends all fail together with 403s, that's a pattern, not a coincidence — and the fix is either a custom UA or dropping them.

The `emit` output is deliberately compatible with your existing `rss_automator.py`: flat URLs, `#` comments for category headers, re-read each cycle so you can append a feed without touching the service.


    """Print catalogue entries, optionally filtered by category."""
    target = FEEDS.get(category) if category else None
    if category and not target:
        print(f"Category not found: {category}", file=sys.stderr)
        print(f"Available: {', '.join(FEEDS.keys())}", file=sys.stderr)
        sys.exit(1)

    if target:
        print(f"\n{category.upper()}")
        print("-" * 60)
        for entry in target:
            confidence_tag = f" [{entry.confidence}]" if entry.confidence != "high" else ""
            print(f"  {entry.name}{confidence_tag}")
            print(f"    {entry.url}")
            if entry.note:
                print(f"    Note: {entry.note}")
        print()
    else:
        # Show all categories
        for cat_name, cat_entries in sorted(FEEDS.items()):
            print(f"\n{cat_name.upper()}")
            print("-" * 60)
            for entry in cat_entries:
                confidence_tag = f" [{entry.confidence}]" if entry.confidence != "high" else ""
                print(f"  {entry.name}{confidence_tag}")
                print(f"    {entry.url}")
            print(f"  Total: {len(cat_entries)} feeds")
        print()


def cmd_validate(min_confidence: str = "low", workers: int = 8, timeout: int = 15, json_path: str | None = None) -> int:
    """Validate feeds and optionally save results to JSON."""
    _require_deps()
    
    # Filter by confidence
    min_level = CONFIDENCE_ORDER.get(min_confidence, 3)
    feeds = [e for e in all_feeds() if CONFIDENCE_ORDER[e.confidence] <= min_level]
    
    print(f"Validating {len(feeds)} feeds ({min_confidence}+) with {workers} workers...")
    print("-" * 70)
    
    start_time = time.time()
    results = validate_all(workers)
    elapsed = time.time() - start_time
    
    # Count results by status
    alive = [r for r in results if r.alive]
    dead = [r for r in results if not r.alive]
    
    print(f"\n{'='*70}")
    print(f"RESULTS in {elapsed:.1f}s: {len(alive)}/{len(results)} feeds working")
    print(f"{'='*70}")
    
    # Sort working feeds by entry count
    alive_sorted = sorted(alive, key=lambda r: r.entries, reverse=True)
    for r in alive_sorted[:20]:
        print(f"  [{r.entries:>3}] {r.name} ({r.confidence})")
    
    if dead:
        print(f"\n--- FAILED ({len(dead)}) ---")
        for r in dead[:15]:
            err = r.error or f"HTTP {r.status_code}"
            print(f"  {r.name} ({r.confidence}): {err}")
    
    if json_path:
        out_path = Path(json_path).expanduser()
        out_path.parent.mkdir(parents=True, exist_ok=True)
        output = [
            {
                "url": r.url,
                "name": r.name,
                "confidence": r.confidence,
                "alive": r.alive,
                "entries": r.entries,
                "status_code": r.status_code,
                "error": r.error,
            }
            for r in results
        ]
        out_path.write_text(json.dumps(output, indent=2), encoding="utf-8")
        print(f"\nFull report saved to: {out_path}")
    
    return 0 if not dead else 1


def cmd_emit(path: str, min_confidence: str = "high", validate_first: bool = False, workers: int = 8) -> int:
    """Write feeds.txt file with validated feeds."""
    out_path = Path(path).expanduser()
    out_path.parent.mkdir(parents=True, exist_ok=True)
    
    # Filter by confidence
    min_level = CONFIDENCE_ORDER.get(min_confidence, 3)
    feeds = [e for e in all_feeds() if CONFIDENCE_ORDER[e.confidence] <= min_level]
    
    if validate_first:
        _require_deps()
        print(f"Validating {len(feeds)} feeds before writing...")
        results = validate_all(workers)
        # Only keep alive feeds
        valid_urls = {r.url for r in results if r.alive}
        feeds = [e for e in feeds if e.url in valid_urls]
        print(f"After validation: {len(feeds)} feeds")
    
    lines = [
        "# Generated by feeds_master.py",
        "# One URL per line. Lines starting with # are ignored.",
        f"# Confidence threshold: {min_confidence}",
        f"# Validated: {validate_first}",
        "",
    ]
    
    current_cat = None
    for entry in feeds:
        # Find category for this feed
        found_cat = None
        for cat_name, cat_entries in FEEDS.items():
            if any(e.url == entry.url for e in cat_entries):
                found_cat = cat_name
                break
        
        if found_cat and found_cat != current_cat:
            lines.append(f"\n# === {found_cat.upper()} ===")
            current_cat = found_cat
        
        lines.append(f"# {entry.name}")
        lines.append(entry.url)
    
    out_path.write_text("\n".join(lines) + "\n", encoding="utf-8")
    print(f"Wrote {len(feeds)} feeds to {out_path}")
    
    # Return exit code
    return 0


def _require_deps() -> None:
    """Check that required modules are installed."""
    missing = []
    if requests is None:
        missing.append("requests")
    if feedparser is None:
        missing.append("feedparser")
    if missing:
        print(f"ERROR: Missing Python modules: {', '.join(missing)}", file=sys.stderr)
        print("Install with: pip install " + " ".join(missing), file=sys.stderr)
        sys.exit(1)


def cmd_check(url: str, timeout: int = 15) -> int:
    """Check a single feed URL."""
    _require_deps()
    
    entry = FeedEntry(name=url, url=url, confidence="unknown", note="")
    result = probe_feed(entry, timeout)
    
    if result.alive:
        print(f"✓ OK: {result.name}")
        print(f"  Entries: {result.entries}")
        print(f"  Status: {result.status_code}")
        return 0
    else:
        print(f"✗ FAIL: {result.name}")
        if result.status_code:
            print(f"  Status: {result.status_code}")
        if result.error:
            print(f"  Error: {result.error}")
        return 1


def main():
    parser = argparse.ArgumentParser(
        description="Master RSS catalogue + validator",
        formatter_class=argparse.RawDescriptionHelpFormatter,
    )
    
    subparsers = parser.add_subparsers(dest="command")
    
    # list command
    p_list = subparsers.add_parser("list", help="List feeds")
    p_list.add_argument("category", nargs="?", help="Filter by category")
    
    # validate command
    p_validate = subparsers.add_parser("validate", help="Validate feeds")
    p_validate.add_argument("--min-confidence", default="low", 
                          choices=["high", "medium", "low", "avoid"])
    p_validate.add_argument("--workers", type=int, default=8)
    p_validate.add_argument("--timeout", type=int, default=15)
    p_validate.add_argument("--json", help="Save results to JSON file")
    
    # emit command
    p_emit = subparsers.add_parser("emit", help="Write feeds.txt")
    p_emit.add_argument("path", help="Output file path")
    p_emit.add_argument("--min-confidence", default="high",
                       choices=["high", "medium", "low"])
    p_emit.add_argument("--validate-first", action="store_true",
                       help="Only emit feeds that pass validation")
    p_emit.add_argument("--workers", type=int, default=8)
    
    # check command
    p_check = subparsers.add_parser("check", help="Check single URL")
    p_check.add_argument("url", help="Feed URL to check")
    p_check.add_argument("--timeout", type=int, default=15)
    
    args = parser.parse_args()
    
    if args.command == "list":
        cmd_list(args.category)
    elif args.command == "validate":
        return cmd_validate(
            args.min_confidence, args.workers, args.timeout, args.json
        )
    elif args.command == "emit":
        return cmd_emit(args.path, args.min_confidence, 
                       args.validate_first, args.workers)
    elif args.command == "check":
        return cmd_check(args.url, args.timeout)
    else:
        parser.print_help()
        return 1


if __name__ == "__main__":
    raise SystemExit(main())
PY

chmod 700 ~/rss-automator/feeds_master.py
```

## Usage Summary

Once complete, run these steps:

| Command | Purpose |
|---------|---------|
| `python ~/rss-automator/feeds_master.py list` | Browse all available feeds |
| `python ~/rss-automator/feeds_master.py list tech` | Show only tech feeds |
| `python ~/rss-automator/feeds_master.py validate` | Test all feeds for uptime |
| `python ~/rss-automator/feeds_master.py validate --json report.json` | Save validation report |
| `python ~/rss-automator/feeds_master.py emit ~/rss-automator/feeds.txt --validate-first` | Create verified feeds.txt |
| `python ~/rss-automator/feeds_master.py check https://example.com/feed` | Test single URL |

Then verify with your existing smoke tests:

```bash
python ~/rss-automator/rss_automator.py --once
sqlite3 ~/rss-automator/state/rss.sqlite3 "SELECT feed_url, COUNT(*) FROM entries GROUP BY 1;"
```
