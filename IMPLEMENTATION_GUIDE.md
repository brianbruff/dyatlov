# Implementation Guide: Adding New Devices to Dyatlov Map

## Overview

This guide provides step-by-step instructions for adding new shortwave radio receivers to the Dyatlov map. The process involves editing the static receiver configuration file and testing the changes.

## Quick Start

### Step 1: Edit static_rx.js

Add your new receiver to the `static_rx` array in `static_rx.js`:

```javascript
var static_rx = [
    // ... existing receivers ...
    {
        name: 'Your Receiver Name, Location, Country',
        url: 'http://your-receiver-url:port/',
        gps: '(latitude,longitude)',
        bands: 'frequency_range_in_hz',
        users_max: 'max_users',      // Optional
        sdr_hw: 'hardware_info',     // Optional  
        antenna: 'antenna_info'      // Optional
    },
];
```

### Step 2: Test Changes

Open `index.html` in a web browser to verify the new receiver appears on the map.

### Step 3: Deploy

Upload the modified `static_rx.js` file to your web server.

## Detailed Implementation Steps

### 1. Prepare Receiver Information

Before adding a receiver, gather the following information:

#### Required Fields
- **Name**: Descriptive name including location (e.g., "0-29 MHz WebSDR, University of Twente, Netherlands")
- **URL**: Direct URL to the receiver interface (must be accessible to end users)
- **GPS Coordinates**: Precise latitude and longitude in decimal degrees

#### Optional Fields  
- **Frequency Range**: Bandwidth in Hz (e.g., "0-29160000" for 0-29.16 MHz)
- **Maximum Users**: Concurrent user limit (e.g., "800", "20")
- **Hardware Description**: SDR hardware and setup info
- **Antenna Description**: Antenna type and specifications

### 2. Format GPS Coordinates

GPS coordinates must be in the format `(latitude,longitude)` with decimal degrees:

```javascript
// Correct formats:
gps: '(52.2381,6.8577)'     // Positive coordinates
gps: '(-33.8688,151.2093)'  // Negative coordinates (Southern/Western hemispheres)
gps: '(40.7128,-74.0060)'   // Mixed positive/negative

// Incorrect formats:
gps: '52.2381,6.8577'       // Missing parentheses
gps: '(52°14\'17"N,6°51\'28"E)' // Degrees/minutes/seconds format
gps: 'latitude: 52.2381, longitude: 6.8577' // Descriptive format
```

### 3. Validate Receiver URL

Ensure the receiver URL is:
- **Accessible**: Publicly available without authentication
- **Functional**: Actually serves a working receiver interface
- **Stable**: Likely to remain available long-term
- **Wideband**: Covers significant frequency range (preferably >5 MHz)

Test the URL in your browser before adding it to the configuration.

### 4. Edit static_rx.js

#### Location
Open `/static_rx.js` in your text editor.

#### Add New Entry
Insert your receiver data into the `static_rx` array. Add it at the end, before the closing bracket:

```javascript
var static_rx = [
    {
        name: '0-29 MHz WebSDR, University of Twente, Enschede, Netherlands',
        url: 'http://websdr.ewi.utwente.nl:8901/',
        gps: '(52.2381,6.8577)',
        bands: '0-29160000',
        users_max: '800',
        sdr_hw: 'WebSDR / custom, high-performance GPU-based setup',
        antenna: 'Mini-Whip',
    },
    // Add your new receiver here:
    {
        name: 'Your New Receiver Name, Location, Country',
        url: 'http://your-receiver.example.com:8901/',
        gps: '(40.7128,-74.0060)',
        bands: '0-30000000',
        users_max: '50',
        sdr_hw: 'RTL-SDR with upconverter',
        antenna: 'Random wire antenna',
    },
];
```

#### JavaScript Syntax Notes
- Each receiver object must end with a comma (`,`)
- All string values must be quoted with single (`'`) or double (`"`) quotes
- The last receiver in the array should NOT have a trailing comma
- Maintain consistent indentation (use tabs as in existing entries)

### 5. Test the Configuration

#### Local Testing
1. Open `index.html` in a web browser
2. Wait for the map to load
3. Look for your new receiver marker on the map
4. Click the marker to verify the info bubble displays correctly
5. Click the receiver name link to test the URL

#### Troubleshooting
If your receiver doesn't appear:

**Check Browser Console**
- Open Developer Tools (F12)
- Look for JavaScript errors in the Console tab
- Common errors include syntax errors in static_rx.js

**Verify Data Format**
- Ensure GPS coordinates are valid decimal degrees
- Check that all required quotes and commas are present
- Verify the URL is accessible from your browser

**Check Map Bounds**
- Very remote coordinates might not be visible in the default map view
- Try zooming out or searching for your receiver's location

### 6. Advanced Configuration

#### Frequency Range Format
The `bands` field accepts various formats:
```javascript
bands: '0-29160000'          // 0 Hz to 29.16 MHz
bands: '3500000-28000000'    // 3.5 MHz to 28 MHz  
bands: '14000000-14350000'   // 20m amateur band only
```

