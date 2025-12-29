# Web Scraping Pipeline - Agent Instructions

Instructions for an LLM agent to build a modular web scraping pipeline for creating local business directories.

---

## When to Use This Template

Use this pipeline pattern when the user needs to:
- Build a directory or database of businesses/entities from web sources
- Collect and structure data from multiple websites
- Create a repeatable process for discovering and validating web data

---

## Pipeline Architecture

Build these components in order. Note that **LLM Generation** and **Enrichment** are optional depending on project needs.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              PIPELINE FLOW                                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  1. SYNC DOMAINS → 2. SEARCH → 3. CRAWL & EVALUATE → 4. CREATE LISTING          │
│                                                                                  │
│         ↓                                                                        │
│                                                                                  │
│  5. [LLM GENERATION] → 6. [ENRICHMENT] → 7. AUDIT → 8. SYNC DOMAINS             │
│       (optional)          (optional)                                             │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

| Step | Script | Purpose | Required? |
|------|--------|---------|-----------|
| 1 | `sync-processed.js` | Initialize/update domain deduplication list | Yes |
| 2 | `search.js` | Find candidate URLs via Google Search API | Yes |
| 3 | `fetch-page.js` | Crawl site, extract content & contacts, classify | Yes |
| 4 | `create-listing.js` | Create listing JSON from evaluated candidates | Yes |
| 5 | `generate.js` | Generate descriptions via LLM (Gemini) | Optional |
| 6 | `enrich.js` | Add Google Places data (ratings, photos, hours) | Optional |
| 7 | `audit.js` | Flag incomplete records, generate report | Yes |
| 8 | `sync-processed.js` | Update domain tracker after adding listings | Yes |

**Utility Scripts:**
| Script | Purpose |
|--------|---------|
| `scrape-contacts.js` | Batch extract missing contacts from audit report |
| `sync-crawled.js` | Crash recovery - sync crawled data back to candidates |

---

## Data Source Matrix

This matrix shows **where each piece of information should be gathered from** and the priority order when multiple sources are available.

### Identity & Location Data

| Field | Web Scraping | LLM Generation | Google Places | Priority Order | Notes |
|-------|:------------:|:--------------:|:-------------:|----------------|-------|
| **Business Name** | ✓ | - | ✓ (authoritative) | GBP → Search Title → Crawl → Domain | GBP is most reliable; search title before `\|` is second best |
| **Website URL** | ✓ | - | ✓ | Scrape (primary) | Original discovered URL |
| **Domain** | ✓ | - | - | Scrape | Normalized from URL |
| **Street Address** | ✓ | - | ✓ (verified) | GBP → Scrape | GBP provides verified, formatted address |
| **City** | ✓ | - | ✓ | GBP → Scrape → Search context | Often from search query context |
| **State** | ✓ | - | ✓ | GBP → Scrape → Search context | Often from search query context |
| **ZIP/Postal Code** | ✓ | - | ✓ | GBP → Scrape | GBP parses from address components |
| **Coordinates (lat/lng)** | - | - | ✓ | GBP only | Required for maps; only from GBP |

### Contact Information

| Field | Web Scraping | LLM Generation | Google Places | Priority Order | Notes |
|-------|:------------:|:--------------:|:-------------:|----------------|-------|
| **Phone** | ✓ | - | ✓ | Scrape → GBP | tel: links most reliable; GBP fills gaps |
| **Email** | ✓ | - | - | Scrape only | NOT available from Google Places |
| **Social Media URLs** | ✓ | - | - | Scrape only | Instagram, Facebook, TikTok, etc. |

### Content & Descriptions

| Field | Web Scraping | LLM Generation | Google Places | Priority Order | Notes |
|-------|:------------:|:--------------:|:-------------:|----------------|-------|
| **Short Summary** | ✓ (meta desc) | ✓ | ✓ (editorial) | LLM → GBP → Meta desc | LLM writes best; GBP has editorial summaries |
| **Long Description** | - | ✓ | - | LLM only | Generated from crawled content |
| **Service Types/Categories** | ✓ (inferred) | ✓ | ✓ (place types) | LLM → GBP types → Scrape | LLM classifies from content |
| **Tags/Keywords** | - | ✓ | - | LLM only | Generated from content analysis |
| **Pricing Info** | ✓ | ✓ (extract) | ✓ (price level) | Scrape → LLM extract | GBP only has $-$$$$ level |
| **Service Area** | ✓ | ✓ (extract) | - | Scrape → LLM | Often on "Areas We Serve" pages |

### Business Metadata

| Field | Web Scraping | LLM Generation | Google Places | Priority Order | Notes |
|-------|:------------:|:--------------:|:-------------:|----------------|-------|
| **Business Hours** | ✓ | - | ✓ (structured) | GBP → Scrape | GBP has structured weekly hours |
| **Year Founded** | ✓ | ✓ (extract) | - | Scrape → LLM | Often on About page |
| **Owner Name** | ✓ | ✓ (extract) | - | Scrape → LLM | Often on About page |
| **Is Chain/Franchise** | ✓ (keywords) | ✓ | - | LLM → Scrape | LLM analyzes content for chain signals |
| **Is Local/Independent** | ✓ (keywords) | ✓ | - | LLM → Scrape | LLM analyzes for local signals |

### Ratings & Reviews

