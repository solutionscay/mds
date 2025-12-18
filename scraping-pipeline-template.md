# Web Scraping Pipeline - Agent Instructions

Instructions for an LLM agent to build a modular web scraping pipeline. Follow these steps to create a complete data collection system.

---

## When to Use This Template

Use this pipeline pattern when the user needs to:
- Build a directory or database of businesses/entities from web sources
- Collect and structure data from multiple websites
- Create a repeatable process for discovering and validating web data

---

## Pipeline Architecture

Build these components in order. Note that **Enrichment is optional** - not all pipelines need external API data.

```
1. SETUP → 2. DISCOVERY → 3. EVALUATE & CREATE → 4. [ENRICHMENT] → 5. VALIDATION → 6. MAINTENANCE
                                                     (optional)
```

| Stage | Script | Purpose | Required? |
|-------|--------|---------|-----------|
| Setup | - | Create structure, config files | Yes |
| Discovery | `search.js` | Find candidate URLs via Google Search API | Yes |
| Evaluate & Create | `fetch-page.js` | Crawl site, review output, create record | Yes |
| Enrichment | `enrich.js` | Add external API data (Places, etc.) | **Optional** |
| Validation | `audit.js` | Flag incomplete records | Yes |
| Maintenance | `sync-processed.js` | Track domains, prevent duplicates | Yes |

**How "Evaluate & Create" works with an agent:**
1. Run `fetch-page.js` on a candidate URL
2. Agent reviews the console output (content, contact info, pages crawled)
3. Agent decides if it qualifies based on inclusion criteria
4. If yes: Agent creates the record JSON file using extracted data
5. Update candidate status in `candidates.json`

This is not a separate script - the agent handles evaluation and record creation based on fetch-page output.

---

## Before Building: Key Decisions

**Ask the user these questions before writing any code:**

### 1. What Are You Looking For?

Define **inclusion criteria** clearly:
- What qualifies as a valid listing?
- What specifically disqualifies a result?
- Are you looking for specific features, offers, or attributes?

**Example (homeschool deals):**
```
INCLUDE: Establishments offering incentives to homeschoolers
  - Homeschool discounts
  - Homeschool days/events
  - Homeschool-specific programs

EXCLUDE:
  - Co-ops (families teaching each other)
  - Generic programs without homeschool incentives
  - Aggregator/directory sites
```

### 2. Geographic Scope

- What states/regions do you cover?
- What metro areas within each state?
- What cities within each metro?

### 3. Categories (if applicable)

- How should records be organized?
- What search terms help find each category?

### 4. Do You Need Enrichment?

**This is the most important optional decision.** Ask the user:

| Need | Use Enrichment | Skip Enrichment |
|------|----------------|-----------------|
| Verified addresses & coordinates | Yes | No |
| Business hours | Yes | No |
| Google ratings/reviews | Yes | No |
| Online-only businesses | No | Yes |
| Address from website is sufficient | No | Yes |
| Cost-sensitive project | No | Yes |

**When to recommend enrichment:**
- Building a map-based directory
- Need verified business hours for display
- Want to show ratings/review counts
- Need lat/lng for distance calculations

**When to skip enrichment:**
- Website already provides good address data
- Listings are events, not physical locations
- Online-only or nationwide services
- Budget constraints (API costs)
- Pilot/MVP phase - can add later

---

## Step 1: Create Project Structure

```
{project}/
├── content/
│   ├── _data/
│   │   └── cityAreas.json     # Enabled metros tracker (site-level)
│   └── listings/              # Record files
├── src/
│   └── _data/
│       └── cityAreas.js       # (Optional) Eleventy data file
└── content-pipeline/
    ├── scripts/
    │   ├── search.js              # Google Search API discovery
    │   ├── fetch-page.js          # Playwright web scraper
    │   ├── audit.js               # Record validation
    │   ├── sync-processed.js      # Domain deduplication
    │   └── enrich.js              # (Optional) External API enrichment
    ├── data/
    │   ├── regions.json           # Geographic coverage + categories
    │   ├── blacklist.json         # Domains/patterns to exclude
    │   ├── candidates.json        # Discovered URLs with status
    │   ├── processed-domains.json # Domains already handled
    │   ├── audit-report.json      # Generated audit results
    │   └── drafts/                # Draft records for review
    ├── .env                       # API keys (add to .gitignore)
    ├── .env.example               # API key template
    ├── package.json
    ├── README.md
    └── MVP.md                     # Implementation spec
```

