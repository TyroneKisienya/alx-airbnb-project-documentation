# AirBnB Clone Backend Requirements Specification

## Table of Contents
1. [User Authentication & Authorization System](#1-user-authentication--authorization-system)
2. [Property Management System](#2-property-management-system)
3. [Booking Management System](#3-booking-management-system)

---

## 1. User Authentication & Authorization System

### 1.1 Functional Requirements

#### 1.1.1 User Registration
**Description:** Allow new users to create accounts with role-based access (guest/host/admin).

**Business Rules:**
- Email must be unique across the system
- Password must meet security requirements (min 8 chars, special chars, numbers)
- Users can register as guests or hosts (admin role assigned manually)
- Account activation via email verification required

#### 1.1.2 User Authentication
**Description:** Secure login system with JWT token-based authentication.

**Business Rules:**
- Support email/password login
- Optional OAuth integration (Google, Facebook)
- JWT tokens expire after 24 hours
- Refresh tokens valid for 30 days
- Account lockout after 5 failed login attempts

#### 1.1.3 Profile Management
**Description:** Users can view and update their profile information.

**Business Rules:**
- Users can only modify their own profiles
- Email changes require verification
- Profile photos limited to 5MB, specific formats
- Certain fields require admin approval for hosts

### 1.2 API Endpoints

#### 1.2.1 POST /api/auth/register
**Purpose:** Register a new user account

**Request Body:**
```json
{
  "first_name": "string (required, 2-50 chars)",
  "last_name": "string (required, 2-50 chars)", 
  "email": "string (required, valid email format)",
  "password": "string (required, min 8 chars)",
  "phone_number": "string (optional, E.164 format)",
  "role": "enum (required: 'guest', 'host')"
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "message": "User registered successfully. Please verify your email.",
  "data": {
    "user_id": "uuid",
    "email": "string",
    "role": "string",
    "verification_required": true
  }
}
```

**Error Responses:**
- `400 Bad Request`: Validation errors
- `409 Conflict`: Email already exists
- `422 Unprocessable Entity`: Invalid data format

#### 1.2.2 POST /api/auth/login
**Purpose:** Authenticate user and return JWT tokens

**Request Body:**
```json
{
  "email": "string (required)",
  "password": "string (required)"
}
```

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "user": {
      "user_id": "uuid",
      "first_name": "string",
      "last_name": "string",
      "email": "string",
      "role": "string"
    },
    "tokens": {
      "access_token": "jwt_string",
      "refresh_token": "jwt_string",
      "expires_in": 86400
    }
  }
}
```

#### 1.2.3 POST /api/auth/refresh
**Purpose:** Refresh expired access token

**Headers:**
```
Authorization: Bearer <refresh_token>
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "access_token": "jwt_string",
    "expires_in": 86400
  }
}
```

#### 1.2.4 GET /api/users/profile
**Purpose:** Retrieve current user's profile

**Headers:**
```
Authorization: Bearer <access_token>
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "user_id": "uuid",
    "first_name": "string",
    "last_name": "string",
    "email": "string",
    "phone_number": "string",
    "role": "string",
    "profile_picture_url": "string",
    "created_at": "timestamp",
    "is_verified": boolean
  }
}
```

#### 1.2.5 PUT /api/users/profile
**Purpose:** Update user profile information

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request Body:**
```json
{
  "first_name": "string (optional, 2-50 chars)",
  "last_name": "string (optional, 2-50 chars)",
  "phone_number": "string (optional, E.164 format)",
  "profile_picture": "file (optional, max 5MB)"
}
```

### 1.3 Validation Rules

| Field | Rules |
|-------|-------|
| first_name | Required, 2-50 characters, alphabetic + spaces |
| last_name | Required, 2-50 characters, alphabetic + spaces |
| email | Required, valid email format, unique |
| password | Min 8 chars, 1 uppercase, 1 lowercase, 1 number, 1 special char |
| phone_number | Optional, E.164 format validation |
| role | Required, enum: ['guest', 'host'] |

### 1.4 Security Requirements

- **Password Hashing:** Use bcrypt with salt rounds ≥ 12
- **JWT Security:** RS256 algorithm, secure secret management
- **Rate Limiting:** 5 requests/minute for auth endpoints per IP
- **HTTPS Only:** All authentication endpoints require TLS
- **CORS:** Strict origin validation
- **Account Security:** Lock account after 5 failed attempts for 15 minutes

### 1.5 Performance Criteria

| Metric | Target |
|--------|--------|
| Registration Response Time | < 500ms (95th percentile) |
| Login Response Time | < 200ms (95th percentile) |
| Profile Fetch Response Time | < 100ms (95th percentile) |
| Concurrent Users | Support 10,000 simultaneous users |
| Database Connections | Pool size: 20-50 connections |

---

## 2. Property Management System

### 2.1 Functional Requirements

#### 2.1.1 Property Creation
**Description:** Hosts can create new property listings with comprehensive details.

**Business Rules:**
- Only verified hosts can create listings
- Maximum 20 properties per host (configurable)
- Property must have at least 1 photo
- Location validation against geocoding service
- Price must be positive value

#### 2.1.2 Property Search and Filtering
**Description:** Advanced search functionality for property discovery.

**Business Rules:**
- Support location-based search (city, coordinates, radius)
- Price range filtering
- Date-based availability checking
- Amenity-based filtering
- Guest capacity filtering
- Pagination required for performance

#### 2.1.3 Property Management
**Description:** Hosts can update and manage their property listings.

**Business Rules:**
- Hosts can only modify their own properties
- Price changes affect future bookings only
- Availability calendar management
- Property deactivation (soft delete)

### 2.2 API Endpoints

#### 2.2.1 POST /api/properties
**Purpose:** Create a new property listing

**Headers:**
```
Authorization: Bearer <access_token>
Content-Type: multipart/form-data
```

**Request Body:**
```json
{
  "name": "string (required, 5-255 chars)",
  "description": "string (required, 50-2000 chars)",
  "location": {
    "address": "string (required)",
    "city": "string (required)",
    "country": "string (required)",
    "latitude": "decimal (optional)",
    "longitude": "decimal (optional)"
  },
  "price_per_night": "decimal (required, > 0, max 99999.99)",
  "max_guests": "integer (required, 1-20)",
  "bedrooms": "integer (required, 0-10)",
  "bathrooms": "decimal (required, 0-10, 0.5 increments)",
  "amenities": ["array of strings (optional)"],
  "house_rules": "string (optional, max 1000 chars)",
  "images": ["array of files (required, min 1, max 10, 5MB each)"]
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "message": "Property created successfully",
  "data": {
    "property_id": "uuid",
    "name": "string",
    "status": "active",
    "created_at": "timestamp"
  }
}
```

#### 2.2.2 GET /api/properties/search
**Purpose:** Search and filter properties

**Query Parameters:**
```
location: string (optional) - City or address
latitude: decimal (optional) - Latitude coordinate  
longitude: decimal (optional) - Longitude coordinate
radius: integer (optional, default: 25) - Search radius in km
check_in: date (optional, YYYY-MM-DD)
check_out: date (optional, YYYY-MM-DD)
guests: integer (optional, default: 1)
min_price: decimal (optional)
max_price: decimal (optional)
amenities: string (optional) - Comma-separated amenity IDs
bedrooms: integer (optional)
bathrooms: decimal (optional)
page: integer (optional, default: 1)
limit: integer (optional, default: 20, max: 100)
sort_by: enum (optional) - 'price_asc', 'price_desc', 'rating', 'distance'
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "properties": [
      {
        "property_id": "uuid",
        "name": "string",
        "description": "string (truncated to 200 chars)",
        "location": "string",
        "price_per_night": "decimal",
        "max_guests": "integer",
        "bedrooms": "integer",
        "bathrooms": "decimal",
        "average_rating": "decimal",
        "review_count": "integer",
        "main_image_url": "string",
        "distance_km": "decimal (if location search)",
        "available": "boolean"
      }
    ],
    "pagination": {
      "current_page": 1,
      "total_pages": 10,
      "total_count": 200,
      "has_next": true,
      "has_previous": false
    }
  }
}
```

#### 2.2.3 GET /api/properties/{property_id}
**Purpose:** Retrieve detailed property information

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "property_id": "uuid",
    "host": {
      "user_id": "uuid",
      "first_name": "string",
      "last_name": "string",
      "profile_picture_url": "string",
      "host_since": "timestamp",
      "response_rate": "decimal",
      "response_time": "string"
    },
    "name": "string",
    "description": "string",
    "location": {
      "address": "string",
      "city": "string",
      "country": "string",
      "latitude": "decimal",
      "longitude": "decimal"
    },
    "price_per_night": "decimal",
    "max_guests": "integer",
    "bedrooms": "integer",
    "bathrooms": "decimal",
    "amenities": ["array of amenity objects"],
    "house_rules": "string",
    "images": ["array of image URLs"],
    "availability_calendar": ["array of available date ranges"],
    "reviews": {
      "average_rating": "decimal",
      "total_count": "integer",
      "recent_reviews": ["array of recent review objects"]
    },
    "created_at": "timestamp",
    "updated_at": "timestamp"
  }
}
```