| Field | Web Scraping | LLM Generation | Google Places | Priority Order | Notes |
|-------|:------------:|:--------------:|:-------------:|----------------|-------|
| **Google Rating** | - | - | ✓ | GBP only | 1-5 star rating |
| **Review Count** | - | - | ✓ | GBP only | Number of Google reviews |
| **Place ID** | - | - | ✓ | GBP only | For future API lookups |
| **Google Maps URL** | - | - | ✓ | GBP only | Direct link to listing |

### Images

| Field | Web Scraping | LLM Generation | Google Places | Priority Order | Notes |
|-------|:------------:|:--------------:|:-------------:|----------------|-------|
| **Photos** | ✓ (possible) | - | ✓ | GBP → Scrape | GBP photos are higher quality, attributed |
| **Logo** | ✓ | - | - | Scrape only | From website |

### Accessibility & Payments

| Field | Web Scraping | LLM Generation | Google Places | Priority Order | Notes |
|-------|:------------:|:--------------:|:-------------:|----------------|-------|
| **Accessibility Features** | - | - | ✓ | GBP only | Wheelchair access, etc. |
| **Payment Methods** | ✓ | - | ✓ | GBP → Scrape | Credit cards, contactless, etc. |

---

### Data Source Summary

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         DATA SOURCE CAPABILITIES                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  WEB SCRAPING (fetch-page.js)                                                   │
│  ├─ Best for: Email, social links, website-specific info                        │
│  ├─ Good for: Phone, address (unverified), business name                        │
│  └─ Methods: tel:/mailto: links, schema.org, microdata, regex                   │
│                                                                                  │
│  LLM GENERATION (generate.js)                                                   │
│  ├─ Best for: Descriptions, summaries, tags, classification                     │
│  ├─ Good for: Extracting structured data from unstructured content              │
│  └─ Input: Crawled page content from data/crawled/{domain}.json                 │
│                                                                                  │
│  GOOGLE PLACES (enrich.js)                                                      │
│  ├─ Best for: Verified address, coordinates, ratings, hours, photos             │
│  ├─ Good for: Business name (authoritative), phone                              │
│  ├─ NOT available: Email, social media, website-specific content                │
│  └─ Cost: Per API call - use selectively                                        │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Recommended Data Collection Strategy

1. **Always scrape first** - Get everything possible from the website
2. **Use LLM for content generation** - Descriptions, classification, extraction
3. **Use GBP to verify and fill gaps** - Authoritative name, verified address, ratings
4. **GBP overwrites scraped data for**: Business name, address, phone (if missing)
5. **Scraping is only source for**: Email, social media links

---

## Before Building: Key Decisions

**Ask the user these questions before writing any code:**

### 1. What Are You Looking For?

Define **inclusion criteria** clearly:
- What qualifies as a valid listing?
- What specifically disqualifies a result?
- Are you looking for specific features, offers, or attributes?

**Example (service provider directory):**
```
INCLUDE: Local service providers
  - Family-owned, locally-owned, or small regional companies
  - Provides services for residential AND commercial customers
  - Has a real business address in the service area

EXCLUDE:
  - National chains (large corporate brands)
  - National franchises
  - Aggregators/lead generation sites
  - Government info pages
  - Wrong business type
```

**Example (class/activity directory):**
```
INCLUDE: Local studios offering classes
  - Niche-related keywords present
  - Local or independent business
  - Offers classes, lessons, or workshops

EXCLUDE:
  - National chains/franchises
  - Social media platforms
  - Aggregator/directory sites
  - Unrelated businesses
```

### 2. Geographic Scope

- What states/regions do you cover?
- Search by STATE or METRO AREA (not individual cities) to avoid duplicates
- Each company is listed ONCE with their service area defined

**Search Strategy by State Size:**
| State Size | Strategy |
|------------|----------|
| Small (RI, DE, CT) | One state-wide search |
| Medium (AL, SC) | State-wide + 1-2 major metros |
| Large (TX, CA, FL) | Search each major metro separately |

### 3. Do You Need LLM Generation?

**When to use LLM generation:**
- Need rich descriptions written from website content
- Want automated classification (tags, categories, types)
- Website content is unstructured and needs summarization

**When to skip:**
- Simple listings with minimal description needs
- Information is already structured on websites
- Cost-sensitive project (API costs)

### 4. Do You Need Enrichment?

**When to use Google Places enrichment:**
- Building a map-based directory (need coordinates)
- Need verified addresses, phone numbers
- Want Google ratings/review counts
- Need business hours for display
- Want photos from Google

**When to skip:**
- Website already provides good data
- Listings are events, not physical locations
- Online-only or nationwide services
- Budget constraints (API costs)

---

## Step 1: Create Project Structure

