# Call Recording Management System - Technical Specification

**Version:** 1.0  
**Date:** October 30, 2025  
**Purpose:** Personal research and evidence collection system for managing ~3000 call recordings

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture](#2-architecture)
3. [Technology Stack](#3-technology-stack)
4. [Database Schema](#4-database-schema)
5. [Backend API Specification](#5-backend-api-specification)
6. [Frontend Application](#6-frontend-application)
7. [File Processing Pipeline](#7-file-processing-pipeline)
8. [Docker Configuration](#8-docker-configuration)
9. [Security Considerations](#9-security-considerations)
10. [Development Workflow](#10-development-workflow)
11. [API Documentation](#11-api-documentation)
12. [Testing Strategy](#12-testing-strategy)
13. [Deployment Instructions](#13-deployment-instructions)
14. [Future Enhancements](#14-future-enhancements)

---

## 1. System Overview

### 1.1 Purpose

This system is designed to index, search, and manage approximately 3000 call recording files (AMR format) for personal research and legal evidence collection purposes. The system provides a web-based interface for efficient searching, filtering, and playback of recordings.

### 1.2 Key Features

- **Automated Indexing**: Scan and index 3000+ AMR recording files
- **Metadata Extraction**: Extract date, time, duration, phone number, file size from filenames and file metadata
- **Contact Integration**: Import VCF contacts and cross-reference with phone numbers
- **Advanced Search & Filtering**: Multi-parameter search (date range, phone number, contact, day of week, duration)
- **Direct Audio Playback**: Stream and play recordings directly in the web interface
- **Local-First Architecture**: All data and processing happens locally on Ubuntu machine
- **Docker-Based Deployment**: Containerized for easy setup and deterministic execution

### 1.3 Filename Format

The system handles the following filename patterns:

```
0521234555-sim1-voicecall-20250619145807.amr
*9074-sim1-voicecall-20250605223634.amr
```

**Format breakdown:**
- **Phone Number**: `0521234555` or `*9074` (for special numbers)
- **SIM Card**: `sim1` (identifier)
- **Call Type**: `voicecall`
- **Timestamp**: `YYYYMMDDHHMMSS` (20250619145807 = June 19, 2025, 14:58:07)

---

## 2. Architecture

### 2.1 System Architecture

```
┌─────────────────────────────────────────────────┐
│              User Browser                       │
│          (http://localhost:3000)                │
└───────────────────┬─────────────────────────────┘
                    │
                    │ HTTP/REST
                    │
┌───────────────────▼─────────────────────────────┐
│         Frontend Container (React)              │
│   Vite + React + TypeScript + Tailwind         │
│              + shadcn/ui                        │
└───────────────────┬─────────────────────────────┘
                    │
                    │ API Calls
                    │
┌───────────────────▼─────────────────────────────┐
│         Backend Container (Flask)               │
│      Python 3.12 + Flask + SQLAlchemy          │
│     + Pydub + FFmpeg + vobject                 │
└───────────────┬─────────────┬───────────────────┘
                │             │
                │             │ Direct File Access
                │             │
┌───────────────▼─────┐   ┌───▼─────────────────┐
│  PostgreSQL         │   │  Local File System  │
│  Container          │   │  (Audio Files)      │
│  (Database)         │   │  Bind Mount         │
└─────────────────────┘   └─────────────────────┘
```

### 2.2 Design Principles

- **Local-First**: No cloud dependencies, all processing local
- **Privacy-Focused**: Sensitive call data stays on local machine
- **Containerized**: Docker ensures consistent, reproducible environment
- **Direct File Access**: Audio files accessed via absolute paths, no duplication
- **Modular**: Each component (frontend, backend, database) is independent
- **Stateless Backend**: API is stateless for scalability

---

## 3. Technology Stack

### 3.1 Frontend

| Technology | Version | Purpose |
|------------|---------|---------|
| **React** | 18.3+ | UI framework |
| **TypeScript** | 5.6+ | Type safety |
| **Vite** | 5.4+ | Build tool & dev server |
| **Tailwind CSS** | 3.4+ | Styling framework |
| **shadcn/ui** | Latest | Component library |
| **Axios** | 1.7+ | HTTP client |
| **React Router** | 6.26+ | Routing |
| **Zustand** | 4.5+ | State management (lightweight alternative to Redux) |

### 3.2 Backend

| Technology | Version | Purpose |
|------------|---------|---------|
| **Python** | 3.12 | Runtime |
| **Flask** | 3.0+ | Web framework |
| **Flask-CORS** | 5.0+ | CORS handling |
| **Flask-SQLAlchemy** | 3.1+ | ORM |
| **Flask-Migrate** | 4.0+ | Database migrations |
| **Pydub** | 0.25.1+ | Audio processing |
| **FFmpeg** | Latest | Audio metadata extraction |
| **vobject** | 0.9.7+ | VCF parsing |
| **Gunicorn** | 22.0+ | Production WSGI server |
| **psycopg2-binary** | 2.9+ | PostgreSQL adapter |

### 3.3 Database

| Technology | Version | Purpose |
|------------|---------|---------|
| **PostgreSQL** | 16 | Primary database |

### 3.4 Infrastructure

| Technology | Version | Purpose |
|------------|---------|---------|
| **Docker** | 24.0+ | Containerization |
| **Docker Compose** | 2.23+ | Multi-container orchestration |

---

## 4. Database Schema

### 4.1 Contacts Table

Stores contact information imported from VCF file.

```sql
CREATE TABLE contacts (
    id SERIAL PRIMARY KEY,
    phone_number VARCHAR(20) NOT NULL UNIQUE,
    contact_name VARCHAR(255),
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX idx_contacts_phone ON contacts(phone_number);
CREATE INDEX idx_contacts_name ON contacts(contact_name);
```

**Column Descriptions:**
- `id`: Auto-incrementing primary key
- `phone_number`: VARCHAR to support international formats and leading zeros
- `contact_name`: Display name from VCF
- `notes`: Optional field for additional information
- `created_at`, `updated_at`: Audit timestamps

### 4.2 Recordings Table

Main table storing all indexed call recordings.

```sql
CREATE TABLE recordings (
    id SERIAL PRIMARY KEY,
    file_path VARCHAR(500) NOT NULL UNIQUE,
    file_name VARCHAR(255) NOT NULL,
    phone_number VARCHAR(20) NOT NULL,
    contact_id INTEGER REFERENCES contacts(id) ON DELETE SET NULL,
    
    -- Date/Time fields
    call_date DATE NOT NULL,
    call_time TIME NOT NULL,
    call_datetime TIMESTAMP NOT NULL,
    day_of_week VARCHAR(20) NOT NULL,
    
    -- Audio metadata
    duration_seconds INTEGER,
    duration_formatted VARCHAR(20),
    
    -- File info
    file_size_bytes BIGINT NOT NULL,
    file_size_formatted VARCHAR(20),
    audio_format VARCHAR(10) DEFAULT 'amr',
    
    -- Audit
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    indexed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Performance Indexes
CREATE INDEX idx_recordings_phone ON recordings(phone_number);
CREATE INDEX idx_recordings_contact ON recordings(contact_id);
CREATE INDEX idx_recordings_date ON recordings(call_date DESC);
CREATE INDEX idx_recordings_datetime ON recordings(call_datetime DESC);
CREATE INDEX idx_recordings_day ON recordings(day_of_week);
CREATE INDEX idx_recordings_duration ON recordings(duration_seconds);

-- Composite index for common queries
CREATE INDEX idx_recordings_phone_date ON recordings(phone_number, call_date DESC);
```

**Column Descriptions:**
- `file_path`: Absolute path to the AMR file on local filesystem
- `file_name`: Original filename
- `phone_number`: Extracted from filename
- `contact_id`: Foreign key to contacts (nullable)
- `call_datetime`: Combined date+time for efficient range queries
- `day_of_week`: Hebrew day name (ראשון, שני, etc.) for filtering
- `duration_seconds`: Audio length in seconds for filtering
- `duration_formatted`: Human-readable duration (HH:MM:SS)
- `file_size_bytes`: Raw file size for sorting/filtering
- `file_size_formatted`: Human-readable size (MB, KB)

### 4.3 Import Logs Table

Tracks import/indexing operations for debugging and auditing.

```sql
CREATE TABLE import_logs (
    id SERIAL PRIMARY KEY,
    import_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    import_type VARCHAR(50),  -- 'contacts' or 'recordings'
    total_files INTEGER,
    successful_imports INTEGER,
    failed_imports INTEGER,
    errors TEXT,
    duration_seconds INTEGER
);
```

---

## 5. Backend API Specification

### 5.1 Project Structure

```
backend/
├── app/
│   ├── __init__.py              # Flask app factory
│   ├── models.py                # SQLAlchemy models
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── contacts.py          # Contact endpoints
│   │   ├── recordings.py        # Recording endpoints
│   │   └── stats.py             # Statistics endpoints
│   ├── services/
│   │   ├── __init__.py
│   │   ├── audio_processor.py   # Audio metadata extraction
│   │   ├── contact_importer.py  # VCF import logic
│   │   └── file_scanner.py      # Directory scanning
│   └── utils/
│       ├── __init__.py
│       ├── validators.py        # Input validation
│       └── helpers.py           # Helper functions
├── migrations/                   # Alembic migrations
├── tests/
│   ├── test_contacts.py
│   ├── test_recordings.py
│   └── test_audio_processor.py
├── config.py                    # Configuration classes
├── requirements.txt             # Python dependencies
├── Dockerfile                   # Backend container
└── entrypoint.sh               # Container startup script
```

### 5.2 Core API Endpoints

#### 5.2.1 Contacts API

**Import VCF File**
```
POST /api/contacts/import
Content-Type: multipart/form-data

Body:
- file: VCF file

Response:
{
  "success": true,
  "imported": 150,
  "skipped": 5,
  "errors": []
}
```

**List Contacts**
```
GET /api/contacts?page=1&per_page=50&search=john

Response:
{
  "contacts": [
    {
      "id": 1,
      "phone_number": "+972501234567",
      "contact_name": "John Doe",
      "recording_count": 12
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 50,
    "total": 150,
    "pages": 3
  }
}
```

**Get Contact Details**
```
GET /api/contacts/{id}

Response:
{
  "id": 1,
  "phone_number": "+972501234567",
  "contact_name": "John Doe",
  "notes": null,
  "created_at": "2025-10-15T10:30:00Z",
  "recording_count": 12
}
```

#### 5.2.2 Recordings API

**Scan Directory**
```
POST /api/recordings/scan
Content-Type: application/json

Body:
{
  "directory_path": "/home/user/recordings"
}

Response:
{
  "success": true,
  "total": 3000,
  "successful": 2987,
  "failed": 13,
  "duration_seconds": 245,
  "errors": [
    "Failed to parse: invalid-filename.amr"
  ]
}
```

**Search Recordings**
```
GET /api/recordings?
  page=1&
  per_page=50&
  phone_number=0521234555&
  contact_id=5&
  date_from=2025-01-01&
  date_to=2025-06-30&
  day_of_week=שלישי&
  duration_min=60&
  duration_max=600&
  sort_by=call_datetime&
  sort_order=desc

Response:
{
  "recordings": [
    {
      "id": 1,
      "file_name": "0521234555-sim1-voicecall-20250619145807.amr",
      "phone_number": "0521234555",
      "contact_name": "John Doe",
      "call_date": "2025-06-19",
      "call_time": "14:58:07",
      "day_of_week": "חמישי",
      "duration": "00:03:45",
      "duration_seconds": 225,
      "file_size": "1.2 MB",
      "file_size_bytes": 1258291
    }
  ],
  "pagination": {
    "page": 1,
    "per_page": 50,
    "total": 245,
    "pages": 5
  },
  "summary": {
    "total_duration_seconds": 15678,
    "total_size_bytes": 987654321
  }
}
```

**Get Recording Details**
```
GET /api/recordings/{id}

Response:
{
  "id": 1,
  "file_path": "/home/user/recordings/0521234555-sim1-voicecall-20250619145807.amr",
  "file_name": "0521234555-sim1-voicecall-20250619145807.amr",
  "phone_number": "0521234555",
  "contact": {
    "id": 5,
    "contact_name": "John Doe"
  },
  "call_date": "2025-06-19",
  "call_time": "14:58:07",
  "call_datetime": "2025-06-19T14:58:07Z",
  "day_of_week": "חמישי",
  "duration": "00:03:45",
  "duration_seconds": 225,
  "file_size": "1.2 MB",
  "file_size_bytes": 1258291,
  "audio_format": "amr",
  "indexed_at": "2025-10-15T10:30:00Z"
}
```

**Stream Audio**
```
GET /api/recordings/{id}/audio

Response:
Content-Type: audio/amr
Content-Length: 1258291
X-File-Name: 0521234555-sim1-voicecall-20250619145807.amr

[Binary audio data stream]
```

#### 5.2.3 Statistics API

**System Overview**
```
GET /api/stats/overview

Response:
{
  "total_recordings": 3000,
  "total_contacts": 150,
  "total_duration_hours": 245.5,
  "total_size_gb": 2.8,
  "date_range": {
    "earliest": "2024-01-15",
    "latest": "2025-10-28"
  },
  "recordings_by_day": {
    "ראשון": 420,
    "שני": 450,
    "שלישי": 438,
    "רביעי": 445,
    "חמישי": 410,
    "שישי": 387,
    "שבת": 450
  }
}
```

**Top Contacts by Call Count**
```
GET /api/stats/top-contacts?limit=10

Response:
{
  "contacts": [
    {
      "contact_name": "John Doe",
      "phone_number": "+972501234567",
      "recording_count": 156,
      "total_duration_seconds": 23456
    }
  ]
}
```

### 5.3 Filename Parsing Logic

```python
import re
from datetime import datetime

def parse_recording_filename(filename: str) -> dict:
    """
    Parse AMR recording filename to extract metadata.
    
    Formats:
    - 0521234555-sim1-voicecall-20250619145807.amr
    - *9074-sim1-voicecall-20250605223634.amr
    
    Returns:
        dict with keys: phone_number, sim_card, call_type, 
                       date, time, datetime_obj, day_of_week
    """
    # Pattern: PHONE-SIMCARD-CALLTYPE-TIMESTAMP.amr
    pattern = r'^([0-9*]+)-sim(\d+)-voicecall-(\d{14})\.amr$'
    
    match = re.match(pattern, filename)
    if not match:
        raise ValueError(f"Invalid filename format: {filename}")
    
    phone_number = match.group(1)
    sim_card = match.group(2)
    timestamp_str = match.group(3)  # YYYYMMDDHHMMSS
    
    # Parse timestamp
    datetime_obj = datetime.strptime(timestamp_str, '%Y%m%d%H%M%S')
    
    # Hebrew day names
    hebrew_days = ['ראשון', 'שני', 'שלישי', 'רביעי', 'חמישי', 'שישי', 'שבת']
    day_of_week = hebrew_days[datetime_obj.weekday()]
    
    return {
        'phone_number': phone_number,
        'sim_card': sim_card,
        'call_type': 'voicecall',
        'date': datetime_obj.date(),
        'time': datetime_obj.time(),
        'datetime_obj': datetime_obj,
        'day_of_week': day_of_week
    }
```

### 5.4 Audio Metadata Extraction

```python
from pydub import AudioSegment
from pydub.utils import mediainfo
import os

def extract_audio_metadata(file_path: str) -> dict:
    """
    Extract audio duration and other metadata from AMR file.
    
    Args:
        file_path: Absolute path to AMR file
        
    Returns:
        dict with duration_seconds and duration_formatted
    """
    try:
        # Load audio file
        audio = AudioSegment.from_file(file_path, format="amr")
        
        # Duration in milliseconds, convert to seconds
        duration_seconds = int(len(audio) / 1000)
        
        # Format as HH:MM:SS
        hours = duration_seconds // 3600
        minutes = (duration_seconds % 3600) // 60
        seconds = duration_seconds % 60
        duration_formatted = f"{hours:02d}:{minutes:02d}:{seconds:02d}"
        
        return {
            'duration_seconds': duration_seconds,
            'duration_formatted': duration_formatted
        }
    except Exception as e:
        print(f"Error extracting audio metadata from {file_path}: {e}")
        return {
            'duration_seconds': 0,
            'duration_formatted': '00:00:00'
        }

def get_file_size(file_path: str) -> dict:
    """Get file size in bytes and formatted."""
    size_bytes = os.path.getsize(file_path)
    
    # Format size
    for unit in ['B', 'KB', 'MB', 'GB']:
        if size_bytes < 1024.0:
            size_formatted = f"{size_bytes:.2f} {unit}"
            break
        size_bytes /= 1024.0
    else:
        size_formatted = f"{size_bytes:.2f} TB"
    
    return {
        'file_size_bytes': os.path.getsize(file_path),
        'file_size_formatted': size_formatted
    }
```

### 5.5 Contact Import (VCF)

```python
import vobject
import re

def normalize_phone_number(phone: str) -> str:
    """
    Normalize phone number for consistent storage.
    - Remove spaces, dashes, parentheses
    - Convert Israeli 0XX to +972XX format
    """
    # Remove all non-digit characters except + and *
    clean = re.sub(r'[^\d+*]', '', phone)
    
    # Handle Israeli numbers starting with 0
    if clean.startswith('0') and len(clean) == 10:
        clean = '+972' + clean[1:]
    
    return clean

def import_contacts_from_vcf(vcf_file_path: str, db_session) -> dict:
    """
    Import contacts from VCF file.
    
    Returns:
        dict with imported count, skipped count, and errors
    """
    with open(vcf_file_path, 'r', encoding='utf-8') as f:
        vcf_content = f.read()
    
    imported = 0
    skipped = 0
    errors = []
    
    try:
        for vcard in vobject.readComponents(vcf_content):
            try:
                name = vcard.fn.value if hasattr(vcard, 'fn') else None
                
                if hasattr(vcard, 'tel'):
                    for tel in vcard.tel_list:
                        phone = normalize_phone_number(tel.value)
                        
                        # Check if already exists
                        existing = db_session.query(Contact).filter_by(
                            phone_number=phone
                        ).first()
                        
                        if existing:
                            skipped += 1
                            continue
                        
                        # Create new contact
                        contact = Contact(
                            phone_number=phone,
                            contact_name=name
                        )
                        db_session.add(contact)
                        imported += 1
                        
            except Exception as e:
                errors.append(f"Error processing vCard: {str(e)}")
        
        db_session.commit()
        
    except Exception as e:
        db_session.rollback()
        errors.append(f"Fatal error: {str(e)}")
    
    return {
        'imported': imported,
        'skipped': skipped,
        'errors': errors
    }
```

---

## 6. Frontend Application

### 6.1 Project Structure

```
frontend/
├── public/
│   └── assets/
├── src/
│   ├── components/
│   │   ├── ui/                    # shadcn/ui components
│   │   │   ├── button.tsx
│   │   │   ├── input.tsx
│   │   │   ├── select.tsx
│   │   │   ├── table.tsx
│   │   │   └── ...
│   │   ├── layout/
│   │   │   ├── Header.tsx
│   │   │   ├── Sidebar.tsx
│   │   │   └── MainLayout.tsx
│   │   ├── recordings/
│   │   │   ├── RecordingsTable.tsx
│   │   │   ├── FilterPanel.tsx
│   │   │   ├── AudioPlayer.tsx
│   │   │   └── RecordingDetails.tsx
│   │   └── contacts/
│   │       ├── ContactsList.tsx
│   │       └── ContactImport.tsx
│   ├── pages/
│   │   ├── Dashboard.tsx
│   │   ├── Recordings.tsx
│   │   ├── Contacts.tsx
│   │   └── Settings.tsx
│   ├── hooks/
│   │   ├── useRecordings.ts
│   │   ├── useContacts.ts
│   │   └── useFilters.ts
│   ├── store/
│   │   ├── recordingsStore.ts
│   │   └── contactsStore.ts
│   ├── services/
│   │   ├── api.ts               # Axios instance
│   │   ├── recordingsApi.ts
│   │   └── contactsApi.ts
│   ├── types/
│   │   ├── recording.ts
│   │   └── contact.ts
│   ├── utils/
│   │   ├── formatters.ts
│   │   └── validators.ts
│   ├── App.tsx
│   ├── main.tsx
│   └── index.css
├── .env.development
├── .env.production
├── tailwind.config.ts
├── tsconfig.json
├── vite.config.ts
├── package.json
└── Dockerfile
```

### 6.2 Key Components

#### 6.2.1 FilterPanel Component

```tsx
// src/components/recordings/FilterPanel.tsx
import { useState } from 'react';
import { Input } from '@/components/ui/input';
import { Select } from '@/components/ui/select';
import { Button } from '@/components/ui/button';
import { Calendar } from '@/components/ui/calendar';

interface FilterPanelProps {
  onApplyFilters: (filters: RecordingFilters) => void;
  onClearFilters: () => void;
}

export function FilterPanel({ onApplyFilters, onClearFilters }: FilterPanelProps) {
  const [phoneNumber, setPhoneNumber] = useState('');
  const [dateFrom, setDateFrom] = useState<Date | null>(null);
  const [dateTo, setDateTo] = useState<Date | null>(null);
  const [dayOfWeek, setDayOfWeek] = useState<string[]>([]);
  const [durationMin, setDurationMin] = useState(0);
  const [durationMax, setDurationMax] = useState(120);
  const [contactId, setContactId] = useState<number | null>(null);

  const handleApply = () => {
    onApplyFilters({
      phoneNumber,
      dateFrom,
      dateTo,
      dayOfWeek,
      durationMin: durationMin * 60, // Convert to seconds
      durationMax: durationMax * 60,
      contactId
    });
  };

  return (
    <div className="w-80 bg-white p-6 shadow-lg rounded-lg space-y-6">
      <h2 className="text-2xl font-bold">סינונים</h2>
      
      {/* Phone Number Search */}
      <div className="space-y-2">
        <label className="text-sm font-medium">חיפוש לפי מספר</label>
        <Input
          type="text"
          placeholder="הזן מספר טלפון..."
          value={phoneNumber}
          onChange={(e) => setPhoneNumber(e.target.value)}
        />
      </div>

      {/* Date Range */}
      <div className="space-y-2">
        <label className="text-sm font-medium">טווח תאריכים</label>
        <div className="grid grid-cols-2 gap-2">
          <Calendar
            selected={dateFrom}
            onSelect={setDateFrom}
            placeholder="מתאריך"
          />
          <Calendar
            selected={dateTo}
            onSelect={setDateTo}
            placeholder="עד תאריך"
          />
        </div>
      </div>

      {/* Day of Week */}
      <div className="space-y-2">
        <label className="text-sm font-medium">יום בשבוע</label>
        <Select
          multiple
          value={dayOfWeek}
          onChange={setDayOfWeek}
          options={[
            { value: 'ראשון', label: 'ראשון' },
            { value: 'שני', label: 'שני' },
            { value: 'שלישי', label: 'שלישי' },
            { value: 'רביעי', label: 'רביעי' },
            { value: 'חמישי', label: 'חמישי' },
            { value: 'שישי', label: 'שישי' },
            { value: 'שבת', label: 'שבת' }
          ]}
        />
      </div>

      {/* Duration Range Slider */}
      <div className="space-y-2">
        <label className="text-sm font-medium">משך (דקות)</label>
        <DualRangeSlider
          min={0}
          max={120}
          valueMin={durationMin}
          valueMax={durationMax}
          onChangeMin={setDurationMin}
          onChangeMax={setDurationMax}
        />
        <div className="text-sm text-gray-600">
          {durationMin} - {durationMax} דקות
        </div>
      </div>

      {/* Action Buttons */}
      <div className="space-y-2">
        <Button onClick={handleApply} className="w-full">
          החל סינונים
        </Button>
        <Button onClick={onClearFilters} variant="outline" className="w-full">
          נקה הכל
        </Button>
      </div>
    </div>
  );
}
```

#### 6.2.2 AudioPlayer Component

```tsx
// src/components/recordings/AudioPlayer.tsx
import { useState, useRef, useEffect } from 'react';
import { Button } from '@/components/ui/button';
import { Slider } from '@/components/ui/slider';
import { Play, Pause, Square } from 'lucide-react';

interface AudioPlayerProps {
  recordingId: number | null;
  fileName: string | null;
}

export function AudioPlayer({ recordingId, fileName }: AudioPlayerProps) {
  const audioRef = useRef<HTMLAudioElement>(null);
  const [isPlaying, setIsPlaying] = useState(false);
  const [currentTime, setCurrentTime] = useState(0);
  const [duration, setDuration] = useState(0);
  const [audioUrl, setAudioUrl] = useState<string | null>(null);

  useEffect(() => {
    if (recordingId) {
      setAudioUrl(`http://localhost:5000/api/recordings/${recordingId}/audio`);
    }
  }, [recordingId]);

  useEffect(() => {
    const audio = audioRef.current;
    if (!audio) return;

    const updateTime = () => setCurrentTime(audio.currentTime);
    const updateDuration = () => setDuration(audio.duration);
    const handleEnded = () => setIsPlaying(false);

    audio.addEventListener('timeupdate', updateTime);
    audio.addEventListener('loadedmetadata', updateDuration);
    audio.addEventListener('ended', handleEnded);

    return () => {
      audio.removeEventListener('timeupdate', updateTime);
      audio.removeEventListener('loadedmetadata', updateDuration);
      audio.removeEventListener('ended', handleEnded);
    };
  }, []);

  const handlePlayPause = () => {
    const audio = audioRef.current;
    if (!audio) return;

    if (isPlaying) {
      audio.pause();
    } else {
      audio.play();
    }
    setIsPlaying(!isPlaying);
  };

  const handleStop = () => {
    const audio = audioRef.current;
    if (!audio) return;

    audio.pause();
    audio.currentTime = 0;
    setIsPlaying(false);
  };

  const handleSeek = (value: number[]) => {
    const audio = audioRef.current;
    if (!audio) return;

    audio.currentTime = value[0];
    setCurrentTime(value[0]);
  };

  const formatTime = (seconds: number) => {
    const mins = Math.floor(seconds / 60);
    const secs = Math.floor(seconds % 60);
    return `${mins.toString().padStart(2, '0')}:${secs.toString().padStart(2, '0')}`;
  };

  if (!recordingId) {
    return (
      <div className="bg-gray-100 p-6 rounded-lg text-center text-gray-500">
        בחר הקלטה לניגון
      </div>
    );
  }

  return (
    <div className="bg-white p-6 shadow-lg rounded-lg">
      <div className="mb-4">
        <div className="text-sm text-gray-600">מנגן כעת:</div>
        <div className="font-medium text-lg truncate">{fileName}</div>
      </div>

      <audio ref={audioRef} src={audioUrl || undefined} preload="metadata" />

      <div className="space-y-4">
        {/* Progress Bar */}
        <div className="space-y-2">
          <Slider
            value={[currentTime]}
            max={duration}
            step={1}
            onValueChange={handleSeek}
            className="w-full"
          />
          <div className="flex justify-between text-sm text-gray-600">
            <span>{formatTime(currentTime)}</span>
            <span>{formatTime(duration)}</span>
          </div>
        </div>

        {/* Controls */}
        <div className="flex justify-center gap-4">
          <Button onClick={handlePlayPause} size="lg">
            {isPlaying ? <Pause className="h-6 w-6" /> : <Play className="h-6 w-6" />}
          </Button>
          <Button onClick={handleStop} variant="outline" size="lg">
            <Square className="h-6 w-6" />
          </Button>
        </div>
      </div>
    </div>
  );
}
```

#### 6.2.3 RecordingsTable Component

```tsx
// src/components/recordings/RecordingsTable.tsx
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from '@/components/ui/table';
import { Button } from '@/components/ui/button';
import { Play } from 'lucide-react';
import type { Recording } from '@/types/recording';

interface RecordingsTableProps {
  recordings: Recording[];
  onPlayRecording: (id: number) => void;
  currentPlayingId: number | null;
}

export function RecordingsTable({ 
  recordings, 
  onPlayRecording,
  currentPlayingId 
}: RecordingsTableProps) {
  return (
    <div className="bg-white rounded-lg shadow">
      <Table>
        <TableHeader>
          <TableRow>
            <TableHead>תאריך</TableHead>
            <TableHead>שעה</TableHead>
            <TableHead>יום בשבוע</TableHead>
            <TableHead>מספר טלפון</TableHead>
            <TableHead>איש קשר</TableHead>
            <TableHead>משך</TableHead>
            <TableHead>גודל</TableHead>
            <TableHead>פעולות</TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          {recordings.map((recording) => (
            <TableRow 
              key={recording.id}
              className={currentPlayingId === recording.id ? 'bg-blue-50' : ''}
            >
              <TableCell>{recording.call_date}</TableCell>
              <TableCell>{recording.call_time}</TableCell>
              <TableCell>{recording.day_of_week}</TableCell>
              <TableCell className="font-mono">{recording.phone_number}</TableCell>
              <TableCell>{recording.contact_name || '-'}</TableCell>
              <TableCell>{recording.duration}</TableCell>
              <TableCell>{recording.file_size}</TableCell>
              <TableCell>
                <Button
                  size="sm"
                  onClick={() => onPlayRecording(recording.id)}
                  variant={currentPlayingId === recording.id ? 'default' : 'outline'}
                >
                  <Play className="h-4 w-4 mr-1" />
                  נגן
                </Button>
              </TableCell>
            </TableRow>
          ))}
        </TableBody>
      </Table>
    </div>
  );
}
```

### 6.3 State Management (Zustand)

```typescript
// src/store/recordingsStore.ts
import { create } from 'zustand';
import type { Recording, RecordingFilters } from '@/types/recording';

interface RecordingsState {
  recordings: Recording[];
  filters: RecordingFilters;
  pagination: {
    page: number;
    perPage: number;
    total: number;
    pages: number;
  };
  sortBy: string;
  sortOrder: 'asc' | 'desc';
  currentPlayingId: number | null;
  
  setRecordings: (recordings: Recording[]) => void;
  setFilters: (filters: RecordingFilters) => void;
  setPagination: (pagination: Partial<RecordingsState['pagination']>) => void;
  setSorting: (sortBy: string, sortOrder: 'asc' | 'desc') => void;
  setCurrentPlayingId: (id: number | null) => void;
  clearFilters: () => void;
}

export const useRecordingsStore = create<RecordingsState>((set) => ({
  recordings: [],
  filters: {},
  pagination: {
    page: 1,
    perPage: 50,
    total: 0,
    pages: 0
  },
  sortBy: 'call_datetime',
  sortOrder: 'desc',
  currentPlayingId: null,
  
  setRecordings: (recordings) => set({ recordings }),
  setFilters: (filters) => set({ filters, pagination: { ...get().pagination, page: 1 } }),
  setPagination: (pagination) => set((state) => ({
    pagination: { ...state.pagination, ...pagination }
  })),
  setSorting: (sortBy, sortOrder) => set({ sortBy, sortOrder }),
  setCurrentPlayingId: (id) => set({ currentPlayingId: id }),
  clearFilters: () => set({ filters: {}, pagination: { ...get().pagination, page: 1 } })
}));
```

---

## 7. File Processing Pipeline

### 7.1 Complete Scanning Workflow

```python
# app/services/file_scanner.py
import os
from pathlib import Path
from typing import List, Dict
from app.models import Recording, Contact
from app.services.audio_processor import (
    parse_recording_filename,
    extract_audio_metadata,
    get_file_size
)

def scan_recordings_directory(
    directory_path: str,
    db_session,
    progress_callback=None
) -> Dict:
    """
    Scan directory and index all AMR recordings.
    
    Args:
        directory_path: Absolute path to recordings directory
        db_session: SQLAlchemy session
        progress_callback: Optional callback(current, total) for progress updates
        
    Returns:
        Dict with results: total, successful, failed, errors
    """
    if not os.path.exists(directory_path):
        raise ValueError(f"Directory not found: {directory_path}")
    
    # Find all AMR files
    amr_files = list(Path(directory_path).rglob('*.amr'))
    total = len(amr_files)
    
    successful = 0
    failed = 0
    errors = []
    
    for idx, file_path in enumerate(amr_files):
        try:
            # Progress callback
            if progress_callback:
                progress_callback(idx + 1, total)
            
            # Parse filename
            file_metadata = parse_recording_filename(file_path.name)
            
            # Extract audio metadata
            audio_metadata = extract_audio_metadata(str(file_path))
            
            # Get file size
            file_info = get_file_size(str(file_path))
            
            # Look up contact
            contact = db_session.query(Contact).filter_by(
                phone_number=file_metadata['phone_number']
            ).first()
            
            # Check if already indexed
            existing = db_session.query(Recording).filter_by(
                file_path=str(file_path)
            ).first()
            
            if existing:
                # Update existing record
                existing.duration_seconds = audio_metadata['duration_seconds']
                existing.duration_formatted = audio_metadata['duration_formatted']
                existing.file_size_bytes = file_info['file_size_bytes']
                existing.file_size_formatted = file_info['file_size_formatted']
                existing.contact_id = contact.id if contact else None
            else:
                # Create new recording
                recording = Recording(
                    file_path=str(file_path),
                    file_name=file_path.name,
                    phone_number=file_metadata['phone_number'],
                    contact_id=contact.id if contact else None,
                    call_date=file_metadata['date'],
                    call_time=file_metadata['time'],
                    call_datetime=file_metadata['datetime_obj'],
                    day_of_week=file_metadata['day_of_week'],
                    duration_seconds=audio_metadata['duration_seconds'],
                    duration_formatted=audio_metadata['duration_formatted'],
                    file_size_bytes=file_info['file_size_bytes'],
                    file_size_formatted=file_info['file_size_formatted'],
                    audio_format='amr'
                )
                db_session.add(recording)
            
            successful += 1
            
            # Commit every 100 files
            if successful % 100 == 0:
                db_session.commit()
                
        except Exception as e:
            failed += 1
            error_msg = f"Failed to process {file_path.name}: {str(e)}"
            errors.append(error_msg)
            print(error_msg)
    
    # Final commit
    db_session.commit()
    
    # Log import
    from app.models import ImportLog
    import_log = ImportLog(
        import_type='recordings',
        total_files=total,
        successful_imports=successful,
        failed_imports=failed,
        errors='\n'.join(errors) if errors else None
    )
    db_session.add(import_log)
    db_session.commit()
    
    return {
        'total': total,
        'successful': successful,
        'failed': failed,
        'errors': errors
    }
```

---

## 8. Docker Configuration

### 8.1 docker-compose.yml

```yaml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:16-alpine
    container_name: call-recordings-db
    restart: unless-stopped
    environment:
      POSTGRES_USER: recordings_user
      POSTGRES_PASSWORD: recordings_pass
      POSTGRES_DB: recordings_db
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    networks:
      - recordings-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U recordings_user -d recordings_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Flask Backend
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: call-recordings-backend
    restart: unless-stopped
    environment:
      FLASK_ENV: production
      DATABASE_URL: postgresql://recordings_user:recordings_pass@postgres:5432/recordings_db
      RECORDINGS_PATH: /recordings
    volumes:
      # Mount local recordings directory (read-only for security)
      - ${HOST_RECORDINGS_PATH}:/recordings:ro
    ports:
      - "5000:5000"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - recordings-network
    command: gunicorn --bind 0.0.0.0:5000 --workers 4 --timeout 300 "app:create_app()"

  # React Frontend
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: call-recordings-frontend
    restart: unless-stopped
    ports:
      - "3000:3000"
    depends_on:
      - backend
    networks:
      - recordings-network
    environment:
      VITE_API_URL: http://localhost:5000

networks:
  recordings-network:
    driver: bridge

volumes:
  postgres_data:
```

### 8.2 Backend Dockerfile

```dockerfile
# backend/Dockerfile
FROM python:3.12-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    ffmpeg \
    libpq-dev \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy requirements first (for layer caching)
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create non-root user for security
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app

USER appuser

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:5000/api/health')"

# Run with gunicorn
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "--timeout", "300", "app:create_app()"]
```

### 8.3 Frontend Dockerfile

```dockerfile
# frontend/Dockerfile

# Build stage
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci

# Copy source code
COPY . .

# Build production app
RUN npm run build

# Production stage
FROM nginx:alpine

# Copy built files from builder
COPY --from=builder /app/dist /usr/share/nginx/html

# Copy nginx configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expose port
EXPOSE 3000

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
```

### 8.4 Environment Configuration

```bash
# .env
# Copy this file and modify values

# Host path to recordings directory (absolute path on your Ubuntu machine)
HOST_RECORDINGS_PATH=/home/user/call-recordings

# PostgreSQL
POSTGRES_USER=recordings_user
POSTGRES_PASSWORD=recordings_pass
POSTGRES_DB=recordings_db

# Backend
FLASK_ENV=production
SECRET_KEY=your-secret-key-here-change-this

# Frontend
VITE_API_URL=http://localhost:5000
```

### 8.5 Running the Application

```bash
# 1. Clone/setup project
cd /path/to/project

# 2. Set recordings path in .env
echo "HOST_RECORDINGS_PATH=/home/user/call-recordings" > .env

# 3. Build containers
docker-compose build

# 4. Start services
docker-compose up -d

# 5. Run database migrations
docker-compose exec backend flask db upgrade

# 6. Check logs
docker-compose logs -f

# 7. Access application
# Frontend: http://localhost:3000
# Backend API: http://localhost:5000

# 8. Stop services
docker-compose down

# 9. Stop and remove all data (including database)
docker-compose down -v
```

---

## 9. Security Considerations

### 9.1 Path Traversal Prevention

```python
from werkzeug.security import safe_join
from flask import abort
import os

def validate_file_path(recording_id: int, db_session) -> str:
    """
    Securely validate and return file path for recording.
    Prevents path traversal attacks.
    """
    recording = db_session.query(Recording).get(recording_id)
    if not recording:
        abort(404, "Recording not found")
    
    # Verify file path is within allowed directory
    base_dir = os.getenv('RECORDINGS_PATH', '/recordings')
    file_path = recording.file_path
    
    # Ensure file_path is absolute and within base_dir
    if not file_path.startswith(base_dir):
        abort(403, "Access denied")
    
    # Verify file exists
    if not os.path.isfile(file_path):
        abort(404, "Audio file not found")
    
    return file_path
```

### 9.2 Input Validation

```python
from flask import request, jsonify
from functools import wraps

def validate_pagination():
    """Middleware to validate pagination parameters."""
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            page = request.args.get('page', 1, type=int)
            per_page = request.args.get('per_page', 50, type=int)
            
            if page < 1:
                return jsonify({'error': 'Page must be >= 1'}), 400
            
            if per_page < 1 or per_page > 200:
                return jsonify({'error': 'per_page must be between 1 and 200'}), 400
            
            return f(*args, **kwargs)
        return decorated_function
    return decorator
```

### 9.3 CORS Configuration

```python
# app/__init__.py
from flask import Flask
from flask_cors import CORS

def create_app():
    app = Flask(__name__)
    
    # Restrict CORS to frontend origin only
    CORS(app, resources={
        r"/api/*": {
            "origins": ["http://localhost:3000"],
            "methods": ["GET", "POST", "PUT", "DELETE"],
            "allow_headers": ["Content-Type"]
        }
    })
    
    return app
```

### 9.4 Rate Limiting

```python
from flask_limiter import Limiter
from flask_limiter.util import get_remote_address

limiter = Limiter(
    app,
    key_func=get_remote_address,
    default_limits=["200 per day", "50 per hour"]
)

@app.route('/api/recordings')
@limiter.limit("10 per minute")
def search_recordings():
    # ...
    pass
```

---

## 10. Development Workflow

### 10.1 Local Development Setup (Without Docker)

```bash
# Backend
cd backend
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
flask db upgrade
flask run

# Frontend (separate terminal)
cd frontend
npm install
npm run dev
```

### 10.2 Development with Docker

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  postgres:
    # ... same as production

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile.dev
    volumes:
      - ./backend:/app  # Hot reload
      - ${HOST_RECORDINGS_PATH}:/recordings:ro
    environment:
      FLASK_ENV: development
      FLASK_DEBUG: 1
    command: flask run --host=0.0.0.0 --port=5000

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    volumes:
      - ./frontend:/app  # Hot reload
      - /app/node_modules
    command: npm run dev -- --host
```

---

## 11. API Documentation

### 11.1 OpenAPI/Swagger Documentation

Add Flask-RESTX or Flasgger for automatic API documentation:

```python
from flask_restx import Api, Resource

api = Api(
    app,
    version='1.0',
    title='Call Recordings API',
    description='API for managing call recordings',
    doc='/api/docs'
)

recordings_ns = api.namespace('recordings', description='Recording operations')

@recordings_ns.route('')
class RecordingsList(Resource):
    @api.doc('list_recordings')
    @api.param('page', 'Page number')
    @api.param('per_page', 'Results per page')
    def get(self):
        """List all recordings"""
        # ...
```

Access documentation at: `http://localhost:5000/api/docs`

---

## 12. Testing Strategy

### 12.1 Backend Tests

```python
# tests/test_audio_processor.py
import pytest
from app.services.audio_processor import parse_recording_filename

def test_parse_standard_filename():
    filename = "0521234555-sim1-voicecall-20250619145807.amr"
    result = parse_recording_filename(filename)
    
    assert result['phone_number'] == '0521234555'
    assert result['sim_card'] == '1'
    assert result['day_of_week'] == 'חמישי'  # Thursday

def test_parse_special_number_filename():
    filename = "*9074-sim1-voicecall-20250605223634.amr"
    result = parse_recording_filename(filename)
    
    assert result['phone_number'] == '*9074'

def test_invalid_filename():
    filename = "invalid-format.amr"
    with pytest.raises(ValueError):
        parse_recording_filename(filename)
```

### 12.2 Frontend Tests

```typescript
// src/components/recordings/__tests__/AudioPlayer.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { AudioPlayer } from '../AudioPlayer';

describe('AudioPlayer', () => {
  it('renders "select recording" message when no recording', () => {
    render(<AudioPlayer recordingId={null} fileName={null} />);
    expect(screen.getByText(/בחר הקלטה לניגון/)).toBeInTheDocument();
  });

  it('renders player controls when recording is selected', () => {
    render(<AudioPlayer recordingId={1} fileName="test.amr" />);
    expect(screen.getByRole('button', { name: /נגן/i })).toBeInTheDocument();
  });
});
```

---

## 13. Deployment Instructions

### 13.1 Production Deployment Checklist

- [ ] Change default passwords in `.env`
- [ ] Set strong `SECRET_KEY` for Flask
- [ ] Configure PostgreSQL backups
- [ ] Set up log rotation
- [ ] Configure firewall (only allow necessary ports)
- [ ] Enable HTTPS (if exposing to network)
- [ ] Test backup/restore procedure
- [ ] Document recovery procedures

### 13.2 Backup Strategy

```bash
# Backup PostgreSQL database
docker-compose exec postgres pg_dump -U recordings_user recordings_db > backup_$(date +%Y%m%d).sql

# Restore from backup
docker-compose exec -T postgres psql -U recordings_user recordings_db < backup_20251030.sql
```

---

## 14. Future Enhancements

### 14.1 Phase 2: Speech-to-Text

- Integrate Whisper AI for automatic transcription
- Index transcripts in database
- Enable full-text search within conversations

### 14.2 Phase 3: Advanced Analytics

- Call duration trends over time
- Contact interaction frequency
- Peak call times visualization
- Export reports to PDF/Excel

### 14.3 Phase 4: Mobile Access

- React Native mobile app
- Responsive web design improvements
- Offline playback support

---

## Appendix A: Technology Version Matrix

| Technology | Development | Production | Notes |
|------------|-------------|------------|-------|
| Python | 3.12.x | 3.12.x | Latest stable |
| PostgreSQL | 16.x | 16.x | Latest major |
| Node.js | 20.x | 20.x | LTS |
| React | 18.3+ | 18.3+ | Latest stable |
| TypeScript | 5.6+ | 5.6+ | Latest stable |
| Docker | 24.0+ | 24.0+ | Latest stable |

## Appendix B: Useful Commands

```bash
# View container logs
docker-compose logs -f backend
docker-compose logs -f frontend

# Access backend shell
docker-compose exec backend /bin/bash

# Access database
docker-compose exec postgres psql -U recordings_user -d recordings_db

# Run database migrations
docker-compose exec backend flask db migrate -m "description"
docker-compose exec backend flask db upgrade

# Rebuild specific service
docker-compose build --no-cache backend

# Remove all containers and volumes
docker-compose down -v

# View resource usage
docker stats
```

---

**END OF SPECIFICATION**

This specification is ready for immediate implementation by an AI development agent. All architectural decisions have been made, file formats are defined, and code examples are provided for key functionality.