# Test Suite for Expert Session Booking System

## Backend Tests

### Setup
```bash
npm install --save-dev jest supertest mongodb-memory-server
```

## 1. Expert Routes Tests

```javascript
// __tests__/routes/expertRoutes.test.js
const request = require('supertest');
const mongoose = require('mongoose');
const { MongoMemoryServer } = require('mongodb-memory-server');
const app = require('../../server');
const Expert = require('../../models/Expert');

let mongoServer;

beforeAll(async () => {
  mongoServer = await MongoMemoryServer.create();
  const mongoUri = mongoServer.getUri();
  await mongoose.connect(mongoUri);
});

afterAll(async () => {
  await mongoose.disconnect();
  await mongoServer.stop();
});

beforeEach(async () => {
  await Expert.deleteMany({});
});

describe('GET /experts', () => {
  test('should return empty array when no experts exist', async () => {
    const response = await request(app).get('/experts');
    expect(response.status).toBe(200);
    expect(response.body).toEqual([]);
  });

  test('should return all experts with pagination', async () => {
    // Create 7 test experts
    const experts = Array.from({ length: 7 }, (_, i) => ({
      name: `Expert ${i + 1}`,
      category: 'Doctor',
      experience: 5 + i,
      rating: 4.5,
      bio: 'Test bio',
      availableSlots: []
    }));

    await Expert.insertMany(experts);

    const response = await request(app).get('/experts?page=1');
    expect(response.status).toBe(200);
    expect(response.body.length).toBe(5); // Default limit is 5
  });

  test('should filter experts by search term (case-insensitive)', async () => {
    await Expert.create({
      name: 'Dr. John Smith',
      category: 'Doctor',
      experience: 10,
      rating: 4.8,
      bio: 'Cardiologist',
      availableSlots: []
    });

    const response = await request(app).get('/experts?search=john');
    expect(response.status).toBe(200);
    expect(response.body.length).toBe(1);
    expect(response.body[0].name).toContain('john');
  });

  test('should filter experts by category', async () => {
    await Expert.create({
      name: 'Dr. Jane Doe',
      category: 'Lawyer',
      experience: 8,
      rating: 4.6,
      bio: 'Criminal lawyer',
      availableSlots: []
    });

    await Expert.create({
      name: 'Dr. John Smith',
      category: 'Doctor',
      experience: 10,
      rating: 4.8,
      bio: 'Cardiologist',
      availableSlots: []
    });

    const response = await request(app).get('/experts?category=Lawyer');
    expect(response.status).toBe(200);
    expect(response.body.length).toBe(1);
    expect(response.body[0].category).toBe('Lawyer');
  });

  test('should handle search and category filter together', async () => {
    await Expert.create({
      name: 'Dr. John Smith',
      category: 'Doctor',
      experience: 10,
      rating: 4.8,
      bio: 'Cardiologist',
      availableSlots: []
    });

    const response = await request(app).get('/experts?search=john&category=Doctor');
    expect(response.status).toBe(200);
    expect(response.body.length).toBe(1);
  });
});

describe('GET /experts/:id', () => {
  test('should return expert by valid ID', async () => {
    const expert = await Expert.create({
      name: 'Dr. John Smith',
      category: 'Doctor',
      experience: 10,
      rating: 4.8,
      bio: 'Cardiologist',
      availableSlots: [
        { date: '2026-05-10', slots: ['09:00', '10:00'] }
      ]
    });

    const response = await request(app).get(`/experts/${expert._id}`);
    expect(response.status).toBe(200);
    expect(response.body.name).toBe('Dr. John Smith');
    expect(response.body.availableSlots.length).toBe(1);
  });

  test('should return 500 error for invalid expert ID format', async () => {
    const response = await request(app).get('/experts/invalidId');
    expect(response.status).toBe(500);
    expect(response.body.message).toBeDefined();
  });
});
```

## 2. Booking Routes Tests

