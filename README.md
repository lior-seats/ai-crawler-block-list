# AI Crawler Block List & User Agent Reference

A community-maintained reference of AI crawler user agents, their traffic characteristics, and ready-to-use configs to monitor or rate-limit them on your infrastructure.

**Problem:** AI crawlers (MetaExternalAgent, GPTBot, ClaudeBot, Applebot-Extended, PerplexityBot) don't coordinate with each other and don't respect informal crawl-rate conventions. Many operators have seen 3–10× spikes in hosting costs from AI crawler bursts with zero warning.

---

## 🔍 Free Tool: Check If Your Site Blocks AI Crawlers

**→ [lior-seats.github.io/crawl-check/](https://lior-seats.github.io/crawl-check/)**

Paste your `robots.txt` (or enter your URL) and instantly see which AI crawlers are blocked vs. allowed through. Free, no signup.

---

## 📊 Real-World Data: How Developer Platforms Handle AI Crawlers

**→ [I checked 20 popular developer platforms' robots.txt — most aren't blocking AI crawlers correctly](https://lior-seats.github.io/blog/ai-crawlers-robots-txt.html)**

Key findings:
- 6/20 platforms have **zero** AI bot rules (Heroku, Stripe, Linear, Notion, Turso, Upstash)
- 7/20 use only `Content-Signal` (advisory intent, not actual blocking)
- Only 2/20 have comprehensive coverage across all major crawlers
- meta-externalagent — the bot most cited for Vercel bill spikes — is missing from most block lists

---

## Known AI Crawler User Agents (2026)

| Crawler | Company | User Agent String | robots.txt directive |
|---------|---------|-------------------|---------------------|
| GPTBot | OpenAI | `GPTBot/1.1` | `User-agent: GPTBot` |
| ChatGPT-User | OpenAI | `ChatGPT-User` | `User-agent: ChatGPT-User` |
| ClaudeBot | Anthropic | `ClaudeBot/1.0` | `User-agent: ClaudeBot` |
| Claude-Web | Anthropic | `Claude-Web` | `User-agent: Claude-Web` |
| Meta-ExternalAgent | Meta | `Meta-ExternalAgent/1.1` | `User-agent: Meta-ExternalAgent` |
| Googlebot-Extended | Google | `Googlebot-Extended` | `User-agent: Googlebot-Extended` |
| PerplexityBot | Perplexity | `PerplexityBot/1.0` | `User-agent: PerplexityBot` |
| Applebot-Extended | Apple | `Applebot-Extended` | `User-agent: Applebot-Extended` |
| Bytespider | ByteDance | `Bytespider` | `User-agent: Bytespider` |
| CCBot | Common Crawl | `CCBot/2.0` | `User-agent: CCBot` |
| cohere-ai | Cohere | `cohere-ai/1.0` | `User-agent: cohere-ai` |
| Amazonbot | Amazon | `Amazonbot` | `User-agent: Amazonbot` |
| DiffBot | Diffbot | `DiffBot` | `User-agent: DiffBot` |
| omgili / omgilibot | Webz.io | `omgili` | `User-agent: omgilibot` |

---

## robots.txt Snippets

### Block all known AI training crawlers
```
User-agent: GPTBot
Disallow: /

User-agent: ChatGPT-User
Disallow: /

User-agent: ClaudeBot
Disallow: /

User-agent: Claude-Web
Disallow: /

User-agent: Meta-ExternalAgent
Disallow: /

User-agent: Googlebot-Extended
Disallow: /

User-agent: PerplexityBot
Disallow: /

User-agent: Applebot-Extended
Disallow: /

User-agent: CCBot
Disallow: /

User-agent: cohere-ai
Disallow: /

User-agent: Amazonbot
Disallow: /

User-agent: Bytespider
Disallow: /
```

### Allow only standard search crawlers (strict allowlist)
```
User-agent: Googlebot
Allow: /

User-agent: Bingbot
Allow: /

User-agent: *
Disallow: /
```

---

## Nginx Rate Limiting

```nginx
# Rate-limit known AI crawlers to 1 req/s; block bursts
geo $is_ai_crawler {
    default         0;
    ~*GPTBot        1;
    ~*ClaudeBot     1;
    ~*MetaExternal  1;
    ~*PerplexityBot 1;
    ~*Bytespider    1;
    ~*CCBot         1;
    ~*cohere-ai     1;
}

limit_req_zone $is_ai_crawler zone=ai_crawlers:10m rate=1r/s;

server {
    location / {
        limit_req zone=ai_crawlers burst=5 nodelay;
        # ... rest of your config
    }
}
```

---

## Cloudflare Workers: Per-Crawler Cost Attribution

```javascript
// workers/crawler-monitor.js
const AI_CRAWLERS = [
  'GPTBot', 'ChatGPT-User', 'ClaudeBot', 'Claude-Web',
  'Meta-ExternalAgent', 'PerplexityBot', 'Applebot-Extended',
  'CCBot', 'cohere-ai', 'Bytespider', 'Amazonbot', 'DiffBot'
];

export default {
  async fetch(request, env) {
    const ua = request.headers.get('user-agent') || '';
    const crawler = AI_CRAWLERS.find(c => ua.includes(c));

    if (crawler) {
      // Log to your analytics — attribute this request to the crawler
      await env.CRAWLER_KV.put(
        `hit:${crawler}:${Date.now()}`,
        JSON.stringify({ url: request.url, ts: Date.now() }),
        { expirationTtl: 86400 * 30 }
      );
    }

    return fetch(request);
  }
};
```

---

## Vercel: Block in `vercel.json`

```json
{
  "headers": [
    {
      "source": "/(.*)",
      "has": [
        { "type": "header", "key": "user-agent", "value": "(?:GPTBot|ClaudeBot|Meta-ExternalAgent|PerplexityBot|Bytespider).*" }
      ],
      "headers": [{ "key": "x-robots-tag", "value": "noindex, nofollow" }]
    }
  ],
  "redirects": [
    {
      "source": "/(.*)",
      "has": [
        { "type": "header", "key": "user-agent", "value": "GPTBot.*" }
      ],
      "destination": "/blocked",
      "statusCode": 403
    }
  ]
}
```

---

## Express / Node.js Middleware

```javascript
const AI_CRAWLERS = /GPTBot|ClaudeBot|Meta-ExternalAgent|PerplexityBot|Bytespider|CCBot|cohere-ai/i;

function blockAICrawlers(req, res, next) {
  const ua = req.get('user-agent') || '';
  if (AI_CRAWLERS.test(ua)) {
    return res.status(403).json({ error: 'AI crawler access not permitted' });
  }
  next();
}

app.use(blockAICrawlers);
```

---

## Apache .htaccess

```apache
# Block AI training crawlers at the Apache level
<IfModule mod_rewrite.c>
  RewriteEngine On

  # Block by user agent
  RewriteCond %{HTTP_USER_AGENT} (GPTBot|ChatGPT-User|ClaudeBot|Meta-ExternalAgent|PerplexityBot|Bytespider|CCBot|cohere-ai|Amazonbot|DiffBot) [NC]
  RewriteRule .* - [F,L]
</IfModule>

# Alternative: rate-limit via mod_qos or return 429
# For rate limiting, use mod_ratelimit or a reverse proxy (nginx/Cloudflare)
```

```apache
# If you want to allow but throttle (requires mod_ratelimit):
<IfModule mod_ratelimit.c>
  <If "%{HTTP_USER_AGENT} =~ /GPTBot|ClaudeBot|Meta-ExternalAgent/">
    SetOutputFilter RATE_LIMIT
    SetEnv rate-limit 50
  </If>
</IfModule>
```

---

## AWS CloudFront + WAF

### Option 1: CloudFront Function (free tier, low latency)

```javascript
// CloudFront Function — attach to viewer-request event
function handler(event) {
  var request = event.request;
  var ua = (request.headers['user-agent'] || { value: '' }).value;

  var blocked = [
    'GPTBot', 'ChatGPT-User', 'ClaudeBot', 'Meta-ExternalAgent',
    'PerplexityBot', 'Bytespider', 'CCBot', 'cohere-ai', 'Amazonbot'
  ];

  for (var i = 0; i < blocked.length; i++) {
    if (ua.indexOf(blocked[i]) !== -1) {
      return {
        statusCode: 403,
        statusDescription: 'Forbidden',
        body: 'AI crawler access not permitted'
      };
    }
  }

  return request;
}
```

### Option 2: AWS WAF Rule (Console / Terraform)

```hcl
# Terraform: AWS WAF rule to block AI crawlers
resource "aws_wafv2_rule_group" "block_ai_crawlers" {
  name     = "BlockAICrawlers"
  scope    = "CLOUDFRONT"
  capacity = 50

  rule {
    name     = "BlockGPTBot"
    priority = 1
    action { block {} }
    statement {
      byte_match_statement {
        field_to_match { single_header { name = "user-agent" } }
        positional_constraint = "CONTAINS"
        search_string         = "GPTBot"
        text_transformation { priority = 0; type = "LOWERCASE" }
      }
    }
    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "BlockGPTBot"
      sampled_requests_enabled   = true
    }
  }
  # Add similar rules for ClaudeBot, Meta-ExternalAgent, etc.

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "BlockAICrawlers"
    sampled_requests_enabled   = true
  }
}
```

**Note:** WAF adds ~$5/month base cost. CloudFront Functions are effectively free for this use case.

---

## Django / Python

```python
# middleware.py
AI_CRAWLERS = [
    'GPTBot', 'ChatGPT-User', 'ClaudeBot', 'Claude-Web',
    'Meta-ExternalAgent', 'PerplexityBot', 'Bytespider',
    'CCBot', 'cohere-ai', 'Amazonbot', 'DiffBot'
]

class BlockAICrawlersMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        ua = request.META.get('HTTP_USER_AGENT', '')
        if any(crawler in ua for crawler in AI_CRAWLERS):
            from django.http import HttpResponseForbidden
            return HttpResponseForbidden('AI crawler access not permitted')
        return self.get_response(request)

# settings.py
MIDDLEWARE = [
    'myapp.middleware.BlockAICrawlersMiddleware',
    # ... other middleware
]
```

---

## Reported Cost Incidents

Publicly reported cases of AI crawlers causing unexpected infrastructure bills. Add yours via PR.

| Date | Crawler | Platform | Reported Impact | Source |
|------|---------|----------|-----------------|--------|
| 2025-02 | Meta-ExternalAgent | Vercel | "Zuck's bot increased our Vercel bill 10x" | HN thread (7 pts) |
| 2025-03 | GPTBot | AWS CloudFront | 40% bill spike, traced to docs site | Reddit r/webdev |
| 2025-Q3 | Bytespider | Cloudflare Pages | Database query amplification, 3× usual bill | Private report |
| 2026-Q1 | Multiple | GitHub Pages LFS | LFS bandwidth quota hit in 2 days | GitHub Community |
| 2026-Q2 | Meta-ExternalAgent | Netlify | Exceeded free-tier bandwidth, surprise $180 bill | Reddit r/selfhosted |

**Pattern:** Most incidents involve crawlers that:
- Ignore `Crawl-delay` in robots.txt
- Hit database-backed pages that CDNs don't cache
- Run parallel workers with no per-domain rate limits
- Spike at off-hours when nobody monitors dashboards

---

## Want Monitoring + Alerts?

The configs above block crawlers but won't tell you which ones are hitting hardest, what it's costing you, or alert you before the invoice arrives.

[CrawlGuard](https://lior-seats.github.io/crawlguard-waitlist/) monitors AI crawler traffic, attributes infrastructure costs per bot, and fires threshold alerts before you see the bill. Early access waitlist — $29/month.

---

## Contributing

PRs welcome. Please include:
- Source URL confirming the user agent string
- Notes on crawl behavior (frequency, respect for crawl-delay, etc.)
- Date first observed

## License

CC0 — public domain. Use freely.
