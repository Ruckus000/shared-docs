# Roadside Helper - API Documentation

## 📋 Overview

The Roadside Helper mobile app uses a dual-API architecture:
- **Primary**: Supabase (PostgreSQL with real-time subscriptions)
- **Secondary**: Admin Dashboard API (custom REST API)

**Base URLs:**
- Supabase: `${EXPO_PUBLIC_SUPABASE_URL}/rest/v1/`
- Admin API: `${EXPO_PUBLIC_ADMIN_API_URL}`

---

## 🔐 Authentication

### Supabase Authentication
**Type**: JWT Bearer tokens via Supabase Auth  
**Header**: `Authorization: Bearer <jwt_token>`  
**Additional**: `apikey: <supabase_anon_key>`

### Admin API Authentication
**Type**: JWT Bearer tokens (same as Supabase)  
**Header**: `Authorization: Bearer <jwt_token>`

---

## 👤 User Authentication

### POST /auth/v1/signup
**Description**: Register a new user account

**Authentication**: None

**Headers**:
```
Content-Type: application/json
apikey: {SUPABASE_ANON_KEY}
```

**Request Body**:
```json
{
  "email": "string - user email address",
  "password": "string - min 6 characters",
  "options": {
    "data": {
      "full_name": "string - optional user display name"
    }
  }
}
```

**Success Response: 200 OK**
```json
{
  "user": {
    "id": "uuid",
    "email": "string",
    "user_metadata": {
      "full_name": "string"
    }
  },
  "session": {
    "access_token": "jwt_string",
    "refresh_token": "string"
  }
}
```

**Error Responses**:
- `400`: Bad Request - Invalid email or weak password
- `422`: Unprocessable Entity - Email already registered

---

### POST /auth/v1/token?grant_type=password
**Description**: Sign in existing user

**Authentication**: None

**Headers**:
```
Content-Type: application/json
apikey: {SUPABASE_ANON_KEY}
```

**Request Body**:
```json
{
  "email": "string",
  "password": "string"
}
```

**Success Response: 200 OK**
```json
{
  "user": {
    "id": "uuid",
    "email": "string"
  },
  "session": {
    "access_token": "jwt_string",
    "refresh_token": "string"
  }
}
```

**Error Responses**:
- `400`: Bad Request - Invalid credentials
- `422`: Unprocessable Entity - Missing email or password

---

### POST /auth/v1/logout
**Description**: Sign out current user

**Authentication**: Required

**Headers**:
```
Authorization: Bearer {jwt_token}
apikey: {SUPABASE_ANON_KEY}
```

**Success Response: 204 No Content**

---

## 👥 User Profiles

### GET /profiles?id=eq.{user_id}&select=*
**Description**: Get user profile information

**Authentication**: Required

**Headers**:
```
Authorization: Bearer {jwt_token}
apikey: {SUPABASE_ANON_KEY}
```

**Success Response: 200 OK**
```json
[{
  "id": "uuid",
  "email": "string",
  "name": "string | null",
  "phone": "string | null",
  "member_since": "string | null",
  "created_at": "string",
  "updated_at": "string"
}]
```

**Error Responses**:
- `401`: Unauthorized - Invalid or expired token
- `403`: Forbidden - RLS policy violation

---

### PATCH /profiles?id=eq.{user_id}
**Description**: Update user profile

**Authentication**: Required

**Headers**:
```
Content-Type: application/json
Authorization: Bearer {jwt_token}
apikey: {SUPABASE_ANON_KEY}
Prefer: return=minimal
```

**Request Body**:
```json
{
  "name": "string - optional",
  "phone": "string - optional",
  "updated_at": "string - ISO timestamp"
}
```

**Success Response: 204 No Content**

**Error Responses**:
- `401`: Unauthorized
- `403`: Forbidden - Can only update own profile
- `422`: Unprocessable Entity - Invalid data format

---

## 🛠️ Service Types

### GET /service_types?select=*&order=name
**Description**: Get all available roadside services

**Authentication**: Optional

**Headers**:
```
apikey: {SUPABASE_ANON_KEY}
Authorization: Bearer {jwt_token} (if authenticated)
```

**Success Response: 200 OK**
```json
[{
  "id": "uuid",
  "name": "string",
  "icon_name": "string",
  "description": "string",
  "base_price": "number - in cents",
  "base_time": "string - estimated duration"
}]
```

