# GitBook Publishing Guide

This guide will help you publish the PulseGuard documentation to GitBook as a unified space.

## Prerequisites

1. **GitBook Account**: Sign up at [gitbook.com](https://gitbook.com)
2. **Git Repository**: The documentation is ready in `/Users/alperzkn/Documents/dev/pulseguard-gitbook/`
3. **GitHub Repository**: You'll need to push this to GitHub for GitBook integration

## Step 1: Prepare Git Repository

### 1. Initialize Git Repository

```bash
cd /Users/alperzkn/Documents/dev/pulseguard-gitbook

# Initialize git repository
git init

# Add GitBook configuration
echo "node_modules/" > .gitignore
echo "_book/" >> .gitignore
echo ".DS_Store" >> .gitignore

# Add all files
git add .
git commit -m "Initial commit: PulseGuard documentation for GitBook"
```

### 2. Create GitHub Repository

1. Go to [GitHub](https://github.com) and create a new repository
2. Name it `pulseguard-documentation` (or similar)
3. Make it **public** (required for GitBook free plan)
4. Don't initialize with README (we already have one)

### 3. Push to GitHub

```bash
# Replace with your GitHub repository URL
git remote add origin https://github.com/YOUR_USERNAME/pulseguard-documentation.git
git branch -M main
git push -u origin main
```

## Step 2: GitBook Setup

### Option A: GitBook Cloud (Recommended)

1. **Sign in to GitBook**
   - Go to [app.gitbook.com](https://app.gitbook.com)
   - Sign in with GitHub for easy integration

2. **Create New Space**
   - Click "New Space" or "Create"
   - Choose "Git Sync" option
   - Select "GitHub" as the provider

3. **Connect GitHub Repository**
   - Authorize GitBook to access your GitHub account
   - Select your `pulseguard-documentation` repository
   - Choose the `main` branch
   - Set the root path to `/` (since files are in root)

4. **Configure Space Settings**
   - **Space Name**: "PulseGuard Documentation"
   - **Description**: "Comprehensive documentation for PulseGuard cryptocurrency data ecosystem"
   - **Visibility**: Public or Private (based on your preference)

5. **Import Documentation**
   - GitBook will automatically detect the `SUMMARY.md` file
   - It will import all the markdown files according to the structure
   - Wait for the import to complete (usually 1-2 minutes)

### Option B: GitBook CLI (Alternative)

If you prefer using the command line:

1. **Install GitBook CLI**
   ```bash
   npm install -g gitbook-cli
   ```

2. **Install GitBook Plugins**
   ```bash
   cd /Users/alperzkn/Documents/dev/pulseguard-gitbook
   gitbook install
   ```

3. **Test Locally**
   ```bash
   gitbook serve
   # Visit http://localhost:4000 to preview
   ```

4. **Build for Deployment**
   ```bash
   gitbook build
   # Generates _book/ directory with static HTML
   ```

## Step 3: Configure GitBook Space

### 1. Customize Appearance

In GitBook settings:
- **Cover Image**: Add PulseGuard logo/banner
- **Favicon**: Upload PulseGuard icon
- **Theme**: Choose appropriate color scheme
- **Typography**: Select readable fonts

### 2. Configure Integrations

**GitHub Integration**:
- Enable automatic sync from GitHub
- Set up webhooks for automatic updates
- Configure branch protection if needed

**Search Configuration**:
- Enable search indexing
- Configure search scopes
- Test search functionality

### 3. Set Up Custom Domain (Optional)

If you have a custom domain:
1. Go to Space Settings → Domain
2. Add your custom domain (e.g., `docs.pulseguard.com`)
3. Configure DNS CNAME record
4. Enable SSL certificate

## Step 4: Content Review and Optimization

### 1. Review Navigation

- Check that all pages load correctly
- Verify the table of contents structure
- Test internal links between pages
- Ensure proper page hierarchy

### 2. Optimize Content

**Fix Image Paths** (if any):
```markdown
<!-- Change relative paths to absolute if needed -->
![Architecture](./images/architecture.png)
# to
![Architecture](https://raw.githubusercontent.com/YOUR_USERNAME/pulseguard-documentation/main/images/architecture.png)
```

**Update Variables**:
Edit the configuration to replace placeholder URLs:
```json
{
  "variables": {
    "api_base_url": "https://your-actual-api-domain.com/api/v1"
  }
}
```

### 3. Test All Features

- Test code syntax highlighting
- Verify all links work
- Check responsive design on mobile
- Test search functionality
- Verify PDF export (if enabled)

## Step 5: Set Up Automatic Updates

### 1. GitHub Webhook Configuration

GitBook automatically sets up webhooks, but verify:
1. Go to your GitHub repository settings
2. Click "Webhooks"
3. Verify GitBook webhook is present and active
4. Test by making a small change and pushing

### 2. Branch Protection

Protect your main branch:
```bash
# In GitHub repository settings
Settings → Branches → Add rule
- Branch name pattern: main
- Require pull request reviews
- Require status checks
```

### 3. Automated Updates

Create a simple update workflow:

**.github/workflows/update-docs.yml**:
```yaml
name: Update Documentation

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Validate GitBook structure
      run: |
        # Check required files exist
        test -f README.md
        test -f SUMMARY.md
        test -f .gitbook.yaml
        echo "✅ GitBook structure validated"
```

## Step 6: Share and Collaborate

### 1. Set Permissions

In GitBook space settings:
- **Admin**: Repository owners
- **Editor**: Team members who can edit
- **Viewer**: Public or selected users

### 2. Enable Comments

- Turn on page comments for feedback
- Configure comment moderation
- Set notification preferences

### 3. Analytics (Optional)

- Enable GitBook analytics
- Add Google Analytics if desired
- Set up conversion tracking

## Step 7: Maintenance

### 1. Regular Updates

Create a maintenance schedule:
- **Weekly**: Review and update any outdated information
- **Monthly**: Check for broken links and fix them
- **Quarterly**: Review structure and add new sections

### 2. Version Management

For major updates:
```bash
# Create version tags
git tag -a v1.0.0 -m "Initial documentation release"
git push origin v1.0.0

# Use GitBook variants for different versions if needed
```

### 3. Backup Strategy

- GitHub serves as primary backup
- Export PDF versions periodically
- Consider mirroring to another documentation platform

## Troubleshooting

### Common Issues

**1. Import Fails**
- Check SUMMARY.md syntax
- Verify all referenced files exist
- Ensure markdown syntax is valid

**2. Images Don't Load**
- Use absolute URLs for images
- Check image file paths are correct
- Verify image permissions on GitHub

**3. Formatting Issues**
- Test locally with GitBook CLI first
- Check for unsupported markdown features
- Validate YAML frontmatter if used

**4. Sync Issues**
- Check GitHub webhook status
- Verify GitBook permissions
- Force manual sync if needed

### Support Resources

- **GitBook Documentation**: [docs.gitbook.com](https://docs.gitbook.com)
- **Community Forum**: [community.gitbook.com](https://community.gitbook.com)
- **GitHub Integration Guide**: [GitBook GitHub Integration](https://docs.gitbook.com/integrations/git-sync)

## Final Checklist

Before going live:

- [ ] All pages load without errors
- [ ] Navigation structure is logical
- [ ] All internal links work
- [ ] Code examples are properly formatted
- [ ] Images and diagrams display correctly
- [ ] Search functionality works
- [ ] Mobile responsiveness is good
- [ ] PDF export works (if enabled)
- [ ] Custom domain is configured (if applicable)
- [ ] Analytics are set up (if desired)
- [ ] Team permissions are configured
- [ ] Backup strategy is in place

## Publishing Command Summary

```bash
# Quick publishing workflow
cd /Users/alperzkn/Documents/dev/pulseguard-gitbook

# Initialize and push to GitHub
git init
git add .
git commit -m "Initial PulseGuard documentation"
git remote add origin https://github.com/YOUR_USERNAME/pulseguard-documentation.git
git push -u origin main

# Then configure GitBook Cloud to sync with the GitHub repository
```

Your PulseGuard documentation will be available at:
`https://YOUR_GITBOOK_SPACE.gitbook.io/pulseguard-documentation/`

Or with custom domain:
`https://docs.your-domain.com/`