```
{project}/
├── content/
│   ├── listings/              # Final listing JSON files
│   │   └── {slug}.json
│   └── images/                # Downloaded photos (optional)
│       └── {slug}/
│           └── photo-1.jpg
├── content-pipeline/          # OR scripts/ at root level
│   ├── scripts/
│   │   ├── search.js              # Google Search API discovery
│   │   ├── fetch-page.js          # Playwright web scraper + classifier
│   │   ├── create-listing.js      # Convert candidates to listings
│   │   ├── generate.js            # (Optional) LLM description generation
│   │   ├── enrich.js              # (Optional) Google Places enrichment
│   │   ├── audit.js               # Record validation
│   │   ├── sync-processed.js      # Domain deduplication
│   │   ├── scrape-contacts.js     # (Utility) Batch contact extraction
│   │   └── contact-extractor.js   # Shared contact extraction module
│   ├── data/
│   │   ├── regions.json           # Geographic coverage + search terms
│   │   ├── blacklist.json         # Domains/patterns to exclude
│   │   ├── candidates.json        # Discovered URLs with status
│   │   ├── processed-domains.json # Domains already handled
│   │   ├── audit-report.json      # Generated audit results
│   │   └── crawled/               # Cached crawl data for LLM
│   │       └── {domain}.json
│   ├── .env                       # API keys (add to .gitignore)
│   ├── .env.example               # API key template
│   └── package.json
└── data/                      # Alternative: listings at root
    └── listings/
        └── {state}/           # Organized by state
            └── {slug}.json
```

**package.json:**
```json
{
  "name": "content-pipeline",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "sync": "node scripts/sync-processed.js",
    "search": "node --env-file=.env scripts/search.js",
    "fetch": "node scripts/fetch-page.js",
    "create": "node scripts/create-listing.js",
    "generate": "node --env-file=.env scripts/generate.js",
    "enrich": "node --env-file=.env scripts/enrich.js",
    "audit": "node scripts/audit.js",
    "contacts": "node scripts/scrape-contacts.js",
    "pipeline": "npm run fetch && npm run create && npm run generate && npm run enrich && npm run audit && npm run sync"
  },
  "dependencies": {
    "playwright": "^1.40.0"
  }
}
```

**.env.example:**
```bash
# Required for search
GOOGLE_CSE_API_KEY=your_api_key_here
GOOGLE_CSE_ID=your_engine_id_here

# Optional - only if using LLM generation
GOOGLE_GEMINI_API_KEY=your_gemini_key_here

# Optional - only if using enrichment
GOOGLE_PLACES_API_KEY=your_places_key_here
```

**API credentials:**
- Search API Key: https://console.cloud.google.com/apis/credentials
- Search Engine: https://programmablesearchengine.google.com/
- Places API: https://console.cloud.google.com/apis/library/places-backend.googleapis.com
- Gemini API: https://aistudio.google.com/apikey

---

## Step 2: Build Configuration Files

### `data/regions.json`

Define geographic coverage and search terms.

```json
{
  "states": {
    "texas": {
      "name": "Texas",
      "abbr": "TX"
    },
    "florida": {
      "name": "Florida",
      "abbr": "FL"
    }
  },
  "metros": {
    "dallas-fort-worth": {
      "name": "Dallas-Fort Worth",
      "state": "texas",
      "cities": ["Dallas", "Fort Worth", "Arlington", "Plano", "Irving", "Frisco"]
    },
    "houston": {
      "name": "Houston",
      "state": "texas",
      "cities": ["Houston", "Katy", "Sugar Land", "Pearland", "The Woodlands"]
    }
  },
  "searchTerms": [
    "local {niche}",
    "family owned {niche}",
    "{niche} near me",
    "best {niche}"
  ],
  "nicheKeywords": [
    "primary keyword",
    "alternate keyword",
    "another variant"
  ]
}
```

### `data/blacklist.json`

Domains and URL patterns to exclude.

```json
{
  "nationalChains": [
    "bigcompany.com",
    "nationalfranchise.com"
  ],
  "aggregators": [
    "yelp.com",
    "yellowpages.com",
    "thumbtack.com",
    "angi.com",
    "homeadvisor.com"
  ],
  "socialPlatforms": [
    "facebook.com",
    "instagram.com",
    "twitter.com",
    "linkedin.com",
    "tiktok.com",
    "youtube.com",
    "pinterest.com"
  ],
  "directories": [
    "tripadvisor.com",
    "bbb.org",
    "manta.com",
    "chamberofcommerce.com"
  ],
  "urlPatterns": [
    "/review",
    "/listing/",
    "/directory/",
    "/business/"
  ],
  "reasons": {
    "yelp.com": "Aggregator - not primary source",
    "bigcompany.com": "National chain"
  }
}
```

---

## Step 3: Build the Search Script

**File:** `scripts/search.js`

**Usage:**
```bash
# Search by state (small states)
node --env-file=.env scripts/search.js --state TX

# Search by metro area (recommended for large states)
node --env-file=.env scripts/search.js --state TX --metro houston
node --env-file=.env scripts/search.js --state TX --metro dallas-fort-worth

# Search specific city
node --env-file=.env scripts/search.js --city "Austin" --state TX

# Dry run (preview without saving)
node --env-file=.env scripts/search.js --state TX --dry-run

# Limit pages (default: 3 pages = 30 results per query)
node --env-file=.env scripts/search.js --state TX --pages 2
```

**Key Features:**

1. **Load filters before searching:**
   - Read `blacklist.json` for exclusions
   - Read `processed-domains.json` for already-handled domains
   - Read existing `candidates.json` for session deduplication

2. **Build focused search queries:**
   ```javascript
   // High-intent queries with primary keywords
   `"${keyword}" "${city}" ${state}`
   `"local ${keyword}" "${metro}" ${state}`
   `"family owned ${keyword}" near "${city}" ${state}`

   // Use intitle: for precision
   `intitle:${keyword} "${city}" ${state}`
   ```