#### 2.2.4 PUT /api/properties/{property_id}
**Purpose:** Update existing property listing

**Headers:**
```
Authorization: Bearer <access_token>
```

**Authorization:** Host must own the property

### 2.3 Validation Rules

| Field | Rules |
|-------|-------|
| name | Required, 5-255 characters, no special chars except spaces/hyphens |
| description | Required, 50-2000 characters |
| price_per_night | Required, decimal(10,2), > 0, ≤ 99999.99 |
| max_guests | Required, integer, 1-20 |
| bedrooms | Required, integer, 0-10 |
| bathrooms | Required, decimal(2,1), 0-10, 0.5 increments |
| location.address | Required, 10-500 characters |
| images | Required, 1-10 files, JPG/PNG, max 5MB each |

### 2.4 Performance Criteria

| Operation | Target Response Time | Throughput |
|-----------|---------------------|------------|
| Property Creation | < 2s (including image upload) | 100 req/min per host |
| Property Search | < 300ms (95th percentile) | 1000 req/min |
| Property Details | < 150ms (95th percentile) | 2000 req/min |
| Property Update | < 1s (95th percentile) | 200 req/min per host |

---

## 3. Booking Management System

### 3.1 Functional Requirements

#### 3.1.1 Booking Creation
**Description:** Guests can book available properties with real-time availability validation.

