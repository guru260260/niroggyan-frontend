// File: src/types.ts
export interface Doctor {
  id: string;
  name: string;
  specialization: string;
  profileImage: string;
  availabilityStatus: 'Available Today' | 'Fully Booked' | 'On Leave';
  schedule: Array<{ date: string; slots: string[] }>; // ISO date strings
}

export interface AppointmentRequest {
  doctorId: string;
  patientName: string;
  email: string;
  date: string; // ISO
  time: string; // e.g. "14:30"
}

// File: src/context/DoctorContext.tsx
import React, { createContext, useContext, useState, ReactNode } from 'react';
import { Doctor, AppointmentRequest } from '../types';

// Example mock data
const MOCK_DOCTORS: Doctor[] = [
  {
    id: 'd1',
    name: 'Dr. Ananya Rao',
    specialization: 'General Physician',
    profileImage: 'https://via.placeholder.com/150',
    availabilityStatus: 'Available Today',
    schedule: [
      { date: '2025-08-02', slots: ['10:00', '11:00', '14:00'] },
      { date: '2025-08-03', slots: ['09:00', '13:00'] },
    ],
  },
  {
    id: 'd2',
    name: 'Dr. Vivek Sharma',
    specialization: 'Dermatologist',
    profileImage: 'https://via.placeholder.com/150',
    availabilityStatus: 'Fully Booked',
    schedule: [
      { date: '2025-08-02', slots: [] },
      { date: '2025-08-04', slots: ['15:00'] },
    ],
  },
];

interface DoctorContextValue {
  doctors: Doctor[];
  searchTerm: string;
  setSearchTerm: (t: string) => void;
  filtered: Doctor[];
  bookAppointment: (appt: AppointmentRequest) => Promise<boolean>;
}

const DoctorContext = createContext<DoctorContextValue | undefined>(undefined);

export const useDoctors = () => {
  const ctx = useContext(DoctorContext);
  if (!ctx) throw new Error('useDoctors must be inside DoctorProvider');
  return ctx;
};

export const DoctorProvider = ({ children }: { children: ReactNode }) => {
  const [doctors] = useState<Doctor[]>(MOCK_DOCTORS);
  const [searchTerm, setSearchTerm] = useState('');

  const filtered = doctors.filter((d) => {
    const lower = searchTerm.toLowerCase();
    return (
      d.name.toLowerCase().includes(lower) ||
      d.specialization.toLowerCase().includes(lower)
    );
  });

  const bookAppointment = async (appt: AppointmentRequest) => {
    // In a real app, POST to backend. Here we simulate delay and success.
    return new Promise<boolean>((resolve) => {
      setTimeout(() => resolve(true), 500);
    });
  };

  return (
    <DoctorContext.Provider
      value={{ doctors, searchTerm, setSearchTerm, filtered, bookAppointment }}
    >
      {children}
    </DoctorContext.Provider>
  );
};

// File: src/components/DoctorCard.tsx
import React from 'react';
import { Doctor } from '../types';
import { Link } from 'react-router-dom';

interface Props {
  doctor: Doctor;
}

const statusColors: Record<string, string> = {
  'Available Today': 'bg-green-100 text-green-800',
  'Fully Booked': 'bg-yellow-100 text-yellow-800',
  'On Leave': 'bg-red-100 text-red-800',
};

export const DoctorCard: React.FC<Props> = ({ doctor }) => {
  return (
    <div className="p-4 border rounded-2xl shadow-sm flex gap-4 items-center">
      <img
        src={doctor.profileImage}
        alt={doctor.name}
        className="w-16 h-16 rounded-full object-cover"
      />
      <div className="flex-1">
        <h2 className="text-lg font-semibold">{doctor.name}</h2>
        <p className="text-sm text-gray-600">{doctor.specialization}</p>
        <div
          className={`inline-block mt-1 px-2 py-1 rounded-full text-xs font-medium ${
            statusColors[doctor.availabilityStatus]
          }`}
        >
          {doctor.availabilityStatus}
        </div>
      </div>
      <Link to={`/doctor/${doctor.id}`}> 
        <button className="px-4 py-2 bg-indigo-600 text-white rounded-xl shadow hover:opacity-90">
          View
        </button>
      </Link>
    </div>
  );
};