**package.json:**
```json
{
  "name": "content-pipeline",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "search": "node --env-file=.env scripts/search.js",
    "fetch": "node scripts/fetch-page.js",
    "audit": "node scripts/audit.js",
    "sync": "node scripts/sync-processed.js"
  },
  "dependencies": {
    "playwright": "^1.40.0"
  }
}
```

**.env.example:**
```bash
# Required
GOOGLE_SEARCH_API_KEY=your_api_key_here
GOOGLE_SEARCH_ENGINE_ID=your_engine_id_here

# Optional - only if using enrichment
GOOGLE_PLACES_API_KEY=your_places_key_here
```

**API credentials:**
- Search API Key: https://console.cloud.google.com/apis/credentials
- Search Engine: https://programmablesearchengine.google.com/
- Places API: https://console.cloud.google.com/apis/library/places-backend.googleapis.com

---

## Step 2: Build Configuration Files

### Metro/City-Area Tracking

**Purpose:** Track which metros are enabled for the live site, separate from the full search coverage.

The pipeline uses TWO related but distinct geographic data structures:

| File | Location | Purpose |
|------|----------|---------|
| `regions.json` | `content-pipeline/data/` | Full search coverage - all metros/cities to search |
| `cityAreas.json` | `content/_data/` | Site tracking - which metros are enabled for the site |

**Why separate files?**
- `regions.json` defines what you CAN search (full potential coverage)
- `cityAreas.json` defines what's LIVE on the site (incrementally expanded)
- Allows gradual rollout: search all of Texas, but only enable DFW initially

### `content/_data/cityAreas.json`

Tracks enabled metros with their metadata. **This is the source of truth for which metros appear on the site.**

```json
{
  "texas/dallas-fort-worth": {
    "slug": "dallas-fort-worth",
    "name": "Dallas-Fort Worth",
    "state": "texas"
  },
  "texas/houston": {
    "slug": "houston",
    "name": "Houston",
    "state": "texas"
  },
  "florida/orlando": {
    "slug": "orlando",
    "name": "Orlando",
    "state": "florida"
  }
}
```

**Key format:** `{state}/{metro-slug}` - enables easy lookup and prevents collisions

**Workflow for adding a new metro:**

1. **Add to `cityAreas.json`:**
   ```json
   "texas/el-paso": {
     "slug": "el-paso",
     "name": "El Paso",
     "state": "texas"
   }
   ```

2. **Ensure metro exists in `regions.json`** (for search coverage)

3. **Run search for the new metro:**
   ```bash
   node --env-file=.env scripts/search.js "El Paso" TX
   ```

4. **Process candidates and create listings**

5. **Site automatically generates pages** for the new metro (if using dynamic page generation)

### Optional: Eleventy Data File

If using Eleventy, create a JS version for template access:

**File:** `src/_data/cityAreas.js`

```javascript
/**
 * City Areas Data
 * Returns array of city areas for catalog page generation
 */
module.exports = [
  // Texas
  { slug: 'dallas-fort-worth', name: 'Dallas-Fort Worth', state: 'texas' },
  { slug: 'houston', name: 'Houston', state: 'texas' },
  // Florida
  { slug: 'orlando', name: 'Orlando', state: 'florida' }
];
```

This allows templates to iterate over metros for navigation, sitemap, etc.

---

### `data/regions.json`

Define geographic coverage and categories.

```json
{
  "regions": {
    "metro-slug": {
      "name": "Display Name",
      "cityArea": "metro-slug",
      "state": "state-slug",
      "cities": ["City1", "City2", "City3"]
    }
  },
  "categories": [
    {
      "name": "Category Name",
      "slug": "category-slug",
      "terms": ["search term 1", "search term 2", "search term 3"]
    }
  ],
  "searchKeywords": [
    "primary search keyword",
    "alternate keyword",
    "another variant"
  ]
}
```

### `data/blacklist.json`

Domains and URL patterns to exclude.

```json
{
  "domains": [
    "facebook.com",
    "instagram.com",
    "twitter.com",
    "yelp.com",
    "tripadvisor.com",
    "yellowpages.com",
    "aggregator-site.com"
  ],
  "urlPatterns": [
    "/review",
    "/listing/",
    "/directory/",
    "/business/"
  ],
  "reasons": {
    "aggregator-site.com": "Aggregator - not primary source"
  }
}
```

**Common blacklist domains by category:**

