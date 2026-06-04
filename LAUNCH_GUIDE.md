# Toolify Core — Complete Launch & Automation Guide

## Part 1: Deploy on Coolify (Step by Step)

---

### Step 1 — Prepare your files on GitHub

1. Create a free GitHub account at github.com if you don't have one
2. Create a new repository called `toolifycore`
3. Upload your entire toolifycore folder (index.html + tools/ folder)
4. Make sure repository is set to **Public**

Your structure should look like:
```
toolifycore/
├── index.html          ← main landing page
├── tools/
│   ├── img-compress.html
│   ├── word-count.html
│   ├── age-calc.html
│   ├── pdf-compress.html
│   └── ... (add more tools here)
└── README.md
```

---

### Step 2 — Connect GitHub to Coolify

1. Open your Coolify dashboard (your VPS IP / coolify)
2. Go to **Sources** → Click **Add Source**
3. Select **GitHub** → Click **Register Now**
4. Follow GitHub OAuth steps to connect your account
5. Once connected, you'll see all your repositories

---

### Step 3 — Create New Application in Coolify

1. Go to **Projects** → **New Project** → name it `Toolify Core`
2. Inside the project, click **New Resource** → **Application**
3. Select **GitHub** as source
4. Choose your `toolifycore` repository
5. Select branch: `main`

