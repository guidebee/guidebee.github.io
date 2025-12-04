# Design Document

## Overview

This design document outlines the technical approach for customizing the Academic Pages Jekyll blog template for guidebee's personal development blog. The customization transforms an academic-focused template into a professional development blog centered on documenting the Solana trading system project, while maintaining the template's core functionality and structure.

The design focuses on configuration-based changes rather than structural modifications, ensuring maintainability and compatibility with future template updates. The primary changes involve updating YAML configuration files, Markdown content files, and selectively disabling academic-focused features.

## Architecture

### Jekyll Site Structure

The Academic Pages template follows a standard Jekyll architecture:

```
guidebee.github.io/
├── _config.yml              # Main site configuration
├── _data/
│   └── navigation.yml       # Navigation menu structure
├── _pages/
│   ├── about.md            # Homepage content
│   ├── cv.md               # Resume/CV page
│   └── ...                 # Other static pages
├── _posts/                 # Blog posts collection
├── _includes/
│   └── author-profile.html # Sidebar author profile template
├── _layouts/               # Page layout templates
├── _publications/          # Academic publications (to be disabled)
├── _talks/                 # Academic talks (to be disabled)
├── _teaching/              # Teaching materials (to be disabled)
└── _portfolio/             # Portfolio items (to be kept)
```

### Configuration Hierarchy

Jekyll uses a hierarchical configuration system:

1. **Site-wide configuration** (`_config.yml`) - Global settings, author information, collections
2. **Data files** (`_data/navigation.yml`) - Structured data for navigation and other components
3. **Page front matter** - Per-page YAML metadata overriding site defaults
4. **Layout templates** - HTML/Liquid templates that render content

## Components and Interfaces

### 1. Site Configuration Component

**File:** `_config.yml`

**Purpose:** Central configuration for site metadata, author information, and Jekyll settings

**Key Sections to Modify:**

```yaml
# Basic Site Settings
title: "guidebee - Solana Trading System Development"
name: "guidebee"
description: "Full-stack developer documenting the journey of building a production-grade Solana trading system"
url: https://guidebee.github.io
repository: "guidebee/guidebee.github.io"

# Author Profile
author:
  avatar: "profile.png"
  name: "guidebee"
  bio: "Innovative full-stack developer with 20+ years of experience. Currently building a high-frequency trading system on Solana. Expert in React, .NET, Python, and cloud platforms."
  location: "" # Remove placeholder
  employer: "" # Remove academic employer
  email: "" # Optional: add real email
  
  # Social Links (keep only relevant)
  github: "guidebee"
  linkedin: "guidebee"
  
  # Remove academic links
  googlescholar: ""
  orcid: ""
  pubmed: ""
  arxiv: ""
  academia: ""
  researchgate: ""
  # ... (set all academic fields to empty)
```

**Interface:**
- Input: YAML configuration values
- Output: Site-wide variables accessible via `site.*` in Liquid templates
- Dependencies: Jekyll build process, GitHub Pages deployment

### 2. Navigation Component

**File:** `_data/navigation.yml`

**Purpose:** Define header navigation menu structure

**Current Structure:**
```yaml
main:
  - title: "Publications"
  - title: "Talks"
  - title: "Teaching"
  - title: "Portfolio"
  - title: "Blog Posts"
  - title: "CV"
  - title: "Guide"
```

**Modified Structure:**
```yaml
main:
  - title: "Blog Posts"
    url: /year-archive/
  - title: "Portfolio"
    url: /portfolio/
  - title: "CV"
    url: /cv/
  - title: "About"
    url: /
```

**Interface:**
- Input: YAML list of navigation items with title and URL
- Output: Rendered navigation menu in site header
- Dependencies: `_includes/masthead.html` template

### 3. Homepage Component

**File:** `_pages/about.md`

**Purpose:** Landing page introducing the blog and Solana trading system project

**Structure:**
```markdown
---
permalink: /
title: "Building a Production-Grade Solana Trading System"
author_profile: true
redirect_from: 
  - /about/
  - /about.html
---

[Introduction to guidebee and the project]
[Overview of the Solana trading system]
[Link to blog posts and repository]
[Technical focus areas]
```

