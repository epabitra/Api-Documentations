# SendGrid Setup Guide for Heart Thrive

This guide will walk you through creating a new SendGrid account and connecting it to your Heart Thrive application.

## Prerequisites

- A valid email address (different from the one that hit the limit)
- Access to the email inbox for verification
- Access to your application configuration files

---

## Step 1: Create a New SendGrid Account

1. **Visit SendGrid Website**
   - Go to [https://sendgrid.com](https://sendgrid.com)
   - Click on **"Start for Free"** or **"Sign Up"** button

2. **Sign Up Process**
   - Enter your email address (use a different email than the one that exceeded limits)
   - Create a strong password
   - Fill in your details (Name, Company, etc.)
   - Accept the terms and conditions
   - Click **"Create Account"**

3. **Email Verification**
   - Check your email inbox for a verification email from SendGrid
   - Click the verification link to activate your account

4. **Complete Account Setup**
   - You may be asked to provide additional information about your use case
   - Select appropriate options (e.g., "Transactional Email", "Marketing Emails", etc.)
   - Complete any required onboarding steps

---

## Step 2: Verify Your Sender Identity

SendGrid requires you to verify your sender email address before you can send emails.

### Option A: Single Sender Verification (Recommended for Development)

1. **Navigate to Settings**
   - Log into your SendGrid dashboard
   - Go to **Settings** → **Sender Authentication** (or **Email API** → **Sender Authentication**)

2. **Verify a Single Sender**
   - Click on **"Verify a Single Sender"** or **"Create a Sender"**
   - Fill in the required information:
     - **From Email Address**: Enter the email you want to use (e.g., `yourname@vidyayug.com`)
     - **From Name**: Enter "Heart Thrive" (or your preferred sender name)
     - **Reply To**: Same as From Email Address
     - **Company Address**: Your company address
     - **City, State, Zip Code**: Your location details
     - **Country**: Select your country
   - Click **"Create"**

3. **Verify Email Address**
   - SendGrid will send a verification email to the address you specified
   - **Important**: Check the inbox of the email address you entered (not your SendGrid account email)
   - Click the verification link in that email
   - Your sender will be marked as "Verified" once confirmed

### Option B: Domain Authentication (Recommended for Production)

If you own a domain (e.g., `vidyayug.com`), you can authenticate the entire domain:

1. **Navigate to Domain Authentication**
   - Go to **Settings** → **Sender Authentication** → **Authenticate Your Domain**

2. **Add Your Domain**
   - Enter your domain name (e.g., `vidyayug.com`)
   - Select your DNS host (e.g., GoDaddy, Cloudflare, etc.)
   - Click **"Next"**

3. **Add DNS Records**
   - SendGrid will provide you with DNS records (CNAME records)
   - Log into your domain's DNS management panel
   - Add the provided CNAME records to your domain's DNS settings
   - Wait for DNS propagation (can take a few minutes to 48 hours)
   - Click **"Verify"** in SendGrid dashboard once DNS records are added

4. **Link Subdomain (Optional)**
   - You may need to link a subdomain for link branding
   - Follow the same process for the subdomain

---

## Step 3: Create an API Key

1. **Navigate to API Keys**
   - In SendGrid dashboard, go to **Settings** → **API Keys**
   - Or click on your profile icon → **API Keys**

2. **Create New API Key**
   - Click **"Create API Key"** button
   - Enter a name for your API key (e.g., "Heart Thrive Dev" or "Heart Thrive Production")
   - Select **API Key Permissions**:
     - For full access: Select **"Full Access"**
     - For restricted access: Select **"Restricted Access"** and enable only:
       - **Mail Send** (required)
       - **Stats** (optional, for monitoring)
   - Click **"Create & View"**

3. **Copy Your API Key**
   - **IMPORTANT**: Copy the API key immediately and save it securely
   - The API key will be shown only once and cannot be retrieved later
   - Format: `SG.xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
   - Store it in a secure location (password manager, secure notes, etc.)

---

## Step 4: Update Application Configuration

Now you need to update your application configuration files with the new SendGrid credentials.

### For Development Environment

1. **Open the configuration file**
   - Navigate to: `src/main/resources/config/application-dev.yml`

2. **Update SendGrid Configuration**
   - Find the `sendgrid:` section (around line 62)
   - Update the following values:
     ```yaml
     sendgrid:
       api-key: SG.YOUR_NEW_API_KEY_HERE   # Replace with your new API key
       from-email: your-verified-email@vidyayug.com   # Replace with your verified sender email
       from-name: Heart Thrive
     ```

3. **Comment out or remove the old configuration**
   - You can comment out the old API key for reference:
     ```yaml
     # Old SendGrid configuration (exceeded limit)
     #sendgrid:
     #  api-key: SG.YOUR_NEW_API_KEY_HERE
     #  from-email: epabitra@vidyayug.com
     #  from-name: Heart Thrive
     ```

### For Production Environment

1. **Open the production configuration file**
   - Navigate to: `src/main/resources/config/application-prod.yml`

2. **Update SendGrid Configuration**
   - Find the `sendgrid:` section (around line 57)
   - Update with the same new credentials:
     ```yaml
     sendgrid:
       api-key: SG.YOUR_NEW_API_KEY_HERE   # Replace with your new API key
       from-email: your-verified-email@vidyayug.com   # Replace with your verified sender email
       from-name: Heart Thrive
     ```

### Security Best Practice (Optional but Recommended)

For better security, consider using environment variables instead of hardcoding the API key:

1. **Set Environment Variable**
   - Windows PowerShell:
     ```powershell
     $env:SENDGRID_API_KEY="SG.YOUR_NEW_API_KEY_HERE"
     ```
   - Windows CMD:
     ```cmd
     set SENDGRID_API_KEY=SG.YOUR_NEW_API_KEY_HERE
     ```
   - Linux/Mac:
     ```bash
     export SENDGRID_API_KEY="SG.YOUR_NEW_API_KEY_HERE"
     ```

2. **Update Configuration File**
   - In `application-dev.yml` and `application-prod.yml`:
     ```yaml
     sendgrid:
       api-key: ${SENDGRID_API_KEY}   # Load from environment variable
       from-email: your-verified-email@vidyayug.com
       from-name: Heart Thrive
     ```

---

## Step 5: Test the Configuration

1. **Restart Your Application**
   - Stop your Spring Boot application if it's running
   - Start it again to load the new configuration

2. **Test Email Sending**
   - Trigger an email from your application (e.g., OTP email, password reset, etc.)
   - Check the application logs for any errors
   - Verify that the email is received successfully

3. **Check SendGrid Dashboard**
   - Log into SendGrid dashboard
   - Go to **Activity** → **Email Activity**
   - You should see your test email in the activity feed
   - Check the status (should show "Delivered" for successful sends)

---

## Step 6: Monitor Your Usage

1. **Check Email Statistics**
   - In SendGrid dashboard, go to **Stats** → **Overview**
   - Monitor your daily email sending volume
   - Free tier typically allows 100 emails per day

2. **Set Up Alerts (Optional)**
   - Go to **Settings** → **Alerts**
   - Set up alerts for:
     - Email volume approaching limits
     - API key usage
     - Bounce rates
     - Spam reports

---

## Troubleshooting

### Issue: "Invalid API Key" Error

**Solution:**
- Verify that you copied the entire API key correctly (it should start with `SG.`)
- Check for any extra spaces or line breaks
- Ensure the API key has the correct permissions (Mail Send)

### Issue: "Sender Not Verified" Error

**Solution:**
- Go to **Settings** → **Sender Authentication**
- Verify that your sender email shows as "Verified"
- If not verified, check your email inbox and click the verification link
- Wait a few minutes after verification before testing

### Issue: "Maximum Credits Exceeded" Error

**Solution:**
- Check your SendGrid dashboard → **Stats** → **Overview**
- Verify your daily/monthly email limits
- Free tier: 100 emails/day
- Consider upgrading to a paid plan if you need more volume

### Issue: Emails Not Being Received

**Solution:**
- Check **Activity** → **Email Activity** in SendGrid dashboard
- Look for bounce or block reasons
- Verify recipient email addresses are valid
- Check spam/junk folders
- Ensure your sender reputation is good

### Issue: Configuration Not Loading

**Solution:**
- Verify YAML syntax (indentation, colons, etc.)
- Ensure you're using the correct profile (dev/prod)
- Restart the application after configuration changes
- Check application logs for configuration errors

---

## SendGrid Plan Limits

### Free Tier
- **100 emails per day** (forever)
- Single sender verification
- Basic email analytics
- API access

### Essentials Plan (Paid)
- Starts at $19.95/month
- 50,000 emails/month
- Domain authentication
- Advanced analytics
- Email API access

### Pro Plan (Paid)
- Starts at $89.95/month
- 100,000 emails/month
- Advanced features
- Dedicated IP option

**Note:** If you're consistently hitting the free tier limit, consider upgrading to a paid plan or implementing email queuing/throttling in your application.

---

## Additional Resources

- [SendGrid Documentation](https://docs.sendgrid.com/)
- [SendGrid API Reference](https://docs.sendgrid.com/api-reference)
- [SendGrid Best Practices](https://docs.sendgrid.com/for-developers/sending-email/best-practices)
- [SendGrid Support](https://support.sendgrid.com/)

---

## Quick Checklist

- [ ] Created new SendGrid account
- [ ] Verified email address
- [ ] Created API key and saved it securely
- [ ] Updated `application-dev.yml` with new credentials
- [ ] Updated `application-prod.yml` with new credentials
- [ ] Tested email sending functionality
- [ ] Verified emails are being delivered
- [ ] Monitored SendGrid dashboard for activity
- [ ] Set up usage alerts (optional)

---

**Important Notes:**
- Keep your API key secure and never commit it to version control if using environment variables
- Monitor your email sending volume to avoid hitting limits
- Consider implementing email queuing if you send high volumes
- Keep your sender reputation high by following email best practices

