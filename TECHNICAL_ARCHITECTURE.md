# LuggageCompensation Platform - Technical Architecture

**Version:** 1.0  
**Date:** February 26, 2026  
**Status:** Pre-Development (Recommended Architecture)

---

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      CLIENT LAYER (Web/Mobile)                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  React/Next.js SPA                                       │  │
│  │  - Claim Wizard                                          │  │
│  │  - Dashboard                                             │  │
│  │  - Knowledge Base                                        │  │
│  │  - Mobile Responsive                                     │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────────────┘
                       │ HTTPS/REST API
┌──────────────────────┴──────────────────────────────────────────┐
│                    API LAYER (Backend)                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Node.js/Express Server                                  │  │
│  │  ├─ /api/auth/* (Authentication)                         │  │
│  │  ├─ /api/claims/* (Claim CRUD)                           │  │
│  │  ├─ /api/generate/* (AI Letter Generation)               │  │
│  │  ├─ /api/calculator/* (Compensation Calc)                │  │
│  │  └─ /api/help/* (Knowledge Base)                         │  │
│  └──────────────────────────────────────────────────────────┘  │
└──────┬──────────────────────────┬──────────────────┬────────────┘
       │                          │                  │
       ├─ PostgreSQL             ├─ OpenAI API      └─ Auth0/Supabase
       │  (User data,            │  (Generate
       │   Claims,               │   Letters)
       │   History)              │
       │                         └─ SendGrid
       │                            (Email)
       │
    ┌──┴─────────────────────────────┐
    │  STORAGE & SERVICES LAYER       │
    ├─────────────────────────────────┤
    │ PostgreSQL                      │
    │ ├─ users                        │
    │ ├─ claims                       │
    │ ├─ claim_items                  │
    │ ├─ airline_guidelines           │
    │ └─ knowledge_base               │
    │                                 │
    │ S3/Cloudinary (Images)          │
    │ ├─ User uploads                 │
    │ ├─ Generated PDFs                │
    │ └─ Cached artifacts              │
    │                                 │
    │ Redis (Optional)                │
    │ ├─ Session cache                │
    │ └─ Rate limiting                │
    └─────────────────────────────────┘
```

---

## Technology Stack Rationale

### Frontend: React/Next.js

**Why Next.js?**
- Built-in SSR/SSG = better SEO (important for "how to claim luggage" content)
- File-based routing = fast development
- API routes = no separate backend needed initially
- Vercel deployment = free tier, simple CI/CD
- TypeScript support = type safety

**Key Libraries:**
- `react-hook-form` - Form handling (wizard)
- `tailwindcss` - Styling (fast)
- `recharts` - Compensation calculator visualization
- `next-auth` or Auth0 - Authentication
- `react-pdf` - PDF generation

**UI/UX:**
- Mobile-first design (most traffic will be mobile + immediate need)
- Dark mode (reduce eye strain for frustrated users)
- Wizard-style form (step-by-step, not overwhelming)
- Clear visual feedback on claim status

### Backend: Node.js/Express

**Why Node.js?**
- JavaScript everywhere = faster development
- Excellent async/await = handles I/O bound tasks (DB, API calls)
- Huge npm ecosystem = quick to integrate tools
- Can handle 10K+ concurrent users without heavy infrastructure

**Why Express (not Nest.js or Fastify)?**
- Lightweight, proven, simple
- Perfect for MVP (don't over-engineer)
- Can upgrade to Nest.js later if needed

**Key Libraries:**
- `express` - Web framework
- `pg` - PostgreSQL driver
- `dotenv` - Environment management
- `cors` - Cross-origin requests
- `helmet` - Security headers
- `express-rate-limit` - API rate limiting
- `winston` - Logging
- `joi` - Input validation
- `openai` - AI integration
- `nodemailer` or `sendgrid` - Email

### Database: PostgreSQL

**Why PostgreSQL?**
- Open source, free tier on Railway/Render
- Strong ACID compliance = critical for financial/claim data
- Full-text search = good for knowledge base
- JSON support = flexible schema
- Excellent for relational data (users, claims, items, airlines)

**Schema (Initial):**

```sql
-- Users
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email VARCHAR UNIQUE NOT NULL,
  password_hash VARCHAR,
  full_name VARCHAR,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  subscription_tier VARCHAR DEFAULT 'free' -- free, premium, enterprise
);

-- Claims
CREATE TABLE claims (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  airline VARCHAR NOT NULL,
  flight_number VARCHAR,
  departure_date DATE NOT NULL,
  arrival_date DATE,
  destination_airport VARCHAR,
  claim_type VARCHAR NOT NULL, -- 'lost', 'delayed', 'damaged'
  claim_status VARCHAR DEFAULT 'draft', -- draft, submitted, responded, resolved
  estimated_compensation DECIMAL,
  actual_compensation DECIMAL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  resolved_at TIMESTAMP
);

-- Claim Items
CREATE TABLE claim_items (
  id UUID PRIMARY KEY,
  claim_id UUID REFERENCES claims(id),
  item_name VARCHAR NOT NULL,
  category VARCHAR, -- 'clothing', 'electronics', 'valuables', etc
  purchase_date DATE,
  original_value DECIMAL,
  estimated_depreciation DECIMAL,
  image_url VARCHAR,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Airline Guidelines (Static Reference)
CREATE TABLE airline_guidelines (
  id UUID PRIMARY KEY,
  airline_code VARCHAR UNIQUE,
  airline_name VARCHAR,
  compensation_limit_domestic DECIMAL,
  compensation_limit_international DECIMAL,
  excluded_items TEXT[], -- JSON array of excluded item types
  notes TEXT,
  last_updated TIMESTAMP
);

-- Knowledge Base Articles
CREATE TABLE kb_articles (
  id UUID PRIMARY KEY,
  title VARCHAR NOT NULL,
  slug VARCHAR UNIQUE,
  content TEXT,
  category VARCHAR, -- 'rights', 'tips', 'airline-specific', 'legal'
  status VARCHAR DEFAULT 'published',
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Email Logs (Optional, for analytics)
CREATE TABLE email_logs (
  id UUID PRIMARY KEY,
  user_id UUID REFERENCES users(id),
  claim_id UUID REFERENCES claims(id),
  email_type VARCHAR, -- 'claim_generated', 'follow_up', 'reminder'
  sent_at TIMESTAMP DEFAULT NOW()
);
```

### AI Integration: OpenAI API

**Why OpenAI (not Anthropic Claude)?**
- More mature production integrations
- Better fine-tuning options
- Lower latency
- Easier integration for users

**Alternative: Anthropic Claude**
- Better at nuanced writing
- Could use Claude directly if you want (your choice!)

**Use Cases:**
1. **Claim Letter Generation:** GPT-4 turbo = $0.01-0.03 per request
2. **Item Categorization:** GPT-3.5 = auto-categorize items user enters
3. **FAQ Responses:** Claude for detailed explanations

**Cost Estimate:** $0.05 per claim generated = sustainable at scale

**Prompt Engineering:**
```
You are an expert in airline luggage compensation. Help users write a 
professional claim letter.

User's luggage was [CONDITION] on [AIRLINE] flight [FLIGHT_NUM].

Items lost/damaged:
[ITEMIZED_LIST]

Jurisdiction: [US/EU/OTHER]
Compensation limit: [AMOUNT]

Generate a persuasive, factual claim letter that:
1. Cites relevant regulations
2. Itemizes losses with depreciation
3. Requests full compensation
4. Suggests compromises if needed
5. Is professional but assertive

Include placeholders for [USER_NAME], [CLAIM_DATE], [AIRLINE_ADDRESS]
```

### Authentication: Auth0 or Supabase Auth

**Why Auth0?**
- Free tier: 1,000 active users
- Handles OAuth (Google, Apple, Microsoft sign-in)
- No need to manage passwords
- Works with email/passwordless
- Easy to scale

**Why Supabase Auth?**
- Open source alternative
- Includes PostgreSQL out of the box
- JWT tokens, simple to integrate
- Lower cost at scale

**Recommendation:** Start with Auth0 free tier, migrate to Supabase Auth if needed

### Hosting & Deployment

**Frontend:** Vercel
- Free tier: unlimited deployments
- Next.js native support
- Automatic HTTPS
- CDN included
- Analytics built-in

**Backend:** Railway or Render
- Free tier: $5/month credits (enough for MVP)
- PostgreSQL included
- Environment variables management
- One-click deployments from GitHub

**Alternative:** AWS (more complex, only if needed)

**Email:** SendGrid or Mailgun
- Free tier: 100 emails/day
- Transactional email (claim confirmations, follow-ups)
- Later: scheduled reminder emails

---

## API Specification (Core Endpoints)

### Authentication
```
POST   /api/auth/signup
POST   /api/auth/login
POST   /api/auth/logout
POST   /api/auth/refresh-token
GET    /api/auth/me
```

### Claims Management
```
POST   /api/claims
GET    /api/claims              (List user's claims)
GET    /api/claims/:id          (Get single claim)
PATCH  /api/claims/:id          (Update claim)
DELETE /api/claims/:id          (Delete draft claim)
POST   /api/claims/:id/submit   (Submit to tracking)
POST   /api/claims/:id/export   (Export as PDF)
```

### AI Features
```
POST   /api/generate/letter     (Generate claim letter)
POST   /api/generate/estimate   (Get compensation estimate)
POST   /api/calculator          (Depreciation calculator)
```

### Knowledge Base
```
GET    /api/help/articles                  (List all articles)
GET    /api/help/articles/:slug            (Get single article)
GET    /api/help/airline/:airline_code     (Airline-specific info)
GET    /api/help/compensation-limits       (All limits by airline)
POST   /api/help/search                    (Search articles)
```

### User Account
```
GET    /api/user/profile
PATCH  /api/user/profile        (Update name, email, etc)
PATCH  /api/user/subscription   (Upgrade to premium)
POST   /api/user/delete         (Delete account and data)
```

---

## Development Phases

### Phase 1: MVP (Weeks 5-12) - Core Features
- [ ] User authentication (Auth0)
- [ ] Basic claim form (wizard)
- [ ] AI claim letter generation
- [ ] Simple dashboard
- [ ] Knowledge base (static pages)
- [ ] PDF export
- [ ] Mobile responsive

**Estimated Lines of Code:** 3K-5K

### Phase 2: Enhancement (Weeks 13-20) - Polish & Growth
- [ ] Claim tracking (connect to airline data via manual entry)
- [ ] Automated reminders (email)
- [ ] User testimonials
- [ ] Improved UX based on beta feedback
- [ ] Analytics (Mixpanel/GA4)
- [ ] SEO optimization (blog content)

### Phase 3: Scale (Months 6+)
- [ ] Insurance partnership integration
- [ ] Advanced claim analytics
- [ ] Lawyer referral network
- [ ] Mobile app
- [ ] Multi-language support

---

## Security Considerations

**Data Protection:**
- All passwords hashed (bcrypt, not stored)
- HTTPS/TLS only
- PII encrypted at rest (claim contents)
- GDPR-compliant data deletion

**API Security:**
- Rate limiting (100 requests/minute per user)
- Input validation (Joi)
- CSRF tokens
- Helmet.js security headers
- Secrets in env vars (never in code)

**Compliance:**
- Terms of Service (review with lawyer)
- Privacy Policy (GDPR, CCPA)
- No claim of legal advice (disclaimer on every page)
- Data retention policy (delete old claims after 2 years)

---

## Scalability Assumptions

**MVP Targets:** 50-200 concurrent users
**Year 1 Target:** 10,000 users
**Database:** PostgreSQL can handle 100K+ queries/second
**API:** Node.js can handle 10K requests/second on single process
**If needed, scale with:** PM2 process manager, horizontal load balancer

---

## Development Timeline (Estimated)

| Task | Duration | Owner | Status |
|------|----------|-------|--------|
| Setup project repo & CI/CD | 0.5 days | Ignatius | Pending |
| Frontend scaffold (Next.js) | 1 day | Ignatius | Pending |
| Backend scaffold (Express) | 1 day | Ignatius | Pending |
| Auth setup | 1 day | Ignatius | Pending |
| Claim form & wizard UI | 3 days | Ignatius | Pending |
| Claim form backend | 2 days | Ignatius | Pending |
| AI letter generation | 2 days | Ignatius | Pending |
| Calculator & compensation logic | 2 days | Ignatius | Pending |
| Dashboard & claim tracking | 2 days | Ignatius | Pending |
| Knowledge base pages | 2 days | Ignatius | Pending |
| PDF generation | 1 day | Ignatius | Pending |
| Testing & bug fixes | 3 days | Ignatius | Pending |
| Deployment setup | 1 day | Ignatius | Pending |
| Beta testing & iteration | 2 weeks | Ignatius + Beta Users | Pending |
| **TOTAL** | **~35 working days** | | |

**Assumptions:** Full-time work, no major blocking issues

---

## Monitoring & Analytics

**What to Track:**
- Form completion rate (goal: >80%)
- Claim submission rate
- User retention (week 1, month 1)
- Email open rates
- AI letter quality (user feedback)
- Feature usage (which claims types are most common)

**Tools:**
- Sentry (error tracking)
- Mixpanel or Plausible (analytics)
- Google Analytics (traffic)
- Datadog/New Relic (infrastructure)

---

## Tech Stack Summary

| Layer | Technology | Cost | Reasoning |
|-------|-----------|------|-----------|
| Frontend | Next.js + React | Free (OSS) | Fast development, SEO, Vercel integration |
| Backend | Node.js + Express | Free (OSS) | Quick prototyping, JavaScript ecosystem |
| Database | PostgreSQL | Free tier available | ACID compliance, scalable, cheap |
| Hosting | Vercel + Railway | Free/low cost | Simple, integrated, grow as needed |
| Auth | Auth0 or Supabase | Free/cheap | Handles complexity, let experts manage it |
| AI | OpenAI API | Pay per use (~$0.05/claim) | Proven, easy integration |
| Email | SendGrid | Free tier available | Reliable, good deliverability |
| Monitoring | Sentry | Free tier available | Catch errors before users do |
| Analytics | Mixpanel | Free tier available | Understand user behavior |

**Estimated Monthly Cost (MVP):** $20-50 (mostly AI usage)

---

## Known Limitations & Future Work

**MVP Limitations:**
- Manual claim submission (not automated filing)
- No real-time airline integration
- No email parsing of airline responses
- US/EU only initially

**Future Enhancements:**
- Lawyer network (referral for complex cases)
- Insurance company integrations
- Automated email parsing (AI reads airline responses)
- Real-time baggage tracking integration
- Mobile app
- Multilingual support
- Blockchain-based claim verification

---

## Questions for Development

1. **Frontend:** React or Vue? (Recommend React for ecosystem)
2. **Styling:** Tailwind or styled-components? (Recommend Tailwind for speed)
3. **AI Model:** GPT-4 turbo or Claude? (Either works)
4. **Database:** PostgreSQL or MongoDB? (PostgreSQL for relational claims data)
5. **Timeline:** Full-time vs part-time development? (Affects feature velocity)

---

**Version:** 1.0  
**Last Updated:** Feb 26, 2026  
**Next Review:** Week 5 (Start of development)
