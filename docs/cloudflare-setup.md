# Cloudflare Pages Setup Guide

This guide walks you through deploying your Hugo blog to Cloudflare Pages.

## Prerequisites

- A Cloudflare account (free tier is sufficient)
- Your blog repository pushed to GitHub
- A custom domain (optional, Cloudflare provides a free subdomain)

## Step 1: Connect GitHub Repository to Cloudflare Pages

1. Log in to your [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. Navigate to **Workers & Pages** in the left sidebar
3. Click **Create application** > **Pages** > **Connect to Git**
4. Authorize Cloudflare to access your GitHub account
5. Select the repository containing your Hugo blog (e.g., `jhyoong/gitblog`)
6. Click **Begin setup**

## Step 2: Configure Build Settings

Configure the following build settings:

### Framework Preset
- Select **Hugo** from the dropdown

### Build Configuration
- **Production branch:** `main` (or your default branch)
- **Build command:** `hugo --gc --minify`
- **Build output directory:** `public`

### Environment Variables
Add the following environment variable to specify the Hugo version:

| Variable Name | Value |
|--------------|-------|
| `HUGO_VERSION` | `0.146.0` |

This ensures Cloudflare uses the same Hugo version as your local environment.

## Step 3: Deploy Your Site

1. Click **Save and Deploy**
2. Cloudflare will build and deploy your site automatically
3. The first deployment takes 1-3 minutes
4. Once complete, you'll see a deployment URL (e.g., `https://gitblog-xxx.pages.dev`)

## Step 4: Configure Custom Domain (Optional)

### If you already have a domain on Cloudflare:

1. Go to your Pages project
2. Click **Custom domains**
3. Click **Set up a custom domain**
4. Enter your desired domain or subdomain (e.g., `blog.yourdomain.com`)
5. Cloudflare will automatically create the required DNS records
6. Click **Activate domain**

### If you need to add a domain to Cloudflare:

1. Go to **Websites** in the Cloudflare dashboard
2. Click **Add a site**
3. Enter your domain name and follow the setup wizard
4. Update your domain's nameservers to Cloudflare's
5. Wait for DNS propagation (can take up to 24 hours)
6. Then follow the steps above to set up the custom domain

## Step 5: Update Hugo Base URL

Once you have your final domain/URL, update `hugo.toml`:

```toml
baseURL = 'https://yourdomain.com/'  # or your Cloudflare Pages URL
```

Commit and push this change to trigger a new deployment.

## Automatic Deployments

Cloudflare Pages automatically deploys your site when you push to your production branch:

1. Make changes locally
2. Commit and push to GitHub
3. Cloudflare automatically builds and deploys
4. Changes are live in 1-3 minutes

## Deployment Status

You can monitor deployments in the Cloudflare dashboard:

1. Go to **Workers & Pages**
2. Click on your project
3. View the **Deployments** tab
4. See build logs, status, and deployment history

## Troubleshooting

### Build Failures

**Issue:** Build fails with "Hugo version mismatch"
- **Solution:** Ensure `HUGO_VERSION` environment variable matches your local version

**Issue:** Theme not found
- **Solution:** Ensure theme is committed to your repository (not just submoduled)

### DNS Issues

**Issue:** Custom domain shows "Not found"
- **Solution:** Wait for DNS propagation (up to 24 hours)
- **Solution:** Verify DNS records are correctly configured in Cloudflare

### Build Takes Too Long

- Cloudflare's free tier has a 500 builds/month limit
- Each push triggers a build
- Consider batching commits or using draft mode locally

## Next Steps

- Set up a deployment webhook for notifications
- Configure preview deployments for pull requests
- Add analytics tracking
- Optimize images and assets for performance

## Additional Resources

- [Cloudflare Pages Documentation](https://developers.cloudflare.com/pages/)
- [Hugo Documentation](https://gohugo.io/documentation/)
- [Ananke Theme Documentation](https://github.com/theNewDynamic/gohugo-theme-ananke)