**Business Rules:**
- Prevent overlapping bookings for same property
- Minimum booking duration: 1 night
- Maximum booking duration: 90 days (configurable)
- Check-in date must be in future
- Real-time availability validation
- Automatic total price calculation
- Booking confirmation within 24-48 hours for hosts

#### 3.1.2 Booking Status Management
**Description:** Track and manage booking lifecycle through various states.

**Business Rules:**
- Initial status: 'pending' (awaiting host approval)
- Host can confirm/decline within 48 hours
- Auto-decline after 48 hours if no response
- Guests can cancel based on cancellation policy
- Status progression: pending → confirmed → completed/canceled

#### 3.1.3 Availability Management
**Description:** Real-time availability checking and calendar management.

**Business Rules:**
- Block dates for confirmed bookings
- Support for host-blocked dates (maintenance, personal use)
- Minimum advance notice for bookings (e.g., 2 hours)
- Check-in/check-out time restrictions (default: 3 PM / 11 AM)

### 3.2 API Endpoints

#### 3.2.1 POST /api/bookings
**Purpose:** Create a new booking request

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request Body:**
```json
{
  "property_id": "uuid (required)",
  "check_in_date": "date (required, YYYY-MM-DD)",
  "check_out_date": "date (required, YYYY-MM-DD)",
  "guests": "integer (required, 1-max_property_guests)",
  "special_requests": "string (optional, max 500 chars)"
}
```

**Response (201 Created):**
```json
{
  "success": true,
  "message": "Booking request created successfully",
  "data": {
    "booking_id": "uuid",
    "property_id": "uuid",
    "user_id": "uuid",
    "check_in_date": "date",
    "check_out_date": "date",
    "guests": "integer",
    "nights": "integer",
    "price_per_night": "decimal",
    "total_price": "decimal",
    "status": "pending",
    "expires_at": "timestamp (48h from creation)",
    "created_at": "timestamp"
  }
}
```

**Error Responses:**
- `400 Bad Request`: Invalid dates or parameters
- `409 Conflict`: Property not available for selected dates
- `403 Forbidden`: Cannot book own property
- `404 Not Found`: Property not found

#### 3.2.2 GET /api/bookings
**Purpose:** Retrieve user's bookings with filtering options

**Headers:**
```
Authorization: Bearer <access_token>
```

**Query Parameters:**
```
status: enum (optional) - 'pending', 'confirmed', 'canceled', 'completed'
role: enum (optional) - 'guest', 'host' (determines perspective)
start_date: date (optional) - Filter bookings from this date
end_date: date (optional) - Filter bookings until this date
page: integer (optional, default: 1)
limit: integer (optional, default: 20, max: 100)
sort_by: enum (optional) - 'created_at', 'check_in_date', 'total_price'
sort_order: enum (optional) - 'asc', 'desc'
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "bookings": [
      {
        "booking_id": "uuid",
        "property": {
          "property_id": "uuid",
          "name": "string",
          "location": "string",
          "main_image_url": "string"
        },
        "guest": {
          "user_id": "uuid",
          "first_name": "string",
          "last_name": "string"
        },
        "check_in_date": "date",
        "check_out_date": "date",
        "guests": "integer",
        "nights": "integer",
        "total_price": "decimal",
        "status": "string",
        "created_at": "timestamp",
        "can_cancel": "boolean",
        "cancellation_deadline": "timestamp"
      }
    ],
    "pagination": {
      "current_page": 1,
      "total_pages": 5,
      "total_count": 95,
      "has_next": true,
      "has_previous": false
    }
  }
}
```