| Category | Domains |
|----------|---------|
| Social | facebook.com, instagram.com, twitter.com, linkedin.com, tiktok.com |
| Reviews | yelp.com, tripadvisor.com, bbb.org |
| Directories | yellowpages.com, manta.com, chamberofcommerce.com |
| Deals | groupon.com, retailmenot.com, slickdeals.net |
| Events | eventbrite.com, meetup.com |
| Jobs | indeed.com, glassdoor.com |

---

## Step 3: Build the Search Script

**File:** `scripts/search.js`

**Usage:**
```bash
node --env-file=.env scripts/search.js "City" ST
node --env-file=.env scripts/search.js "City" ST --category="Category"
node --env-file=.env scripts/search.js "City" ST --pages=3
```

**Key features:**

1. **Load filters before searching:**
   - Read `blacklist.json` for exclusions
   - Read `processed-domains.json` for already-handled domains
   - Read `regions.json` for category search terms

2. **Build focused search queries:**
   ```javascript
   // High-intent queries with primary keywords
   `"${keyword}" "${city}" ${state}`
   `intitle:${keyword} "${city}" ${state}`

   // Category-specific queries
   `"${keyword}" "${categoryTerm}" "${city}" ${state}`
   ```

3. **Execute paginated searches:**
   - Use Google Custom Search API
   - Fetch 2-3 pages per query (10 results each)
   - **Rate limit: 500ms+ between requests**

4. **Filter and deduplicate:**
   - Extract domain from URL (remove `www.`)
   - Skip blacklisted domains
   - Skip processed domains
   - Skip duplicates within session
   - Prefer specific URLs over root domains

5. **Check for existing records:**
   - Match by domain against existing record files
   - Mark as "update" if exists, "create" if new

6. **Save candidates:**
   ```json
   {
     "url": "https://example.com/specific-page",
     "domain": "example.com",
     "title": "Search result title",
     "snippet": "Search snippet text",
     "city": "Dallas",
     "state": "TX",
     "category": "Museums",
     "status": "pending",
     "verdict": "pending",
     "reason": null,
     "existingListing": null,
     "action": "create",
     "addedAt": "2025-01-01T00:00:00.000Z"
   }
   ```

**Status flow:**
```
pending → processed (qualifies/not_qualified)
              ↓
      skipped (with reason)
```

---

## Step 4: Build the Web Scraper

**File:** `scripts/fetch-page.js`

**Usage:**
```bash
node scripts/fetch-page.js https://example.com/target-page
```

**Key features:**

1. **Configure Playwright:**
   - Headless chromium
   - Realistic user agent
   - 30 second timeout

2. **Multi-page crawl strategy:**

   Start at the discovered URL (your target page), then crawl:
   - Homepage (company info)
   - About page (background, year founded)
   - Contact page (phone, email, address)
   - Programs/pricing page (offerings, costs)

3. **Page pattern matching:**
   ```javascript
   const PAGE_PATTERNS = {
     home: [/^\/?$/, /^\/?(index|home)/i],
     contact: [/contact/i, /reach-us/i, /get-in-touch/i],
     about: [/about/i, /who-we-are/i, /our-story/i],
     programs: [/program/i, /pricing/i, /services/i, /schedule/i]
   };
   ```

4. **Content extraction:**
   - Remove script, style, noscript, iframe, svg tags
   - Extract main content area text
   - Normalize whitespace
   - Limit to 50,000 chars per page

5. **Contact info extraction (regex):**
   ```javascript
   // Phone
   /\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}/g

   // Email
   /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g

   // Address (customize states)
   /\d+\s+[\w\s]+(?:Street|St|Avenue|Ave|Road|Rd|...)[\w\s,#.-]*(?:TX|FL|CA|...)[\s,]*\d{5}/gi
   ```

6. **Output for human review:**
   - Content from each page (truncated)
   - Extracted phones, emails, addresses
   - Pages crawled list
   - Keyword validation (did we find expected terms?)

---

## Step 5: Define Record Schema

Define what fields a record should have. **Customize for your use case.**

**Minimum required fields:**
```json
{
  "slug": "company-name",
  "company": "Company Name",
  "website": "https://example.com/specific-page",
  "createdAt": "2025-01-01",
  "updatedAt": "2025-01-01"
}
```

**Location fields (for local businesses):**
```json
{
  "nationwide": false,
  "state": "texas",
  "cityArea": "dallas-fort-worth",
  "city": "dallas",
  "location": "Dallas, TX",
  "address": "123 Main St, Dallas, TX 75001"
}
```

