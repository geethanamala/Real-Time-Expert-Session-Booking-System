# API Documentation - Expert Session Booking System

## Base URL
```
http://localhost:5000
```

---

## Authentication
Currently no authentication is implemented. Consider adding JWT tokens for production.

---

## Endpoints

### Experts API

#### 1. Get All Experts
```
GET /experts
```

**Query Parameters:**
| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `search` | string | Search by expert name (case-insensitive) | `?search=John` |
| `category` | string | Filter by category | `?category=Doctor` |
| `page` | number | Page number for pagination | `?page=2` |

**Response (200 OK):**
```json
[
  {
    "_id": "507f1f77bcf86cd799439011",
    "name": "Dr. John Smith",
    "category": "Doctor",
    "experience": 10,
    "rating": 4.8,
    "bio": "Experienced cardiologist with 10 years of practice",
    "availableSlots": [
      {
        "date": "2026-05-10",
        "slots": ["09:00", "10:00", "14:00", "15:00"]
      },
      {
        "date": "2026-05-11",
        "slots": ["09:00", "11:00", "13:00"]
      }
    ]
  }
]
```

**Example Request:**
```bash
curl -X GET "http://localhost:5000/experts?search=John&category=Doctor&page=1"
```

**Error Response (500):**
```json
{
  "message": "Database connection error"
}
```

---

#### 2. Get Expert by ID
```
GET /experts/:id
```

**Path Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | string | MongoDB ObjectId of expert |

**Response (200 OK):**
```json
{
  "_id": "507f1f77bcf86cd799439011",
  "name": "Dr. John Smith",
  "category": "Doctor",
  "experience": 10,
  "rating": 4.8,
  "bio": "Experienced cardiologist",
  "availableSlots": [
    {
      "date": "2026-05-10",
      "slots": ["09:00", "10:00"]
    }
  ]
}
```

**Error Response (400):**
```json
{
  "message": "Invalid expert ID format"
}
```

**Error Response (404):**
```json
{
  "message": "Expert not found"
}
```

**Example Request:**
```bash
curl -X GET "http://localhost:5000/experts/507f1f77bcf86cd799439011"
```

---

### Bookings API

#### 3. Create Booking
```
POST /bookings
```

**Request Body:**
```json
{
  "expertId": "507f1f77bcf86cd799439011",
  "name": "Jane Doe",
  "email": "jane@example.com",
  "phone": "555-1234567",
  "date": "2026-05-10",
  "timeSlot": "09:00",
  "notes": "First consultation, have medical history ready"
}
```

**Validation Rules:**
- `expertId`: Must be valid MongoDB ObjectId
- `name`: Required, non-empty string
- `email`: Required, valid email format
- `phone`: Required, 10-15 digits
- `date`: Required, format YYYY-MM-DD, cannot be past date
- `timeSlot`: Required, format HH:MM, must exist in expert's availableSlots
- `notes`: Optional string

**Response (201 Created):**
```json
{
  "_id": "607f1f77bcf86cd799439012",
  "expertId": "507f1f77bcf86cd799439011",
  "name": "Jane Doe",
  "email": "jane@example.com",
  "phone": "555-1234567",
  "date": "2026-05-10",
  "timeSlot": "09:00",
  "notes": "First consultation, have medical history ready",
  "status": "Pending",
  "createdAt": "2026-05-08T15:30:00Z",
  "updatedAt": "2026-05-08T15:30:00Z"
}
```

**Error Response (400 - Validation Failed):**
```json
{
  "message": "Invalid email format"
}
```

**Error Response (400 - Duplicate Booking):**
```json
{
  "message": "This slot is already booked. Please choose another time."
}
```

**Error Response (404 - Expert Not Found):**
```json
{
  "message": "Expert not found"
}
```

**Error Response (400 - Slot Not Available):**
```json
{
  "message": "Selected time slot is not available"
}
```

**Example Request:**
```bash
curl -X POST "http://localhost:5000/bookings" \
  -H "Content-Type: application/json" \
  -d '{
    "expertId": "507f1f77bcf86cd799439011",
    "name": "Jane Doe",
    "email": "jane@example.com",
    "phone": "5551234567",
    "date": "2026-05-10",
    "timeSlot": "09:00",
    "notes": "First consultation"
  }'
```

---

#### 4. Get All Bookings
```
GET /bookings
```

**Query Parameters:**
| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `status` | string | Filter by status | `?status=Pending` |
| `expertId` | string | Filter by expert | `?expertId=507f1f77bcf86cd799439011` |
| `page` | number | Page number | `?page=1` |

**Allowed Status Values:** `Pending`, `Confirmed`, `Completed`

**Response (200 OK):**
```json
[
  {
    "_id": "607f1f77bcf86cd799439012",
    "expertId": {
      "_id": "507f1f77bcf86cd799439011",
      "name": "Dr. John Smith",
      "category": "Doctor"
    },
    "name": "Jane Doe",
    "email": "jane@example.com",
    "phone": "555-1234567",
    "date": "2026-05-10",
    "timeSlot": "09:00",
    "status": "Pending",
    "createdAt": "2026-05-08T15:30:00Z",
    "updatedAt": "2026-05-08T15:30:00Z"
  }
]
```

**Example Request:**
```bash
curl -X GET "http://localhost:5000/bookings?status=Confirmed&page=1"
```

---

