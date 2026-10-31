"""
Free AI podcast on autopilot.
Topic (Google Sheet) -> Gemini script -> edge-tts audio -> cover -> RSS on GitHub Pages.
Spotify & Apple Podcasts read the RSS feed automatically.
"""
import asyncio, csv, io, json, os, re, subprocess, sys
from datetime import datetime, timezone
from email.utils import format_datetime
from xml.sax.saxutils import escape

import edge_tts
import requests
from PIL import Image, ImageDraw, ImageFont

# ---------- SETTINGS (set as GitHub Secrets / Variables) ----------
GEMINI_API_KEY = os.environ["GEMINI_API_KEY"]
SHEET_CSV_URL  = os.environ["SHEET_CSV_URL"]           # Google Sheet published as CSV
BASE_URL       = os.environ["BASE_URL"].rstrip("/")    # e.g. https://username.github.io/ai-podcast
LANG           = os.environ.get("PODCAST_LANG", "en")  # en | fr | ar
TITLE          = os.environ.get("PODCAST_TITLE", "My AI Podcast")
AUTHOR         = os.environ.get("PODCAST_AUTHOR", "Rachid")
EMAIL          = os.environ.get("PODCAST_EMAIL", "you@example.com")
DESCRIPTION    = os.environ.get("PODCAST_DESCRIPTION", "An AI-generated podcast. Episodes are researched, written and voiced with AI.")
CATEGORY       = os.environ.get("PODCAST_CATEGORY", "Technology")
MODEL          = os.environ.get("GEMINI_MODEL", "gemini-2.5-flash")

VOICES = {
    "en": ("en-US-AriaNeural", "en-US-GuyNeural"),
    "fr": ("fr-FR-DeniseNeural", "fr-FR-HenriNeural"),
    "ar": ("ar-MA-MounaNeural", "ar-MA-JamalNeural"),
}
LANG_NAME = {"en": "English", "fr": "French", "ar": "Arabic"}

DB = "episodes.json"
EP_DIR = "episodes"


def load_db():
    with open(DB, encoding="utf-8") as f:
        return json.load(f)


def next_topic(db):
    done = {e["topic"].strip().lower() for e in db["episodes"]}
    text = requests.get(SHEET_CSV_URL, timeout=30).content.decode("utf-8")
    if text.lstrip().lower().startswith("<!doctype") or "<html" in text[:500].lower():
        raise RuntimeError("SHEET_CSV_URL returns a web page, not CSV. Publish the sheet as CSV.")
    for raw in csv.DictReader(io.StringIO(text)):
        row = {(k or "").strip().lower(): (v or "") for k, v in raw.items()}
        topic = row.get("topic", "").strip()
        approved = row.get("approved", "").strip().lower()
        if topic and approved in ("yes", "y", "oui", "1") and topic.lower() not in done:
            return topic
    return None


def gemini_script(topic):
    prompt = f"""You write a natural, engaging two-host podcast episode in {LANG_NAME[LANG]}.
Host A is curious and asks questions; Host B is the expert who explains clearly.
Topic: {topic}
Length: about 1200-1500 words of dialogue. Be accurate; if unsure of a fact, say so instead of inventing.
No sound effects, no stage directions, no markdown.
Return ONLY JSON:
{{"title": "...", "description": "2-3 sentences", "cover_text": "max 5 words",
  "dialogue": [{{"speaker": "A", "text": "..."}}, {{"speaker": "B", "text": "..."}}]}}"""
    url = f"https://generativelanguage.googleapis.com/v1beta/models/{MODEL}:generateContent"
    r = requests.post(
        url,
        headers={"x-goog-api-key": GEMINI_API_KEY, "Content-Type": "application/json"},
        json={"contents": [{"parts": [{"text": prompt}]}],
              "generationConfig": {"responseMimeType": "application/json", "temperature": 0.8}},
        timeout=180,
    )
    r.raise_for_status()
    text = r.json()["candidates"][0]["content"]["parts"][0]["text"]
    text = re.sub(r"^```json|```$", "", text.strip()).strip()
    return json.loads(text)


async def tts(dialogue, out_mp3, workdir):
    voice_a, voice_b = VOICES[LANG]
    parts = []
    for i, line in enumerate(dialogue):
        txt = line["text"].strip()
        if not txt:
            continue
        voice = voice_a if line["speaker"].upper() == "A" else voice_b
        p = os.path.join(workdir, f"seg_{i:04d}.mp3")
        await edge_tts.Communicate(txt, voice).save(p)
        parts.append(p)
    lst = os.path.join(workdir, "list.txt")
    with open(lst, "w") as f:
        for p in parts:
            f.write(f"file '{os.path.abspath(p)}'\n")
    subprocess.run(["ffmpeg", "-y", "-loglevel", "error", "-f", "concat", "-safe", "0",
                    "-i", lst, "-ac", "1", "-b:a", "64k", out_mp3], check=True)