**Interface:**
- Input: Markdown content with YAML front matter
- Output: Rendered HTML homepage with author sidebar
- Dependencies: `_layouts/single.html`, `_includes/author-profile.html`

### 4. Author Profile Component

**File:** `_includes/author-profile.html` (read-only, configured via `_config.yml`)

**Purpose:** Sidebar displaying author information and social links

**Behavior:**
- Reads author data from `site.author` configuration
- Conditionally renders social links based on non-empty values
- Displays avatar, name, bio, location, employer
- Renders icon links for GitHub, LinkedIn, email, etc.

**Configuration Strategy:**
- Set academic fields to empty strings in `_config.yml` to hide them
- Populate only relevant fields (GitHub, LinkedIn, bio)
- No template modification required

### 5. CV/Resume Component

**File:** `_pages/cv.md`

**Purpose:** Display comprehensive professional resume

**Structure:**
```markdown
---
layout: archive
title: "CV"
permalink: /cv/
author_profile: true
redirect_from:
  - /resume
---

## Profile Summary
[20+ years full-stack development experience]

## Technical Skills
### Languages
- TypeScript, JavaScript, C#, Python, C++, Java

### Cloud Platforms
- Azure, AWS, Google Cloud

### Mobile Platforms
- Android, iOS, Flutter, Xamarin, React Native, Unity3D

[... continue with all sections from resume]

## Certifications
- AWS Solutions Architect
- CompTIA Security+
- Microsoft certifications
[... full list]

## Professional Experience
[Detailed work history]

## Education
- Master of Science: Computer Integrated Manufacturing System
- Bachelor of Science: Mechanical and Electronic Engineering

## Demo Projects
- Solana NFT Playground
- OpenAI Australia Stock Exchange Env
- Android NFC Ticketing System
[... full list]
```

**Interface:**
- Input: Markdown content with structured sections
- Output: Rendered CV page with author sidebar
- Dependencies: `_layouts/archive.html`

### 6. Collections Management

**Purpose:** Control which content collections are active

**Collections to Disable:**
- `_publications/` - Academic publications
- `_talks/` - Conference talks
- `_teaching/` - Teaching materials

**Collections to Keep:**
- `_posts/` - Blog posts (primary content)
- `_portfolio/` - Project showcases

**Implementation Strategy:**
```yaml
# In _config.yml
collections:
  # Disable by setting output: false
  teaching:
    output: false
  publications:
    output: false
  talks:
    output: false
  
  # Keep active
  portfolio:
    output: true
    permalink: /:collection/:path/
```

**Note:** Setting `output: false` prevents Jekyll from generating pages for these collections, but they remain in the configuration for potential future use.

## Data Models

### Site Configuration Model

```yaml
Site:
  locale: string
  title: string
  name: string
  description: string
  url: string
  baseurl: string
  repository: string
  author: Author
  collections: Collection[]
  defaults: Default[]
  
Author:
  avatar: string
  name: string
  pronouns: string (optional)
  bio: string
  location: string (optional)
  employer: string (optional)
  email: string (optional)
  github: string (optional)
  linkedin: string (optional)
  [... other social links]
  
Collection:
  name: string
  output: boolean
  permalink: string
  
Default:
  scope:
    path: string
    type: string
  values:
    layout: string
    author_profile: boolean
    [... other defaults]
```

### Navigation Model

```yaml
Navigation:
  main: NavigationItem[]
  
NavigationItem:
  title: string
  url: string
```

### Page Front Matter Model

```yaml
PageFrontMatter:
  layout: string
  title: string
  permalink: string
  author_profile: boolean
  redirect_from: string[] (optional)
  date: date (for posts)
  categories: string[] (for posts)
  tags: string[] (for posts)
```

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Property 1: Author information consistency

*For any* page that displays the author profile, the author name SHALL be "guidebee" and the bio SHALL describe a full-stack developer with 20+ years of experience focused on Solana trading systems.

**Validates: Requirements 1.2, 1.3, 4.1, 4.2**