**Example Response**:
```json
[
  {
    "id": "123e4567-e89b-12d3-a456-426614174000",
    "name": "Towing",
    "icon_name": "truck",
    "description": "Vehicle towing for breakdowns",
    "base_price": 7500,
    "base_time": "20-30 min"
  },
  {
    "id": "123e4567-e89b-12d3-a456-426614174001",
    "name": "Battery Jump",
    "icon_name": "zap",
    "description": "Jump start & replacement",
    "base_price": 3500,
    "base_time": "15-20 min"
  }
]
```

---

## 📋 Bookings

### GET /bookings?user_id=eq.{user_id}&select=*,service_types(*),booking_offers(*,service_providers(*))&order=created_at.desc
**Description**: Get user's booking history with full details

**Authentication**: Required

**Headers**:
```
Authorization: Bearer {jwt_token}
apikey: {SUPABASE_ANON_KEY}
```

**Success Response: 200 OK**
```json
[{
  "id": "uuid",
  "user_id": "uuid",
  "service_type_id": "uuid",
  "location_latitude": "number",
  "location_longitude": "number", 
  "location_address": "string",
  "notes": "string",
  "status": "pending|offered|accepted|in_progress|completed|cancelled",
  "created_at": "string",
  "updated_at": "string",
  "service_types": {
    "name": "string",
    "icon_name": "string",
    "description": "string",
    "base_price": "number",
    "base_time": "string"
  },
  "booking_offers": [{
    "id": "uuid",
    "booking_id": "uuid",
    "provider_id": "uuid",
    "price": "number - in cents",
    "estimated_arrival": "string",
    "status": "pending|accepted|declined",
    "created_at": "string",
    "service_providers": {
      "name": "string",
      "rating": "number",
      "review_count": "number"
    }
  }]
}]
```

**Error Responses**:
- `401`: Unauthorized
- `403`: Forbidden - Can only access own bookings

---

### POST /bookings
**Description**: Create a new service booking

**Authentication**: Required

**Headers**:
```
Content-Type: application/json
Authorization: Bearer {jwt_token}
apikey: {SUPABASE_ANON_KEY}
Prefer: return=representation
```

**Request Body**:
```json
{
  "user_id": "uuid",
  "service_type_id": "uuid",
  "location_latitude": "number",
  "location_longitude": "number",
  "location_address": "string",
  "notes": "string",
  "status": "pending"
}
```

**Success Response: 201 Created**
```json
{
  "id": "uuid",
  "user_id": "uuid",
  "service_type_id": "uuid",
  "location_latitude": "number",
  "location_longitude": "number",
  "location_address": "string", 
  "notes": "string",
  "status": "pending",
  "created_at": "string",
  "updated_at": "string"
}
```

**Error Responses**:
- `400`: Bad Request - Invalid service_type_id
- `401`: Unauthorized
- `422`: Unprocessable Entity - Missing required fields

---

### PATCH /bookings?id=eq.{booking_id}&user_id=eq.{user_id}
**Description**: Update booking status

**Authentication**: Required

**Headers**:
```
Content-Type: application/json
Authorization: Bearer {jwt_token}
apikey: {SUPABASE_ANON_KEY}
Prefer: return=minimal
```

**Request Body**:
```json
{
  "status": "accepted|in_progress|completed|cancelled",
  "updated_at": "string - ISO timestamp"
}
```

**Success Response: 204 No Content**

**Error Responses**:
- `401`: Unauthorized
- `403`: Forbidden - Can only update own bookings
- `404`: Not Found - Booking doesn't exist

---

### GET /bookings?id=eq.{booking_id}&user_id=eq.{user_id}&select=*,service_types(*),booking_offers(*,service_providers(*))
**Description**: Get specific booking details

**Authentication**: Required

**Headers**:
```
Authorization: Bearer {jwt_token}
apikey: {SUPABASE_ANON_KEY}
```

**Success Response: 200 OK**
```json
{
  "id": "uuid",
  "user_id": "uuid",
  "service_type_id": "uuid",
  "location_latitude": "number",
  "location_longitude": "number",
  "location_address": "string",
  "notes": "string", 
  "status": "string",
  "created_at": "string",
  "updated_at": "string",
  "service_types": {
    "name": "string",
    "icon_name": "string", 
    "description": "string",
    "base_price": "number",
    "base_time": "string"
  },
  "booking_offers": [...]
}
```

