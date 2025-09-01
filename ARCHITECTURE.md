# Dyatlov Map Maker - Technical Architecture Documentation

## Overview

The Dyatlov Map Maker is a client-side JavaScript web application that displays wideband shortwave radio receivers on an interactive world map. It provides a simple, modular architecture for visualizing receiver networks with real-time status information.

## System Architecture

### High-Level Architecture Diagram

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   index.html    │────│   dyatlov.js    │────│  Map Toolkit    │
│  (Entry Point)  │    │  (Core Logic)   │    │ (Google/Leaflet)│
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │                       │                       │
    ┌─────────┐             ┌─────────┐             ┌─────────┐
    │ Data    │             │ Device  │             │ Map     │
    │ Sources │             │ Models  │             │ Markers │
    └─────────┘             └─────────┘             └─────────┘
         │                       │                       │
    ┌─────────┐             ┌─────────┐             ┌─────────┐
    │static_rx│             │   RX    │             │Colored  │
    │kiwisdr  │             │ Class   │             │Icons    │
    └─────────┘             └─────────┘             └─────────┘
```

### Core Components

#### 1. Entry Point (`index.html`)
- **Purpose**: Main web page that orchestrates the application
- **Responsibilities**:
  - Loads map toolkit libraries (Google Maps or Leaflet)
  - Loads optional add-ons (day/night overlay, moment.js)
  - Loads data sources (static_rx.js, kiwisdr_com.js)
  - Loads core dyatlov.js library
  - Initializes the Dyatlov map instance

#### 2. Core Engine (`dyatlov.js`)
- **Purpose**: Main application logic and abstraction layer
- **Key Classes**:
  - `Dyatlov`: Main orchestrator class
  - `RX`: Device/receiver model class
  - `GoogleMaps`/`Leaflet`: Map toolkit implementations

#### 3. Data Sources
- **`static_rx.js`**: Manually curated list of receivers (hand-picked quality receivers)
- **`kiwisdr_com.js`**: Dynamically generated KiwiSDR network data (auto-updated)

## Data Flow Architecture

### Data Processing Pipeline

```
Data Sources → Merge → Validate → Transform → Render
     │            │        │          │         │
┌─────────┐  ┌─────────┐ ┌──────┐ ┌─────────┐ ┌────────┐
│static_rx│  │receivers│ │ RX   │ │ coords  │ │ Map    │
│kiwisdr  │─→│  array  │→│class │→│ colors  │→│Markers │
│  data   │  │         │ │      │ │ icons   │ │        │
└─────────┘  └─────────┘ └──────┘ └─────────┘ └────────┘
```

### Data Flow Steps

1. **Data Loading**: Static and dynamic data sources are loaded as JavaScript variables
2. **Data Merging**: `receivers()` function concatenates all available data sources
3. **Object Creation**: Each receiver data object is wrapped in an `RX` class instance
4. **Validation**: Invalid receivers (missing required fields) are filtered out
5. **Coordinate Processing**: GPS coordinates are parsed and validated
6. **Marker Creation**: Each valid receiver becomes a map marker with status-based styling
7. **Map Rendering**: Markers are placed on the map with appropriate clustering and z-indexing

## Device Data Model

### Current Device Structure

```javascript
// Static receiver format (static_rx.js)
{
    name: 'Display name for the receiver',
    url: 'http://example.com:8901/',
    gps: '(latitude,longitude)',        // Format: "(52.2381,6.8577)"
    bands: 'frequency_range',           // Format: "0-29160000" (Hz)
    users_max: 'max_concurrent_users',  // Optional
    sdr_hw: 'hardware_description',     // Optional
    antenna: 'antenna_description'      // Optional
}

