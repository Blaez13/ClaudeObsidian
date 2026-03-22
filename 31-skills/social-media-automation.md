# Skill: Social Media Automation Pipeline

> Content created → graphic/video generated → posted to platforms.
> No browser. No manual uploads. Code and API calls only.

---

## WHEN TO USE

- Any time social content needs to be created AND posted
- Scheduled content calendar execution
- Client social media management (SR)
- Paw & Pantry brand content publishing
- Repurposing recipes, blog posts, or tips into social posts

---

## THE PIPELINE

```
Step 1: CONTENT (Lyra/Cass write it)
   ↓
Step 2: VISUAL (generate graphic or video via API)
   ↓
Step 3: POST (publish to platforms via API)
   ↓
Step 4: TRACK (check engagement, adjust)
```

Every step happens through code. No browser. No manual uploads.
An agent can run this entire pipeline autonomously.

---

## STEP 1 — CONTENT CREATION

Lyra or Cass writes the post content using their existing skills:
- `social-media` skill for content calendar planning
- `copywriting` skill for captions
- `content-strategy` skill for editorial direction

**Output:** A content brief per post:
```yaml
post:
  platform: [instagram, facebook, tiktok, pinterest]
  type: image | video | carousel
  caption: "Your dog deserves Sunday dinners too. 🐾"
  hashtags: "#homemadedogfood #pawandpantry #dogmom"
  visual_prompt: "Warm kitchen scene, golden retriever looking
    at a bowl of fresh chicken and sweet potato, soft lighting,
    overhead angle, food photography style"
  schedule: "2026-03-25T11:00:00-05:00"
  client: "Paw & Pantry"  # or client name for SR
```

---

## STEP 2 — VISUAL GENERATION

### For Static Images: Creatomate API

Best for: recipe cards, quote graphics, product announcements,
tip cards, branded templates.

**How it works:**
1. Design a template once (recipe card, quote card, tip card)
2. API fills in dynamic content (recipe name, image, caption)
3. Returns a finished image URL ready to post

```python
import requests

response = requests.post(
    "https://api.creatomate.com/v1/renders",
    headers={"Authorization": f"Bearer {CREATOMATE_API_KEY}"},
    json={
        "template_id": "recipe-card-template-id",
        "modifications": {
            "recipe_name": "Chicken & Sweet Potato Bowl",
            "recipe_image": "https://storage.example.com/recipe.jpg",
            "prep_time": "15 min",
            "tagline": "Real food, made with love."
        }
    }
)
image_url = response.json()[0]["url"]
```

**Cost:** Starts free (limited renders), then ~$19/month
**Setup:** Sign up at creatomate.com, design templates in
their editor, get API key

**Alternative — Orshot API:**
Similar concept but also handles posting in the same call.
Can import Canva/Figma templates directly as API templates.
```python
response = requests.post(
    "https://api.orshot.com/v1/studio/render",
    headers={"Authorization": f"Bearer {ORSHOT_API_KEY}"},
    json={
        "templateId": 1234,
        "modifications": {
            "title": "Chicken & Sweet Potato Bowl",
            "description": "15 min prep • Grain-free • Adult dogs"
        },
        "response": {"type": "url", "format": "png"},
        "publish": {
            "accounts": [1],  # connected social account
            "content": "Your dog deserves Sunday dinners too. 🐾"
        }
    }
)
```

### For Video: Google Veo 3.1 API (Gemini)

Best for: recipe cooking process clips, ingredient showcases,
dog reaction videos, short-form content for Reels/TikTok/Shorts.

**Why Veo 3.1:**
- Already in your Google Cloud ecosystem (same API key)
- Generates native audio (sound effects, music)
- Supports 9:16 portrait format (Reels, TikTok, Shorts)
- Supports 16:9 landscape (YouTube, Facebook)
- Up to 8 seconds per clip, extensible for longer content
- Image-to-video: feed a recipe photo, get a cooking scene
- Python SDK via google-genai package