---

## 💼 Booking Offers

### PATCH /booking_offers?id=eq.{offer_id}
**Description**: Accept a provider's offer

**Authentication**: Required

**Headers**:
```
Content-Type: application/json
Authorization: Bearer {jwt_token}
apikey: {SUPABASE_ANON_KEY}
Prefer: return=minimal
```

**Request Body**:
```json
{
  "status": "accepted"
}
```

**Success Response: 204 No Content**

**Error Responses**:
- `401`: Unauthorized
- `403`: Forbidden - Can only accept offers for own bookings
- `404`: Not Found - Offer doesn't exist

---

## 🏢 Admin Dashboard API

### POST /api/bookings/notify
**Description**: Notify admin dashboard of new booking

**Authentication**: Required

**Headers**:
```
Content-Type: application/json
Authorization: Bearer {jwt_token}
```

**Request Body**:
```json
{
  "bookingId": "string",
  "serviceType": "string",
  "customerName": "string",
  "customerPhone": "string",
  "customerEmail": "string",
  "location": {
    "address": "string",
    "latitude": "number",
    "longitude": "number"
  },
  "notes": "string",
  "emergency": "boolean",
  "createdAt": "string - ISO timestamp"
}
```

**Success Response: 200 OK**
```json
{
  "success": true,
  "message": "Booking notification received",
  "bookingId": "string"
}
```

**Error Responses**:
- `400`: Bad Request - Invalid payload
- `401`: Unauthorized
- `500`: Internal Server Error

---

### GET /api/bookings/{bookingId}/offers
**Description**: Get booking with current offers from admin

**Authentication**: Required

**Headers**:
```
Authorization: Bearer {jwt_token}
```

**Success Response: 200 OK**
```json
{
  "bookingId": "string",
  "offers": [{
    "id": "string",
    "providerId": "string", 
    "providerName": "string",
    "price": "number - in cents",
    "estimatedArrival": "string",
    "rating": "number",
    "createdAt": "string"
  }],
  "lastUpdated": "string"
}
```

---

### POST /api/bookings/{bookingId}/offers/{offerId}/accept
**Description**: Accept an offer from admin dashboard

**Authentication**: Required

**Headers**:
```
Content-Type: application/json
Authorization: Bearer {jwt_token}
```

**Success Response: 200 OK**
```json
{
  "success": true,
  "message": "Offer accepted",
  "offerId": "string"
}
```

---

### PUT /api/bookings/{bookingId}/status
**Description**: Update booking status from admin

**Authentication**: Required

**Headers**:
```
Content-Type: application/json
Authorization: Bearer {jwt_token}
```

**Request Body**:
```json
{
  "status": "string",
  "metadata": "object - optional"
}
```

**Success Response: 200 OK**
```json
{
  "success": true,
  "bookingId": "string",
  "status": "string"
}
```

---

## 📍 Provider Services

### GET /api/providers/{providerId}/location
**Description**: Get real-time provider location

**Authentication**: Required

**Headers**:
```
Authorization: Bearer {jwt_token}
```

**Success Response: 200 OK**
```json
{
  "providerId": "string",
  "location": {
    "latitude": "number",
    "longitude": "number"
  },
  "timestamp": "string",
  "accuracy": "number - optional"
}
```

---

### PUT /api/providers/{providerId}/location
**Description**: Update provider location (for provider app)

**Authentication**: Required

**Headers**:
```
Content-Type: application/json
Authorization: Bearer {jwt_token}
```

**Request Body**:
```json
{
  "lat": "number",
  "lng": "number"
}
```

**Success Response: 200 OK**
```json
{
  "success": true,
  "providerId": "string",
  "updated": "string"
}
```

---

## 🔔 Notifications

### POST /api/notifications/register
**Description**: Register device for push notifications

**Authentication**: Required

**Headers**:
```
Content-Type: application/json
Authorization: Bearer {jwt_token}
```

**Request Body**:
```json
{
  "deviceToken": "string",
  "platform": "ios|android"
}
```

**Success Response: 200 OK**
```json
{
  "success": true,
  "deviceId": "string"
}
```