### Property 2: Academic link exclusion

*For any* rendered author profile, academic-specific links (Google Scholar, ORCID, PubMed, arXiv, ResearchGate) SHALL NOT be displayed.

**Validates: Requirements 1.4, 4.4**

### Property 3: Navigation menu composition

*For any* page navigation menu, it SHALL include "Blog Posts", "Portfolio", and "CV" links, and SHALL NOT include "Publications", "Talks", or "Teaching" links.

**Validates: Requirements 2.1, 2.2, 2.3, 2.4, 2.5**

### Property 4: Homepage content relevance

*For any* visitor accessing the homepage, the content SHALL introduce the Solana trading system project and SHALL NOT contain Academic Pages template boilerplate text.

**Validates: Requirements 3.1, 3.2, 3.3, 3.4, 3.5**

### Property 5: Repository reference accuracy

*For any* site configuration or metadata, the repository field SHALL reference "guidebee/guidebee.github.io".

**Validates: Requirements 5.1, 5.2, 5.3, 5.4**

### Property 6: Blog post visibility

*For any* blog posts page access, the "Getting Started: Building a Solana Trading System" post SHALL be displayed with author "guidebee" and relevant tags.

**Validates: Requirements 6.1, 6.2, 6.3, 6.4, 6.5**

### Property 7: Collection output control

*For any* Jekyll build, the publications, talks, and teaching collections SHALL have output disabled, while posts and portfolio collections SHALL remain active.

**Validates: Requirements 7.1, 7.2, 7.3, 7.4, 7.5**

### Property 8: CV content completeness

*For any* CV page access, the content SHALL include professional experience spanning 20+ years, technical skills sections, certifications, and demo projects including Solana NFT Playground.

**Validates: Requirements 8.1, 8.2, 8.3, 8.4, 8.5**

## Error Handling

### Configuration Errors

**Invalid YAML Syntax:**
- **Detection:** Jekyll build will fail with syntax error
- **Prevention:** Use YAML validators before committing
- **Recovery:** Review error message, fix syntax, rebuild

**Missing Required Fields:**
- **Detection:** Jekyll may build but render incorrectly
- **Prevention:** Validate all required fields are populated
- **Recovery:** Add missing fields to `_config.yml`

### Content Errors

**Broken Internal Links:**
- **Detection:** Manual testing or link checker tools
- **Prevention:** Use Jekyll's `link` tag for internal links
- **Recovery:** Update URLs in navigation and content files

**Missing Front Matter:**
- **Detection:** Page renders with default layout or fails
- **Prevention:** Include required front matter in all pages
- **Recovery:** Add front matter to affected files

### Build Errors

**GitHub Pages Build Failure:**
- **Detection:** Email notification from GitHub
- **Prevention:** Test locally with `bundle exec jekyll serve`
- **Recovery:** Review build log, fix errors, push again

**Plugin Compatibility:**
- **Detection:** Build warnings or failures
- **Prevention:** Use only GitHub Pages-compatible plugins
- **Recovery:** Remove incompatible plugins or use alternatives

## Testing Strategy

### Manual Testing Approach

Since this is a static site customization with no dynamic logic, testing will be primarily manual and visual:

#### 1. Local Development Testing

**Setup:**
```bash
# Install dependencies
bundle install

# Run local server
bundle exec jekyll serve

# Access at http://localhost:4000
```

**Test Cases:**

**TC1: Author Profile Display**
- Navigate to any page
- Verify sidebar shows "guidebee" as name
- Verify bio mentions "20+ years" and "Solana trading systems"
- Verify only GitHub and LinkedIn links appear
- Verify no academic links (Google Scholar, ORCID, etc.)

**TC2: Navigation Menu**
- Check header navigation on all pages
- Verify "Blog Posts", "Portfolio", "CV" links present
- Verify "Publications", "Talks", "Teaching" links absent
- Click each link to verify correct destination

**TC3: Homepage Content**
- Navigate to homepage (/)
- Verify title mentions Solana trading system
- Verify content introduces the project
- Verify no Academic Pages boilerplate text
- Verify link to solana-trading-system repository