**Contact fields:**
```json
{
  "phone": "(555) 123-4567",
  "email": "contact@example.com"
}
```

**Content fields:**
```json
{
  "title": "Listing Title",
  "shortSummary": "Brief description (10-200 chars)",
  "description": "Full description (100+ chars)",
  "tags": [],
  "activities": []
}
```

**Enrichment fields (optional - only if using Places API):**
```json
{
  "externalRating": 4.5,
  "reviewCount": 127,
  "placeId": "ChIJ...",
  "coordinates": { "lat": 32.7767, "lng": -96.7970 },
  "enrichedAt": "2025-01-01"
}
```

**Audit fields (auto-added):**
```json
{
  "needsReview": false,
  "reviewIssues": [],
  "lastAuditDate": "2025-01-01"
}
```

---

## Step 6: Build the Audit Script

**File:** `scripts/audit.js`

**Usage:**
```bash
node scripts/audit.js              # Apply audit flags
node scripts/audit.js --dry-run    # Report only
```

**Key features:**

1. **Define severity levels:**

   | Severity | Meaning | Action |
   |----------|---------|--------|
   | critical | Unusable record | Must fix |
   | warning | Should fix | Review soon |
   | info | Nice to have | Low priority |

2. **Define audit rules:**

   **Critical:**
   - Missing website
   - Missing company/name
   - Missing slug

   **Warning:**
   - Missing phone
   - Missing address
   - Description too short (< 100 chars)
   - Missing category/type
   - Using root domain URL (should use specific page)

   **Info:**
   - Missing email
   - Missing scheduling info
   - Missing cost info

3. **Update records with audit flags:**
   ```json
   {
     "needsReview": true,
     "reviewIssues": [
       { "severity": "warning", "field": "phone", "message": "Missing phone number" }
     ],
     "lastAuditDate": "2025-01-01"
   }
   ```

4. **Generate audit report:**
   - Summary: total, clean, needs review
   - Issues by severity
   - Breakdown by state/category
   - List of records with issues

---

## Step 7: Build the Deduplication Tracker

**File:** `scripts/sync-processed.js`

**Usage:**
```bash
node scripts/sync-processed.js
```

**Key features:**

1. **Collect domains from:**
   - All records (extract from `website` field)
   - Candidates with status "processed" or "skipped"

2. **Normalize domains:**
   - Remove `www.` prefix
   - Lowercase

3. **Save to `data/processed-domains.json`:**
   ```json
   ["domain1.com", "domain2.com", "domain3.com"]
   ```

4. **Run before search operations** to prevent re-discovering known domains

---

## Step 8 (Optional): Build Enrichment Script

**Only implement this if the user needs verified location data, ratings, or business hours.**

**File:** `scripts/enrich.js`

**Usage:**
```bash
node --env-file=.env scripts/enrich.js --category texas
node --env-file=.env scripts/enrich.js --file path/to/record.json
node --env-file=.env scripts/enrich.js --stale 30  # Re-enrich after 30 days
```

**Key features:**

1. **Query Google Places API:**
   - Build query: `"${company}" ${city} ${state}`
   - Find best match by name similarity

2. **Add enrichment fields:**
   ```json
   {
     "placeId": "ChIJ...",
     "externalRating": 4.5,
     "reviewCount": 127,
     "coordinates": { "lat": 32.7767, "lng": -96.7970 },
     "formattedAddress": "Verified address from Google",
     "businessHours": { ... },
     "enrichedAt": "2025-01-01"
   }
   ```

3. **Rate limit:** 1 second between API calls

4. **Cost awareness:**
   - Places API charges per request
   - Consider caching results
   - Only enrich when necessary

---

## Workflow Guide

### Initial Setup

```bash
cd content-pipeline
npm install
cp .env.example .env
# Edit .env with API keys

node scripts/sync-processed.js  # Initialize domain tracker
```

**Create initial `content/_data/cityAreas.json`:**
```json
{
  "texas/dallas-fort-worth": {
    "slug": "dallas-fort-worth",
    "name": "Dallas-Fort Worth",
    "state": "texas"
  }
}
```

Start with one or two metros, expand as you add listings.

### Daily Discovery Workflow

```bash
# 1. Search for candidates in a city
node --env-file=.env scripts/search.js "Dallas" TX

# 2. For each pending candidate, fetch and evaluate
node scripts/fetch-page.js https://museum.com/homeschool-day
```

