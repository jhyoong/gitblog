# Product Requirements Document: Event-Driven Personal Blog System

## Executive Summary

A serverless personal blog platform leveraging Cloudflare's free tier services, featuring event-driven deployments, mTLS-protected admin interface, and local content storage. The system ensures public traffic never directly reaches the origin server while maintaining secure administrative access.

---

## 1. Project Overview

### 1.1 Purpose
Build a free, secure, and automated blog platform that:
- Hosts content on Cloudflare's edge network
- Stores blog data locally on a Linux server
- Provides secure admin access via mTLS authentication
- Deploys automatically when content is published

### 1.2 Key Constraints
- Must use only Cloudflare's free tier services
- Public traffic must not hit the origin Linux server
- Updates can have up to 24-hour latency (but event-driven approach reduces this to minutes)
- Low traffic expectations (personal blog scale)

### 1.3 Target Scale
- Up to hundreds of blog posts
- Low to moderate traffic volume
- Single administrator (blog owner)

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Internet Users                       │
└────────────────────────────┬────────────────────────────────┘
                             │
                             │ HTTPS
                             ▼
                    ┌────────────────────┐
                    │  Cloudflare Pages  │
                    │   (Static Blog)    │
                    │  blog.domain.com   │
                    └────────────────────┘
                             ▲
                             │ Auto-deploy
                             │
                    ┌────────────────────┐
                    │   Git Repository   │
                    │  (GitHub/GitLab)   │
                    └────────────────────┘
                             ▲
                             │ Git Push (on publish)
                             │
┌────────────────────────────┼────────────────────────────────┐
│                            │                                 │
│  ┌─────────────────────────┴──────────┐                    │
│  │      Linux Server (Origin)         │                    │
│  │                                     │                    │
│  │  ┌──────────────────────────────┐  │                    │
│  │  │   Blog API Server            │  │                    │
│  │  │   (Admin Operations)         │◄─┼────────┐          │
│  │  └──────────────────────────────┘  │        │          │
│  │                                     │        │          │
│  │  ┌──────────────────────────────┐  │        │          │
│  │  │   Markdown Storage           │  │        │          │
│  │  │   (Blog Posts & Drafts)      │  │        │          │
│  │  └──────────────────────────────┘  │        │          │
│  │                                     │        │          │
│  │  ┌──────────────────────────────┐  │        │          │
│  │  │   Static Site Generator      │  │        │          │
│  │  │   (Markdown → HTML)          │  │        │          │
│  │  └──────────────────────────────┘  │        │          │
│  │                                     │        │          │
│  │  ┌──────────────────────────────┐  │        │          │
│  │  │   Deployment Script          │  │        │          │
│  │  │   (Git Commit & Push)        │  │        │          │
│  │  └──────────────────────────────┘  │        │          │
│  └─────────────────────────────────────┘        │          │
│                                                  │          │
└──────────────────────────────────────────────────┼──────────┘
                                                   │
                                                   │ mTLS
                                                   │ Authenticated
                                                   │
                                          ┌────────┴─────────┐
                                          │ Cloudflare Worker│
                                          │  (Admin Gateway) │
                                          │ api.domain.com   │
                                          └──────────────────┘
                                                   ▲
                                                   │ mTLS
                                                   │
                                          ┌────────┴─────────┐
                                          │  Blog Admin      │
                                          │  (Your Device)   │
                                          └──────────────────┘
