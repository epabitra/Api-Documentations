# Parking Management System
## Complete User Documentation

**Version:** 1.0.0  
**Last Updated:** January 2025  
**Documentation Type:** User Guide & Technical Reference

---

## Table of Contents

1. [Introduction](#introduction)
2. [System Overview](#system-overview)
3. [Getting Started](#getting-started)
4. [User Roles and Permissions](#user-roles-and-permissions)
5. [Core Features](#core-features)
6. [Detailed Feature Guides](#detailed-feature-guides)
7. [Multi-Company Architecture](#multi-company-architecture)
8. [Technical Specifications](#technical-specifications)
9. [Troubleshooting](#troubleshooting)
10. [Best Practices](#best-practices)
11. [Appendix](#appendix)

---

## 1. Introduction

### 1.1 What is the Parking Management System?

The Parking Management System is a comprehensive, cloud-based solution designed to streamline vehicle parking operations for parking centers, commercial establishments, and multi-location businesses. This modern web application enables efficient tracking, registration, and discharge of vehicles with advanced security features, multi-company support, and real-time analytics.

### 1.2 Key Benefits

- **Efficient Vehicle Tracking:** Real-time monitoring of parked and discharged vehicles
- **Multi-Company Support:** Manage multiple parking centers from a single platform
- **Secure Verification:** Multiple discharge verification methods (OTP, Image, Manual)
- **Mobile-Friendly:** Responsive design works seamlessly on mobile, tablet, and desktop
- **Image Management:** Capture and store vehicle images using device cameras
- **Real-Time Analytics:** Comprehensive dashboard with statistics and insights
- **Role-Based Access:** Granular permissions for different user types
- **Customer History:** Track complete parking history for each customer

### 1.3 System Requirements

**For Users:**
- Modern web browser (Chrome, Firefox, Safari, Edge - latest versions)
- Internet connection
- JavaScript enabled
- Mobile devices: iOS 12+ or Android 8+

**For Administrators:**
- Google account (for Google Apps Script backend)
- Google Sheets (for data storage)
- Firebase account (for image storage - optional)
- WhatsApp Business API or SMS service (for OTP delivery - optional)

---

## 2. System Overview

### 2.1 Architecture

The Parking Management System follows a modern, scalable architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                    Frontend (React)                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Web App    │  │  Mobile Web  │  │  Desktop Web │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ HTTPS
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              Google Apps Script (Backend API)                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Authentication│  │ Business Logic│  │ Data Validation│    │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ Google Sheets│  │ Firebase     │  │ WhatsApp/SMS │
│  (Database)  │  │  (Storage)   │  │   (OTP)      │
└──────────────┘  └──────────────┘  └──────────────┘
```

### 2.2 Key Components

1. **Frontend Application (React)**
   - User interface and user experience
   - Client-side routing and navigation
   - Form validation and data presentation
   - Image capture and upload

2. **Backend API (Google Apps Script)**
   - Authentication and authorization
   - Business logic and data processing
   - Integration with Google Sheets
   - OTP generation and verification

3. **Data Storage**
   - Google Sheets: Primary database for vehicles, employees, companies
   - Firebase Storage: Image storage for vehicle photos

4. **External Services**
   - WhatsApp Business API: OTP delivery
   - SMS Gateway: OTP fallback option

### 2.3 Data Flow

1. User performs an action in the frontend
2. Frontend sends authenticated request to Google Apps Script API
3. Backend validates request and processes business logic
4. Data is stored/retrieved from Google Sheets
5. Response is sent back to frontend
6. Frontend updates UI with new data

---

## 3. Getting Started

### 3.1 First-Time Access

#### For Company Administrators

1. **Navigate to Home Page**
   - Open the application URL in your web browser
   - You'll see the public landing page

2. **Register Your Company**
   - Click "Register Company" button
   - Fill in company details:
     - Company Name (required)
     - Address (optional)
     - Contact Information (optional)
   - Fill in Admin User details:
     - Full Name (required)
     - Email Address (required - will be your login)
     - Password (required - minimum 8 characters)
     - Mobile Number (required)
     - Timezone (required - select your timezone)
   - Click "Register Company"
   - You'll be automatically redirected to the login page

3. **Login to Your Account**
   - Enter your email address
   - Enter your password
   - Click "Sign In"
   - You'll be redirected to the Dashboard

#### For Employees

1. **Receive Credentials**
   - Your company administrator will create your account
   - You'll receive your email and temporary password

2. **First Login**
   - Navigate to the login page
   - Enter your email and password
   - If password change is required, you'll be prompted to change it
   - After changing password, you'll access the dashboard

### 3.2 Navigation Overview

The application uses a sidebar navigation menu with the following sections:

- **Dashboard:** Overview of statistics and quick actions
- **Register Vehicle:** Add new vehicles to the system
- **Vehicle List:** View and manage all vehicles
- **Delivery Request:** Discharge vehicles with verification
- **Employees:** Manage team members (Admin only)
- **Companies:** Manage companies (Super Admin only)
- **Change Password:** Update your password

### 3.3 Understanding the Interface

**Sidebar Navigation:**
- Left side of the screen (desktop)
- Collapsible menu for more screen space
- Mobile-friendly hamburger menu

**Main Content Area:**
- Displays the current page content
- Responsive design adapts to screen size

**Header/Company Name:**
- Shows your company name at the top of the sidebar
- Truncates with ellipsis if name is too long
- Hover to see full company name

**User Profile:**
- Bottom of sidebar shows your profile
- Displays name, email, and company information

---

## 4. User Roles and Permissions

### 4.1 Role Types

The system supports three types of users:

#### 4.1.1 Super Admin
- **Access Level:** Global access to all companies
- **Capabilities:**
  - View and manage all companies
  - Create, edit, and delete companies
  - Access all company data
  - Register vehicles for any company
  - Discharge vehicles from any company
  - View cross-company analytics
  - Manage employees across all companies

#### 4.1.2 Company Admin
- **Access Level:** Full access within their company
- **Capabilities:**
  - Register vehicles for their company
  - Discharge vehicles from their company
  - View all vehicles in their company
  - Manage employees within their company
  - View company-specific analytics
  - Edit company information (if permitted)

#### 4.1.3 Employee
- **Access Level:** Standard user within their company
- **Capabilities:**
  - Register vehicles for their company
  - Discharge vehicles from their company
  - View vehicles in their company
  - View company-specific analytics
  - Cannot manage employees or company settings

### 4.2 Permission Matrix

| Feature | Super Admin | Company Admin | Employee |
|---------|------------|--------------|----------|
| View Dashboard | ✅ | ✅ | ✅ |
| Register Vehicle | ✅ (any company) | ✅ (own company) | ✅ (own company) |
| View Vehicles | ✅ (all companies) | ✅ (own company) | ✅ (own company) |
| Edit Vehicle | ✅ (all companies) | ✅ (own company) | ✅ (own company) |
| Discharge Vehicle | ✅ (any company) | ✅ (own company) | ✅ (own company) |
| Manage Employees | ✅ (all companies) | ✅ (own company) | ❌ |
| Manage Companies | ✅ | ❌ | ❌ |
| View Analytics | ✅ (all companies) | ✅ (own company) | ✅ (own company) |
| Change Password | ✅ | ✅ | ✅ |

---

## 5. Core Features

### 5.1 Vehicle Registration

Register new vehicles entering your parking facility with complete customer and vehicle information.

**Key Features:**
- Capture vehicle images using device camera or file upload
- Store customer contact information
- Automatic timestamp recording
- Company-specific data isolation
- Bulk registration support

### 5.2 Vehicle Discharge

Securely discharge vehicles with multiple verification methods to ensure authorized release.

**Verification Methods:**
1. **OTP Verification:** Send OTP to customer's mobile number
2. **Image Verification:** Compare registered vehicle image with current vehicle
3. **Manual Discharge:** Direct discharge without verification (for trusted scenarios)

**Additional Features:**
- Optional discharge image capture (photo of person taking the vehicle)
- Batch discharge support (multiple vehicles at once)
- Automatic status update

### 5.3 Vehicle Management

Comprehensive vehicle tracking and management capabilities.

**Features:**
- View all vehicles with filtering and search
- Edit vehicle information
- View vehicle images in modal popups
- Track parking history per customer
- Server-side pagination for large datasets
- Sort by various fields
- Date range filtering

### 5.4 Dashboard and Analytics

Real-time insights into your parking operations.

**Statistics Provided:**
- Today's registrations, discharges, and pending vehicles
- Weekly statistics
- Monthly statistics
- Total counts
- Company-specific data (for regular users)
- Cross-company data (for super admins)

### 5.5 Employee Management

Manage your team members with role-based access control.

**Features:**
- Create new employee accounts
- Edit employee information
- Deactivate/delete employees
- Assign roles (Admin or Employee)
- Company-specific employee management

### 5.6 Company Management (Super Admin Only)

Manage multiple companies from a single interface.

**Features:**
- Create new companies
- Edit company information
- View all companies
- Manage company status (active/inactive)
- View company-specific statistics

### 5.7 Customer History

Track complete parking history for individual customers.

**Features:**
- View all parking records for a mobile number
- Company-specific history for regular users
- Cross-company history for super admins
- Summary statistics
- Detailed vehicle records

---

## 6. Detailed Feature Guides

### 6.1 Vehicle Registration

#### 6.1.1 Accessing Vehicle Registration

1. Log in to your account
2. Click "Register Vehicle" in the sidebar menu
3. The registration form will appear

#### 6.1.2 Filling the Registration Form

**Required Fields:**
- **Vehicle Number:** Enter the vehicle registration number
  - Format: Alphanumeric (e.g., "ABC1234", "MH12AB1234")
  - Maximum 20 characters
- **Customer Name:** Full name of the vehicle owner/customer
  - Maximum 100 characters
- **Mobile Number:** Customer's 10-digit mobile number
  - Format: 10 digits (e.g., "9876543210")
  - Used for OTP verification during discharge

**Optional Fields:**
- **Address:** Customer's address (optional)
  - Maximum 200 characters
- **Vehicle Image:** Photo of the vehicle
  - Can be uploaded from files or captured using camera
  - Maximum file size: 5MB
  - Supported formats: JPEG, PNG, GIF, WebP

**For Super Admins:**
- **Company Selection:** Select which company this vehicle belongs to
  - Required field for super admins
  - Regular users automatically use their company

#### 6.1.3 Adding Vehicle Image

**Option 1: Upload from Files**
1. Click "📁 Upload from Files" button
2. Select an image file from your device
3. Wait for upload to complete
4. Preview will appear below

**Option 2: Capture from Camera**
1. Click "📷 Capture from Camera" button
2. Allow camera access when prompted
3. Position the vehicle in the camera view
4. Click "📷 Capture Photo"
5. Image will be automatically uploaded
6. Wait for upload to complete

**Tips:**
- Ensure good lighting for clear images
- Capture the full vehicle, especially license plate
- Images are stored securely in Firebase Storage
- You can change the image before submitting

#### 6.1.4 Submitting the Form

1. Review all entered information
2. Verify the vehicle image (if added)
3. Click "Register Vehicle" button
4. Wait for confirmation message
5. Vehicle will be registered with status "Parked"
6. You'll be redirected to the Vehicle List page

**What Happens:**
- Vehicle record is created in the system
- Status is set to "Parked"
- Registration timestamp is recorded
- Company association is established
- Image is stored in Firebase (if provided)

### 6.2 Vehicle Discharge

#### 6.2.1 Accessing Delivery Request

1. Log in to your account
2. Click "Delivery Request" in the sidebar menu
3. The delivery/discharge interface will appear

#### 6.2.2 Selecting Vehicles

**Filtering Vehicles:**
- Use filters to find vehicles:
  - Vehicle Number
  - Mobile Number
  - Date Range (From Date and To Date)
  - Company (Super Admin only)
- Click "Apply Filters" to search
- Click "Clear All Filters" to reset

**Selecting Vehicles:**
1. Browse the filtered list of parked vehicles
2. Check the checkbox next to vehicles you want to discharge
3. You can select multiple vehicles for batch discharge
4. Selected vehicles will be highlighted

#### 6.2.3 Choosing Verification Method

Three verification methods are available:

**Method 1: OTP Verification**
- Most secure method
- OTP is sent to customer's registered mobile number
- Customer provides OTP for verification
- Steps:
  1. Select "OTP" as verification method
  2. Enter customer's mobile number
  3. Click "Send OTP"
  4. Wait for OTP to be delivered (via WhatsApp or SMS)
  5. Enter the 6-digit OTP received
  6. Click "Verify OTP"
  7. Once verified, proceed to discharge

**Method 2: Image Verification**
- Visual comparison method
- Compare registered vehicle image with current vehicle
- Steps:
  1. Select "Image" as verification method
  2. View the registered vehicle image(s)
  3. Verify that the current vehicle matches the image
  4. Click "Verify Image"
  5. Once verified, proceed to discharge

**Method 3: Manual Discharge**
- Direct discharge without verification
- Use for trusted scenarios or when verification is not possible
- Steps:
  1. Select "Manual" as verification method
  2. Proceed directly to discharge (no verification needed)

#### 6.2.4 Adding Discharge Image (Optional)

You can capture a photo of the person taking the vehicle:

1. Click "📁 Upload from Files" or "📷 Capture from Camera"
2. Upload or capture the image
3. Image will be associated with the discharge record
4. This helps track who collected the vehicle

#### 6.2.5 Completing Discharge

1. Ensure verification is complete (if using OTP or Image method)
2. Review selected vehicles
3. Add discharge image if needed (optional)
4. Click "Discharge Vehicle(s)" button
5. Confirm the action
6. Wait for success message
7. Vehicles will be marked as "Discharged"
8. Discharge timestamp will be recorded

**What Happens:**
- Vehicle status changes from "Parked" to "Discharged"
- Discharge timestamp is recorded
- Verification method is logged
- Discharge image is stored (if provided)
- Vehicle is removed from active parking list

### 6.3 Vehicle List and Management

#### 6.3.1 Accessing Vehicle List

1. Log in to your account
2. Click "Vehicle List" in the sidebar menu
3. All vehicles will be displayed

#### 6.3.2 Understanding the Vehicle List

**Columns Displayed:**
- **Vehicle Number:** Registration number
- **Customer Name:** Owner's name
- **Mobile Number:** Contact number
- **Status:** Parked or Discharged
- **Registered Image:** Thumbnail of vehicle photo (click to view full image)
- **Discharge Image:** Thumbnail of discharge photo (if available, click to view)
- **Registered At:** Registration date and time
- **Discharged At:** Discharge date and time (if discharged)
- **Actions:** Edit button

#### 6.3.3 Filtering and Searching

**Available Filters:**
- **Vehicle Number:** Search by registration number
- **Mobile Number:** Search by customer mobile number
- **Name:** Search by customer name
- **Date Range:** Filter by registration date
  - From Date: Start date
  - To Date: End date
- **Company:** Filter by company (Super Admin only)

**How to Filter:**
1. Enter search terms in filter fields
2. Select date range if needed
3. Click "Apply Filters" button
4. Results will update automatically

**Filter Behavior:**
- Vehicle Number, Mobile Number, and Name filters work on blur (when you click away) or Enter key
- Date filters apply automatically when selected
- Mobile number filter shows customer parking history

#### 6.3.4 Pagination

**Pagination Controls:**
- **Page Size:** Select how many records per page (default: 20)
  - Options: 10, 20, 50, 100
- **Page Navigation:**
  - Previous button: Go to previous page
  - Next button: Go to next page
  - Page indicator: Shows current page and total pages

**How to Use:**
1. Select desired page size from dropdown
2. Use Previous/Next buttons to navigate
3. Page information shows: "Page X of Y"

#### 6.3.5 Sorting

**Sortable Fields:**
- Registration Date (default)
- Vehicle Number
- Customer Name
- Status

**How to Sort:**
1. Click column header to sort
2. Click again to reverse sort order
3. Sort indicator shows current sort field and direction

#### 6.3.6 Viewing Images

**Registered Vehicle Image:**
1. Click on the vehicle image thumbnail
2. Full-size image opens in a modal popup
3. Click outside or close button to dismiss

**Discharge Image:**
1. Click on the discharge image thumbnail (if available)
2. Full-size image opens in a modal popup
3. Click outside or close button to dismiss

#### 6.3.7 Customer History (Mobile Number Filter)

When you filter by mobile number:
- A customer history card appears
- Shows summary statistics:
  - Total parkings
  - Total discharged
  - Currently parked
- For Super Admins: Shows company-wise breakdown
- For Regular Users: Shows only their company's data
- Lists all vehicles for that customer

#### 6.3.8 Editing Vehicles

1. Click "Edit" button in the Actions column
2. Edit form opens with pre-filled data
3. Modify any field (except vehicle number)
4. Update vehicle image if needed
5. Click "Update Vehicle" to save changes
6. Changes are immediately reflected in the list

**Note:** Vehicle number cannot be changed after registration.

### 6.4 Dashboard

#### 6.4.1 Accessing Dashboard

1. Log in to your account
2. Dashboard is the default landing page
3. Or click "Dashboard" in the sidebar menu

#### 6.4.2 Dashboard Overview

The dashboard provides a comprehensive overview of your parking operations:

**Statistics Cards:**
- **Today's Statistics:**
  - Registered: Vehicles registered today
  - Discharged: Vehicles discharged today
  - Pending: Currently parked vehicles

- **This Week:**
  - Registered: Vehicles registered this week
  - Discharged: Vehicles discharged this week

- **This Month:**
  - Registered: Vehicles registered this month
  - Discharged: Vehicles discharged this month

- **Total:**
  - Registered: All-time registered vehicles
  - Discharged: All-time discharged vehicles
  - Pending: Currently parked vehicles

**Quick Actions:**
- Links to common tasks:
  - Register Vehicle
  - View Vehicles
  - Delivery Request

#### 6.4.3 Understanding Statistics

**Time Zones:**
- All timestamps are displayed in your timezone
- Statistics are calculated based on your timezone
- Set your timezone during registration or in profile

**Company-Specific Data:**
- Regular users see only their company's statistics
- Super admins see aggregated data from all companies
- Company name is displayed in the dashboard header

### 6.5 Employee Management

#### 6.5.1 Accessing Employee Management

**Note:** Only Company Admins and Super Admins can access this feature.

1. Log in as Admin
2. Click "Employees" in the sidebar menu
3. Employee list will be displayed

#### 6.5.2 Viewing Employees

**Employee List Shows:**
- Employee Name
- Email Address
- Role (Admin or Employee)
- Company Name
- Status (Active/Inactive)
- Actions (Edit/Delete)

**Filtering:**
- View all employees in your company
- Super admins can see employees from all companies

#### 6.5.3 Adding New Employees

1. Click "Add Employee" button
2. Fill in the form:
   - **Name:** Full name (required)
   - **Email:** Email address (required, must be unique)
   - **Password:** Initial password (required, minimum 8 characters)
   - **Mobile:** Mobile number (optional)
   - **Role:** Select Admin or Employee (required)
   - **Timezone:** Select timezone (required)
   - **Company:** Select company (Super Admin only, auto-filled for regular admins)
3. Click "Create Employee"
4. Employee account will be created
5. Employee can log in with provided credentials

**Important:**
- Employee will be prompted to change password on first login
- Email must be unique across the system
- Password should be strong (minimum 8 characters)

#### 6.5.4 Editing Employees

1. Click "Edit" button next to an employee
2. Modify any field (except email)
3. Update role if needed
4. Click "Update Employee" to save
5. Changes take effect immediately

**Note:** Email address cannot be changed after creation.

#### 6.5.5 Deleting Employees

1. Click "Delete" button next to an employee
2. Confirm the deletion
3. Employee account will be removed
4. Employee will no longer be able to log in

**Warning:** This action cannot be undone. Ensure the employee no longer needs access.

### 6.6 Company Management (Super Admin Only)

#### 6.6.1 Accessing Company Management

**Note:** Only Super Admins can access this feature.

1. Log in as Super Admin
2. Click "Companies" in the sidebar menu
3. Company list will be displayed

#### 6.6.2 Viewing Companies

**Company List Shows:**
- Company Name
- Address
- Contact Information
- Admin Email
- Status (Active/Inactive)
- Created Date
- Actions (Edit/Delete)

#### 6.6.3 Adding New Companies

1. Click "Add Company" button
2. Fill in the form:
   - **Name:** Company name (required)
   - **Address:** Company address (optional)
   - **Contact Info:** Contact details (optional)
   - **Admin Email:** Email for company admin (required)
   - **Status:** Active or Inactive (default: Active)
3. Click "Create Company"
4. Company will be created
5. Admin account will be created automatically

#### 6.6.4 Editing Companies

1. Click "Edit" button next to a company
2. Modify any field
3. Update status if needed
4. Click "Update Company" to save
5. Changes take effect immediately

#### 6.6.5 Deleting Companies

1. Click "Delete" button next to a company
2. Confirm the deletion
3. Company and all associated data will be removed

**Warning:** This action cannot be undone. All vehicles, employees, and data associated with the company will be deleted.

### 6.7 Change Password

#### 6.7.1 Accessing Change Password

1. Log in to your account
2. Click "Change Password" in the sidebar menu
3. Change password form will appear

#### 6.7.2 Changing Your Password

1. Enter your current password
2. Enter your new password (minimum 8 characters)
3. Confirm your new password
4. Click "Change Password"
5. Password will be updated
6. You'll be logged out and need to log in again

**Password Requirements:**
- Minimum 8 characters
- Should include letters and numbers
- Avoid common passwords
- Don't reuse recent passwords

**Security Tips:**
- Use a strong, unique password
- Don't share your password
- Change password regularly
- Log out when using shared devices

---

## 7. Multi-Company Architecture

### 7.1 Understanding Multi-Company Support

The system is designed to support multiple parking companies from a single application instance. Each company's data is completely isolated, ensuring privacy and security.

### 7.2 How It Works

**Data Isolation:**
- Each vehicle is associated with a company
- Each employee belongs to a specific company
- Regular users can only see their company's data
- Super admins can see all companies' data

**Company Registration:**
- Companies can self-register through the public registration page
- Super admins can also create companies manually
- Each company gets an admin account automatically

**User Experience:**
- Company name is displayed in the sidebar header
- All operations are scoped to the user's company
- Super admins can switch between companies when registering/discharging vehicles

### 7.3 Company-Specific Features

**For Regular Users:**
- All vehicles belong to their company
- All employees are from their company
- Statistics show only their company's data
- Cannot see other companies' information

**For Super Admins:**
- Can view all companies
- Can manage all companies
- Can register vehicles for any company
- Can discharge vehicles from any company
- Can see cross-company analytics
- Can manage employees across all companies

### 7.4 Customer History Across Companies

**Regular Users:**
- Customer history shows only vehicles from their company
- Filtered by company automatically

**Super Admins:**
- Customer history shows vehicles from all companies
- Grouped by company for easy viewing
- Can see complete parking history across all companies

---

## 8. Technical Specifications

### 8.1 Image Management

**Supported Formats:**
- JPEG (.jpg, .jpeg)
- PNG (.png)
- GIF (.gif)
- WebP (.webp)

**Size Limits:**
- Maximum file size: 5MB per image
- Recommended size: Under 2MB for faster uploads

**Storage:**
- Images are stored in Firebase Storage
- Organized by folders: "vehicles" and "discharge"
- Secure URLs are generated for access
- Images are accessible only through the application

**Camera Capture:**
- Works on mobile, tablet, and desktop
- Uses device camera (back camera preferred on mobile)
- Automatic image compression
- Real-time preview before capture

### 8.2 OTP System

**OTP Generation:**
- 6-digit numeric code
- Randomly generated
- Expires after 10 minutes
- One-time use only

**Delivery Methods:**
1. **WhatsApp (Primary):**
   - Uses WhatsApp Business API
   - Instant delivery
   - Professional message format

2. **SMS (Fallback):**
   - Used if WhatsApp is not configured
   - Requires SMS gateway configuration

**OTP Verification:**
- Entered by user during discharge
- Validated against stored OTP
- Automatically expires after use or timeout
- Cannot be reused

### 8.3 Data Storage

**Google Sheets Structure:**
- **Vehicles Sheet:**
  - Vehicle information
  - Customer details
  - Status and timestamps
  - Company association

- **Employees Sheet:**
  - Employee information
  - Role and permissions
  - Company association
  - Password hashes (encrypted)

- **Companies Sheet:**
  - Company information
  - Admin details
  - Status

- **Sessions Sheet:**
  - Authentication tokens
  - Session management

- **OTP Codes Sheet:**
  - OTP generation and verification
  - Expiration tracking

### 8.4 Security Features

**Authentication:**
- JWT-based token authentication
- Token expiration (24 hours default)
- Secure password hashing (bcrypt)
- Session management

**Authorization:**
- Role-based access control (RBAC)
- Company-based data isolation
- Permission checks on all operations

**Data Protection:**
- HTTPS encryption for all communications
- Secure password storage
- Token-based API authentication
- Input validation and sanitization

### 8.5 Performance Optimizations

**Frontend:**
- Server-side pagination
- Lazy loading of images
- Optimized React components
- Efficient state management

**Backend:**
- Efficient Google Sheets queries
- Caching where appropriate
- Optimized data retrieval
- Batch operations support

---

## 9. Troubleshooting

### 9.1 Common Issues and Solutions

#### Issue: Cannot Log In

**Symptoms:**
- Login fails with error message
- "Invalid credentials" error

**Solutions:**
1. Verify email address is correct
2. Check password (case-sensitive)
3. Ensure Caps Lock is not enabled
4. Try resetting password (contact admin)
5. Clear browser cache and cookies
6. Try a different browser

#### Issue: Images Not Uploading

**Symptoms:**
- Upload fails
- Error message about file size or format

**Solutions:**
1. Check file size (must be under 5MB)
2. Verify file format (JPEG, PNG, GIF, WebP)
3. Check internet connection
4. Try a different image
5. Clear browser cache
6. Check Firebase Storage configuration

#### Issue: OTP Not Received

**Symptoms:**
- OTP not delivered to mobile
- Verification fails

**Solutions:**
1. Verify mobile number is correct
2. Check mobile network connection
3. Wait a few minutes (delivery can be delayed)
4. Check spam/junk folder (for SMS)
5. Verify WhatsApp number (for WhatsApp delivery)
6. Contact administrator if issue persists

#### Issue: Vehicle Not Appearing in List

**Symptoms:**
- Registered vehicle not showing
- Filter not working

**Solutions:**
1. Refresh the page
2. Clear filters
3. Check date range filters
4. Verify company selection (Super Admin)
5. Check vehicle status
6. Contact administrator if issue persists

#### Issue: Cannot Discharge Vehicle

**Symptoms:**
- Discharge button not working
- Verification fails

**Solutions:**
1. Ensure vehicle is in "Parked" status
2. Complete verification (OTP or Image)
3. Check internet connection
4. Refresh the page
5. Try again after a few moments
6. Contact administrator if issue persists

#### Issue: Dashboard Not Loading

**Symptoms:**
- Statistics not showing
- Loading spinner continues

**Solutions:**
1. Check internet connection
2. Refresh the page
3. Clear browser cache
4. Try a different browser
5. Check if other pages work
6. Contact administrator if issue persists

### 9.2 Browser Compatibility

**Supported Browsers:**
- Chrome (latest version) - Recommended
- Firefox (latest version)
- Safari (latest version)
- Edge (latest version)

**Mobile Browsers:**
- Chrome Mobile (Android)
- Safari Mobile (iOS)
- Firefox Mobile

**Unsupported:**
- Internet Explorer (any version)
- Very old browser versions

**Recommendations:**
- Keep your browser updated
- Enable JavaScript
- Allow cookies
- Enable camera access (for image capture)

### 9.3 Performance Issues

**Slow Loading:**
- Check internet connection speed
- Clear browser cache
- Close unnecessary browser tabs
- Disable browser extensions
- Try a different browser

**Large Dataset:**
- Use pagination (reduce page size)
- Apply filters to narrow results
- Use date range filters
- Contact administrator for optimization

---

## 10. Best Practices

### 10.1 Vehicle Registration

**Best Practices:**
1. **Accurate Information:**
   - Double-check vehicle number
   - Verify mobile number (10 digits)
   - Use full customer name

2. **Image Quality:**
   - Ensure good lighting
   - Capture full vehicle
   - Include license plate clearly
   - Use camera when possible (better quality)

3. **Timeliness:**
   - Register vehicles immediately upon entry
   - Don't delay registration
   - Update information if customer details change

### 10.2 Vehicle Discharge

**Best Practices:**
1. **Verification:**
   - Always verify before discharging
   - Use OTP for high-value vehicles
   - Use Image verification when OTP unavailable
   - Use Manual only for trusted scenarios

2. **Documentation:**
   - Capture discharge image when possible
   - Verify customer identity
   - Check vehicle matches registration

3. **Security:**
   - Never skip verification for valuable vehicles
   - Verify mobile number matches
   - Confirm OTP with customer directly

### 10.3 Data Management

**Best Practices:**
1. **Regular Updates:**
   - Keep vehicle information current
   - Update customer details when changed
   - Correct any errors immediately

2. **Organization:**
   - Use consistent naming conventions
   - Maintain accurate records
   - Review and clean up old data periodically

3. **Backup:**
   - Regular data exports (if available)
   - Keep records of important transactions
   - Document any manual changes

### 10.4 Security

**Best Practices:**
1. **Password Management:**
   - Use strong passwords
   - Change passwords regularly
   - Don't share passwords
   - Use unique passwords

2. **Access Control:**
   - Grant admin access only to trusted employees
   - Review employee access regularly
   - Deactivate unused accounts
   - Monitor for suspicious activity

3. **Data Privacy:**
   - Protect customer information
   - Don't share data outside the system
   - Follow data protection regulations
   - Secure device access

### 10.5 User Management

**Best Practices:**
1. **Employee Onboarding:**
   - Provide proper training
   - Set strong initial passwords
   - Require password change on first login
   - Assign appropriate roles

2. **Access Management:**
   - Grant minimum necessary permissions
   - Review access regularly
   - Deactivate former employees immediately
   - Monitor user activity

3. **Training:**
   - Train employees on system usage
   - Provide documentation
   - Answer questions promptly
   - Keep training materials updated

---

## 11. Appendix

### 11.1 Keyboard Shortcuts

Currently, the application does not have specific keyboard shortcuts, but standard browser shortcuts work:
- **Ctrl/Cmd + R:** Refresh page
- **Ctrl/Cmd + F:** Find/Search
- **Tab:** Navigate between form fields
- **Enter:** Submit forms
- **Esc:** Close modals

### 11.2 Glossary

**Admin:** User with administrative privileges within their company

**Company:** A parking business entity in the system

**Discharge:** Process of releasing a vehicle from parking

**Employee:** Standard user with basic permissions

**OTP:** One-Time Password used for verification

**Parked:** Vehicle status indicating vehicle is currently in parking

**Register:** Process of adding a vehicle to the system

**Super Admin:** User with access to all companies

**Vehicle:** Any vehicle registered in the system

### 11.3 Contact and Support

**For Technical Issues:**
- Contact your system administrator
- Check this documentation first
- Review troubleshooting section

**For Feature Requests:**
- Contact your system administrator
- Provide detailed description
- Include use case examples

### 11.4 Version History

**Version 1.0.0 (Current)**
- Initial release
- Multi-company support
- Vehicle registration and discharge
- Employee management
- Dashboard and analytics
- Image management
- OTP verification
- Customer history tracking

### 11.5 Additional Resources

**Related Documentation:**
- Setup Guide (for administrators)
- API Documentation (for developers)
- Deployment Guide (for system administrators)

**External Resources:**
- Google Sheets Help
- Firebase Documentation
- WhatsApp Business API Documentation

---

## Document Information

**Document Title:** Parking Management System - Complete User Documentation  
**Version:** 1.0.0  
**Last Updated:** January 2025  
**Author:** Technical Documentation Team  
**Status:** Current  

**Revision History:**
- v1.0.0 (January 2025): Initial comprehensive documentation

---

**End of Documentation**

For questions or feedback about this documentation, please contact your system administrator.