```python
from google import genai
from google.genai import types
import time

client = genai.Client()

operation = client.models.generate_videos(
    model="veo-3.1-generate-preview",
    prompt="""Close-up of a golden retriever watching eagerly as
    fresh chicken and sweet potatoes are placed in a beautiful
    ceramic dog bowl. Warm kitchen lighting, shallow depth of
    field, food photography style. The dog's tail wags.""",
    config=types.GenerateVideosConfig(
        aspect_ratio="9:16",  # Portrait for Reels/TikTok
        # aspect_ratio="16:9",  # Landscape for YouTube/Facebook
    ),
)

# Wait for generation (typically 1-3 minutes)
while not operation.done:
    time.sleep(20)
    operation = client.operations.get(operation)

video = operation.result.generated_videos[0]
client.files.download(file=video.video)
video.video.save("recipe_reel.mp4")
```

**Cost:** Token-based through Gemini API. Free tier available
in AI Studio for development. Production pricing via Vertex AI.
**Setup:** Already have GEMINI_API_KEY configured

### For Image Generation: Google Imagen 3 (Nano Banana)

Best for: product shots, ingredient flatlays, lifestyle imagery
when you don't have real photos.

```python
from google import genai

client = genai.Client()

response = client.models.generate_images(
    model="imagen-3.0-generate-002",
    prompt="""Overhead shot of fresh dog food ingredients on a
    marble kitchen counter: raw chicken breast, sweet potatoes,
    blueberries, spinach, a small bowl of fish oil. Warm natural
    lighting, lifestyle food photography, millennial aesthetic.""",
    config={"number_of_images": 4}
)

for i, image in enumerate(response.generated_images):
    image.image.save(f"ingredient_shot_{i}.png")
```

---

## STEP 3 — POSTING TO PLATFORMS

### Option A: Late/Zernio API (Developer-First, Multi-Platform)

Best for: Paw & Pantry (your own brand), any scenario where
agents need to post autonomously without GHL.

**Platforms:** Instagram, Facebook, TikTok, LinkedIn, X/Twitter,
YouTube, Threads, Pinterest, Reddit, Bluesky, Telegram, Snapchat,
Google Business Profile — 14 platforms from one API.

**Pricing:**
- Free: 20 posts/month, 2 profiles
- Build: $19/month, 100 posts/month
- Accelerate: $49/month, 500 posts/month

```python
import requests

# Post to multiple platforms in one call
response = requests.post(
    "https://getlate.dev/api/v1/posts",
    headers={"Authorization": f"Bearer {LATE_API_KEY}"},
    json={
        "content": "Your dog deserves Sunday dinners too. 🐾\n\n"
                   "#homemadedogfood #pawandpantry #dogmom",
        "profileId": "PROFILE_ID",
        "platforms": [
            {"platform": "instagram", "accountId": "IG_ACCOUNT_ID"},
            {"platform": "facebook", "accountId": "FB_ACCOUNT_ID"},
            {"platform": "tiktok", "accountId": "TT_ACCOUNT_ID"},
            {"platform": "pinterest", "accountId": "PIN_ACCOUNT_ID"},
        ],
        "mediaUrls": ["https://storage.example.com/recipe_card.png"],
        "scheduledFor": "2026-03-25T16:00:00Z",
        "timezone": "America/Chicago"
    }
)
```

### Option B: GHL Social Planner (For SR Clients)

Best for: Stellaris Ridge client social media. Everything stays
in the client's GHL sub-account dashboard.

**Platforms:** Facebook Pages, Instagram Business, LinkedIn,
Google Business Profile, TikTok

**How agents use it:**
- Bulk CSV upload (up to 90 posts per batch)
- GHL REST API for individual post scheduling
- Recurring posts for evergreen content
- Review auto-posting (turns Google/Facebook reviews into posts)

```python
# Bulk upload approach — agent generates CSV, uploads to GHL
import csv

posts = [
    {
        "platform": "facebook,instagram",
        "date": "2026-03-25",
        "time": "11:00",
        "caption": "Spring HVAC maintenance special! Book now.",
        "media_url": "https://storage.example.com/hvac_spring.png",
        "hashtags": "#hvac #springmaintenance #plymouthwi"
    },
    # ... more posts
]

with open("social_calendar.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=posts[0].keys())
    writer.writeheader()
    writer.writerows(posts)

# Upload CSV to GHL Social Planner via dashboard or API
```

