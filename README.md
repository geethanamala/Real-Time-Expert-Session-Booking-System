# Real-Time-Expert-Session-Booking-System
{
  "name": "backend",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "dev": "nodemon server.js",
    "start": "node server.js"
  },
  "dependencies": {
    "cors": "^2.8.5",
    "dotenv": "^16.4.5",
    "express": "^4.19.2",
    "mongoose": "^8.5.1",
    "nodemon": "^3.1.4",
    "socket.io": "^4.7.5"
  }
}
PORT=5000
MONGO_URI=your_mongodb_connection 
const mongoose = require('mongoose');

const connectDB = async () => {

    try {

        await mongoose.connect(process.env.MONGO_URI);

        console.log("MongoDB Connected");

    } catch (error) {

        console.log(error);

        process.exit(1);
    }
};

module.exports = connectDB;
const mongoose = require('mongoose');

const expertSchema = new mongoose.Schema({

    name: String,

    category: String,

    experience: Number,

    rating: Number,

    bio: String,

    availableSlots: [
        {
            date: String,
            slots: [String]
        }
    ]
});

module.exports = mongoose.model("Expert", expertSchema);
const mongoose = require('mongoose');

const bookingSchema = new mongoose.Schema({

    expertId: {
        type: mongoose.Schema.Types.ObjectId,
        ref: 'Expert'
    },

    name: String,

    email: String,

    phone: String,

    date: String,

    timeSlot: String,

    notes: String,

    status: {
        type: String,
        enum: ['Pending', 'Confirmed', 'Completed'],
        default: 'Pending'
    }

}, { timestamps: true });

bookingSchema.index(
    {
        expertId: 1,
        date: 1,
        timeSlot: 1
    },
    {
        unique: true
    }
);

module.exports = mongoose.model("Booking", bookingSchema);
const express = require('express');
const router = express.Router();

const Expert = require('../models/Expert');

router.get('/', async (req, res) => {

    try {

        const page = parseInt(req.query.page) || 1;

        const limit = 5;

        const search = req.query.search || '';

        const category = req.query.category || '';

        const query = {};

        if (search) {

            query.name = {
                $regex: search,
                $options: 'i'
            };
        }

        if (category) {

            query.category = category;
        }

        const experts = await Expert.find(query)
            .skip((page - 1) * limit)
            .limit(limit);

        res.json(experts);

    } catch (error) {

        res.status(500).json({
            message: error.message
        });
    }
});

router.get('/:id', async (req, res) => {

    try {

        const expert = await Expert.findById(req.params.id);

        res.json(expert);

    } catch (error) {

        res.status(500).json({
            message: error.message
        });
    }
});

module.exports = router;
require('dotenv').config();

const express = require('express');
const cors = require('cors');
const http = require('http');

const { Server } = require('socket.io');

const connectDB = require('./config/db');

const expertRoutes = require('./routes/expertRoutes');

const bookingRoutes = require('./routes/bookingRoutes');

const app = express();

const server = http.createServer(app);

const io = new Server(server, {
    cors: {
        origin: "*"
    }
});

global.io = io;

connectDB();

app.use(cors());

app.use(express.json());

app.use('/experts', expertRoutes);

app.use('/bookings', bookingRoutes);

io.on('connection', (socket) => {

    console.log('User Connected');
});

const PORT = process.env.PORT || 5000;

server.listen(PORT, () => {

    console.log(`Server Running On ${PORT}`);
});
require('dotenv').config();

const express = require('express');
const cors = require('cors');
const http = require('http');

const { Server } = require('socket.io');

const connectDB = require('./config/db');

const expertRoutes = require('./routes/expertRoutes');

const bookingRoutes = require('./routes/bookingRoutes');

const app = express();

const server = http.createServer(app);

const io = new Server(server, {
    cors: {
        origin: "*"
    }
});

global.io = io;

connectDB();

app.use(cors());

app.use(express.json());

app.use('/experts', expertRoutes);

app.use('/bookings', bookingRoutes);

io.on('connection', (socket) => {

    console.log('User Connected');
});

const PORT = process.env.PORT || 5000;

server.listen(PORT, () => {

    console.log(`Server Running On ${PORT}`);
});
{
  "name": "frontend",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "axios": "^1.7.2",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "react-router-dom": "^6.24.1",
    "react-scripts": "5.0.1",
    "socket.io-client": "^4.7.5"
  },
  "scripts": {
    "start": "react-scripts start"
  }
}
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';
import './App.css';

const root = ReactDOM.createRoot(document.getElementById('root'));

root.render(
    <App />
);
import { BrowserRouter, Routes, Route } from 'react-router-dom';

import Experts from './pages/Experts';