#### Hardware Descriptions
Be descriptive but concise:
```javascript
sdr_hw: 'WebSDR / custom, high-performance GPU-based setup'
sdr_hw: 'RTL-SDR with Ham It Up upconverter'
sdr_hw: 'KiwiSDR / BeagleBone + Kiwi cape'
sdr_hw: 'OpenWebRX / HackRF One'
```

#### Special Characters
Escape special characters in strings:
```javascript
name: 'Receiver with "quotes" in name'    // Escape quotes
url: 'http://example.com/path\\with\\backslashes'  // Escape backslashes
```

### 7. Quality Assurance

#### Pre-Deployment Checklist
- [ ] Receiver URL is publicly accessible
- [ ] GPS coordinates are accurate (use Google Maps to verify)
- [ ] Name is descriptive and includes location
- [ ] No JavaScript syntax errors (test in browser console)
- [ ] Marker appears in correct location on map
- [ ] Info bubble displays properly formatted information
- [ ] Receiver link works when clicked

#### Testing Multiple Browsers
Test your changes in multiple browsers to ensure compatibility:
- Chrome/Chromium
- Firefox
- Safari (if on macOS)
- Edge (if on Windows)

### 8. Deployment

#### File Upload
1. Upload the modified `static_rx.js` to your web server
2. Clear your browser cache to ensure the new version loads
3. Verify changes are live by visiting your map URL

#### Version Control
If using Git:
```bash
git add static_rx.js
git commit -m "Add [Receiver Name] to static receiver list"
git push
```

## Example: Adding a Complete Receiver

Here's a complete example of adding a new receiver:

### Before (existing static_rx.js):
```javascript
var static_rx = [
    {
        name: '0-29 MHz WebSDR, University of Twente, Enschede, Netherlands',
        url: 'http://websdr.ewi.utwente.nl:8901/',
        gps: '(52.2381,6.8577)',
        bands: '0-29160000',
        users_max: '800',
        sdr_hw: 'WebSDR / custom, high-performance GPU-based setup',
        antenna: 'Mini-Whip',
    },
];
```

### After (with new receiver):
```javascript
var static_rx = [
    {
        name: '0-29 MHz WebSDR, University of Twente, Enschede, Netherlands',
        url: 'http://websdr.ewi.utwente.nl:8901/',
        gps: '(52.2381,6.8577)',
        bands: '0-29160000',
        users_max: '800',
        sdr_hw: 'WebSDR / custom, high-performance GPU-based setup',
        antenna: 'Mini-Whip',
    },
    {
        name: '0-30 MHz WebSDR, Example University, New York, USA',
        url: 'http://websdr.example.edu:8901/',
        gps: '(40.7128,-74.0060)',
        bands: '0-30000000',
        users_max: '100',
        sdr_hw: 'WebSDR / USRP B200 with custom frontend',
        antenna: 'Active loop antenna on rooftop',
    },
];
```

## Common Issues and Solutions

### Issue: Receiver doesn't appear on map
**Causes:**
- JavaScript syntax error in static_rx.js
- Invalid GPS coordinates format
- Coordinates outside visible map area

**Solutions:**
- Check browser console for errors
- Verify GPS coordinate format: `(lat,lng)`
- Zoom out on map to find receiver

### Issue: Marker appears but link doesn't work  
**Causes:**
- Incorrect URL format
- Receiver server is down
- Firewall/CORS restrictions

**Solutions:**
- Test URL directly in browser
- Verify URL includes protocol (http:// or https://)
- Contact receiver operator

### Issue: Info bubble displays incorrectly
**Causes:**
- Unescaped special characters in name/description
- Missing quotes around strings
- HTML injection in receiver name

**Solutions:**
- Escape quotes and special characters
- Ensure all strings are properly quoted
- Use plain text in receiver names

## Performance Considerations

### File Size
- Each receiver adds ~200-500 bytes to static_rx.js
- 1000 receivers ≈ 200-500KB additional download
- Consider splitting into multiple files for very large datasets

### Map Performance
- 100+ markers may impact initial render time
- 500+ markers should use clustering
- Consider regional maps for large deployments

### Browser Compatibility
- Modern browsers handle 1000+ markers well
- Older browsers may struggle with large datasets
- Test on target audience's browsers

## Future Enhancement Opportunities

### Dynamic Loading
Convert to REST API for real-time updates:
```javascript
// Future concept:
async function loadReceivers() {
    const response = await fetch('/api/receivers');
    return response.json();
}
```

### Database Integration
Store receivers in a database for better management:
```sql
-- Future schema concept:
CREATE TABLE receivers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    url VARCHAR(500) NOT NULL,
    latitude DECIMAL(10,8) NOT NULL,
    longitude DECIMAL(11,8) NOT NULL,
    bands VARCHAR(100),
    max_users INTEGER,
    hardware TEXT,
    antenna TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

### Admin Interface
Web-based management interface:
- Add/edit/delete receivers through web forms
- Validate URLs automatically
- Preview changes before publishing
- User authentication and permissions