### Option C: N8N Workflow (Complex Multi-Step)

Best for: Workflows that need multiple steps — generate content,
create image, resize for each platform, post to each platform
with platform-specific captions.

N8N has native nodes for: Canva, Facebook, Instagram, LinkedIn,
Twitter/X, Google Business, plus HTTP nodes for any API.

Use when GHL Social Planner or Late API aren't flexible enough.

---

## STEP 4 — TRACKING & OPTIMIZATION

### Through GHL (SR Clients)
GHL Social Planner tracks: impressions, clicks, likes, comments
per post. Sufficient for client reporting.

### Through Late API (P&P)
Late provides: likes, shares, reach, impressions, clicks, views
unified across all platforms via API.

```python
# Get analytics for a post
response = requests.get(
    f"https://getlate.dev/api/v1/posts/{post_id}/analytics",
    headers={"Authorization": f"Bearer {LATE_API_KEY}"}
)
```

### Using Corey's Skills
- `analytics` skill — set up tracking and interpret results
- `social-media` skill — adjust strategy based on performance

---

## FULL AUTONOMOUS WORKFLOW EXAMPLE (Paw & Pantry)

```
1. TRIGGER: Weekly content calendar says "Post recipe card Tuesday"

2. LYRA reads the recipe from C:\PawAndPantry\content\recipes\
   Writes caption + hashtags + visual prompt

3. AGENT calls Creatomate API with recipe card template
   → Returns branded image URL

4. AGENT calls Veo 3.1 API with cooking scene prompt
   → Returns 8-second 9:16 video with audio

5. AGENT calls Late API twice:
   - Image post → Instagram feed, Facebook, Pinterest
   - Video post → Instagram Reels, TikTok, YouTube Shorts

6. AGENT logs what was posted in
   C:\PawAndPantry\content\social\posted-log.md

7. SAGE reviews the post content (brand voice check)
   Note: Sage reviews BEFORE posting in production.
   During testing, post-review is acceptable.
```

---

## TOOL SETUP CHECKLIST

### Phase 5 (Paw & Pantry)

```
☐ Sign up for Late/Zernio (getlate.dev) — free tier
☐ Connect P&P Instagram, Facebook, TikTok, Pinterest accounts
☐ Store LATE_API_KEY in environment variables
☐ Sign up for Creatomate (creatomate.com) — free tier
☐ Design 3 templates: recipe card, tip card, quote card
☐ Store CREATOMATE_API_KEY in environment variables
☐ Test Veo 3.1 in Google AI Studio (aistudio.google.com)
☐ Test full pipeline: content → image → post
☐ Install Python dependencies:
  pip install google-genai requests --break-system-packages
```

### Phase 4 (Stellaris Ridge Clients)

```
☐ GHL Social Planner configured per client sub-account
☐ Client social accounts connected in GHL
☐ CSV template created for bulk scheduling
☐ Test: bulk upload 5 posts via CSV → verify they publish
```

---

## CANVA — WHERE IT FITS

Canva is excellent for manual template creation but has limited
agent automation. The API is enterprise-only for direct access.

**Best use of Canva in your workflow:**
- YOU use Canva Pro ($13/month) to design initial templates
- Export those templates to Creatomate or Orshot for API access
- Agents then use the API versions for automated generation

**Canva + Make.com automation** is possible (export designs →
post to social) but adds another tool subscription ($9+/month)
and complexity. Late API is simpler for the posting step.

**Verdict:** Use Canva for YOUR manual design work. Use
Creatomate/Orshot + Late API for agent automation.

---

## TOOLS NOT RECOMMENDED

| Tool | Why Skip |
|------|----------|
| Buffer/Hootsuite | Consumer-grade, no API for agent use |
| SocialBee | Great AI Copilot but not API-first |
| Predis | Too consumer-focused, limited API |
| PostEverywhere | New, unproven reliability |
| Zapier for posting | Works but Late API is cheaper and simpler for the same job |

---

*The goal: Lyra writes it, the API makes it beautiful,
the API posts it everywhere, and nobody opens a browser.*
