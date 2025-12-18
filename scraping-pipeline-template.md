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

Build these 7 components in order:

```
1. SETUP → 2. DISCOVERY → 3. EVALUATION → 4. EXTRACTION → 5. ENRICHMENT → 6. VALIDATION → 7. MAINTENANCE
```

---

## Step 1: Create Project Structure

Create this directory structure:

```
{project}/
├── scripts/                    # One script per pipeline stage
├── data/
│   ├── reference.json          # Lookup data (cities, categories, etc.)
│   ├── blacklist.json          # Domains/entities to exclude
│   ├── candidates.json         # Discovered URLs with status
│   ├── processed.json          # Domains already handled (for deduplication)
│   └── records/
│       └── {CATEGORY}/         # Final records organized by category
│           └── {slug}.json
├── .env                        # API keys (add to .gitignore)
└── package.json
```

**Create package.json:**
- Set `"type": "module"` for ES modules
- Add dependencies: `playwright`, `dotenv`

**Create .env with required API keys:**
- `SEARCH_API_KEY` - Google Custom Search API
- `SEARCH_ENGINE_ID` - Custom Search Engine ID
- `ENRICHMENT_API_KEY` - Google Places API (or similar)

---

## Step 2: Build the Blacklist

Create `data/blacklist.json` to filter unwanted results.

**Structure:**
```json
{
  "domains": ["specific-domain-to-block.com"],
  "patterns": ["yelp.com", "facebook.com", "yellowpages.com"],
  "reasons": {
    "specific-domain.com": "Why this is excluded"
  }
}
```

**What to include:**
- National chains (if looking for local businesses)
- Aggregator/directory sites
- Lead generation sites
- Social media platforms
- Review sites (Yelp, Google reviews pages)

Ask the user what types of entities should be excluded for their use case.

---

## Step 3: Build the Search Script

Create a script that discovers candidate URLs via Google Custom Search API.

**Required functionality:**

1. **Accept CLI arguments:** Category (required), SubCategory (optional)
   ```
   node search.js TX Houston
   ```

2. **Load filters before searching:**
   - Read `blacklist.json`
   - Read `processed.json` (domains already handled)

3. **Build multiple search queries** to maximize coverage:
   - Include location terms
   - Include business-type keywords
   - Use variations: "local", "independent", "family owned"
   - Example: `"Houston TX" local dumpster rental family owned`

4. **Execute paginated searches:**
   - Use Google Custom Search API endpoint
   - Fetch 2-3 pages per query (10 results each)
   - Add 500ms delay between requests

5. **Filter and deduplicate results:**
   - Extract domain from each URL (remove `www.`)
   - Skip if domain is in blacklist
   - Skip if domain is in processed.json
   - Skip if domain already seen this session

6. **Save candidates to `data/candidates.json`:**
   ```json
   {
     "url": "https://example.com",
     "domain": "example.com",
     "title": "Search result title",
     "snippet": "Search snippet text",
     "searchedCategory": "TX",
     "searchedSubCategory": "Houston",
     "status": "pending",
     "addedAt": "2024-01-15T10:30:00Z"
   }
   ```

