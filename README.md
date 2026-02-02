# Retell AI Voice Agent + n8n Appointment Booking System

A comprehensive voice-based appointment booking system that combines **Retell AI** voice agents with **n8n** automation workflows. This system enables automated phone-based appointment booking, rescheduling, and cancellation for various business types.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Components](#components)
  - [Retell AI Voice Agents](#retell-ai-voice-agents)
  - [n8n Workflows](#n8n-workflows)
- [Supported CRM Integrations](#supported-crm-integrations)
- [Setup Guide](#setup-guide)
- [API Endpoints](#api-endpoints)
- [Function Reference](#function-reference)
- [Database Schema](#database-schema)
- [Configuration](#configuration)

---

## Overview

This system provides an end-to-end solution for businesses to automate their appointment booking process via phone calls. A customer calls in, speaks with an AI voice agent (powered by Retell AI and GPT-4o), and can:

- **Book** new appointments
- **Reschedule** existing appointments
- **Cancel** appointments
- Get **availability** information

The voice agent connects to backend n8n workflows that handle the business logic and integrate with various CRM/calendar systems.

---

## Architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│   Phone Call    │────▶│   Retell AI      │────▶│   n8n Workflows     │
│   (Customer)    │     │   Voice Agent    │     │   (Automation)      │
└─────────────────┘     └──────────────────┘     └─────────────────────┘
                                                          │
                        ┌─────────────────────────────────┼─────────────────────────────────┐
                        │                                 │                                 │
                        ▼                                 ▼                                 ▼
               ┌─────────────────┐              ┌─────────────────┐              ┌─────────────────┐
               │   Square API    │              │  GoHighLevel    │              │ Google Calendar │
               │   (Bookings)    │              │  (GHL API)      │              │      API        │
               └─────────────────┘              └─────────────────┘              └─────────────────┘
                        │                                 │                                 │
                        └─────────────────────────────────┼─────────────────────────────────┘
                                                          │
                                                          ▼
                                                 ┌─────────────────┐
                                                 │    Supabase     │
                                                 │   (Database)    │
                                                 └─────────────────┘
```

---

## Components

### Retell AI Voice Agents

Three pre-configured voice agents for different business types:

| Agent | File | Business Type | Client ID |
|-------|------|---------------|-----------|
| **Lumina MedSpa** | `GHL-LuminaMedSpaOakville.json` | Medical Spa | `LuminaMedSpaOakville` |
| **Alder & Ash Toronto** | `GC-AlderAshToronto__Restaurant_.json` | Restaurant | `AlderAshToronto` |
| **Toronto Pets** | `Square_Agent_Pet-updated.json` | Pet Grooming | `TorontoPets` |

#### Agent Features

- **Voice**: Custom voice with natural speech patterns
- **Model**: GPT-4o for conversation handling
- **Language**: English (en-US)
- **Backchannel**: Enabled with natural responses ("okay", "uh-huh", "mhmm", "yeah")
- **Ambient Sound**: Coffee shop background
- **Call Duration**: Max ~10 minutes
- **Call Transfer**: Warm transfer to human agents when needed

#### Agent Personality

All agents use the persona "Kate" with these characteristics:
- Warm, enthusiastic, and conversational tone
- Natural speech with occasional pauses ("umm...", "let me see...")
- Spell-checking for names and emails
- Calm handling of upset customers
- Immediate action on detected intent

### n8n Workflows

#### Core Booking Workflows

| Workflow | File | Webhook Path | Purpose |
|----------|------|--------------|---------|
| **User Check** | `g_user_check.json` | `/webhook/g_user_check` | Verify if customer exists in system |
| **Create User** | `g_create_user.json` | `/webhook/g_user_create` | Register new customers |
| **Check Availability** | `g_availability.json` | `/webhook/g_availability` | Query available time slots |
| **Create Booking** | `g_create_booking.json` | `/webhook/g_create_booking` | Book new appointments |
| **Get Booked Slots** | `g_Get_booked_slots.json` | `/webhook/g_Get_booked_slots` | Retrieve customer's existing bookings |
| **Reschedule** | `g_Reschedule.json` | `/webhook/g_Rechedule` | Modify existing appointments |
| **Cancel Booking** | `g_cancel_booking.json` | `/webhook/g_cancel_booking` | Cancel appointments |

#### OAuth Token Management (GoHighLevel)

| Workflow | File | Purpose |
|----------|------|---------|
| **Token Generate** | `Token_Generate_ghl.json` | Initial OAuth token acquisition |
| **Token Refresh** | `Re_New_Token_ghl.json` | Daily automated token refresh (runs at 23:00) |

---

## Supported CRM Integrations

### 1. Square Appointments
- Customer management via Square Customers API
- Booking management via Square Bookings API
- Service catalog integration
- Team member assignment

### 2. GoHighLevel (GHL)
- Contact management
- Calendar event integration
- OAuth 2.0 authentication with automatic token refresh

### 3. Google Calendar
- Free/busy slot checking
- Event creation and management
- OAuth 2.0 authentication

---

## Setup Guide

### Prerequisites

1. **n8n Instance**: Self-hosted n8n (recommended: `n8n.srv899746.hstgr.cloud` pattern)
2. **Supabase Account**: For database storage
3. **Retell AI Account**: For voice agent hosting
4. **CRM Account(s)**: Square, GoHighLevel, or Google Calendar

### Step 1: Database Setup (Supabase)

Create the required tables:

```sql
-- Client configuration table
CREATE TABLE client (
    id SERIAL PRIMARY KEY,
    client_id VARCHAR(255) UNIQUE NOT NULL,
    crm_type VARCHAR(50), -- 'Square', 'ghl', 'google'
    crm_api_key TEXT,
    calendar_id VARCHAR(255),
    locations VARCHAR(255),
    agent_id VARCHAR(255)
);

-- Customer records table
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    first_name VARCHAR(255),
    last_name VARCHAR(255),
    phone VARCHAR(50),
    client_id VARCHAR(255),
    agent_id VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Step 2: Import n8n Workflows

1. Access your n8n instance
2. Import each workflow JSON file
3. Configure credentials:
   - **Supabase API**: Database connection
   - **Google Calendar OAuth2**: For calendar integration
   - **Square API Key**: For Square clients (stored in `client.crm_api_key`)
   - **GHL OAuth**: Generated via `Token_Generate_ghl` workflow

### Step 3: Configure Client Records

Insert client configuration into Supabase:

```sql
INSERT INTO client (client_id, crm_type, crm_api_key, calendar_id, locations)
VALUES 
    ('LuminaMedSpaOakville', 'ghl', '[OAuth JSON]', 'calendar@email.com', 'location_id'),
    ('AlderAshToronto', 'google', NULL, 'calendar@email.com', NULL),
    ('TorontoPets', 'Square', 'sq_api_key_here', NULL, 'location_id');
```

### Step 4: Import Retell AI Agents

1. Access Retell AI dashboard
2. Create new agent from each JSON configuration
3. Update webhook URLs to match your n8n instance
4. Link agent IDs back to Supabase `client.agent_id`

---

## API Endpoints

All endpoints accept POST requests with JSON body.

### User Check
```
POST /webhook/g_user_check
```
**Trigger**: Inbound call webhook from Retell AI

**Request Body**:
```json
{
  "body": {
    "call_inbound": {
      "agent_id": "agent_xxx"
    },
    "args": {
      "phone": "+1234567890"
    }
  }
}
```

**Response** (Existing Customer):
```json
{
  "call_inbound": {
    "dynamic_variables": {
      "first_name": "John",
      "last_name": "Doe",
      "customer_status": "Customer_Exists",
      "customer_name": "John Doe"
    }
  }
}
```

### Create User
```
POST /webhook/g_user_create
```

**Request Body**:
```json
{
  "body": {
    "args": {
      "clientId": "TorontoPets",
      "first_name": "John",
      "last_name": "Doe",
      "phone": "+1234567890"
    }
  }
}
```

### Check Availability
```
POST /webhook/g_availability
```

**Request Body**:
```json
{
  "body": {
    "args": {
      "clientId": "LuminaMedSpaOakville",
      "start_at": "2025-09-02T09:00:00Z",
      "end_at": "2025-09-02T17:00:00Z",
      "Service_name": "Dermal Filler"
    }
  }
}
```

**Response**:
```json
[
  { "start_at": "2025-09-02T09:00:00Z" },
  { "start_at": "2025-09-02T09:30:00Z" },
  { "start_at": "2025-09-02T10:00:00Z" }
]
```

### Create Booking
```
POST /webhook/g_create_booking
```

**Request Body**:
```json
{
  "body": {
    "args": {
      "clientId": "TorontoPets",
      "phone": "+1234567890",
      "starttime": "2025-09-02T10:00:00Z",
      "endtime": "2025-09-02T11:00:00Z",
      "title": "Cat Full Grooming - +1234567890",
      "description": "Phone: +1234567890",
      "first_name": "John",
      "last_name": "Doe"
    }
  }
}
```

### Get Booked Slots
```
POST /webhook/g_Get_booked_slots
```

**Request Body**:
```json
{
  "body": {
    "args": {
      "clientId": "LuminaMedSpaOakville",
      "phone": "+1234567890"
    }
  }
}
```

### Reschedule
```
POST /webhook/g_Rechedule
```

**Request Body**:
```json
{
  "body": {
    "args": {
      "clientId": "LuminaMedSpaOakville",
      "phone": "+1234567890",
      "targetStart": "2025-09-02 10:00:00",
      "starttime": "2025-09-03T14:00:00Z",
      "endtime": "2025-09-03T15:00:00Z"
    }
  }
}
```

### Cancel Booking
```
POST /webhook/g_cancel_booking
```

**Request Body**:
```json
{
  "body": {
    "args": {
      "clientId": "AlderAshToronto",
      "phone": "+1234567890",
      "targetStart": "2025-09-02 10:00:00"
    }
  }
}
```

---

## Function Reference

### Voice Agent Tools

| Function | Description | When Called |
|----------|-------------|-------------|
| `end_call` | Terminates the call | After booking completion or user request |
| `transfer_call` | Warm transfer to human agent | Booking failures or upset customers |
| `extract_status` | Extracts customer status variable | Internal flow control |
| `create_user` | Creates new customer profile | After collecting name/email/phone |
| `Booking` | Creates new appointment | After availability confirmation |
| `Check_Booked_Slots` | Retrieves existing bookings | Reschedule/cancel intent detected |
| `Availability_Google` | Checks available time slots | After service selection |
| `Reschedule` | Modifies existing booking | After new time selection |
| `Cancel_Booked_Slots` | Cancels existing booking | After cancel confirmation |

---

## Database Schema

### `client` Table

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL | Primary key |
| `client_id` | VARCHAR(255) | Unique business identifier |
| `crm_type` | VARCHAR(50) | CRM type: `Square`, `ghl`, `google` |
| `crm_api_key` | TEXT | API key or OAuth token JSON |
| `calendar_id` | VARCHAR(255) | Google Calendar ID (for Google/GHL) |
| `locations` | VARCHAR(255) | Square location ID |
| `agent_id` | VARCHAR(255) | Retell AI agent ID |

### `customers` Table

| Column | Type | Description |
|--------|------|-------------|
| `id` | SERIAL | Primary key |
| `first_name` | VARCHAR(255) | Customer first name |
| `last_name` | VARCHAR(255) | Customer last name |
| `phone` | VARCHAR(50) | Phone number (E.164 format) |
| `client_id` | VARCHAR(255) | Associated business |
| `agent_id` | VARCHAR(255) | Agent that created record |
| `created_at` | TIMESTAMP | Record creation time |

---

## Configuration

### Environment Variables

| Variable | Description |
|----------|-------------|
| `SUPABASE_URL` | Supabase project URL |
| `SUPABASE_API_KEY` | Supabase service role key |
| `GHL_CLIENT_ID` | GoHighLevel OAuth client ID |
| `GHL_CLIENT_SECRET` | GoHighLevel OAuth client secret |
| `SQUARE_VERSION` | Square API version (e.g., `2025-07-16`) |

### GoHighLevel OAuth Configuration

**OAuth Scopes Required**:
- `locations.readonly`
- `contacts.readonly`
- `contacts.write`
- `calendars.readonly`
- `calendars.write`
- `calendars/events.readonly`
- `calendars/events.write`

**Token Refresh**: Automatic daily refresh at 23:00 UTC via `Re_New_Token_ghl` workflow.

---

## Business-Specific Configurations

### Medical Spa (Lumina MedSpa)
- **Special Rule**: First-time injectable treatments require complimentary 15-minute consultation
- **Services**: Neuromodulators, Dermal Fillers, Skin Boosters, etc.
- **Location**: 310 Cornwall Rd, Oakville (Dummy)

### Restaurant (Alder & Ash Toronto)
- **Party Size Rules**:
  - 1-6 guests: Direct booking via AI
  - 7-12 guests: Email referral (reservations@alderandash.ca)
  - 13-24 guests: Private dining email referral
  - 24+ guests: Cannot accommodate

### Pet Grooming (Toronto Pets)
- **Location**: Mississauga (Dummy)
- **Services**: Cat/Dog grooming with various packages

---

## Error Handling

The system handles various error conditions:

| Error | Response | Agent Behavior |
|-------|----------|----------------|
| Client not found | `"Client Not Exists"` | Apologize and offer transfer |
| Invalid time | `"Time is Wrong"` | Request valid time |
| No availability | `"No slots available"` | Offer alternative dates |
| Customer not found | `"Not Exists"` | Ask for name/alternative phone |
| Booking failure | Error details | Transfer to human agent |

---

## License

This system configuration is proprietary. Contact the system administrator for usage rights.

---

## Support

For technical issues:
- **Transfer Number**: +92 333 8122531
- **Email**: muhammadnadeem51200@gmail.com