---

## 💳 Payments

### POST /api/payments/create-intent
**Description**: Create Stripe payment intent

**Authentication**: Required

**Headers**:
```
Content-Type: application/json
Authorization: Bearer {jwt_token}
```

**Request Body**:
```json
{
  "bookingId": "string",
  "amount": "number - in cents"
}
```

**Success Response: 200 OK**
```json
{
  "clientSecret": "string",
  "paymentIntentId": "string"
}
```

---

### POST /api/payments/confirm
**Description**: Confirm payment completion

**Authentication**: Required

**Headers**:
```
Content-Type: application/json
Authorization: Bearer {jwt_token}
```

**Request Body**:
```json
{
  "paymentIntentId": "string",
  "bookingId": "string"
}
```

**Success Response: 200 OK**
```json
{
  "success": true,
  "paymentStatus": "succeeded"
}
```

---

## 🔄 Real-time Subscriptions

### WebSocket: Booking Updates
**Description**: Real-time booking status updates via Supabase subscriptions

**Channel**: `user-bookings-{user_id}`
**Table**: `bookings`
**Filter**: `user_id=eq.{user_id}`

**Event Types**: INSERT, UPDATE, DELETE

**Payload Structure**:
```json
{
  "eventType": "INSERT|UPDATE|DELETE",
  "new": "object - new record data",
  "old": "object - old record data", 
  "table": "bookings",
  "schema": "public"
}
```

---

### WebSocket: Offer Updates  
**Description**: Real-time offer updates for specific booking

**Channel**: `booking-{booking_id}`
**Table**: `booking_offers`
**Filter**: `booking_id=eq.{booking_id}`

**Event Types**: INSERT, UPDATE, DELETE

---

## 🌐 Environment Configuration

### Required Environment Variables
```bash
# Supabase Configuration
EXPO_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
EXPO_PUBLIC_SUPABASE_ANON_KEY=your-anon-key

# Admin Dashboard API  
EXPO_PUBLIC_ADMIN_API_URL=https://your-admin-dashboard.vercel.app/api
```

### API Base URLs
- **Development**: Use environment variables
- **Production**: Same environment variables with production URLs

---

## 📊 Error Handling

### Standard Error Format
```json
{
  "error": {
    "message": "string - human readable error",
    "code": "string - error code", 
    "details": "object - additional context"
  }
}
```

### Common HTTP Status Codes
- `200`: Success
- `201`: Created
- `204`: No Content
- `400`: Bad Request - Invalid input
- `401`: Unauthorized - Authentication required
- `403`: Forbidden - Permission denied
- `404`: Not Found - Resource doesn't exist
- `422`: Unprocessable Entity - Validation failed
- `500`: Internal Server Error

---

## 🛡️ Security Notes

### Row Level Security (RLS)
All Supabase tables have RLS policies that ensure users can only access their own data.

### Rate Limiting
- Supabase: Built-in rate limiting per plan
- Admin API: Custom rate limiting (implementation dependent)

### Data Validation
- All inputs are validated on both client and server
- SQL injection protection via parameterized queries
- XSS protection via proper encoding

---

## 🧪 Testing Endpoints

### cURL Examples

**Create Booking:**
```bash
curl -X POST https://your-project.supabase.co/rest/v1/bookings \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "apikey: YOUR_SUPABASE_ANON_KEY" \
  -H "Prefer: return=representation" \
  -d '{
    "user_id": "user-uuid",
    "service_type_id": "service-uuid", 
    "location_latitude": 40.7128,
    "location_longitude": -74.0060,
    "location_address": "New York, NY",
    "notes": "Battery died",
    "status": "pending"
  }'
```

**Notify Admin:**
```bash
curl -X POST https://your-admin.vercel.app/api/bookings/notify \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d '{
    "bookingId": "booking-uuid",
    "serviceType": "Battery Jump",
    "customerName": "John Doe", 
    "customerPhone": "(555) 123-4567",
    "customerEmail": "john@example.com",
    "location": {
      "address": "New York, NY",
      "latitude": 40.7128,
      "longitude": -74.0060
    },
    "notes": "Battery died",
    "emergency": false,
    "createdAt": "2024-01-15T10:30:00Z"
  }'
```

---

*Last Updated: December 2024*  
*Version: 1.0.0*