// File: src/components/BookingForm.tsx
import React, { useState } from 'react';
import { AppointmentRequest, Doctor } from '../types';
import { useDoctors } from '../context/DoctorContext';

interface Props {
  doctor: Doctor;
  onSuccess?: () => void;
}

export const BookingForm: React.FC<Props> = ({ doctor, onSuccess }) => {
  const { bookAppointment } = useDoctors();
  const [patientName, setPatientName] = useState('');
  const [email, setEmail] = useState('');
  const [date, setDate] = useState('');
  const [time, setTime] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);
  const [confirmed, setConfirmed] = useState(false);

  const validate = () => {
    if (!patientName.trim()) return 'Patient name is required';
    if (!email.trim() || !email.includes('@')) return 'Valid email is required';
    if (!date) return 'Date is required';
    if (!time) return 'Time is required';
    return '';
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const v = validate();
    if (v) {
      setError(v);
      return;
    }
    setError('');
    setLoading(true);
    const appt: AppointmentRequest = {
      doctorId: doctor.id,
      patientName,
      email,
      date,
      time,
    };
    const success = await bookAppointment(appt);
    setLoading(false);
    if (success) {
      setConfirmed(true);
      onSuccess?.();
    } else {
      setError('Failed to book. Try again.');
    }
  };

  if (confirmed) {
    return (
      <div className="p-4 bg-green-50 rounded-lg">
        <h3 className="font-semibold">Appointment Confirmed!</h3>
        <p>
          You have booked with {doctor.name} on {date} at {time}. A confirmation has been sent to {email}.
        </p>
      </div>
    );
  }

  return (
    <form onSubmit={handleSubmit} className="space-y-3 max-w-md">
      {error && <div className="text-sm text-red-600">{error}</div>}
      <div>
        <label className="block text-sm font-medium">Patient Name</label>
        <input
          type="text"
          value={patientName}
          onChange={(e) => setPatientName(e.target.value)}
          className="w-full border rounded px-3 py-2 mt-1"
          required
        />
      </div>
      <div>
        <label className="block text-sm font-medium">Email</label>
        <input
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          className="w-full border rounded px-3 py-2 mt-1"
          required
        />
      </div>
      <div>
        <label className="block text-sm font-medium">Date</label>
        <input
          type="date"
          value={date}
          onChange={(e) => setDate(e.target.value)}
          className="w-full border rounded px-3 py-2 mt-1"
          required
        />
      </div>
      <div>
        <label className="block text-sm font-medium">Time</label>
        <input
          type="time"
          value={time}
          onChange={(e) => setTime(e.target.value)}
          className="w-full border rounded px-3 py-2 mt-1"
          required
        />
      </div>
      <button
        type="submit"
        disabled={loading}
        className="mt-2 px-5 py-2 bg-indigo-600 text-white rounded-lg shadow disabled:opacity-50"
      >
        {loading ? 'Booking...' : 'Book Appointment'}
      </button>
    </form>
  );
};

// File: src/components/DoctorProfile.tsx
import React from 'react';
import { useParams, useNavigate } from 'react-router-dom';
import { useDoctors } from '../context/DoctorContext';
import { BookingForm } from './BookingForm';