3. **Execute paginated searches:**
   - Use Google Custom Search API
   - Fetch 2-3 pages per query (10 results each)
   - **Rate limit: 500-600ms between requests**

4. **Filter and deduplicate:**
   - Extract domain from URL (normalize: remove `www.`, lowercase)
   - Skip blacklisted domains
   - Skip processed domains
   - Skip duplicates within session

5. **Save candidates:**
   ```json
   {
     "url": "https://example.com/specific-page",
     "domain": "example.com",
     "title": "Search result title",
     "snippet": "Search snippet text",
     "city": "Dallas",
     "state": "TX",
     "metro": "dallas-fort-worth",
     "query": "local service provider Dallas TX",
     "status": "pending",
     "discoveredAt": "2025-01-01T00:00:00.000Z"
   }
   ```

**Candidate Status Flow:**
```
pending → evaluated → listed
              ↓
          rejected (with reason)
              ↓
          error (crawl failed)
```

---

## Step 4: Build the Web Scraper & Classifier

**File:** `scripts/fetch-page.js`

**Usage:**
```bash
# Crawl single URL (for manual review)
node scripts/fetch-page.js https://example.com/target-page

# Process pending candidates (batch mode)
node scripts/fetch-page.js --status pending --limit 10

# Process specific metro
node scripts/fetch-page.js --metro dallas-fort-worth --limit 20

# Dry run
node scripts/fetch-page.js --status pending --dry-run
```

**Key Features:**

1. **Configure Playwright:**
   ```javascript
   const browser = await chromium.launch({ headless: true });
   const context = await browser.newContext({
     userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
   });
   const page = await context.newPage();
   page.setDefaultTimeout(30000);
   ```

2. **Multi-page crawl strategy:**
   - Start at discovered URL
   - Find and crawl related pages:
   ```javascript
   const PAGE_PATTERNS = {
     contact: [/contact/i, /reach-us/i, /get-in-touch/i],
     about: [/about/i, /who-we-are/i, /our-story/i],
     services: [/service/i, /program/i, /class/i, /pricing/i, /schedule/i]
   };
   ```

3. **Content extraction:**
   - Remove script, style, noscript, iframe, svg tags
   - Extract clean text content
   - Limit to 50,000 chars per page
   - Wait 1000-1500ms for dynamic content

4. **Contact extraction (multi-method):**

   **Phone (4 methods):**
   ```javascript
   // 1. Tel: links (most reliable)
   document.querySelectorAll('a[href^="tel:"]')

   // 2. Data attributes
   document.querySelectorAll('[data-phone]')

   // 3. Schema.org/JSON-LD
   JSON.parse(script.textContent).telephone

   // 4. Regex patterns
   /\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}/g
   ```

   **Email (4 methods):**
   ```javascript
   // 1. Mailto: links
   document.querySelectorAll('a[href^="mailto:"]')

   // 2. Schema.org
   JSON.parse(script.textContent).email

   // 3. Standard regex
   /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g

   // 4. Obfuscated (decode)
   "info [at] company [dot] com" → "info@company.com"
   ```

   **Address (3 methods):**
   ```javascript
   // 1. Schema.org/JSON-LD
   { "@type": "PostalAddress", "streetAddress": "...", "addressLocality": "..." }

   // 2. Microdata
   document.querySelectorAll('[itemtype*="PostalAddress"]')

   // 3. Regex
   /\d+\s+[\w\s]+(?:Street|St|Avenue|Ave|Road|Rd|...)[\w\s,#.-]*(?:TX|FL|CA|...)[\s,]*\d{5}/gi
   ```

5. **Classification (scoring system):**
   ```javascript
   // Niche relevance scoring
   const NICHE_KEYWORDS = ['keyword1', 'keyword2', 'keyword3'];
   let nicheScore = 0;
   for (const keyword of NICHE_KEYWORDS) {
     if (content.toLowerCase().includes(keyword)) nicheScore++;
   }

   // Chain detection
   const CHAIN_KEYWORDS = ['franchise', 'locations nationwide', '100+ locations'];
   let chainScore = 0;
   for (const keyword of CHAIN_KEYWORDS) {
     if (content.toLowerCase().includes(keyword)) chainScore++;
   }

   // Classification result
   const classification = {
     isRelevant: nicheScore >= 2,
     relevanceScore: nicheScore,
     isPotentialChain: chainScore >= 2,
     confidence: nicheScore >= 3 ? 'high' : nicheScore >= 2 ? 'medium' : 'low'
   };
   ```

6. **Save crawled content for LLM:**
   ```javascript
   // Save to data/crawled/{domain}.json
   {
     "domain": "example.com",
     "url": "https://example.com",
     "crawledAt": "2025-01-01T00:00:00.000Z",
     "pages": [
       {
         "url": "https://example.com/",
         "title": "Page Title",
         "bodyText": "Extracted content...",
         "h1": "Main Heading"
       },
       {
         "url": "https://example.com/about",
         "title": "About Us",
         "bodyText": "About content..."
       }
     ],
     "contacts": {
       "phones": ["(555) 123-4567"],
       "emails": ["info@example.com"],
       "address": "123 Main St, Dallas, TX 75001"
     }
   }
   ```