def duration_seconds(mp3):
    out = subprocess.run(["ffprobe", "-v", "error", "-show_entries", "format=duration",
                          "-of", "csv=p=0", mp3], capture_output=True, text=True, check=True)
    return int(float(out.stdout.strip()))


def font(size):
    for path in ["/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf",
                 "/usr/share/fonts/truetype/liberation/LiberationSans-Bold.ttf"]:
        if os.path.exists(path):
            return ImageFont.truetype(path, size)
    return ImageFont.load_default()


def make_cover(text, path, subtitle=TITLE):
    W = 3000
    img = Image.new("RGB", (W, W))
    d = ImageDraw.Draw(img)
    for y in range(W):  # vertical gradient purple -> dark blue
        t = y / W
        d.line([(0, y), (W, y)], fill=(int(120 - 100 * t), int(40 + 10 * t), int(200 - 80 * t)))
    d.ellipse([1900, -400, 3400, 1100], fill=(255, 120, 80))
    # wrap title
    f = font(260)
    words, lines, cur = text.upper().split(), [], ""
    for w in words:
        test = (cur + " " + w).strip()
        if d.textlength(test, font=f) > W - 400:
            lines.append(cur); cur = w
        else:
            cur = test
    lines.append(cur)
    y = 1500 - len(lines) * 150
    for ln in lines:
        d.text((200, y), ln, font=f, fill="white"); y += 300
    d.text((200, 2650), subtitle, font=font(130), fill=(255, 220, 200))
    img.save(path, "PNG", optimize=True)


def build_feed(db):
    items = []
    for e in reversed(db["episodes"]):
        items.append(f"""    <item>
      <title>{escape(e['title'])}</title>
      <description>{escape(e['description'])}</description>
      <itunes:summary>{escape(e['description'])}</itunes:summary>
      <enclosure url="{BASE_URL}/{e['audio']}" length="{e['bytes']}" type="audio/mpeg"/>
      <guid isPermaLink="false">{e['id']}</guid>
      <pubDate>{e['pubDate']}</pubDate>
      <itunes:duration>{e['duration']}</itunes:duration>
      <itunes:image href="{BASE_URL}/{e['cover']}"/>
      <itunes:explicit>false</itunes:explicit>
      <itunes:episodeType>full</itunes:episodeType>
    </item>""")
    feed = f"""<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:itunes="http://www.itunes.com/dtds/podcast-1.0.dtd" xmlns:atom="http://www.w3.org/2005/Atom">
  <channel>
    <title>{escape(TITLE)}</title>
    <link>{BASE_URL}</link>
    <atom:link href="{BASE_URL}/feed.xml" rel="self" type="application/rss+xml"/>
    <language>{LANG}</language>
    <description>{escape(DESCRIPTION)}</description>
    <itunes:author>{escape(AUTHOR)}</itunes:author>
    <itunes:owner><itunes:name>{escape(AUTHOR)}</itunes:name><itunes:email>{escape(EMAIL)}</itunes:email></itunes:owner>
    <itunes:image href="{BASE_URL}/cover.png"/>
    <itunes:category text="{escape(CATEGORY)}"/>
    <itunes:explicit>false</itunes:explicit>
    <itunes:type>episodic</itunes:type>
{chr(10).join(items)}
  </channel>
</rss>
"""
    with open("feed.xml", "w", encoding="utf-8") as f:
        f.write(feed)


def main():
    db = load_db()
    if not os.path.exists("cover.png"):
        make_cover(TITLE, "cover.png", subtitle=AUTHOR)

    topic = next_topic(db)
    if not topic:
        print("No new approved topic in the sheet. Nothing to do.")
        build_feed(db)
        return

    print(f"Topic: {topic}")
    ep = gemini_script(topic)
    n = len(db["episodes"]) + 1
    slug = f"ep{n:03d}"
    work = f"/tmp/{slug}"; os.makedirs(work, exist_ok=True)

    audio = f"{EP_DIR}/{slug}.mp3"
    cover = f"{EP_DIR}/{slug}.png"
    asyncio.run(tts(ep["dialogue"], audio, work))
    make_cover(ep.get("cover_text") or ep["title"], cover)

    db["episodes"].append({
        "id": f"{slug}-{int(datetime.now().timestamp())}",
        "topic": topic,
        "title": f"{n}. {ep['title']}",
        "description": ep["description"] + " (AI-generated episode.)",
        "audio": audio,
        "cover": cover,
        "bytes": os.path.getsize(audio),
        "duration": duration_seconds(audio),
        "pubDate": format_datetime(datetime.now(timezone.utc)),
    })
    with open(DB, "w", encoding="utf-8") as f:
        json.dump(db, f, ensure_ascii=False, indent=2)
    build_feed(db)
    print(f"Published {slug}: {ep['title']}")


if __name__ == "__main__":
    try:
        main()
    except Exception as exc:
        print(f"ERROR: {exc}")
        sys.exit(1)
