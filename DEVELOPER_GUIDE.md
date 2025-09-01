# Developer Guide: Adding New Device URLs to Dyatlov Map

## Quick Reference

### Adding a Receiver (TL;DR)
1. Edit `static_rx.js`
2. Add receiver object with required fields
3. Test locally by opening `index.html`
4. Deploy updated file

### Required Fields
```javascript
{
    name: 'Receiver Name, Location, Country',
    url: 'http://receiver-url:port/',
    gps: '(latitude,longitude)'
}
```

## Table of Contents

1. [Understanding the Architecture](#architecture)
2. [Step-by-Step Implementation](#implementation)
3. [Testing and Validation](#testing)
4. [Security Considerations](#security)
5. [Troubleshooting](#troubleshooting)
6. [Performance Guidelines](#performance)
7. [Future Migration Path](#migration)

## Understanding the Architecture {#architecture}

### System Overview
Dyatlov is a client-side JavaScript application that displays shortwave receivers on an interactive map. It merges data from two sources:

- **Static receivers** (`static_rx.js`): Hand-curated, high-quality receivers
- **KiwiSDR network** (`kiwisdr_com.js`): Auto-generated from KiwiSDR.com

### Data Flow
```
static_rx.js + kiwisdr_com.js → dyatlov.js → Map Markers
```

### Key Files
- `index.html`: Entry point and configuration
- `dyatlov.js`: Core application logic
- `static_rx.js`: **← You edit this file**
- `doc/*.svg`: Marker icon files

## Step-by-Step Implementation {#implementation}

### Step 1: Gather Receiver Information

**Required Information:**
- **Name**: Descriptive name with location
- **URL**: Direct link to receiver interface
- **Coordinates**: Precise latitude/longitude

**Optional Information:**
- Frequency range (in Hz)
- Maximum concurrent users
- Hardware description
- Antenna description

### Step 2: Format the Data

```javascript
// Template for new receiver
{
    name: 'Frequency Range WebSDR, Institution, City, Country',
    url: 'http://receiver.example.com:8901/',
    gps: '(latitude,longitude)',           // Required
    bands: 'min_hz-max_hz',               // Optional
    users_max: 'number',                  // Optional
    sdr_hw: 'hardware description',       // Optional
    antenna: 'antenna description'        // Optional
}
```

**GPS Coordinate Format:**
- Use decimal degrees: `(52.2381,6.8577)`
- Include parentheses and comma
- Negative values for South/West: `(-33.8688,151.2093)`

### Step 3: Edit static_rx.js

**File Location:** `/static_rx.js`

**Before editing:**
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
    // ... other receivers
];
```

**After adding your receiver:**
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
    // ... other receivers
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

### Step 4: Syntax Guidelines

**JavaScript Syntax Rules:**
- All strings must be quoted (`'` or `"`)
- Each object must end with a comma `,`
- Last object should NOT have trailing comma
- Maintain consistent indentation

**Common Mistakes:**
```javascript
// ❌ Wrong - missing quotes
name: My Receiver Name,

// ❌ Wrong - missing comma
name: 'My Receiver'
url: 'http://...'

// ❌ Wrong - trailing comma on last item
{
    name: 'Last Receiver',
    url: 'http://...'
}, // ← Remove this comma

// ✅ Correct
{
    name: 'My Receiver Name',
    url: 'http://example.com/',
    gps: '(40.7,-74.0)',
},
```

## Testing and Validation {#testing}

### Local Testing

**Method 1: Direct File Opening**
1. Open `index.html` in web browser
2. Look for your receiver in the list or map
3. Click the receiver link to test URL

**Method 2: Local Web Server**
```bash
# Python 3
python3 -m http.server 8000

# Python 2
python -m SimpleHTTPServer 8000

# Node.js (if you have it)
npx http-server

# Then open http://localhost:8000
```

### Validation Checklist

**Before Deployment:**
- [ ] Receiver appears in list/map
- [ ] GPS coordinates are correct location
- [ ] Receiver name displays properly
- [ ] URL link works when clicked
- [ ] No JavaScript errors in browser console
- [ ] Special characters display correctly

**Testing in Multiple Browsers:**
- [ ] Chrome/Chromium
- [ ] Firefox
- [ ] Safari (if available)
- [ ] Edge (if available)

### Common Issues and Fixes

**Issue: Receiver doesn't appear**
```bash
# Check browser console for errors (F12)
# Common causes:
# - Syntax error in static_rx.js
# - Missing comma or quote
# - Invalid GPS coordinate format
```

**Issue: Map shows wrong location**
```javascript
// Check GPS format - must be decimal degrees
gps: '(52.2381,6.8577)'  // ✅ Correct
gps: '52.2381,6.8577'    // ❌ Missing parentheses  
gps: '(52°14\'N,6°51\'E)' // ❌ Wrong format
```

**Issue: Link doesn't work**
```javascript
// Ensure URL is complete and accessible
url: 'http://websdr.example.com:8901/'  // ✅ Good
url: 'websdr.example.com'               // ❌ Missing protocol
url: 'https://private-server/'          // ❌ Not publicly accessible
```

## Security Considerations {#security}

### URL Security

**Safe URL Patterns:**
```javascript
// ✅ Safe URLs
'http://websdr.example.com:8901/'
'https://sdr.university.edu/receiver'
'http://192.168.1.100:8901/'  // OK for local testing

// ⚠️ Potentially dangerous
'javascript:alert("xss")'     // Script injection
'data:text/html,<script>'     // Data URI XSS
'file:///etc/passwd'          // Local file access
```

**Validation Function:**
```javascript
function isValidReceiverURL(url) {
    try {
        const parsed = new URL(url);
        
        // Only allow HTTP/HTTPS
        if (!['http:', 'https:'].includes(parsed.protocol)) {
            return false;
        }
        
        // Basic accessibility check
        return true;
    } catch {
        return false;
    }
}
```

### Input Sanitization

**Name Field Guidelines:**
- Avoid HTML tags: `<script>`, `<iframe>`, etc.
- Use plain text descriptions
- Keep names descriptive but concise
- Include location for clarity

**Safe Name Examples:**
```javascript
// ✅ Good names
'0-29 MHz WebSDR, University of Twente, Netherlands'
'2-16 MHz WebSDR, Radio Club, Warsaw, Poland'
'HF WebSDR, Amateur Radio Station, Tokyo, Japan'

// ⚠️ Potentially problematic
'<b>Best</b> WebSDR Ever!'           // HTML tags
'Free WebSDR - Click here!'          // Promotional language
'javascript:alert("name")'           // Script content
```

## Performance Guidelines {#performance}

### File Size Considerations

**Current Status:**
- Each receiver adds ~200-500 bytes
- 100 receivers ≈ 20-50KB
- 1000 receivers ≈ 200-500KB

**Recommendations:**
- Keep descriptions concise
- Avoid duplicate entries
- Consider regional split for 500+ receivers

### Browser Performance

**Marker Limits:**
- < 100 receivers: No performance issues
- 100-500 receivers: Minor impact on initial load
- 500+ receivers: Consider map clustering
- 1000+ receivers: May need optimization

**Optimization Techniques:**
```javascript
// For large datasets, consider lazy loading
// or geographic bounds filtering
function loadReceiversInBounds(north, south, east, west) {
    return static_rx.filter(rx => {
        const coords = parseGPS(rx.gps);
        return coords.lat >= south && coords.lat <= north &&
               coords.lng >= west && coords.lng <= east;
    });
}
```

## Troubleshooting {#troubleshooting}

### Debug Mode

**Enable Browser Developer Tools:**
1. Press F12 or right-click → "Inspect"
2. Go to Console tab
3. Look for error messages
4. Refresh page and watch for errors

### Common Error Messages

**"SyntaxError: Unexpected token"**
```javascript
// Usually means missing comma or quote
// Check around the line number mentioned

// Wrong:
{
    name: 'Receiver'
    url: 'http://...'  // Missing comma after 'Receiver'
}

// Right:
{
    name: 'Receiver',
    url: 'http://...'
}
```

**"ReferenceError: static_rx is not defined"**
```html
<!-- Check that static_rx.js is loaded in index.html -->
<script src="static_rx.js"></script>
```

**"TypeError: Cannot read property of undefined"**
```javascript
// Usually means malformed GPS coordinates
gps: '(52.2381,6.8577)'  // Correct format
```

### Testing Tools

**GPS Coordinate Validation:**
```javascript
// Test coordinates in browser console
function testGPS(gpsString) {
    const match = gpsString.match(/\(([\d\.-]+).*?[, ].*?([\d\.-]+)\)/);
    if (!match) {
        console.error('Invalid GPS format');
        return false;
    }
    
    const lat = parseFloat(match[1]);
    const lng = parseFloat(match[2]);
    
    console.log(`Latitude: ${lat}, Longitude: ${lng}`);
    console.log(`Valid: ${lat >= -90 && lat <= 90 && lng >= -180 && lng <= 180}`);
    
    return true;
}

// Usage:
testGPS('(52.2381,6.8577)');
```

**URL Testing:**
```bash
# Test URL accessibility
curl -I http://websdr.example.com:8901/

# Should return HTTP 200 or 302 for working receivers
```

## Future Migration Path {#migration}

### Database Migration

**When to Migrate:**
- More than 100 static receivers
- Multiple editors needed
- Real-time status updates required
- Admin interface desired

**Migration Steps:**
1. Set up database (MySQL/PostgreSQL)
2. Create migration script
3. Implement REST API
4. Build admin interface
5. Gradual cutover from static files

### API Design Preview

```javascript
// Future API endpoints
GET    /api/receivers              // List all receivers
POST   /api/receivers              // Add new receiver  
PUT    /api/receivers/:id          // Update receiver
DELETE /api/receivers/:id          // Remove receiver
POST   /api/receivers/validate     // Validate URL

// Example usage:
const response = await fetch('/api/receivers', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
        name: 'New WebSDR, Location, Country',
        url: 'http://example.com:8901/',
        latitude: 52.2381,
        longitude: 6.8577
    })
});
```

### Admin Interface Preview

**Features:**
- Web-based receiver management
- Bulk import/export
- URL validation and testing
- Geographic visualization
- User permissions and audit trail

## Best Practices Summary

### Data Quality
1. **Descriptive Names**: Include frequency range, location, country
2. **Accurate Coordinates**: Use precise GPS coordinates
3. **Working URLs**: Test URLs before adding
4. **Complete Information**: Fill optional fields when available

### Code Quality
1. **Consistent Formatting**: Follow existing indentation style
2. **Proper Syntax**: Check commas, quotes, brackets
3. **Validation**: Test in multiple browsers
4. **Documentation**: Comment any unusual configurations

### Security
1. **URL Validation**: Only HTTP/HTTPS protocols
2. **Input Sanitization**: Avoid HTML in names
3. **Access Control**: Limit who can edit static_rx.js
4. **Regular Review**: Periodically check receiver status

### Performance
1. **Reasonable Limits**: Consider splitting large datasets
2. **Optimized Data**: Keep descriptions concise
3. **Regular Cleanup**: Remove dead receivers
4. **Monitoring**: Track page load performance

## Example: Complete Implementation

**Scenario:** Adding a university WebSDR receiver

**Step 1: Gather Information**
- Name: "0-30 MHz WebSDR, Example University, Boston, USA"  
- URL: http://websdr.example.edu:8901/
- Location: Boston (42.3601° N, 71.0589° W)
- Frequency: 0-30 MHz
- Users: 25 max
- Hardware: RTL-SDR array with upconverters
- Antenna: Outdoor longwire antenna

**Step 2: Format Data**
```javascript
{
    name: '0-30 MHz WebSDR, Example University, Boston, USA',
    url: 'http://websdr.example.edu:8901/',
    gps: '(42.3601,-71.0589)',
    bands: '0-30000000',
    users_max: '25',
    sdr_hw: 'RTL-SDR array with upconverters',
    antenna: 'Outdoor longwire antenna',
}
```

**Step 3: Add to static_rx.js**
```javascript
var static_rx = [
    // ... existing receivers ...
    {
        name: '0-30 MHz WebSDR, Example University, Boston, USA',
        url: 'http://websdr.example.edu:8901/',
        gps: '(42.3601,-71.0589)',
        bands: '0-30000000',
        users_max: '25',
        sdr_hw: 'RTL-SDR array with upconverters',
        antenna: 'Outdoor longwire antenna',
    },
];
```

**Step 4: Test and Deploy**
1. Open index.html locally
2. Verify receiver appears near Boston
3. Click link to test connectivity  
4. Check browser console for errors
5. Upload static_rx.js to server

## Support and Resources

### Documentation
- [ARCHITECTURE.md](ARCHITECTURE.md): Complete system architecture
- [IMPLEMENTATION_GUIDE.md](IMPLEMENTATION_GUIDE.md): Detailed implementation steps  
- [MIGRATION_STRATEGY.md](MIGRATION_STRATEGY.md): Future enhancement plans
- [SECURITY_ASSESSMENT.md](SECURITY_ASSESSMENT.md): Security considerations

### Community Resources
- [GitHub Repository](https://github.com/priyom/dyatlov): Official source code
- [Priyom.org](http://priyom.org/): Project background and community
- [KiwiSDR.com](http://kiwisdr.com/public/): KiwiSDR receiver network

### Getting Help
1. Check browser console for error messages
2. Review this guide and documentation
3. Search existing GitHub issues
4. Create new GitHub issue with details:
   - Browser and version
   - Error messages
   - Steps to reproduce
   - Receiver data you're trying to add