// KiwiSDR format (kiwisdr_com.js) - auto-generated
{
    name: "receiver_name",
    url: "receiver_url",
    gps: "coordinates",
    users: "current_users",
    users_max: "max_users",
    offline: "yes/no",
    status: "active/offline/inactive",
    updated: "timestamp",
    // ... other KiwiSDR-specific fields
}
```

### Device Processing in RX Class

The `RX` class provides standardized processing for both data formats:

- **Coordinate parsing**: Converts various GPS formats to `{lat, lng}` objects
- **Status determination**: Calculates online/offline state, availability, quality scores
- **Marker styling**: Determines color coding based on receiver state
- **Info bubble**: Generates HTML content for marker click events

## Map Integration

### Supported Map Toolkits

1. **Google Maps API**
   - Requires API key and billing account
   - Provides satellite imagery by default
   - Supports various map types and overlays

2. **Leaflet + OpenStreetMap**
   - Free and open source
   - No satellite imagery without additional providers
   - Extensible with various tile providers

### Map Features

- **Marker Color Coding**:
  - 🔴 Red: Available receivers with open slots (brightness indicates quality)
  - ⚫ Gray: Available receivers (quality not rated)
  - 🟡 Yellow: Receivers at maximum capacity
  - 🟢 Green: Generally available receivers (no dynamic status)
  - 🟣 Purple: Offline or inaccessible receivers

- **Interactive Features**:
  - Click markers to open info bubbles with receiver details
  - Automatic clustering for overlapping markers
  - Day/night overlay (optional)
  - Zoom and pan controls

## Current Limitations

### Technical Limitations
1. **Static Configuration**: Adding receivers requires manual code changes
2. **No Persistence**: No database or server-side storage
3. **Client-Side Only**: All processing happens in the browser
4. **No Authentication**: No access control for receiver management
5. **Limited Validation**: Minimal URL and data validation

### Scalability Concerns
1. **File Size**: Large receiver lists could impact page load times
2. **Update Frequency**: Static data requires manual updates and redeployment
3. **Concurrent Users**: No server-side infrastructure to manage load
4. **Data Synchronization**: No real-time updates for static receiver status

### Security Considerations
1. **XSS Vulnerability**: User-provided URLs and names are displayed without sanitization
2. **CORS Issues**: External receiver URLs may have cross-origin restrictions
3. **Code Injection**: Dynamic KiwiSDR data could potentially contain malicious scripts
4. **No Rate Limiting**: No protection against excessive requests

## Performance Characteristics

### Loading Performance
- **Initial Load**: Depends on map toolkit and data file sizes
- **Map Rendering**: Scales with number of receivers (500+ may need clustering)
- **Memory Usage**: Grows linearly with receiver count

### Runtime Performance
- **Marker Updates**: No real-time updates, only on page refresh
- **User Interactions**: Click events and info bubbles are responsive
- **Map Navigation**: Performance depends on selected map toolkit

## Development Workflow

### Current Development Process
1. Edit `static_rx.js` to add/modify receivers
2. Test changes by opening `index.html` in browser
3. Deploy updated files to web server
4. Optionally update KiwiSDR data using `kiwisdr_com-update` script

### Build Requirements
- **None**: Pure client-side application with no build step
- **Dependencies**: Map toolkit libraries (loaded via CDN or local files)
- **Server Requirements**: Static file hosting (Apache, nginx, GitHub Pages, etc.)

## File Structure and Dependencies

```
dyatlov/
├── index.html              # Main entry point and configuration
├── dyatlov.js             # Core application logic
├── static_rx.js           # Static receiver data
├── kiwisdr_com-update     # KiwiSDR data fetch script
├── kiwisdr_com-parse      # KiwiSDR data parser
├── kiwisdr_com.js         # Generated KiwiSDR data (optional)
├── doc/                   # Marker icons (SVG files)
│   ├── FD7567.svg        # Red marker
│   ├── 807567.svg        # Gray marker
│   ├── FFFF6E.svg        # Yellow marker
│   ├── 00E74C.svg        # Green marker
│   └── 9067FD.svg        # Purple marker
├── leaflet.js            # Leaflet library (optional)
├── leaflet.css           # Leaflet styles (optional)
├── moment.js             # Time library (optional)
└── daynightoverlay.js    # Day/night overlay (optional)
```

## External Dependencies

### Required
- Map toolkit: Google Maps API OR Leaflet library

### Optional
- Day/night overlay library
- Moment.js for time manipulation
- Various Leaflet tile providers for satellite imagery

### KiwiSDR Integration
- Perl interpreter (for kiwisdr_com-parse)
- wget utility (for kiwisdr_com-update)
- Regular cron job for data updates