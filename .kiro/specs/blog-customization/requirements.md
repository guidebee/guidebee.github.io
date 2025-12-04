# Requirements Document

## Introduction

This document specifies the requirements for customizing the Academic Pages Jekyll blog template for guidebee's personal blog focused on documenting the development of a Solana trading system. The blog is hosted on GitHub Pages at guidebee.github.io and needs to be personalized with the correct author information, social links, and navigation structure appropriate for a technical development blog rather than an academic portfolio.

## Glossary

- **Jekyll**: A static site generator that transforms Markdown files into HTML websites
- **GitHub Pages**: A free hosting service for static websites from GitHub repositories
- **Academic Pages**: The Jekyll template currently being used for the blog
- **Front Matter**: YAML metadata at the top of Markdown files that controls page behavior
- **Navigation**: The menu structure displayed in the website header
- **Author Profile**: The sidebar information displayed on pages showing author details
- **Collection**: A Jekyll feature for grouping related content (e.g., posts, publications, talks)

## Requirements

### Requirement 1

**User Story:** As a blog visitor, I want to see guidebee's correct personal information and social links, so that I can learn about the author and connect with them on various platforms.

#### Acceptance Criteria

1. WHEN the site configuration is loaded THEN the system SHALL set the site title to "guidebee - Solana Trading System Development"
2. WHEN the site configuration is loaded THEN the system SHALL set the author name to "guidebee" in all locations
3. WHEN the author profile is displayed THEN the system SHALL show the GitHub username as "guidebee"
4. WHEN the author profile is displayed THEN the system SHALL show the LinkedIn profile as "guidebee"
5. WHEN the site URL is configured THEN the system SHALL use "https://guidebee.github.io" as the base URL

### Requirement 2

**User Story:** As a blog visitor, I want to see navigation appropriate for a development blog, so that I can easily find blog posts and project information without being confused by academic sections.

#### Acceptance Criteria

1. WHEN the navigation menu is rendered THEN the system SHALL include a "Blog Posts" link
2. WHEN the navigation menu is rendered THEN the system SHALL include a "Portfolio" link for project showcases
3. WHEN the navigation menu is rendered THEN the system SHALL remove the "Publications" link
4. WHEN the navigation menu is rendered THEN the system SHALL remove the "Talks" link
5. WHEN the navigation menu is rendered THEN the system SHALL remove the "Teaching" link

### Requirement 3

**User Story:** As the blog owner, I want the homepage to introduce my Solana trading system project, so that visitors immediately understand the blog's purpose and content focus.

#### Acceptance Criteria

1. WHEN a visitor accesses the homepage THEN the system SHALL display a title about the Solana trading system development blog
2. WHEN the homepage content is rendered THEN the system SHALL include an introduction to the project goals
3. WHEN the homepage content is rendered THEN the system SHALL reference the solana-trading-system repository
4. WHEN the homepage content is rendered THEN the system SHALL provide context about documenting the development journey
5. WHEN the homepage content is rendered THEN the system SHALL remove all Academic Pages template boilerplate text

### Requirement 4

**User Story:** As the blog owner, I want the author bio to reflect my 20+ years of full-stack development experience and current focus on Solana trading systems, so that visitors understand my technical background and expertise.

#### Acceptance Criteria

1. WHEN the author profile bio is displayed THEN the system SHALL describe guidebee as an innovative full-stack developer with over 20 years of experience specializing in web applications, cloud platforms, and currently focused on Solana trading systems
2. WHEN the author profile name is rendered THEN the system SHALL display "guidebee" as the author name
3. WHEN the author profile is rendered THEN the system SHALL remove placeholder text like "Your Sidebar Name", "Red Brick University", and "Earth"
4. WHEN the author profile is rendered THEN the system SHALL remove academic-specific fields like employer, Google Scholar, ORCID, PubMed, arxiv, and other academic links
5. WHEN the author profile is rendered THEN the system SHALL keep only relevant social links (GitHub as "guidebee", LinkedIn as "guidebee", and optionally email)

### Requirement 5

**User Story:** As the blog owner, I want the repository reference to point to my actual blog repository, so that the GitHub integration features work correctly.

#### Acceptance Criteria

1. WHEN the site configuration is loaded THEN the system SHALL set the repository field to "guidebee/guidebee.github.io"
2. WHEN GitHub Pages builds the site THEN the system SHALL use the correct repository for edit links
3. WHEN the site footer is rendered THEN the system SHALL display the correct repository information
4. WHEN site metadata is generated THEN the system SHALL reference the correct GitHub repository

### Requirement 6

**User Story:** As a blog visitor, I want to see the existing Solana trading system blog post prominently, so that I can start reading about the project development.

#### Acceptance Criteria

1. WHEN the blog posts page is accessed THEN the system SHALL display the "Getting Started: Building a Solana Trading System" post
2. WHEN the homepage is rendered THEN the system SHALL include a link or reference to recent blog posts
3. WHEN blog posts are listed THEN the system SHALL show posts in reverse chronological order
4. WHEN a blog post is displayed THEN the system SHALL show the correct author as "guidebee"
5. WHEN blog post metadata is rendered THEN the system SHALL display relevant tags like "solana", "trading", "hft"

### Requirement 7

**User Story:** As the blog owner, I want to remove or hide academic-focused collections, so that the blog structure is clean and focused on development content.

#### Acceptance Criteria

1. WHEN the site is built THEN the system SHALL disable or hide the publications collection
2. WHEN the site is built THEN the system SHALL disable or hide the talks collection
3. WHEN the site is built THEN the system SHALL disable or hide the teaching collection
4. WHEN the navigation is rendered THEN the system SHALL not display links to disabled collections
5. WHEN the site structure is evaluated THEN the system SHALL maintain only the posts and portfolio collections as active

### Requirement 8

**User Story:** As a blog visitor, I want to access guidebee's professional resume and technical skills, so that I can understand their comprehensive background and expertise.

#### Acceptance Criteria

1. WHEN the CV page is accessed THEN the system SHALL display guidebee's professional experience spanning 20+ years
2. WHEN the CV page is rendered THEN the system SHALL include technical skills sections covering languages, cloud platforms, mobile platforms, and certifications
3. WHEN the CV page is displayed THEN the system SHALL list key certifications including AWS Solutions Architect, CompTIA Security+, and Microsoft certifications
4. WHEN the CV page is rendered THEN the system SHALL showcase demo projects and spare time projects including Solana NFT Playground and OpenAI Australia Stock Exchange
5. WHEN the CV navigation link is clicked THEN the system SHALL navigate to the updated CV page with guidebee's actual resume content