```javascript
// __tests__/routes/bookingRoutes.test.js
const request = require('supertest');
const mongoose = require('mongoose');
const app = require('../../server');
const Expert = require('../../models/Expert');
const Booking = require('../../models/Booking');

let expert;

beforeEach(async () => {
  await Booking.deleteMany({});
  await Expert.deleteMany({});

  expert = await Expert.create({
    name: 'Dr. John Smith',
    category: 'Doctor',
    experience: 10,
    rating: 4.8,
    bio: 'Cardiologist',
    availableSlots: [
      { date: '2026-05-10', slots: ['09:00', '10:00', '14:00'] }
    ]
  });
});

describe('POST /bookings', () => {
  test('should create a new booking', async () => {
    const bookingData = {
      expertId: expert._id.toString(),
      name: 'Jane Doe',
      email: 'jane@example.com',
      phone: '555-1234',
      date: '2026-05-10',
      timeSlot: '09:00',
      notes: 'First consultation'
    };

    const response = await request(app)
      .post('/bookings')
      .send(bookingData);

    expect(response.status).toBe(201);
    expect(response.body.expertId).toBe(expert._id.toString());
    expect(response.body.status).toBe('Pending');
    expect(response.body.createdAt).toBeDefined();
  });

  test('should prevent duplicate bookings for same expert, date, and time slot', async () => {
    const bookingData = {
      expertId: expert._id.toString(),
      name: 'Jane Doe',
      email: 'jane@example.com',
      phone: '555-1234',
      date: '2026-05-10',
      timeSlot: '09:00',
      notes: 'First consultation'
    };

    // First booking should succeed
    await request(app).post('/bookings').send(bookingData);

    // Duplicate booking should fail
    const duplicateResponse = await request(app)
      .post('/bookings')
      .send(bookingData);

    expect(duplicateResponse.status).toBe(400);
    expect(duplicateResponse.body.message).toContain('E11000');
  });

  test('should allow bookings for same expert at different times', async () => {
    const booking1 = {
      expertId: expert._id.toString(),
      name: 'Jane Doe',
      email: 'jane@example.com',
      phone: '555-1234',
      date: '2026-05-10',
      timeSlot: '09:00',
      notes: 'First consultation'
    };

    const booking2 = {
      ...booking1,
      timeSlot: '10:00'
    };

    const response1 = await request(app).post('/bookings').send(booking1);
    const response2 = await request(app).post('/bookings').send(booking2);

    expect(response1.status).toBe(201);
    expect(response2.status).toBe(201);
  });

  test('should validate required fields', async () => {
    const incompleteData = {
      name: 'Jane Doe'
      // Missing other required fields
    };

    const response = await request(app)
      .post('/bookings')
      .send(incompleteData);

    expect(response.status).toBe(400);
  });
});

describe('GET /bookings', () => {
  test('should return all bookings', async () => {
    await Booking.create({
      expertId: expert._id,
      name: 'Jane Doe',
      email: 'jane@example.com',
      phone: '555-1234',
      date: '2026-05-10',
      timeSlot: '09:00',
      notes: 'First consultation'
    });

    const response = await request(app).get('/bookings');
    expect(response.status).toBe(200);
    expect(response.body.length).toBeGreaterThan(0);
  });
});

describe('GET /bookings/:id', () => {
  test('should return booking by ID', async () => {
    const booking = await Booking.create({
      expertId: expert._id,
      name: 'Jane Doe',
      email: 'jane@example.com',
      phone: '555-1234',
      date: '2026-05-10',
      timeSlot: '09:00',
      notes: 'First consultation'
    });

    const response = await request(app).get(`/bookings/${booking._id}`);
    expect(response.status).toBe(200);
    expect(response.body.name).toBe('Jane Doe');
  });
});

describe('PATCH /bookings/:id', () => {
  test('should update booking status', async () => {
    const booking = await Booking.create({
      expertId: expert._id,
      name: 'Jane Doe',
      email: 'jane@example.com',
      phone: '555-1234',
      date: '2026-05-10',
      timeSlot: '09:00',
      notes: 'First consultation'
    });

    const response = await request(app)
      .patch(`/bookings/${booking._id}`)
      .send({ status: 'Confirmed' });

    expect(response.status).toBe(200);
    expect(response.body.status).toBe('Confirmed');
  });

  test('should reject invalid status values', async () => {
    const booking = await Booking.create({
      expertId: expert._id,
      name: 'Jane Doe',
      email: 'jane@example.com',
      phone: '555-1234',
      date: '2026-05-10',
      timeSlot: '09:00',
      notes: 'First consultation'
    });

    const response = await request(app)
      .patch(`/bookings/${booking._id}`)
      .send({ status: 'InvalidStatus' });

    expect(response.status).toBe(400);
  });
});

describe('DELETE /bookings/:id', () => {
  test('should delete booking', async () => {
    const booking = await Booking.create({
      expertId: expert._id,
      name: 'Jane Doe',
      email: 'jane@example.com',
      phone: '555-1234',
      date: '2026-05-10',
      timeSlot: '09:00'
    });

    const response = await request(app).delete(`/bookings/${booking._id}`);
    expect(response.status).toBe(200);

    const deletedBooking = await Booking.findById(booking._id);
    expect(deletedBooking).toBeNull();
  });
});
```