7. **Update candidate status:**
   ```json
   {
     "status": "evaluated",
     "crawledAt": "2025-01-01T00:00:00.000Z",
     "pagesCrawled": 4,
     "classification": {
       "isRelevant": true,
       "relevanceScore": 5,
       "isPotentialChain": false,
       "confidence": "high"
     },
     "contacts": {
       "phone": "(555) 123-4567",
       "email": "info@example.com",
       "address": "123 Main St, Dallas, TX 75001"
     },
     "businessName": "Example Company"
   }
   ```

---

## Step 5: Build the Listing Creator

**File:** `scripts/create-listing.js`

**Usage:**
```bash
# Create listings from evaluated candidates
node scripts/create-listing.js

# Limit number created
node scripts/create-listing.js --limit 10

# Force recreate existing
node scripts/create-listing.js --force

# Dry run
node scripts/create-listing.js --dry-run
```

**Key Features:**

### Business Name Extraction Strategy

Getting accurate business names is critical. Use a multi-source approach:

**Priority Order:**
1. **Google Search Title** (highest priority)
   - Extract part before first `|` or `-` separator
   - Most reliable because Google normalizes titles
   ```javascript
   const title = candidate.title; // "Joe's Services | Houston TX"
   const name = title.split('|')[0].trim(); // "Joe's Services"
   ```

2. **Crawled businessName** (fallback)
   - Extracted from HTML during crawl
   - Sources: schema.org, og:site_name, logo alt, copyright, title tag, h1

3. **Domain Name** (last resort)
   - Convert domain to readable name
   ```javascript
   "joesservices.com" → "Joes Services"
   "mainstreetcompany.com" → "Main Street Company"
   ```

**Validation - Reject Bad Names:**
```javascript
const BAD_PATTERNS = [
  /^\(\d{3}\)/, // Phone numbers
  /call today/i, /book now/i, // CTAs
  /^RSS:/i, // RSS artifacts
  /^(service|class|program)/i, // Generic descriptions
  /^(about|contact|home)/i, // Page titles
  /\.(com|net|org)$/i // Domains
];
```

### Listing Schema

```json
{
  "slug": "company-name-city-st",
  "name": "Company Name",
  "domain": "example.com",
  "website": "https://example.com/",

  "state": "texas",
  "stateAbbr": "TX",
  "cityArea": "dallas-fort-worth",
  "cityAreaName": "Dallas-Fort Worth",
  "city": "Dallas",
  "address": "123 Main St, Dallas, TX 75001",
  "latitude": null,
  "longitude": null,

  "phone": "(555) 123-4567",
  "email": "info@example.com",

  "title": "Company Name - Dallas Services",
  "shortSummary": "Brief description (10-200 chars)",
  "description": null,

  "tags": [],
  "serviceTypes": [],
  "targetAudience": [],

  "instagramUrl": null,
  "facebookUrl": null,
  "tiktokUrl": null,
  "youtubeUrl": null,

  "gbpPlaceId": null,
  "gbpUrl": null,
  "gbpRating": null,
  "gbpReviewCount": null,
  "gbpVerified": false,

  "images": [],
  "primaryImageUrl": null,

  "isChain": false,
  "needsReview": true,
  "reviewIssues": ["Missing description", "Missing email"],

  "createdAt": "2025-01-01",
  "updatedAt": "2025-01-01",
  "enrichedAt": null,
  "lastAuditAt": null
}
```

### Alternative: Service Area Focused Schema

```json
{
  "slug": "company-name-city-st",
  "domain": "www.example.com",
  "companyName": "Company Name",
  "phone": "(555) 123-4567",
  "address": "123 Main St",
  "primaryCity": "Houston",
  "state": "TX",
  "email": "info@example.com",
  "website": "https://example.com/",

  "shortSummary": "One sentence summary",
  "longSummary": "Three paragraphs about the company",
  "pricingInfo": "Pricing details",
  "serviceOptions": "Option A, Option B, Option C",
  "yearFounded": "2010",

  "serviceAreaDescription": "Greater Houston area, 50-mile radius",
  "serviceAreaCities": ["Houston", "Katy", "Sugar Land"],
  "serviceAreaZips": ["77001", "77002"],

  "googleRating": 4.8,
  "reviewCount": 127,
  "googlePlaceId": "ChIJ...",
  "googleMapsUrl": "https://maps.google.com/?cid=...",
  "ratingsLastUpdated": "2025-01-01",

  "needsReview": false,
  "reviewIssues": [],
  "lastAuditDate": "2025-01-01"
}
```

---

## Step 6 (Optional): Build LLM Generation Script

**File:** `scripts/generate.js`

**Usage:**
```bash
# Generate descriptions for listings without them
node --env-file=.env scripts/generate.js

# Process all (even those with descriptions)
node --env-file=.env scripts/generate.js --all

# Limit number processed
node --env-file=.env scripts/generate.js --limit 10

# Process specific listing
node --env-file=.env scripts/generate.js --slug company-name-city

# Dry run
node --env-file=.env scripts/generate.js --dry-run
```

**Key Features:**