**After fetch-page runs, the agent:**
- Reviews the output (content from each page, extracted contact info)
- Decides if the site qualifies based on inclusion criteria
- If yes: Creates the record JSON file in the appropriate location
- Updates the candidate status in `candidates.json` to "processed" with verdict

### Maintenance Workflow

```bash
# After adding records, update domain tracker
node scripts/sync-processed.js

# Audit all records for completeness
node scripts/audit.js --dry-run  # Preview
node scripts/audit.js            # Apply flags

# Review audit-report.json, fix issues
```

### Optional: Enrichment Workflow

```bash
# Enrich records in a category
node --env-file=.env scripts/enrich.js --category texas

# Re-enrich stale records (older than 30 days)
node --env-file=.env scripts/enrich.js --stale 30
```

### Expanding to New Metros

When ready to add a new metro to the site:

1. **Add to `content/_data/cityAreas.json`:**
   ```json
   "texas/el-paso": {
     "slug": "el-paso",
     "name": "El Paso",
     "state": "texas"
   }
   ```

2. **Verify metro exists in `content-pipeline/data/regions.json`**
   - If not, add the region with cities list

3. **Search the new metro:**
   ```bash
   node --env-file=.env scripts/search.js "El Paso" TX
   ```

4. **Process candidates** - run fetch-page, evaluate, create listings

5. **Sync processed domains:**
   ```bash
   node scripts/sync-processed.js
   ```

6. **Run audit on new listings:**
   ```bash
   node scripts/audit.js --dry-run
   ```

**Tip:** Add metros incrementally. It's better to have 10 solid listings in one metro than 2 listings each across 5 metros.

---

## Implementation Rules

1. **Always rate limit** - 500ms-1000ms between API requests
2. **Domain is the unique key** - One record per domain, deduplicate everywhere
3. **Support incremental processing** - Filter by category, staleness, or single file
4. **Always support dry-run** - Test before modifying files
5. **Log errors but continue** - Don't stop batch on single failures
6. **Scripts must be idempotent** - Safe to re-run without side effects
7. **Track timestamps** - When discovered, enriched, audited, updated
8. **ES modules** - Use `"type": "module"` in package.json
9. **No external dependencies** - Use built-in `fetch`, `fs`, `path`

---

## Common Issues & Solutions

### Search Returns Too Many Irrelevant Results
- Add more domains to blacklist
- Use more specific search keywords
- Use `intitle:` operator

### Missing Contact Info from Scraper
- Site may not have it publicly listed
- Try additional page patterns
- Check if info is in images (won't extract)
- Consider Places API enrichment

### Duplicate Records
- Run sync-processed.js after adding records
- Check candidates.json before creating new records
- Domain is the unique identifier

### Rate Limiting / Quota Errors
- Google Custom Search: 100 queries/day (free tier)
- Add longer delays between requests
- Consider upgrading for higher volume

---

## Customization Checklist

Before building, confirm with the user:

- [ ] Inclusion/exclusion criteria defined
- [ ] Geographic scope (states, metros, cities)
- [ ] Categories and search terms
- [ ] Record schema fields needed
- [ ] Enrichment needed? (Places API)
- [ ] Quality rules for audit
- [ ] Blacklist domains identified
- [ ] Initial metros for `cityAreas.json` (can expand later)

---

## Example Implementations

### Homeschool Deals Directory
- **Target:** Establishments offering homeschool incentives
- **Regions:** Texas metros, Florida metros
- **Categories:** Museums, Zoos, Farms, Sports, Arts
- **Enrichment:** No (address from website sufficient)
- **Keywords:** "homeschool day", "homeschool program", "homeschool discount"
- **Metro tracking:** `cityAreas.json` with 14 metros (10 TX, 4 FL)
- **Live example:** [homeschooldeals.org](https://homeschooldeals.org)

### Local Service Provider Directory
- **Target:** Local independent businesses
- **Regions:** Single state, multiple metros
- **Categories:** By service type
- **Enrichment:** Yes (need verified addresses, ratings)
- **Keywords:** Service-specific terms + "local" + "family owned"

### Event Venue Finder
- **Target:** Venues hosting specific event types
- **Regions:** National or regional
- **Categories:** By venue type
- **Enrichment:** Yes (need coordinates, capacity)
- **Keywords:** Event-specific terms + venue types
