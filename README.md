# Sarat Chandra's Personal Website

A modern, clean personal website built with [Zola](https://www.getzola.org/) static site generator.

## Recent Improvements

### Configuration Updates

- Updated `config.toml` for latest Zola version compatibility
- Changed `generate_feed` to `generate_feeds`
- Added proper feed configuration with `feed_filenames`
- Moved markdown configuration to proper `[markdown]` section
- Added sitemap and robots.txt generation

### Design Enhancements

- **Modern CSS**: Replaced Chota CSS framework with custom modern CSS
- **Improved Typography**: Better font stack using system fonts
- **Enhanced Visual Hierarchy**: Better spacing, colors, and layout
- **Responsive Design**: Mobile-first approach with proper breakpoints
- **Accessibility**: Added proper ARIA labels, focus states, and semantic HTML
- **Performance**: Optimized CSS, removed external dependencies
- **Modern Card Design**: Blog posts now have card-style layout with shadows and hover effects
- **Better Date Display**: Enhanced date styling with gradient backgrounds
- **Improved Navigation**: Cleaner navigation with hover states and active indicators

### Template Improvements

- **Semantic HTML**: Updated templates to use proper HTML5 semantic elements
- **Better Structure**: Improved template organization and consistency
- **Enhanced Metadata**: Better handling of page descriptions and meta information
- **Improved Tags**: Better tag display and navigation
- **Modern Icons**: Updated social media icons with better SVG implementations

### Key Features

- ✅ Modern, clean design
- ✅ Fully responsive
- ✅ Fast loading
- ✅ Accessible
- ✅ SEO optimized
- ✅ RSS feed support
- ✅ Tag system
- ✅ Syntax highlighting
- ✅ Social media links

## Building the Site

```bash
# Check the site for errors
zola check

# Build the site
zola build

# Serve locally for development
zola serve
```

## File Structure

```
├── config.toml          # Site configuration
├── content/             # Markdown content
│   ├── blog/           # Blog posts
│   └── book-reviews/   # Book reviews
├── static/             # Static assets
│   └── css/           # Custom CSS
├── templates/          # Zola templates
│   ├── macros/        # Template macros
│   └── tags/          # Tag-related templates
└── public/            # Generated site (after build)
```

## License

Personal website content. Please respect copyright for written content while code/templates can be referenced.