1. **Read crawled content:**
   ```javascript
   const crawledPath = `data/crawled/${listing.domain}.json`;
   const crawled = JSON.parse(fs.readFileSync(crawledPath));

   // Combine all page content (limit each to 3000 chars)
   const pageContent = crawled.pages
     .map(p => `### ${p.title}\n${p.bodyText.slice(0, 3000)}`)
     .join('\n\n');
   ```

2. **Build prompt for LLM:**
   ```javascript
   const prompt = `
   You are analyzing a business website to create a directory listing.

   Business: ${listing.name}
   Location: ${listing.city}, ${listing.stateAbbr}
   Website: ${listing.website}

   Website Content:
   ${pageContent}

   Based ONLY on the website content above, provide:

   1. shortSummary: 1-2 sentence description (max 160 chars)
   2. description: 2-3 paragraph detailed description
   3. serviceTypes: Array of service types offered
   4. targetAudience: Array of target audiences
   5. tags: Array of relevant tags
   6. isChain: Boolean - is this a franchise or chain?

   Return as JSON only. Only include facts from the website.
   `;
   ```

3. **Call Gemini API:**
   ```javascript
   const response = await fetch(
     `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key=${API_KEY}`,
     {
       method: 'POST',
       headers: { 'Content-Type': 'application/json' },
       body: JSON.stringify({
         contents: [{ parts: [{ text: prompt }] }],
         generationConfig: { responseMimeType: 'application/json' }
       })
     }
   );
   ```

4. **Validate and update listing:**
   ```javascript
   const generated = JSON.parse(response.text);

   // Update listing
   listing.shortSummary = generated.shortSummary || listing.shortSummary;
   listing.description = generated.description;
   listing.serviceTypes = generated.serviceTypes || [];
   listing.targetAudience = generated.targetAudience || [];
   listing.tags = generated.tags || [];
   listing.isChain = generated.isChain || false;
   listing.generatedAt = new Date().toISOString();
   listing.updatedAt = new Date().toISOString();

   // Remove "Missing description" from issues
   listing.reviewIssues = listing.reviewIssues
     .filter(i => !i.includes('description'));
   ```

5. **Rate limit:** 1.5 seconds between requests

---

## Step 7 (Optional): Build Enrichment Script

**File:** `scripts/enrich.js`

**Usage:**
```bash
# Enrich listings needing enrichment
node --env-file=.env scripts/enrich.js --needs-enrichment

# Enrich specific state
node --env-file=.env scripts/enrich.js --state TX

# Enrich single listing
node --env-file=.env scripts/enrich.js --slug company-name-city

# Re-enrich stale records (older than 30 days)
node --env-file=.env scripts/enrich.js --stale 30

# Skip photos (ratings only)
node --env-file=.env scripts/enrich.js --skip-photos

# Photos only
node --env-file=.env scripts/enrich.js --photos-only
```

**Key Features:**

1. **Search Google Places API:**
   ```javascript
   const searchQuery = `${listing.name} ${listing.city} ${listing.stateAbbr}`;

   const response = await fetch(
     'https://places.googleapis.com/v1/places:searchText',
     {
       method: 'POST',
       headers: {
         'Content-Type': 'application/json',
         'X-Goog-Api-Key': API_KEY,
         'X-Goog-FieldMask': 'places.id,places.displayName,places.formattedAddress,places.rating,places.userRatingCount,places.websiteUri,places.nationalPhoneNumber,places.location,places.photos,places.regularOpeningHours,places.priceLevel,places.accessibilityOptions,places.paymentOptions'
       },
       body: JSON.stringify({ textQuery: searchQuery })
     }
   );
   ```

2. **Match verification (avoid false positives):**
   ```javascript
   // Priority 1: Website domain matching (most reliable)
   const matchByDomain = results.find(place => {
     if (!place.websiteUri) return false;
     const placeDomain = new URL(place.websiteUri).hostname.replace('www.', '');
     return placeDomain === listing.domain;
   });

   // Priority 2: Name + city matching
   const matchByName = results.find(place => {
     const nameMatch = similarity(place.displayName.text, listing.name) > 0.5;
     const cityMatch = place.formattedAddress.includes(listing.city);
     return nameMatch && cityMatch;
   });

   const match = matchByDomain || matchByName;
   ```

3. **Update listing with enriched data:**
   ```javascript
   if (match) {
     // GBP Data
     listing.gbpPlaceId = match.id;
     listing.gbpUrl = `https://www.google.com/maps/place/?q=place_id:${match.id}`;
     listing.gbpRating = match.rating;
     listing.gbpReviewCount = match.userRatingCount;
     listing.gbpVerified = true;

     // Location (fill if missing)
     if (!listing.address) listing.address = match.formattedAddress;
     if (!listing.latitude) listing.latitude = match.location.latitude;
     if (!listing.longitude) listing.longitude = match.location.longitude;

     // Contact (fill if missing)
     if (!listing.phone) listing.phone = match.nationalPhoneNumber;

     // Business name (authoritative source)
     listing.name = match.displayName.text;

     // Hours
     if (match.regularOpeningHours) {
       listing.businessHours = parseHours(match.regularOpeningHours);
     }

     // Photos (download up to 5)
     if (match.photos && match.photos.length > 0) {
       listing.images = await downloadPhotos(match.photos.slice(0, 5), listing.slug);
       listing.primaryImageUrl = listing.images[0]?.url;
     }

     listing.enrichedAt = new Date().toISOString();
   } else {
     listing.gbpVerified = false;
     listing.reviewIssues.push('Not found on Google Places');
   }
   ```

4. **Rate limit:** 1 second between API calls

---

## Step 8: Build the Audit Script

**File:** `scripts/audit.js`

**Usage:**
```bash
# Audit all listings and update them
node scripts/audit.js

