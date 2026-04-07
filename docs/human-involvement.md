# Actions Required from User (Human-in-the-Loop)

> 🤖 **IMPORTANT NOTE FOR AI:**
> Before executing specific milestones, check this file and prompt the user to provide the necessary inputs listed below. You cannot proceed with certain implementations until this information is supplied.

---

This document outlines everything an AI *cannot* do and requires human involvement. Treat this as your checklist for the project.

## 1. Secrets & Credentials
*The `.env` file contains placeholders. An AI cannot generate real credentials.*

- [ ] **OpenAI API Key**: Obtain from OpenAI platform and paste into `.env`.
- [ ] **FluentCRM API Credentials**: Obtain `API_URL`, `API_KEY`, and `API_SECRET` from WordPress admin panel and paste into `.env`.
- [ ] **Webhook Bearer Token**: Generate a secure random string and paste it into `.env`. You must also configure the Fluent Forms webhook inside WordPress to use this token.
- [ ] **Payment Provider Details**: Register with Stripe/PayPal. Obtain the Webhook Signing Secret and Secret Key and paste into `.env`.
- [ ] **DigitalOcean Spaces API Keys**: Create a space, generate access & secret keys, and paste into `.env`. Needs `BUCKET`, `REGION`, `ENDPOINT`, `KEY`, and `SECRET`.
- [ ] **Admin Auth Secrets**: Generate a strong signature for `JWT_SECRET` and set the default admin password in `.env`.

## 2. Client Assets & Domain Specifics
*Information pending from the client.*

- [ ] **SafeRelate Brand Assets**: Provide the logo image (transparent PNG or SVG), primary/secondary hex color codes, and any specific fonts.
- [ ] **Assessment Questions & Scoring Logic**: Provide the actual set of questions, how they map to dimensions (subscales), the weights of each question, reverse-scored items, the logic for score bands, and conditional logic for recommendations.
- [ ] **Multi-Brand Requirements**: Define the exact number of brands going live at launch.
- [ ] **Report URL Expiry**: Confirm the preferred expiration duration for signed PDF links (e.g., 7 days or 30 days).

## 3. Operations & Infrastructure
*Tasks requiring account access.*

- [ ] **Domain Setup**: Purchase/Configure a domain (e.g., `api.saferelate.com`) and point its DNS records to the DigitalOcean Droplet IP address.
- [ ] **WordPress Configuration**: Go into the WordPress admin panel and configure Fluent Forms to fire a POST webhook to the new API endpoint whenever the assessment is submitted, specifying the Bearer token in the headers.

## 4. Business Approvals

- [ ] **Prompts & Models**: Review the default narratives and approve the active prompt versions in the Admin Portal.
- [ ] **Brand Switching**: Admin must manually switch or activate brands in the production database.