**TC4: CV Page**
- Navigate to /cv/
- Verify professional experience section present
- Verify technical skills section with languages, cloud platforms
- Verify certifications section with AWS, CompTIA, Microsoft
- Verify demo projects section with Solana NFT Playground

**TC5: Blog Posts**
- Navigate to /year-archive/
- Verify "Getting Started: Building a Solana Trading System" post visible
- Click post to verify full content loads
- Verify author shows as "guidebee"
- Verify tags include "solana", "trading", "hft"

**TC6: Collection Disabling**
- Attempt to navigate to /publications/
- Attempt to navigate to /talks/
- Attempt to navigate to /teaching/
- Verify these pages return 404 or redirect

#### 2. GitHub Pages Deployment Testing

**After pushing to GitHub:**

**TC7: Live Site Verification**
- Visit https://guidebee.github.io
- Repeat TC1-TC6 on live site
- Verify GitHub Pages build succeeded (check repository Actions tab)

**TC8: Mobile Responsiveness**
- Access site on mobile device or browser dev tools
- Verify navigation menu collapses correctly
- Verify author profile displays properly
- Verify content is readable on small screens

**TC9: Cross-Browser Testing**
- Test on Chrome, Firefox, Safari, Edge
- Verify consistent rendering
- Verify all links work

#### 3. Content Validation

**TC10: Link Checking**
- Use tool like `html-proofer` or manual checking
- Verify all internal links resolve correctly
- Verify external links (GitHub, LinkedIn) work
- Verify repository links point to correct repos

**TC11: Metadata Verification**
- View page source
- Verify `<title>` tags are correct
- Verify Open Graph metadata for social sharing
- Verify canonical URLs

### Testing Checklist

Before considering customization complete:

- [ ] Local build succeeds without errors
- [ ] All author profile information correct
- [ ] Navigation menu shows only relevant links
- [ ] Homepage content introduces Solana project
- [ ] CV page displays complete resume
- [ ] Blog post visible and properly formatted
- [ ] Academic collections disabled
- [ ] GitHub Pages deployment succeeds
- [ ] Live site matches local preview
- [ ] Mobile responsive design works
- [ ] All links functional
- [ ] No console errors in browser

### Regression Testing

When making future updates:

1. Re-run all test cases above
2. Verify no unintended changes to working features
3. Check that new blog posts display correctly
4. Ensure author profile remains consistent

### Performance Testing

While not critical for a static blog, verify:

- Page load times are reasonable (< 3 seconds)
- Images are optimized
- No excessive JavaScript blocking render

## Implementation Notes

### File Modification Strategy

1. **Configuration files** (`_config.yml`, `_data/navigation.yml`):
   - Direct editing with careful YAML syntax
   - Commit after each logical change
   - Test locally before pushing

2. **Content files** (`_pages/*.md`, `_posts/*.md`):
   - Update front matter and content
   - Preserve existing structure where possible
   - Use consistent formatting

3. **Template files** (`_includes/*.html`, `_layouts/*.html`):
   - **Avoid modifying** unless absolutely necessary
   - Configuration-based approach preferred
   - If modifications needed, document thoroughly

### Deployment Process

1. Make changes locally
2. Test with `bundle exec jekyll serve`
3. Verify all test cases pass
4. Commit changes with descriptive message
5. Push to GitHub
6. Monitor GitHub Actions for build success
7. Verify live site at guidebee.github.io
8. If issues, rollback and fix

### Maintenance Considerations

- **Template updates:** Periodically check for Academic Pages template updates
- **Content updates:** Add new blog posts regularly to document Solana project progress
- **Link maintenance:** Periodically verify external links still work
- **Image optimization:** Compress images before adding to repository
- **Backup:** Repository is backed up on GitHub, but consider local backups of important content

### Future Enhancements

Potential improvements not in current scope:

- Custom domain name
- Google Analytics integration
- Comment system for blog posts
- Search functionality
- RSS feed customization
- Custom theme colors
- Additional portfolio items
- Project showcase pages