# Preview only (dry run)
node scripts/audit.js --dry-run

# Generate detailed report
node scripts/audit.js --report
```

**Key Features:**

1. **Define severity levels:**

   | Severity | Meaning | Action |
   |----------|---------|--------|
   | critical | Unusable record | Must fix before publish |
   | warning | Should fix | Review soon |
   | info | Nice to have | Low priority |

2. **Define audit rules:**

   **Critical (blocks publication):**
   - Missing name/company name
   - Missing website URL
   - Missing city
   - Missing state

   **Warning (needs review):**
   - Missing phone number
   - Missing email address
   - Missing street address
   - Missing description
   - No GBP match found
   - Flagged as potential chain
   - Few service area cities (<3)
   - Missing pricing info

   **Info (low priority):**
   - No images available
   - No social media links
   - No service types specified
   - No business hours
   - Low review count (<5)
   - Missing year founded
   - Short summary too brief

3. **Update listings with audit flags:**
   ```json
   {
     "needsReview": true,
     "reviewIssues": [
       "Missing phone number",
       "Missing email address",
       "No GBP match found"
     ],
     "lastAuditAt": "2025-01-01T00:00:00.000Z"
   }
   ```

4. **Generate audit report:**
   ```json
   {
     "summary": {
       "total": 150,
       "clean": 120,
       "needsReview": 30,
       "critical": 5,
       "warning": 20,
       "info": 25
     },
     "byRule": {
       "Missing phone number": 15,
       "Missing email address": 12,
       "No GBP match found": 8
     },
     "byState": {
       "TX": { "total": 80, "needsReview": 15 },
       "FL": { "total": 70, "needsReview": 15 }
     },
     "listings": [
       {
         "slug": "company-name-city",
         "issues": ["Missing phone number"],
         "severity": "warning"
       }
     ],
     "generatedAt": "2025-01-01T00:00:00.000Z"
   }
   ```

---

## Step 9: Build the Domain Sync Script

**File:** `scripts/sync-processed.js`

**Usage:**
```bash
# Sync processed domains
node scripts/sync-processed.js

# Dry run
node scripts/sync-processed.js --dry-run
```

**Key Features:**

1. **Collect domains from:**
   - All listing files (extract from `domain` or `website` field)
   - Candidates with status: `listed`, `rejected`, `error`

2. **Normalize domains:**
   - Remove `www.` prefix
   - Lowercase

3. **Save to `data/processed-domains.json`:**
   ```json
   {
     "domains": ["domain1.com", "domain2.com", "domain3.com"],
     "count": 150,
     "lastSynced": "2025-01-01T00:00:00.000Z"
   }
   ```

4. **Run before search operations** to prevent re-discovering known domains

---

## Utility: Contact Scraper (Batch Mode)

**File:** `scripts/scrape-contacts.js`

**Usage:**
```bash
# Single URL - extract and display
node scripts/scrape-contacts.js https://example.com

# Batch mode - process listings from audit report
node scripts/scrape-contacts.js --from-audit

# Batch mode with auto-update
node scripts/scrape-contacts.js --from-audit --apply
```

**Key Features:**

1. **Reads `audit-report.json`** to find listings with missing contacts
2. **Visits each website** and extracts using robust multi-method approach
3. **Only updates fields that are currently missing**
4. **Rate limited:** 2 seconds between listings
5. **Re-run audit after** to verify fixes

---

## Utility: Crash Recovery

**File:** `scripts/sync-crawled.js`

If the pipeline crashes during crawling, crawled data is saved to `data/crawled/` but `candidates.json` may not be updated.

**Usage:**
```bash
node scripts/sync-crawled.js
```

**Features:**
- Reads all files in `data/crawled/`
- Updates matching candidates with crawled data
- Sets status to "evaluated" or "rejected" based on classification
- Useful for recovering from mid-crawl failures

---

## Workflow Guide

### Initial Setup

```bash
cd content-pipeline
npm install
cp .env.example .env
# Edit .env with API keys

# Initialize domain tracker
npm run sync
```

### Complete Pipeline Execution

```bash
# 1. Sync domains (ensure deduplication is current)
npm run sync

# 2. Search for candidates
npm run search -- --state TX --metro houston

# 3. Crawl and evaluate candidates
npm run fetch -- --status pending --limit 20

# 4. Create listings from evaluated candidates
npm run create

# 5. (Optional) Generate descriptions with LLM
npm run generate

# 6. (Optional) Enrich with Google Places
npm run enrich -- --needs-enrichment

# 7. Audit all listings
npm run audit -- --report

# 8. (Optional) Fix missing contacts
npm run contacts -- --from-audit --apply

# 9. Re-audit after fixes
npm run audit