**Build settings:**
- Build Pack: **Static** (since it's pure HTML/CSS/JS)
- Publish Directory: `/` (root of repo)
- No build command needed

---

### Step 4 — Configure Domain

1. In Application settings → **Domains**
2. Add your domain: `toolifycore.com` (or your actual domain)
3. Enable **HTTPS / SSL** → Coolify auto-handles Let's Encrypt

If you bought the domain on Namecheap or Godaddy:
- Go to your domain DNS settings
- Add an **A Record**: `@` → your VPS IP address
- Add a **CNAME Record**: `www` → `toolifycore.com`
- DNS propagates in 10–30 minutes

---

### Step 5 — Deploy

1. Click **Deploy** in Coolify
2. Watch the build log — should say "Deployment successful" in under 60 seconds
3. Visit your domain — your site is live

**Auto-deploy on push:**
- In Coolify Application settings → enable **Auto Deploy**
- Now every time you push to GitHub, your site updates automatically
- Add a new tool HTML file → push → live in 60 seconds

---

### Step 6 — Add essential pages (required for AdSense)

Create these pages before applying for AdSense:

**about.html** — Who you are, what Toolify Core is, why it exists
**privacy.html** — Standard privacy policy (use freeprivacypolicy.com)
**contact.html** — Simple contact form or email address
**blog/** folder — Start with 5 SEO articles about your tools

---

## Part 2: n8n Integration — How It Connects

n8n is already running on your Coolify VPS. Here's exactly how to connect it to your tools website.

---

### What n8n does for a tools website (different from content site)

For a tools website, n8n does NOT generate tool content (tools are static HTML). 
Instead, n8n handles everything around the tools:

```
n8n jobs for Toolify Core:
├── Blog content generation (SEO articles about each tool)
├── Social media posting (announce new tools, tips)
├── Analytics monitoring (which tools get most traffic)
├── Uptime monitoring (alert if site goes down)
├── New tool deployment (auto-push to GitHub)
└── Affiliate link tracking report
```

---

### Workflow 1 — Auto-generate SEO blog posts for each tool

This is the most valuable workflow. Each tool page needs a supporting blog article
like "How to compress images for free online" to capture long-tail search traffic.

**n8n nodes in order:**

```
[Schedule trigger: Every Monday 9am]
        ↓
[Set node: Define this week's tool]
  topic: "How to compress PNG images"
  tool_url: "/tools/img-compress.html"
  target_keyword: "compress png image free"
        ↓
[HTTP Request: Call Claude/OpenAI API]
  POST https://api.anthropic.com/v1/messages
  Headers: x-api-key: YOUR_KEY
  Body: {
    model: "claude-sonnet-4-6",
    messages: [{
      role: "user",
      content: "Write a 700-word SEO blog post about: {{topic}}
                Target keyword: {{target_keyword}}
                Include: what it is, how to do it step by step,
                common mistakes, FAQ section.
                Link to: {{tool_url}}
                Tone: helpful, clear, no fluff.
                Output: HTML ready for WordPress."
    }]
  }
        ↓
[WordPress node: Create post as DRAFT]
  Title: "{{topic}}"
  Status: draft  ← IMPORTANT: never auto-publish
  Category: Blog
  Tags: auto-extracted by Claude
        ↓
[Telegram node: Notify you]
  "New draft ready: {{topic}} - review and publish"
```

**Result:** You get a Telegram message every Monday.
You open WordPress, review the draft (5 min), click Publish.
One SEO article per week = 52 articles per year, fully automated.

---

### Workflow 2 — Social media posting when new tool launches

**When to trigger:** Manually via n8n webhook, OR when you push to GitHub.

```
[Webhook trigger: POST /webhook/new-tool]
  Receives: { tool_name, tool_url, tool_desc }
        ↓
[HTTP Request: Generate social posts]
  Ask Claude to write 3 versions:
  - Twitter/X: 280 chars with hashtags
  - Facebook: 2 paragraph engaging post
  - LinkedIn: professional angle
        ↓
[Split in Batches]
        ↓
[HTTP Request: Post to Facebook Page API]
[HTTP Request: Post to Twitter API]
        ↓
[Wait 2 hours between posts]
        ↓
[Telegram: Confirm "Tool announced on all platforms"]
```

---

### Workflow 3 — Weekly analytics report

Connects Google Analytics 4 to your Telegram every Sunday.

```
[Schedule: Every Sunday 8am]
        ↓
[HTTP Request: GA4 API]
  GET top 10 pages by pageviews (last 7 days)
  GET total sessions
  GET traffic sources breakdown
        ↓
[HTTP Request: Google Search Console API]
  GET top 10 queries this week
  GET average position for main keywords
        ↓
[Code node: Format report]
  Build readable text summary
        ↓
[Telegram: Send weekly digest]
  "Week Report:
   Top tool: Image Compressor (2,341 views)
   Total visits: 8,432
   Top keyword: 'compress image free'
   Best position: #4 for 'age calculator Pakistan'"
```

---

### Workflow 4 — Uptime monitoring

Simple but critical. Know before your users do.

```
[Schedule: Every 5 minutes]
        ↓
[HTTP Request: GET toolifycore.com]
  Expected: 200 status
        ↓
[IF node: Status !== 200]
        ↓ Yes (site is down)
[Telegram: URGENT — Toolify Core is DOWN]
[Email: Send alert to your email]
        ↓ No (all good)
[Do nothing]
```

---

### Workflow 5 — Trending tool ideas (automated research)

Run weekly to find what tools people are searching for.

```
[Schedule: Every Friday]
        ↓
[HTTP Request: Google Trends API]
  Search: "free online tool" + your categories
        ↓
[HTTP Request: Answer Socrates API or SerpAPI]
  "People also ask" questions for your niche
        ↓
[Google Sheets: Append new tool ideas]
  Columns: Tool idea | Search volume | Difficulty | Status
        ↓
[Telegram: "5 new tool ideas added to your sheet"]
```

---

### How to connect n8n to your Coolify site

n8n runs on your same VPS. Access it at:
`http://your-vps-ip:5678` or `https://n8n.toolifycore.com` (set up subdomain)

**Step-by-step:**

1. Open n8n dashboard
2. Go to **Credentials** → Add new credential for each service:
   - Anthropic API key (from console.anthropic.com)
   - WordPress application password (WP Admin → Users → Application Passwords)
   - Telegram Bot token (create bot via @BotFather on Telegram)
   - Google Analytics service account JSON
3. Import the workflow JSONs (available from n8n template library)
4. Activate each workflow

**WordPress REST API setup:**
- In WordPress admin: Users → Your Profile → Application Passwords
- Add name: "n8n Integration" → Generate password
- n8n uses: `https://toolifycore.com/wp-json/wp/v2/posts`
- Auth: Basic auth with your WP username + app password

---

## Part 3: Value Addition — The Overlooked Extras

These are the things competitors skip that compound your earnings over time.

---

### 1. Tool result sharing (viral loop)

After every tool result, add a "Share this result" button.

Example: After age calculator shows result, display:
"Share your age: I am exactly 28 years, 3 months, 14 days old! 🎂 Calculate yours at toolifycore.com/tools/age-calc.html"

One-click share to WhatsApp, Twitter, Facebook.
This generates free organic backlinks and traffic.

---

### 2. Embed widget (for backlinks)

Add an embed option to every tool:
"Embed this calculator on your website"

```html
<iframe src="https://toolifycore.com/tools/age-calc.html" 
        width="100%" height="500" frameborder="0">
</iframe>
```

Pakistani bloggers and teachers will embed your tools.
Every embed = a backlink = better SEO = more traffic.

---

### 3. Bookmark prompt (return visitors)

After a user uses a tool, show:
"Bookmark this tool: Press Ctrl+D (or Cmd+D on Mac)"

Tools have 40-70% return visitor rates when bookmarked.
Return visitors = higher RPM because Google rewards engaged audiences.

---

### 4. Dark/light mode toggle

Save preference in localStorage. Users who find their preferred mode
stay longer and come back more. Simple but impactful for time-on-site metrics.

---

### 5. Tool usage counter (social proof)

Show on each tool card:
"Used 12,847 times this month"

Store counter in a simple PHP endpoint or use a free counter API.
Social proof increases click-through rate from the homepage by 15-30%.

---

### 6. Urdu language toggle

Add an Urdu translation button on each tool page.
Zero competition for Urdu-language tool searches in Pakistan.
A separate sitemap for Urdu pages doubles your indexed content
targeting a completely different keyword set.

---

### 7. AdSense placement strategy (avoid suspension)

Safe ad positions for tools sites:
- ONE banner above the tool (728x90 or responsive)
- ONE in-content ad below the results/output
- ONE sidebar ad (300x250) — desktop only
- NO ads inside the tool interface itself

Total: 3 ads maximum per page. This is the safe zone.
Exceeding this or placing ads where users click accidentally
is the most common reason for tools site AdSense suspension.

---

### 8. RSS feed for blog

Create /blog/feed.xml — Google Discover picks up fresh content from RSS.
n8n can ping Google's indexing API after each new blog post.

```
n8n: After WordPress publish
→ HTTP POST https://indexing.googleapis.com/v3/urlNotifications:publish
→ Body: { url: "https://toolifycore.com/blog/post-slug", type: "URL_UPDATED" }
```

Pages get indexed within hours instead of weeks.

---

## Quick launch checklist

- [ ] GitHub repo created with all HTML files
- [ ] Coolify application deployed and domain connected
- [ ] SSL certificate active (https working)
- [ ] About, Privacy, Contact pages published
- [ ] Google Search Console verified
- [ ] Google Analytics 4 tracking code added to all pages
- [ ] First 5 blog posts published (one per tool)
- [ ] n8n Workflows 1-4 activated
- [ ] Telegram bot connected for alerts
- [ ] AdSense application submitted (after 30 days + real traffic)
- [ ] First affiliate links added (Grammarly, remove.bg, NordVPN)

---

*Guide version 1.0 — Toolify Core Launch*