import ExpertDetails from './pages/ExpertDetails';

import MyBookings from './pages/MyBookings';

import Navbar from './components/Navbar';

function App() {

    return (

        <BrowserRouter>

            <Navbar />

            <Routes>

                <Route path="/" element={<Experts />} />

                <Route path="/expert/:id" element={<ExpertDetails />} />

                <Route path="/bookings" element={<MyBookings />} />

            </Routes>

        </BrowserRouter>
    );
}

export default App;
import { useEffect, useState } from 'react';

import axios from 'axios';

import { Link } from 'react-router-dom';

function Experts() {

    const [experts, setExperts] = useState([]);

    const [search, setSearch] = useState('');

    const [category, setCategory] = useState('');

    useEffect(() => {

        fetchExperts();

    }, [search, category]);

    const fetchExperts = async () => {

        const res = await axios.get(
            `http://localhost:5000/experts?search=${search}&category=${category}`
        );

        setExperts(res.data);
    };

    return (

        <div className="container">

            <h2>Experts</h2>

            <input
                placeholder="Search"
                onChange={(e) => setSearch(e.target.value)}
            />

            <select onChange={(e) => setCategory(e.target.value)}>

                <option value="">All</option>

                <option value="Doctor">Doctor</option>

                <option value="Lawyer">Lawyer</option>

                <option value="Teacher">Teacher</option>

            </select>

            {
                experts.map((expert) => (

                    <div className="card" key={expert._id}>

                        <h3>{expert.name}</h3>

                        <p>{expert.category}</p>

                        <p>{expert.experience} Years</p>

                        <p>⭐ {expert.rating}</p>

                        <Link to={`/expert/${expert._id}`}>
                            View Details
                        </Link>

                    </div>
                ))
            }

        </div>
    );
}

export default Experts;
import { useEffect, useState } from 'react';

import axios from 'axios';

import { useParams } from 'react-router-dom';

import io from 'socket.io-client';

const socket = io('http://localhost:5000');

function ExpertDetails() {

    const { id } = useParams();

    const [expert, setExpert] = useState(null);

    const [formData, setFormData] = useState({

        name: '',
        email: '',
        phone: '',
        date: '',
        timeSlot: '',
        notes: ''
    });

    useEffect(() => {

        fetchExpert();

        socket.on("slotBooked", () => {

            fetchExpert();
        });

    }, []);

    const fetchExpert = async () => {

        const res = await axios.get(
            `http://localhost:5000/experts/${id}`
        );

        setExpert(res.data);
    };

    const handleBooking = async () => {

        try {

            await axios.post(
                'http://localhost:5000/bookings',
                {
                    ...formData,
                    expertId: id
                }
            );

            alert('Booking Successful');

        } catch (error) {

            alert(error.response.data.message);
        }
    };

    if (!expert) return <h2>Loading...</h2>;

    return (

        <div className="container">

            <h2>{expert.name}</h2>

            <p>{expert.bio}</p>

            {
                expert.availableSlots.map((item, index) => (

                    <div key={index}>

                        <h4>{item.date}</h4>

                        {
                            item.slots.map((slot, i) => (

                                <button
                                    key={i}
                                    onClick={() =>
                                        setFormData({
                                            ...formData,
                                            date: item.date,
                                            timeSlot: slot
                                        })
                                    }
                                >
                                    {slot}
                                </button>
                            ))
                        }

                    </div>
                ))
            }

            <input
                placeholder="Name"
                onChange={(e) =>
                    setFormData({
                        ...formData,
                        name: e.target.value
                    })
                }
            />

            <input
                placeholder="Email"
                onChange={(e) =>
                    setFormData({
                        ...formData,
                        email: e.target.value
                    })
                }
            />

            <input
                placeholder="Phone"
                onChange={(e) =>
                    setFormData({
                        ...formData,
                        phone: e.target.value
                    })
                }
            />

            <textarea
                placeholder="Notes"
                onChange={(e) =>
                    setFormData({
                        ...formData,
                        notes: e.target.value
                    })
                }
            />

            <button onClick={handleBooking}>
                Book Slot
            </button>

        </div>
    );
}

export default ExpertDetails;
body {
    font-family: Arial;
    margin: 0;
    background: #f5f5f5;
}

.navbar {
    background: black;
    padding: 15px;
}

.navbar a {
    color: white;
    margin-right: 20px;
    text-decoration: none;
}

.container {
    padding: 20px;
}

.card {
    background: white;
    padding: 20px;
    margin-top: 20px;
    border-radius: 10px;
}

input,
select,
textarea,
button {
    display: block;
    margin-top: 10px;
    padding: 10px;
    width: 300px;
}

button {
    cursor: pointer;
}
