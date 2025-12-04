# Implementation Plan

- [x] 1. Update site configuration and author profile



  - Update `_config.yml` with guidebee's personal information, site title, description, and repository reference
  - Configure author profile with name, bio, GitHub, and LinkedIn
  - Remove all academic-specific fields (Google Scholar, ORCID, PubMed, etc.)
  - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 4.1, 4.2, 4.3, 4.4, 4.5, 5.1_

- [x] 2. Update navigation menu structure




  - Modify `_data/navigation.yml` to remove academic links (Publications, Talks, Teaching)
  - Reorder navigation to prioritize Blog Posts, Portfolio, and CV
  - Remove or comment out the Guide link
  - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

- [x] 3. Customize homepage content





  - Update `_pages/about.md` with introduction to Solana trading system project
  - Remove all Academic Pages template boilerplate text
  - Add links to the solana-trading-system repository
  - Include brief overview of blog purpose and technical focus
  - _Requirements: 3.1, 3.2, 3.3, 3.4, 3.5_
-

- [x] 4. Create comprehensive CV page




  - Update `_pages/cv.md` with guidebee's complete professional resume
  - Add Profile Summary section highlighting 20+ years of experience
  - Include Technical Skills section (Languages, Cloud Platforms, Mobile Platforms, etc.)
  - Add Certifications section with all credentials
  - Include Professional Experience section with detailed work history
  - Add Education section
  - Include Demo Projects and Spare Time Projects section
  - _Requirements: 8.1, 8.2, 8.3, 8.4, 8.5_

- [x] 5. Disable academic collections





  - Update `_config.yml` collections section to set `output: false` for publications, talks, and teaching
  - Verify posts and portfolio collections remain active
  - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5_

- [x] 6. Verify existing blog post configuration





  - Check that `_posts/2025-12-04-getting-started-building-solana-trading-system.md` has correct author metadata
  - Verify tags and categories are properly set
  - Ensure post displays correctly in blog listing
  - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

- [x] 7. Test locally and verify all changes




  - Run `bundle exec jekyll serve` to build site locally
  - Verify author profile displays correctly on all pages
  - Check navigation menu shows only relevant links
  - Verify homepage content is customized
  - Test CV page displays complete resume
  - Verify blog post is visible and properly formatted
  - Check that academic collection pages return 404
  - Test all internal and external links
  - _Requirements: All_

- [ ] 8. Deploy to GitHub Pages and final verification
  - Commit all changes with descriptive commit messages
  - Push to GitHub repository
  - Monitor GitHub Actions for successful build
  - Verify live site at https://guidebee.github.io
  - Test on mobile devices and different browsers
  - Verify all functionality works on production site
  - _Requirements: All_