## Frontend Tests

### Setup
```bash
npm install --save-dev @testing-library/react @testing-library/jest-dom jest-mock-axios
```

## 3. Experts Component Tests

```javascript
// src/__tests__/pages/Experts.test.js
import { render, screen, waitFor, fireEvent } from '@testing-library/react';
import axios from 'axios';
import Experts from '../../pages/Experts';
import { BrowserRouter } from 'react-router-dom';

jest.mock('axios');

const mockExperts = [
  {
    _id: '1',
    name: 'Dr. John Smith',
    category: 'Doctor',
    experience: 10,
    rating: 4.8
  },
  {
    _id: '2',
    name: 'Jane Lawyer',
    category: 'Lawyer',
    experience: 8,
    rating: 4.6
  }
];

describe('Experts Component', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('should render experts list', async () => {
    axios.get.mockResolvedValue({ data: mockExperts });

    render(
      <BrowserRouter>
        <Experts />
      </BrowserRouter>
    );

    await waitFor(() => {
      expect(screen.getByText('Dr. John Smith')).toBeInTheDocument();
      expect(screen.getByText('Jane Lawyer')).toBeInTheDocument();
    });
  });

  test('should filter experts by search term', async () => {
    axios.get.mockResolvedValue({ data: [mockExperts[0]] });

    render(
      <BrowserRouter>
        <Experts />
      </BrowserRouter>
    );

    const searchInput = screen.getByPlaceholderText('Search');
    fireEvent.change(searchInput, { target: { value: 'John' } });

    await waitFor(() => {
      expect(axios.get).toHaveBeenCalledWith(
        expect.stringContaining('search=John')
      );
    });
  });

  test('should filter experts by category', async () => {
    axios.get.mockResolvedValue({ data: [mockExperts[0]] });

    render(
      <BrowserRouter>
        <Experts />
      </BrowserRouter>
    );

    const categorySelect = screen.getByDisplayValue('All');
    fireEvent.change(categorySelect, { target: { value: 'Doctor' } });

    await waitFor(() => {
      expect(axios.get).toHaveBeenCalledWith(
        expect.stringContaining('category=Doctor')
      );
    });
  });

  test('should display expert cards with details', async () => {
    axios.get.mockResolvedValue({ data: mockExperts });

    render(
      <BrowserRouter>
        <Experts />
      </BrowserRouter>
    );

    await waitFor(() => {
      expect(screen.getByText('Doctor')).toBeInTheDocument();
      expect(screen.getByText('10 Years')).toBeInTheDocument();
      expect(screen.getByText('⭐ 4.8')).toBeInTheDocument();
    });
  });
});
```