7. **Merge with existing candidates** (don't overwrite, append new)

---

## Step 4: Build the Web Scraper

Create a Playwright-based scraper to extract content from candidate websites.

**Required functionality:**

1. **Accept URL as CLI argument:**
   ```
   node fetch-page.js https://example.com
   ```

2. **Configure Playwright browser:**
   - Use chromium in headless mode
   - Set realistic user agent
   - Set 30 second timeout

3. **Fetch multiple pages per site:**
   - Main/home page
   - Contact page (find link matching `/contact|get.?in.?touch/i`)
   - About page (find link matching `/about|who.?we.?are|our.?story/i`)
   - Services/pricing page (find link matching `/services|pricing|products/i`)
   - Add 1 second delay between pages

4. **Clean extracted content:**
   - Remove script, style, noscript, iframe, svg tags
   - Extract text from main content area
   - Normalize whitespace
   - Limit to 50,000 characters per page

5. **Extract structured data using regex:**
   - Phone: `(\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4})`
   - Email: `[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}`
   - Address: Pattern matching street + city + state + zip

6. **Output results to console** for human review:
   - Show content from each page (truncated)
   - Show extracted phones, emails, addresses
   - Human decides if candidate qualifies

---

## Step 5: Define the Record Schema

Create `data/schema.json` defining the final record structure.

**Ask the user what fields they need.** Common fields include:

**Required fields:**
- `slug` - URL-friendly identifier (kebab-case)
- `name` - Entity name
- `primaryCity` - Main location
- `state` - State/region code
- `website` - Full URL
- `domain` - Domain only (for deduplication)

**Optional fields:**
- `phone`, `email`, `address`
- `description` - Summary text
- `yearFounded`
- `serviceAreas` - Array of locations served
- `tags` - Categorization array
- `externalRating`, `reviewCount` - From enrichment
- `needsReview`, `reviewIssues` - From auditing

**Slug format:** `{name-kebab}-{city}-{state}`
Example: `joes-dumpsters-houston-tx`

---

## Step 6: Build the Enrichment Script

Create a script that adds external data (ratings, reviews) to records.

**Required functionality:**

1. **Accept filtering options:**
   ```
   node enrich.js --category TX
   node enrich.js --file path/to/record.json
   node enrich.js --stale 30  # Only records older than 30 days
   ```

2. **Iterate through records to process**

3. **Query external API** (e.g., Google Places):
   - Build query: `"{name} {city} {state}"`
   - Find best match (exact name + city match preferred)

4. **Add enrichment fields:**
   - `externalRating` - 1-5 rating
   - `reviewCount` - Number of reviews
   - `externalId` - ID for future lookups
   - `externalUrl` - Link to external profile
   - `enrichedAt` - Date stamp

5. **Rate limit:** 1 second between API calls

6. **Save updated record back to file**

---

## Step 7: Build the Audit Script

Create a script that validates records and flags issues.

**Required functionality:**

1. **Define audit rules by severity:**

   **Critical (unusable record):**
   - Missing phone number
   - Missing website

   **Warning (should fix):**
   - Missing/incomplete address
   - Missing external rating
   - Missing email
   - Description too short (< 50 chars)
   - Too few service areas (< 3)

   **Info (nice to have):**
   - Missing year founded
   - Short description (< 200 chars)

2. **Support dry-run mode:**
   ```
   node audit.js --dry-run
   ```

3. **For each record:**
   - Run all audit rules
   - Set `needsReview: true` if any critical/warning issues
   - Set `reviewIssues` array with severity, field, message
   - Set `lastAuditDate`

4. **Generate audit report** (`data/audit-report.json`):
   - Summary: total, needs review count, issues by severity
   - Breakdown by category
   - List of records with issues

---

## Step 8: Build the Deduplication Tracker

Create a script that syncs processed domains.

**Required functionality:**

1. **Collect domains from:**
   - All records in `data/records/` (use `domain` field or extract from `website`)
   - Candidates in `candidates.json` with status "processed" or "skipped"

2. **Deduplicate and sort**

3. **Save to `data/processed.json`** (simple array of domains)

4. **Run this before search operations** to prevent re-discovering known domains

---

## Step 9: Build the Migration Script

Create a script for schema updates to existing records.

**Required functionality:**

1. **Define migrations as functions:**
   ```javascript
   const MIGRATIONS = [
     {
       name: 'rename-field',
       migrate: (record) => {
         if (record.oldField) {
           record.newField = record.oldField;
           delete record.oldField;
         }
         return record;
       }
     }
   ];
   ```

2. **Support dry-run mode**

3. **Apply all migrations to all records**

4. **Only save if record actually changed**

---

## Workflow Instructions

### Initial Setup
```bash
npm install
# Create .env with API keys
# Create blacklist.json with exclusions
node sync-processed.js  # Initialize tracker
```

### Discovery Cycle
```bash
node search.js CATEGORY [SUBCATEGORY]
# Review candidates.json
node fetch-page.js <URL>  # Evaluate each candidate
# If qualifies: create record file in data/records/CATEGORY/
# Update candidate status to "processed" or "skipped"
```

### Enrichment & Validation
```bash
node enrich.js --category CATEGORY
node audit.js --dry-run  # Preview issues
node audit.js            # Apply audit flags
# Review audit-report.json
# Fix critical/warning issues manually
```

### Maintenance
```bash
node sync-processed.js   # Update deduplication tracker
node migrate.js          # Apply schema changes
```

---

## Key Implementation Rules

1. **Always rate limit API calls** - 500ms-1000ms between requests
2. **Domain is the primary key** - One record per domain, deduplicate everywhere
3. **Support incremental processing** - Filter by category, staleness, or single file
4. **Always support dry-run** - Test before modifying files
5. **Log errors but continue** - Don't stop batch processing on single failures
6. **Scripts must be idempotent** - Safe to re-run without side effects
7. **Track timestamps** - When discovered, enriched, audited

---

## Customization Checklist

Before building, ask the user about:

1. **Target domain** - What type of entities are being collected?
2. **Categories** - How should records be organized? (by state, type, etc.)
3. **Required fields** - What data must each record have?
4. **Quality criteria** - What makes a record "complete"?
5. **Exclusion criteria** - What domains/types should be blacklisted?
6. **Enrichment sources** - What external APIs to use for additional data?
7. **Search keywords** - What terms find the target entities?

---

## Example: Local Business Directory

For a directory of local businesses (like dumpster rental companies):

- **Categories:** US state abbreviations (TX, CA, NY)
- **Blacklist:** National chains, franchise brands, aggregator sites
- **Search queries:** `"{city} {state}" local {business type} family owned`
- **Required fields:** name, city, state, phone, website
- **Enrichment:** Google Places API for ratings
- **Audit rules:** Must have phone, should have address and rating