```

### 2.2 Component Overview

| Component | Purpose | Technology | Hosting |
|-----------|---------|------------|---------|
| **Cloudflare Pages** | Static blog hosting | Static HTML/CSS/JS | Cloudflare Edge |
| **Cloudflare Worker** | mTLS authentication gateway | JavaScript/TypeScript | Cloudflare Edge |
| **Git Repository** | Version control & deployment trigger | Git | GitHub/GitLab (free) |
| **Linux Server** | Content storage & API server | Language-agnostic | Self-hosted |
| **Static Site Generator** | Converts markdown to HTML | Hugo/Jekyll/Custom | Linux Server |

---

## 3. Detailed Component Specifications

### 3.1 Cloudflare Pages (Public Blog)

**Purpose:** Serve the static blog to public visitors

**Configuration:**
- Domain: `blog.yourdomain.com` or `yourdomain.com`
- Build command: Automatic from Git repository
- Deploy trigger: Git push to main/master branch
- Framework: Static HTML or SSG output

**Requirements:**
- Must stay within Cloudflare Pages free tier:
  - 500 builds per month
  - 1 build at a time
  - Unlimited bandwidth
  - Unlimited requests

**Content Structure:**
- Index page with blog post listings
- Individual blog post pages
- Archive/category pages
- RSS feed (optional)
- About page

---

### 3.2 Cloudflare Worker (Admin Gateway)

**Purpose:** Authenticate admin requests via mTLS and proxy to Linux server

**Configuration:**
- Route: `api.yourdomain.com/admin/*`
- Free tier limits:
  - 100,000 requests per day
  - 10ms CPU time per request

**Responsibilities:**
1. Validate mTLS client certificate
2. Extract and verify certificate details
3. Proxy authenticated requests to Linux server
4. Return responses to admin client
5. Block unauthenticated requests

**Security Requirements:**
- Client certificate validation must be strict
- Must verify certificate against uploaded CA
- Should check certificate expiration
- Should log authentication attempts
- Must use HTTPS for backend communication

---

### 3.3 Linux Server (Origin)

**Purpose:** Host blog API, content storage, and deployment automation

**Components:**

#### 3.3.1 Blog API Server
- Listens on localhost or private network only
- Accessed only via Cloudflare Worker proxy
- Handles CRUD operations for blog posts
- Triggers deployment on publish events

#### 3.3.2 Markdown Storage
- Directory structure for blog posts
- Separate directories for drafts and published posts
- Metadata stored in frontmatter or separate files
- Version control ready (Git)

#### 3.3.3 Static Site Generator
- Converts markdown files to static HTML
- Applies templates and styling
- Generates index and archive pages
- Creates RSS feed

#### 3.3.4 Deployment Script
- Triggered by publish/delete events
- Runs static site generator
- Commits changes to Git repository
- Pushes to remote (triggers Cloudflare Pages deployment)

**Directory Structure Example:**
```
/blog-content/
├── drafts/
│   └── unpublished-post.md
├── published/
│   ├── 2025-01-15-first-post.md
│   └── 2025-02-01-second-post.md
├── templates/
│   ├── base.html
│   └── post.html
└── static/
    ├── css/
    └── images/
```

---

## 4. API Specifications

### 4.1 Admin API Endpoints

Base URL: `https://api.yourdomain.com/admin`

All endpoints require mTLS client certificate authentication.

#### 4.1.1 List Posts
- **Endpoint:** `GET /posts`
- **Query Parameters:**
  - `status`: `draft` | `published` | `all` (default: `all`)
  - `limit`: Number of posts to return
  - `offset`: Pagination offset
- **Response:** Array of post metadata (id, title, status, created, updated)

#### 4.1.2 Get Post
- **Endpoint:** `GET /posts/{id}`
- **Response:** Full post content and metadata

#### 4.1.3 Create Post
- **Endpoint:** `POST /posts`
- **Request Body:**
  - `title`: Post title
  - `content`: Markdown content
  - `status`: `draft` (default)
  - `tags`: Array of tags (optional)
  - `category`: Category (optional)
- **Response:** Created post with id

#### 4.1.4 Update Post
- **Endpoint:** `PUT /posts/{id}`
- **Request Body:** Same as create (partial updates allowed)
- **Response:** Updated post

#### 4.1.5 Publish Post
- **Endpoint:** `POST /posts/{id}/publish`
- **Action:** 
  - Move from drafts to published
  - **Trigger deployment pipeline**
- **Response:** Published post with deployment status

#### 4.1.6 Unpublish Post
- **Endpoint:** `POST /posts/{id}/unpublish`
- **Action:**
  - Move from published to drafts
  - **Trigger deployment pipeline**
- **Response:** Unpublished post with deployment status

#### 4.1.7 Delete Post
- **Endpoint:** `DELETE /posts/{id}`
- **Action:** 
  - Delete post file
  - **Trigger deployment pipeline** (if published)
- **Response:** Success status with deployment status

#### 4.1.8 Get Deployment Status
- **Endpoint:** `GET /deploy/status`
- **Response:** Last deployment status and timestamp

### 4.2 Response Format

All responses should follow a consistent format:

**Success Response:**
```json
{
  "success": true,
  "data": { ... },
  "message": "Operation completed successfully"
}
```

**Error Response:**
```json
{
  "success": false,
  "error": "Error message",
  "code": "ERROR_CODE"
}
```

---

## 5. mTLS Security Requirements

### 5.1 Certificate Infrastructure

#### 5.1.1 Certificate Authority (CA)
- Self-signed CA for issuing client certificates
- CA certificate uploaded to Cloudflare for validation
- CA private key stored securely on admin device
- Validity period: 10+ years

#### 5.1.2 Client Certificate
- Issued by the CA
- Contains identifying information (CN, O, OU)
- Validity period: 1-2 years
- Stored securely on admin device
- Used by admin client for API requests

### 5.2 Cloudflare Worker mTLS Configuration

**Required Validations:**
1. Certificate must be signed by uploaded CA
2. Certificate must not be expired
3. Certificate must not be revoked (if CRL implemented)
4. Certificate CN or SAN must match expected values

**Configuration Steps:**
1. Upload CA certificate to Cloudflare
2. Enable mTLS authentication for `api.yourdomain.com`
3. Configure Worker to validate certificate
4. Implement certificate extraction and verification logic

### 5.3 Certificate Management

**Generation Requirements:**
- Use strong key lengths (RSA 4096-bit or EC P-384)
- Include appropriate key usage flags
- Set proper certificate purposes
- Include necessary extensions

**Renewal Process:**
- Certificate expiration monitoring
- Renewal procedure before expiration
- Certificate revocation capability (if needed)

---

## 6. Deployment Pipeline Specification

### 6.1 Pipeline Triggers

The deployment pipeline is triggered by:
1. `POST /posts/{id}/publish` - Publishing a post
2. `POST /posts/{id}/unpublish` - Unpublishing a post
3. `DELETE /posts/{id}` - Deleting a published post

### 6.2 Pipeline Steps

#### Step 1: Static Site Generation
- Input: All markdown files in `published/` directory
- Process: Convert markdown to HTML using site generator
- Output: Static HTML files ready for deployment
- Duration: Seconds to minutes (depending on post count)

#### Step 2: Git Operations
- Stage all changes in the output directory
- Create commit with meaningful message (e.g., "Publish: Post Title")
- Push to remote Git repository
- Duration: Seconds

#### Step 3: Cloudflare Pages Deployment
- Automatically triggered by Git push
- Cloudflare pulls latest code
- Builds and deploys to edge network
- Duration: 1-3 minutes typically

### 6.3 Pipeline Monitoring

**Requirements:**
- Log each deployment attempt
- Track deployment status (pending, in_progress, success, failed)
- Store deployment timestamps
- Provide status endpoint for querying

**Status Information:**
- Last deployment time
- Deployment result (success/failure)
- Affected post(s)
- Error messages (if failed)

### 6.4 Error Handling

**Failure Scenarios:**
1. Static site generation fails
   - Log error details
   - Return error to API caller
   - Preserve previous state

2. Git operations fail
   - Retry once
   - Log error
   - Alert administrator

3. Cloudflare Pages deployment fails
   - Logged by Cloudflare
   - Verify via webhook or API check
   - Manual intervention may be required

---

## 7. Data Models

### 7.1 Blog Post Structure

**Markdown File Format:**

```markdown
---
id: unique-post-id
title: Post Title
slug: post-title-url-slug
author: Author Name
created: 2025-01-15T10:00:00Z
updated: 2025-01-16T14:30:00Z
published: 2025-01-16T14:30:00Z
status: published
tags: [tag1, tag2, tag3]
category: Category Name
excerpt: Brief description of the post
---

# Post Content

Markdown content here...
```

**Required Fields:**
- `id`: Unique identifier
- `title`: Post title
- `slug`: URL-friendly slug
- `created`: Creation timestamp
- `status`: `draft` or `published`

**Optional Fields:**
- `author`: Author name
- `updated`: Last update timestamp
- `published`: Publication timestamp
- `tags`: Array of tags
- `category`: Category
- `excerpt`: Short description

### 7.2 Deployment Log Entry

```json
{
  "id": "deployment-id",
  "timestamp": "2025-01-16T14:30:00Z",
  "trigger": "publish",
  "post_id": "post-id",
  "status": "success|failed|in_progress",
  "duration_seconds": 120,
  "error": "Error message if failed",
  "git_commit": "commit-hash"
}
```

---

## 8. User Flows

### 8.1 Create and Publish Flow

1. **Create Draft**
   - Admin sends `POST /posts` with title and content
   - API saves markdown file to `drafts/` directory
   - Returns post ID and metadata

2. **Edit Draft** (optional, can repeat)
   - Admin sends `PUT /posts/{id}` with updates
   - API updates markdown file
   - No deployment triggered

3. **Publish Post**
   - Admin sends `POST /posts/{id}/publish`
   - API moves file from `drafts/` to `published/`
   - **Deployment pipeline triggered**
   - Returns deployment status

4. **View Published Post**
   - After 1-3 minutes, post visible on public blog
   - Available at `blog.yourdomain.com/posts/{slug}`

### 8.2 Edit Published Post Flow

1. **Update Content**
   - Admin sends `PUT /posts/{id}` with changes
   - API updates markdown file in `published/` directory
   - **Deployment pipeline triggered automatically**
   - Changes propagate within minutes

### 8.3 Delete Post Flow

1. **Delete Request**
   - Admin sends `DELETE /posts/{id}`
   - API removes markdown file
   - If post was published, **deployment pipeline triggered**
   - Returns success status

2. **Verification**
   - After deployment, post no longer appears on blog
   - URLs return 404 (or redirect to home)

---

## 9. Technical Requirements

### 9.1 Cloudflare Requirements

**Pages:**
- GitHub or GitLab integration configured
- Automatic deployments enabled
- Custom domain configured
- HTTPS enforced

**Workers:**
- mTLS authentication enabled
- CA certificate uploaded
- Worker route configured for `api.yourdomain.com/admin/*`
- Environment variables for backend URL

**DNS:**
- CNAME for `blog.yourdomain.com` → Cloudflare Pages
- CNAME for `api.yourdomain.com` → Cloudflare Workers

### 9.2 Linux Server Requirements

**System:**
- Git installed and configured
- SSH key for Git repository access
- Static site generator installed
- Language runtime (Python/Go/Node.js) for API server

**Network:**
- API server listens on localhost or private IP
- Firewall blocks public access to API port
- Outbound HTTPS access for Git operations

**Storage:**
- Sufficient disk space for markdown files (minimal)
- Git repository initialized and configured
- Proper file permissions

### 9.3 Git Repository Requirements

**Platform:**
- GitHub, GitLab, or other Git hosting (free tier)
- Webhook integration with Cloudflare Pages

**Configuration:**
- Automatic deployment enabled
- Deploy branch configured (e.g., `main`)
- Build settings configured (if needed)

**Access:**
- SSH key or deploy token configured on Linux server
- Push access for automated deployments

---

## 10. Security Considerations

### 10.1 Authentication & Authorization

**mTLS Layer:**
- Only valid client certificates can access admin API
- Certificate validation at Cloudflare edge
- No authentication bypass possible

**API Layer:**
- Additional validation of certificate details
- Rate limiting on admin endpoints
- Logging of all admin operations

### 10.2 Data Protection

**In Transit:**
- HTTPS for all public traffic
- mTLS for admin traffic
- TLS for Git operations

**At Rest:**
- Markdown files stored on Linux server
- Server access controlled via SSH
- Regular backups recommended

### 10.3 Exposure Minimization

**Public Attack Surface:**
- Only static files exposed via Cloudflare Pages
- No server IPs or APIs publicly accessible
- No server software version disclosure

**Admin Attack Surface:**
- Admin API only accessible via mTLS
- API server not publicly accessible
- Worker acts as security boundary

---

## 11. Free Tier Compliance

### 11.1 Cloudflare Pages Limits

| Resource | Free Tier Limit | Expected Usage |
|----------|----------------|----------------|
| Builds/month | 500 | ~30-60 (1-2 per day) |
| Concurrent builds | 1 | 1 |
| Sites | 100 | 1 |
| Requests | Unlimited | Low (personal blog) |
| Bandwidth | Unlimited | Low (personal blog) |

**Compliance:** Well within limits for personal blog usage.

### 11.2 Cloudflare Workers Limits

| Resource | Free Tier Limit | Expected Usage |
|----------|----------------|----------------|
| Requests/day | 100,000 | <100 (admin only) |
| CPU time | 10ms/request | ~5ms (simple proxy) |
| Workers | 100 | 1 |

**Compliance:** Minimal usage for admin operations only.

### 11.3 Git Repository Limits

**GitHub Free:**
- Unlimited public/private repositories
- Unlimited collaborators for public repos
- 2,000 Actions minutes/month (if using Actions)

**GitLab Free:**
- Unlimited repositories
- 400 CI/CD minutes/month

**Compliance:** Blog content doesn't require CI/CD, just Git hosting.

---

## 12. Success Criteria

### 12.1 Functional Requirements

- [ ] Public blog accessible and loads within 3 seconds
- [ ] Admin can create, edit, publish, and delete posts via API
- [ ] Only mTLS-authenticated requests can access admin API
- [ ] Published posts appear on public blog within 5 minutes
- [ ] All operations completed without manual intervention

### 12.2 Non-Functional Requirements

- [ ] Zero monthly costs (all free tiers)
- [ ] No public traffic to Linux server
- [ ] 99.9% uptime for public blog (Cloudflare SLA)
- [ ] Markdown files stored locally as single source of truth
- [ ] Deployment pipeline succeeds >95% of the time

### 12.3 Security Requirements

- [ ] mTLS authentication working correctly
- [ ] No unauthorized access to admin API
- [ ] Client certificate validation enforced
- [ ] No sensitive data exposed in public blog
- [ ] Git repository access controlled

---

## 13. Implementation Phases

### Phase 1: Infrastructure Setup
1. Configure Cloudflare DNS records
2. Set up Git repository
3. Configure Cloudflare Pages with Git integration
4. Set up basic static site structure

### Phase 2: Admin API Development
1. Develop blog API server
2. Implement CRUD operations for posts
3. Integrate static site generator
4. Create deployment script

### Phase 3: mTLS Security
1. Generate CA and client certificates
2. Upload CA to Cloudflare
3. Develop Cloudflare Worker for mTLS validation
4. Test authentication flow

### Phase 4: Deployment Pipeline
1. Implement event-driven deployment logic
2. Test publish/unpublish/delete triggers
3. Add deployment status tracking
4. Implement error handling

### Phase 5: Testing & Refinement
1. End-to-end testing of all flows
2. Security testing of mTLS
3. Performance testing
4. Documentation

---

## 14. Future Considerations

### 14.1 Potential Enhancements

**Content Features:**
- Image upload and management
- Draft preview functionality
- Scheduled publishing
- Post versioning/history

**Admin Features:**
- Web-based admin interface (with mTLS)
- Mobile admin app
- Bulk operations
- Content analytics

**Deployment:**
- Incremental builds (only changed posts)
- Blue-green deployments
- Rollback capability
- Deployment notifications (webhook/email)

**Security:**
- Certificate revocation list (CRL)
- Multiple admin users
- Audit logging
- Intrusion detection

### 14.2 Scalability Considerations

While the current design targets low traffic, the architecture can scale:
- Cloudflare edge network handles traffic spikes
- Static content is highly cacheable
- Can add Cloudflare R2 for media storage (still free tier)
- Can implement content CDN for heavy media

---

## 15. Risk Assessment

### 15.1 Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Cloudflare free tier changes | Low | High | Monitor ToS, have migration plan |
| Git repository quota exceeded | Low | Medium | Use storage efficiently, monitor usage |
| Deployment pipeline failure | Medium | Medium | Error handling, manual fallback |
| Certificate expiration | Medium | High | Monitoring, renewal procedures |
| Linux server downtime | Low | Low | Only affects admin ops, not public blog |

### 15.2 Security Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| mTLS misconfiguration | Medium | High | Thorough testing, security review |
| Client certificate compromise | Low | High | Certificate revocation capability |
| Git repository compromise | Low | High | 2FA, SSH keys, access monitoring |
| Worker vulnerability | Low | Medium | Regular updates, security best practices |

---

## 16. Operational Considerations

### 16.1 Maintenance

**Regular Tasks:**
- Monitor deployment pipeline status
- Check Git repository size and cleanup if needed
- Review Cloudflare usage against free tier limits
- Update dependencies (site generator, API server)

**Periodic Tasks:**
- Renew client certificates before expiration
- Backup markdown files and Git repository
- Review and rotate credentials
- Update documentation

### 16.2 Monitoring

**Key Metrics:**
- Deployment success rate
- Deployment duration
- Admin API error rate
- Public blog availability
- Free tier usage percentages

**Alerting:**
- Deployment failures
- Certificate expiration warnings
- Free tier limit approaching
- Unusual API access patterns

### 16.3 Backup Strategy

**Content Backup:**
- Markdown files in Git repository (primary)
- Local backup on Linux server
- Optional: Periodic backup to cloud storage

**Configuration Backup:**
- Cloudflare Worker code in version control
- mTLS certificates stored securely
- API server configuration

---

## 17. Documentation Requirements

### 17.1 User Documentation

**Admin Guide:**
- How to create and publish posts
- How to use the API endpoints
- Certificate installation instructions
- Troubleshooting common issues

**Developer Documentation:**
- Architecture overview
- API reference
- Deployment pipeline details
- Configuration guide

### 17.2 Operational Documentation

**Runbooks:**
- Certificate renewal procedure
- Deployment failure recovery
- Git repository maintenance
- Cloudflare configuration updates

**Architecture Decision Records (ADRs):**
- Why Cloudflare Pages over alternatives
- Event-driven vs cron-based deployment
- mTLS for authentication
- Markdown as content format

---

## 18. Glossary

- **mTLS (Mutual TLS):** Authentication method where both client and server verify each other's certificates
- **CA (Certificate Authority):** Entity that issues digital certificates
- **SSG (Static Site Generator):** Tool that converts content files to static HTML
- **Edge Network:** Distributed servers close to users (Cloudflare's CDN)
- **Frontmatter:** Metadata section at the top of markdown files (YAML format)
- **Slug:** URL-friendly version of a post title

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-12-12 | System Architect | Initial PRD creation |

---

## Appendix A: Technology Alternatives

### Alternative Approaches Considered

**A.1 Cron-based Deployment**
- *Pros:* Simple, predictable
- *Cons:* Slower updates, unnecessary builds
- *Decision:* Rejected in favor of event-driven

**A.2 Cloudflare Workers KV for Storage**
- *Pros:* Fully serverless, no Linux server needed
- *Cons:* 1GB storage limit, harder to manage locally
- *Decision:* Rejected to keep content locally controlled

**A.3 Basic Auth for Admin**
- *Pros:* Simpler setup
- *Cons:* Less secure than mTLS, password management
- *Decision:* Rejected in favor of certificate-based auth

**A.4 Cloudflare R2 for Content Storage**
- *Pros:* Cloud-based, scalable
- *Cons:* Harder to edit locally, requires API for access
- *Decision:* Rejected to maintain local-first approach

---

## Appendix B: Estimated Timeline

**Assuming 10-20 hours of development time:**

| Phase | Duration | Tasks |
|-------|----------|-------|
| Infrastructure Setup | 2-3 hours | DNS, Git, Cloudflare Pages |
| Admin API Development | 4-6 hours | API server, CRUD, generator integration |
| mTLS Implementation | 2-3 hours | Certificates, Worker, validation |
| Deployment Pipeline | 2-3 hours | Event triggers, Git automation |
| Testing & Refinement | 2-4 hours | E2E testing, fixes, documentation |

**Total:** ~10-20 hours spread over 1-2 weeks

---

**END OF DOCUMENT**