## 4. ExpertDetails Component Tests

```javascript
// src/__tests__/pages/ExpertDetails.test.js
import { render, screen, waitFor, fireEvent } from '@testing-library/react';
import axios from 'axios';
import ExpertDetails from '../../pages/ExpertDetails';
import { BrowserRouter, Route, Routes } from 'react-router-dom';

jest.mock('axios');
jest.mock('socket.io-client');

const mockExpert = {
  _id: '1',
  name: 'Dr. John Smith',
  bio: 'Experienced cardiologist',
  availableSlots: [
    {
      date: '2026-05-10',
      slots: ['09:00', '10:00', '14:00']
    }
  ]
};

describe('ExpertDetails Component', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('should display expert details', async () => {
    axios.get.mockResolvedValue({ data: mockExpert });

    render(
      <BrowserRouter>
        <Routes>
          <Route path="/" element={<ExpertDetails />} />
        </Routes>
      </BrowserRouter>,
      { initialRoute: '/' }
    );

    await waitFor(() => {
      expect(screen.getByText('Dr. John Smith')).toBeInTheDocument();
      expect(screen.getByText('Experienced cardiologist')).toBeInTheDocument();
    });
  });

  test('should display available time slots', async () => {
    axios.get.mockResolvedValue({ data: mockExpert });

    render(
      <BrowserRouter>
        <Routes>
          <Route path="/" element={<ExpertDetails />} />
        </Routes>
      </BrowserRouter>
    );

    await waitFor(() => {
      expect(screen.getByText('2026-05-10')).toBeInTheDocument();
      expect(screen.getByText('09:00')).toBeInTheDocument();
      expect(screen.getByText('10:00')).toBeInTheDocument();
    });
  });

  test('should allow booking with valid data', async () => {
    axios.get.mockResolvedValue({ data: mockExpert });
    axios.post.mockResolvedValue({ data: { success: true } });

    window.alert = jest.fn();

    render(
      <BrowserRouter>
        <Routes>
          <Route path="/" element={<ExpertDetails />} />
        </Routes>
      </BrowserRouter>
    );

    await waitFor(() => {
      expect(screen.getByText('Dr. John Smith')).toBeInTheDocument();
    });

    // Fill form
    fireEvent.change(screen.getByPlaceholderText('Name'), {
      target: { value: 'Jane Doe' }
    });
    fireEvent.change(screen.getByPlaceholderText('Email'), {
      target: { value: 'jane@example.com' }
    });

    // Click time slot and submit
    fireEvent.click(screen.getByText('09:00'));
    fireEvent.click(screen.getByText('Book Slot'));

    await waitFor(() => {
      expect(window.alert).toHaveBeenCalledWith('Booking Successful');
    });
  });

  test('should display error on booking failure', async () => {
    axios.get.mockResolvedValue({ data: mockExpert });
    axios.post.mockRejectedValue({
      response: { data: { message: 'Slot already booked' } }
    });

    window.alert = jest.fn();

    render(
      <BrowserRouter>
        <Routes>
          <Route path="/" element={<ExpertDetails />} />
        </Routes>
      </BrowserRouter>
    );

    await waitFor(() => {
      fireEvent.click(screen.getByText('Book Slot'));
    });

    await waitFor(() => {
      expect(window.alert).toHaveBeenCalledWith('Slot already booked');
    });
  });
});
```

## Running Tests

```bash
# Backend tests
npm test -- __tests__/

# Frontend tests
npm test -- src/__tests__/

# With coverage
npm test -- --coverage

# Watch mode
npm test -- --watch
```

## Test Coverage Goals
- Backend: 80%+ coverage
- Frontend: 75%+ coverage
- Critical paths: 100% coverage