#### 5. Get Booking by ID
```
GET /bookings/:id
```

**Path Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | string | MongoDB ObjectId of booking |

**Response (200 OK):**
```json
{
  "_id": "607f1f77bcf86cd799439012",
  "expertId": {
    "_id": "507f1f77bcf86cd799439011",
    "name": "Dr. John Smith",
    "category": "Doctor"
  },
  "name": "Jane Doe",
  "email": "jane@example.com",
  "phone": "555-1234567",
  "date": "2026-05-10",
  "timeSlot": "09:00",
  "notes": "First consultation",
  "status": "Pending",
  "createdAt": "2026-05-08T15:30:00Z",
  "updatedAt": "2026-05-08T15:30:00Z"
}
```

**Error Response (404):**
```json
{
  "message": "Booking not found"
}
```

**Example Request:**
```bash
curl -X GET "http://localhost:5000/bookings/607f1f77bcf86cd799439012"
```

---

#### 6. Update Booking Status
```
PATCH /bookings/:id
```

**Path Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | string | MongoDB ObjectId of booking |

**Request Body:**
```json
{
  "status": "Confirmed"
}
```

**Valid Status Values:**
- `Pending` (initial status)
- `Confirmed` (expert confirmed the booking)
- `Completed` (session completed)

**Response (200 OK):**
```json
{
  "_id": "607f1f77bcf86cd799439012",
  "expertId": "507f1f77bcf86cd799439011",
  "name": "Jane Doe",
  "email": "jane@example.com",
  "phone": "555-1234567",
  "date": "2026-05-10",
  "timeSlot": "09:00",
  "status": "Confirmed",
  "createdAt": "2026-05-08T15:30:00Z",
  "updatedAt": "2026-05-08T15:35:00Z"
}
```

**Error Response (400 - Invalid Status):**
```json
{
  "message": "Invalid status. Must be Pending, Confirmed, or Completed"
}
```

**Error Response (404):**
```json
{
  "message": "Booking not found"
}
```

**Example Request:**
```bash
curl -X PATCH "http://localhost:5000/bookings/607f1f77bcf86cd799439012" \
  -H "Content-Type: application/json" \
  -d '{"status": "Confirmed"}'
```

---

#### 7. Cancel/Delete Booking
```
DELETE /bookings/:id
```

**Path Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `id` | string | MongoDB ObjectId of booking |

**Response (200 OK):**
```json
{
  "message": "Booking cancelled successfully"
}
```

**Error Response (404):**
```json
{
  "message": "Booking not found"
}
```

**Example Request:**
```bash
curl -X DELETE "http://localhost:5000/bookings/607f1f77bcf86cd799439012"
```

---

## Real-Time Events (Socket.io)

### Connection
```javascript
const socket = io('http://localhost:5000');

socket.on('connect', () => {
  console.log('Connected to server');
});
```

### Events Emitted by Server

#### 1. Slot Booked
Fired when a new booking is created
```javascript
socket.on('booking:created', (booking) => {
  console.log('New booking:', booking);
  // Update UI to reflect booked slot
});
```

**Data:**
```json
{
  "_id": "607f1f77bcf86cd799439012",
  "expertId": "507f1f77bcf86cd799439011",
  "date": "2026-05-10",
  "timeSlot": "09:00",
  "status": "Pending"
}
```

#### 2. Booking Updated
Fired when booking status changes
```javascript
socket.on('booking:updated', (booking) => {
  console.log('Booking updated:', booking);
});
```

#### 3. Booking Cancelled
Fired when booking is deleted
```javascript
socket.on('booking:cancelled', (booking) => {
  console.log('Booking cancelled:', booking);
  // Refresh available slots
});
```

---

## Common Status Codes

| Code | Meaning | Example |
|------|---------|---------|
| 200 | OK | Successful GET or PATCH |
| 201 | Created | Successful POST |
| 400 | Bad Request | Validation error, duplicate booking |
| 404 | Not Found | Resource doesn't exist |
| 500 | Server Error | Database error |

---

## Error Handling Best Practices

Always check the response status:

```javascript
try {
  const response = await axios.post('/bookings', bookingData);
  console.log('Booking successful:', response.data);
} catch (error) {
  if (error.response) {
    // Server responded with error status
    console.error('Error:', error.response.data.message);
  } else if (error.request) {
    // Request made but no response
    console.error('No response from server');
  } else {
    console.error('Error:', error.message);
  }
}
```

---

## Rate Limiting
Currently not implemented. Recommended for production:
- 100 requests per 15 minutes per IP
- Special limits for booking endpoint to prevent abuse

---

## Pagination
- Default page: 1
- Items per page: 5 (experts), 10 (bookings)
- Calculate total pages: `Math.ceil(totalCount / itemsPerPage)`

---

## Data Types Reference

| Type | Format | Example |
|------|--------|---------|
| Email | RFC 5322 | `user@example.com` |
| Phone | 10-15 digits | `5551234567` or `+1-555-123-4567` |
| Date | YYYY-MM-DD | `2026-05-10` |
| Time | HH:MM (24-hour) | `14:30` |
| ObjectId | 24 hex chars | `507f1f77bcf86cd799439011` |

---

## Testing with Postman

1. Import endpoints into Postman
2. Set environment variable: `base_url = http://localhost:5000`
3. Test each endpoint with sample data
4. Verify error responses

Example Postman collection available in `postman_collection.json`