export const DoctorProfile: React.FC = () => {
  const { doctorId } = useParams<{ doctorId: string }>();
  const { doctors } = useDoctors();
  const doctor = doctors.find((d) => d.id === doctorId);
  const navigate = useNavigate();

  if (!doctor) {
    return <div className="p-6">Doctor not found.</div>;
  }

  return (
    <div className="max-w-4xl mx-auto p-6 space-y-6">
      <button onClick={() => navigate(-1)} className="text-sm underline">
        &larr; Back
      </button>
      <div className="flex flex-col md:flex-row gap-6">
        <div className="flex-shrink-0">
          <img
            src={doctor.profileImage}
            alt={doctor.name}
            className="w-32 h-32 rounded-full object-cover"
          />
        </div>
        <div className="flex-1 space-y-2">
          <h1 className="text-2xl font-bold">{doctor.name}</h1>
          <p className="text-gray-700">{doctor.specialization}</p>
          <div className="inline-block mt-1 px-3 py-1 rounded-full bg-indigo-100 text-indigo-800 text-sm">
            {doctor.availabilityStatus}
          </div>
          <div className="mt-4">
            <h2 className="font-semibold">Availability Schedule</h2>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4 mt-2">
              {doctor.schedule.map((s) => (
                <div key={s.date} className="p-3 border rounded">
                  <div className="text-sm font-medium">{s.date}</div>
                  <div className="mt-1 flex flex-wrap gap-2">
                    {s.slots.length ? (
                      s.slots.map((slot) => (
                        <div
                          key={slot}
                          className="text-xs px-2 py-1 bg-green-50 rounded"
                        >
                          {slot}
                        </div>
                      ))
                    ) : (
                      <div className="text-xs text-gray-500">No slots</div>
                    )}
                  </div>
                </div>
              ))}
            </div>
          </div>
        </div>
      </div>
      <div className="bg-white p-6 rounded-2xl shadow">
        <h2 className="text-xl font-semibold mb-3">Book Appointment</h2>
        <BookingForm doctor={doctor} />
      </div>
    </div>
  );
};

// File: src/App.tsx
import React from 'react';
import { BrowserRouter, Routes, Route, Link } from 'react-router-dom';
import { DoctorProvider, useDoctors } from './context/DoctorContext';
import { DoctorCard } from './components/DoctorCard';
import { DoctorProfile } from './components/DoctorProfile';

const LandingPage: React.FC = () => {
  const { filtered, searchTerm, setSearchTerm } = useDoctors();
  return (
    <div className="p-6 max-w-5xl mx-auto space-y-6">
      <header className="flex flex-col md:flex-row justify-between items-start md:items-center gap-4">
        <div>
          <h1 className="text-3xl font-bold">Find a Doctor</h1>
          <p className="text-gray-600 mt-1">Book healthcare appointments quickly.</p>
        </div>
        <div className="w-full md:w-1/3">
          <input
            aria-label="Search doctors"
            placeholder="Search by name or specialization"
            value={searchTerm}
            onChange={(e) => setSearchTerm(e.target.value)}
            className="w-full border rounded px-4 py-2"
          />
        </div>
      </header>
      <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
        {filtered.map((d) => (
          <DoctorCard key={d.id} doctor={d} />
        ))}
        {filtered.length === 0 && <div>No doctors match your search.</div>}
      </div>
    </div>
  );
};

const App: React.FC = () => {
  return (
    <DoctorProvider>
      <BrowserRouter>
        <div className="min-h-screen bg-gray-50">
          <Routes>
            <Route path="/" element={<LandingPage />} />
            <Route path="/doctor/:doctorId" element={<DoctorProfile />} />
          </Routes>
        </div>
      </BrowserRouter>
    </DoctorProvider>
  );
};

export default App;

// File: src/index.tsx
import React from 'react';
import { createRoot } from 'react-dom/client';
import App from './App';
import './index.css'; // Tailwind should be imported here

const container = document.getElementById('root');
if (container) {
  createRoot(container).render(<App />);
}

/*
  Tailwind setup (assumed):
  - tailwind.config.js with content paths including ./src/**/*.{ts,tsx}
  - index.css includes:
      @tailwind base;
      @tailwind components;
      @tailwind utilities;
  Suggested scripts in package.json:
    "start": "vite",
    "build": "vite build"
  Dependencies: react, react-dom, react-router-dom, typescript, @types/react, @types/react-dom, @types/react-router-dom
*/