#### 3.2.3 PATCH /api/bookings/{booking_id}/status
**Purpose:** Update booking status (host approval/decline, guest cancellation)

**Headers:**
```
Authorization: Bearer <access_token>
```

**Request Body:**
```json
{
  "status": "enum (required: 'confirmed', 'canceled')",
  "reason": "string (optional for cancellations, max 500 chars)"
}
```

**Authorization Rules:**
- Hosts can confirm/decline their property bookings in 'pending' status
- Guests can cancel their own bookings (subject to policy)
- Admins can modify any booking status

**Response (200 OK):**
```json
{
  "success": true,
  "message": "Booking status updated successfully",
  "data": {
    "booking_id": "uuid",
    "status": "string",
    "updated_at": "timestamp",
    "refund_amount": "decimal (if applicable)"
  }
}
```

#### 3.2.4 GET /api/properties/{property_id}/availability
**Purpose:** Check property availability for specific date range

**Query Parameters:**
```
start_date: date (required, YYYY-MM-DD)
end_date: date (required, YYYY-MM-DD)
```

**Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "property_id": "uuid",
    "is_available": "boolean",
    "blocked_dates": ["array of blocked date ranges"],
    "price_breakdown": {
      "base_price": "decimal",
      "nights": "integer", 
      "subtotal": "decimal",
      "service_fee": "decimal",
      "taxes": "decimal",
      "total": "decimal"
    },
    "minimum_stay": "integer",
    "maximum_stay": "integer"
  }
}
```

### 3.3 Validation Rules

| Field | Rules |
|-------|-------|
| property_id | Required, valid UUID, must exist and be active |
| check_in_date | Required, date format, must be future date, ≥ minimum notice |
| check_out_date | Required, date format, > check_in_date, ≤ max booking duration |
| guests | Required, integer, 1 ≤ guests ≤ property.max_guests |
| special_requests | Optional, max 500 characters |

### 3.4 Business Logic Requirements

#### 3.4.1 Availability Validation Algorithm
```
1. Validate date range (future dates, logical order)
2. Check property exists and is active
3. Verify guest count within property limits
4. Check for overlapping confirmed bookings
5. Validate against host-blocked dates
6. Verify minimum/maximum stay requirements
7. Calculate total price including taxes/fees
```

#### 3.4.2 Cancellation Policy Engine
```
- Free cancellation: Up to X days before check-in
- Moderate cancellation: 50% refund up to Y days before
- Strict cancellation: No refund within Z days
- Host cancellation penalties apply
- Emergency cancellation overrides (admin only)
```

### 3.5 Performance Criteria

| Operation | Target Response Time | Throughput | Notes |
|-----------|---------------------|------------|--------|
| Booking Creation | < 500ms (95th percentile) | 500 req/min | Includes availability check |
| Availability Check | < 100ms (95th percentile) | 2000 req/min | Critical for search |
| Booking Search | < 200ms (95th percentile) | 1000 req/min | With proper indexing |
| Status Update | < 150ms (95th percentile) | 300 req/min | Includes notifications |

### 3.6 Data Consistency Requirements

- **ACID Compliance:** All booking operations must be transactional
- **Race Condition Prevention:** Use database-level locks for availability checks
- **Eventual Consistency:** Notification dispatch can be asynchronous
- **Backup Strategy:** Point-in-time recovery for booking data
- **Audit Trail:** Log all status changes with timestamps and reasons

---

## Global Technical Requirements

### Database Requirements
- **Connection Pooling:** Min 10, Max 50 connections
- **Query Optimization:** All queries must use appropriate indexes
- **Transaction Management:** Use database transactions for multi-table operations
- **Foreign Key Constraints:** Enforce referential integrity

### Caching Strategy
- **Redis Configuration:**
  - Property search results: 15 minutes TTL
  - User sessions: 24 hours TTL  
  - Availability data: 5 minutes TTL
  - Property details: 1 hour TTL

### Error Handling Standards
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed for the request",
    "details": [
      {
        "field": "email",
        "message": "Email format is invalid"
      }
    ],
    "request_id": "uuid"
  }
}
```

### Monitoring and Logging
- **Request Logging:** All API calls with duration and status
- **Error Logging:** Structured logging with stack traces
- **Performance Monitoring:** Track P95/P99 response times
- **Business Metrics:** Booking conversion rates, search success rates

### Security Requirements
- **Input Sanitization:** Prevent SQL injection, XSS attacks
- **Rate Limiting:** Per-user and per-IP limits
- **API Versioning:** Support multiple API versions
- **Data Encryption:** Encrypt PII data at rest