# 10. Sync processed domains
npm run sync
```

### Alternative: One-Command Pipeline

After initial search, run the full processing pipeline:
```bash
npm run pipeline
```

### Expanding to New Metros

1. **Add to `regions.json`** if not present
2. **Search the new metro:**
   ```bash
   npm run search -- --state TX --metro austin
   ```
3. **Process candidates** - run fetch, create, generate, enrich, audit
4. **Sync processed domains:**
   ```bash
   npm run sync
   ```

---

## Data Flow Diagram

```
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│   regions     │     │   blacklist   │     │     .env      │
│    .json      │     │    .json      │     │   API keys    │
└───────┬───────┘     └───────┬───────┘     └───────┬───────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │   search.js     │ ──────────► Google CSE API
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ candidates.json │  status: "pending"
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ fetch-page.js   │ ──────────► Playwright
                    └────────┬────────┘
                             │
                    ┌────────┴────────┐
                    ▼                 ▼
          ┌─────────────────┐  ┌──────────────────┐
          │ candidates.json │  │ data/crawled/    │
          │ status:evaluated│  │ {domain}.json    │
          └────────┬────────┘  └────────┬─────────┘
                   │                    │
                   ▼                    │
          ┌───────────────────┐         │
          │ create-listing.js │         │
          └─────────┬─────────┘         │
                    │                   │
                    ▼                   │
          ┌───────────────────┐         │
          │ listings/{slug}   │         │
          │     .json         │◄────────┘
          └─────────┬─────────┘    (used by generate.js)
                    │
                    ▼
          ┌─────────────────┐
          │  generate.js    │ ──────────► Google Gemini API
          └────────┬────────┘             (reads crawled content)
                   │
                   ▼
          ┌─────────────────┐
          │   enrich.js     │ ──────────► Google Places API
          └────────┬────────┘
                   │
                   ▼
          ┌─────────────────┐
          │    audit.js     │ ──────────► audit-report.json
          └────────┬────────┘
                   │
                   ▼
          ┌─────────────────┐
          │sync-processed.js│ ──────────► processed-domains.json
          └─────────────────┘
```

---

## Implementation Rules

1. **Always rate limit** - 500ms-1500ms between API requests
2. **Domain is the unique key** - One listing per domain, deduplicate everywhere
3. **Search by region** - Use state/metro, not individual cities
4. **Support incremental processing** - Filter by status, limit, metro, or single item
5. **Always support dry-run** - Test before modifying files
6. **Log errors but continue** - Don't stop batch on single failures
7. **Scripts must be idempotent** - Safe to re-run without side effects
8. **Track timestamps** - `discoveredAt`, `crawledAt`, `createdAt`, `enrichedAt`, `lastAuditAt`
9. **ES modules** - Use `"type": "module"` in package.json
10. **Minimal dependencies** - Use built-in `fetch`, `fs`, `path`; only Playwright for crawling
11. **Save crawled content** - Store full page content for LLM use
12. **Multi-method extraction** - Use 3-4 methods for contacts (schema.org, links, regex)

---

## Common Issues & Solutions

### Search Returns Too Many Irrelevant Results
- Add more domains to blacklist
- Use more specific search keywords
- Use `intitle:` operator
- Search by metro, not state

### Missing Contact Info from Scraper
- Site may not have it publicly listed
- Try additional page patterns
- Check if info is in images (won't extract)
- Use Places API enrichment
- Use `scrape-contacts.js --from-audit` for batch re-extraction

### Duplicate Records
- Run `sync-processed.js` after adding listings
- Check `candidates.json` before creating
- Domain is the unique identifier

### Rate Limiting / Quota Errors
- Google Custom Search: 100 queries/day (free tier)
- Add longer delays between requests
- Consider upgrading for higher volume

### Wrong Business Names
- Check Google Search title first (before `|`)
- Validate against bad name patterns
- Google Places enrichment provides authoritative names

### Pipeline Crashes Mid-Crawl
- Run `sync-crawled.js` to recover crawled data
- Restart with `--status pending` to continue

---

## Customization Checklist

Before building, confirm with the user:

- [ ] Inclusion/exclusion criteria defined
- [ ] Geographic scope (states, metros)
- [ ] Niche keywords and search terms
- [ ] Listing schema fields needed
- [ ] LLM generation needed?
- [ ] Enrichment needed? (Places API)
- [ ] Quality rules for audit
- [ ] Blacklist domains identified
- [ ] Classification scoring keywords

---

## Example Directory Types

### Service Provider Directory
- **Target:** Local service companies
- **Regions:** All US states, major metros
- **Search strategy:** State/metro level to avoid duplicates
- **Schema focus:** Service area cities/zips, pricing, service options
- **LLM generation:** Optional (manual summaries work too)
- **Enrichment:** Yes (Google ratings, reviews)
- **Keywords:** "local [service]", "family owned [service]"

### Class/Activity Directory
- **Target:** Local studios offering classes
- **Regions:** Major metros in target states
- **Search strategy:** Metro-level searches
- **Schema focus:** Class types, age groups, social media
- **LLM generation:** Yes (descriptions, classification)
- **Enrichment:** Yes (ratings, photos, hours)
- **Keywords:** "[activity] classes", "[activity] lessons", "[activity] studio"
- **Classification:** Niche score, chain detection scoring

### Venue/Attraction Directory
- **Target:** Establishments offering specific experiences
- **Regions:** Multiple states, metros
- **Categories:** By venue type
- **LLM generation:** Optional
- **Enrichment:** Optional (address from website often sufficient)
- **Keywords:** "[experience] event", "[experience] program", "[experience] discount"
