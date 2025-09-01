# Security Assessment: Dyatlov Map Maker

## Executive Summary

This security assessment evaluates the current Dyatlov Map Maker implementation and identifies security risks associated with adding new receiver URLs to the map. The assessment covers current vulnerabilities, risk levels, and recommended security controls for both the existing system and future enhancements.

### Risk Level: MEDIUM
- **Current Implementation**: Several medium-risk vulnerabilities identified
- **Adding New URLs**: Introduces additional attack vectors without proper controls
- **Future Enhancements**: Significant security improvements possible with proper implementation

## Current Security Posture

### Architecture Overview
- **Client-Side Application**: Pure JavaScript with no server-side components
- **Data Sources**: Static JavaScript files with embedded receiver data
- **External Dependencies**: Third-party map APIs and optional libraries
- **User Input**: Limited to URL navigation (clicking receiver links)

### Existing Security Controls
1. **Static Content Delivery**: Reduces server-side attack surface
2. **Read-Only Data**: No user input modification of receiver data
3. **External URL Handling**: Browser security model applies to receiver links
4. **No Authentication**: Eliminates credential-based attacks

### Current Vulnerabilities

#### 1. Cross-Site Scripting (XSS) - MEDIUM RISK
**Description**: Receiver names and URLs are rendered directly in HTML without sanitization.

**Attack Vector**:
```javascript
// Malicious receiver entry in static_rx.js
{
    name: 'Legitimate Receiver<script>alert("XSS")</script>',
    url: 'javascript:alert("XSS")',
    gps: '(40.7,-74.0)'
}
```

**Impact**: 
- Code execution in user browsers
- Session hijacking
- Credential theft from other sites
- Malware distribution

**Current Code Analysis**:
```javascript
// From dyatlov.js - vulnerable HTML generation
bubble_HTML: function() {
    return '<a href="' + this.xml_escape(this.raw.url) + 
           '" style="color:teal;font-weight:bold;text-decoration:none" title="' + 
           this.xml_escape(this.title) + '">' + 
           this.xml_escape(this.raw.name) + '</a>';
},
```

**Note**: The code does use `xml_escape()` function, which provides some protection, but needs verification.

#### 2. Malicious URL Injection - HIGH RISK
**Description**: Receiver URLs can point to malicious sites or use dangerous protocols.

**Attack Vectors**:
```javascript
// Dangerous URL examples
{
    url: 'javascript:maliciousFunction()',        // JavaScript execution
    url: 'data:text/html,<script>alert(1)</script>', // Data URI XSS
    url: 'file:///etc/passwd',                    // Local file access attempt
    url: 'http://malware-site.com/download.exe'   // Malware distribution
}
```

**Impact**:
- Malware downloads
- Phishing attacks
- Local file system access attempts
- Browser exploitation

#### 3. Content Security Policy (CSP) Absence - MEDIUM RISK
**Description**: No CSP headers restrict resource loading or script execution.

**Impact**:
- Increased XSS impact
- External resource injection
- Data exfiltration channels
- Mixed content vulnerabilities

#### 4. External Dependency Risks - LOW RISK
**Description**: Third-party libraries loaded from external sources.

**Current Dependencies**:
- Google Maps API
- Leaflet library
- Day/night overlay libraries
- Moment.js

**Risks**:
- Supply chain attacks
- CDN compromise
- Version-specific vulnerabilities
- Network interception

#### 5. Information Disclosure - LOW RISK
**Description**: Receiver data exposes geographical and technical information.

**Exposed Information**:
- Exact GPS coordinates
- Hardware specifications
- Software versions
- Network configurations
- Operational patterns

## Risk Assessment Matrix

| Vulnerability | Likelihood | Impact | Risk Level | Exploitability |
|---------------|------------|--------|------------|----------------|
| XSS via receiver names | Medium | High | **MEDIUM** | Medium |
| Malicious URL injection | High | Medium | **HIGH** | High |
| CSP absence | Medium | Medium | **MEDIUM** | Low |
| External dependencies | Low | Medium | **LOW** | Low |
| Information disclosure | High | Low | **LOW** | High |

## URL Validation Security Requirements

### 1. Protocol Allowlisting
```javascript
// Secure URL validation
function validateReceiverURL(url) {
    try {
        const parsed = new URL(url);
        
        // Only allow HTTP/HTTPS protocols
        if (!['http:', 'https:'].includes(parsed.protocol)) {
            throw new Error('Invalid protocol. Only HTTP and HTTPS are allowed.');
        }
        
        // Block dangerous ports
        const dangerousPorts = ['22', '23', '25', '53', '135', '139', '445'];
        if (dangerousPorts.includes(parsed.port)) {
            throw new Error('Port not allowed for security reasons.');
        }
        
        // Block localhost and private IPs in production
        if (isProduction && isPrivateIP(parsed.hostname)) {
            throw new Error('Private IP addresses not allowed in production.');
        }
        
        return true;
    } catch (error) {
        throw new Error(`Invalid URL: ${error.message}`);
    }
}

function isPrivateIP(hostname) {
    const privateRanges = [
        /^127\./,           // Localhost
        /^10\./,            // Private Class A
        /^192\.168\./,      // Private Class C
        /^172\.(1[6-9]|2[0-9]|3[0-1])\./  // Private Class B
    ];
    
    return privateRanges.some(range => range.test(hostname));
}
```

### 2. Content Validation
```javascript
// Validate receiver response content
async function validateReceiverContent(url) {
    try {
        const response = await fetch(url, {
            method: 'HEAD',  // Only fetch headers
            timeout: 5000,
            redirect: 'manual'  // Handle redirects explicitly
        });
        
        const contentType = response.headers.get('content-type');
        
        // Expect HTML content
        if (!contentType?.includes('text/html')) {
            throw new Error('Receiver must serve HTML content');
        }
        
        // Check for suspicious redirects
        if (response.status >= 300 && response.status < 400) {
            const location = response.headers.get('location');
            if (location && !isValidRedirect(location)) {
                throw new Error('Suspicious redirect detected');
            }
        }
        
        return true;
    } catch (error) {
        throw new Error(`Content validation failed: ${error.message}`);
    }
}
```

### 3. Input Sanitization
```javascript
// Enhanced input sanitization
function sanitizeReceiverInput(receiver) {
    return {
        name: sanitizeHTML(receiver.name),
        url: sanitizeURL(receiver.url),
        gps: sanitizeGPS(receiver.gps),
        bands: sanitizeBands(receiver.bands),
        users_max: sanitizeNumber(receiver.users_max),
        sdr_hw: sanitizeHTML(receiver.sdr_hw),
        antenna: sanitizeHTML(receiver.antenna)
    };
}

function sanitizeHTML(input) {
    if (!input) return '';
    
    // Remove HTML tags and encode special characters
    return input
        .replace(/<[^>]*>/g, '')  // Strip HTML tags
        .replace(/&/g, '&amp;')   // Encode ampersands
        .replace(/</g, '&lt;')    // Encode less-than
        .replace(/>/g, '&gt;')    // Encode greater-than
        .replace(/"/g, '&quot;')  // Encode quotes
        .replace(/'/g, '&#x27;')  // Encode apostrophes
        .trim()
        .substring(0, 255);       // Limit length
}

function sanitizeURL(url) {
    try {
        const parsed = new URL(url);
        return parsed.toString();
    } catch {
        throw new Error('Invalid URL format');
    }
}
```

## CORS Security Considerations

### Current CORS Configuration
The application loads receiver content in iframes or new windows, which relies on browser CORS policies.

### Recommended CORS Headers
```javascript
// For future API implementation
const corsHeaders = {
    'Access-Control-Allow-Origin': 'https://your-domain.com',
    'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
    'Access-Control-Allow-Headers': 'Content-Type, Authorization',
    'Access-Control-Max-Age': '86400',
    'Access-Control-Allow-Credentials': 'true'
};
```

### CORS Security Best Practices
1. **Specific Origins**: Never use `Access-Control-Allow-Origin: *` in production
2. **Credential Handling**: Carefully control `Access-Control-Allow-Credentials`
3. **Method Restrictions**: Only allow necessary HTTP methods
4. **Header Validation**: Validate all allowed headers

## Rate Limiting and DDoS Protection

### Client-Side Rate Limiting
```javascript
// Simple client-side rate limiting for URL validation
class RateLimiter {
    constructor(maxRequests = 10, windowMs = 60000) {
        this.requests = [];
        this.maxRequests = maxRequests;
        this.windowMs = windowMs;
    }
    
    canMakeRequest() {
        const now = Date.now();
        this.requests = this.requests.filter(time => now - time < this.windowMs);
        
        if (this.requests.length >= this.maxRequests) {
            return false;
        }
        
        this.requests.push(now);
        return true;
    }
}

// Usage in URL validation
const rateLimiter = new RateLimiter(5, 60000); // 5 requests per minute

async function validateWithRateLimit(url) {
    if (!rateLimiter.canMakeRequest()) {
        throw new Error('Rate limit exceeded. Please wait before validating more URLs.');
    }
    
    return await validateReceiverContent(url);
}
```

## Content Security Policy Implementation

### Recommended CSP Header
```html
<meta http-equiv="Content-Security-Policy" content="
    default-src 'self';
    script-src 'self' 'unsafe-inline' https://maps.googleapis.com https://cdn.jsdelivr.net;
    style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
    img-src 'self' data: https://*.googleapis.com https://*.gstatic.com;
    connect-src 'self' https://maps.googleapis.com;
    frame-src 'none';
    object-src 'none';
    base-uri 'self';
    form-action 'none';
">
```

### CSP Violation Reporting
```html
<meta http-equiv="Content-Security-Policy" content="
    ...; 
    report-uri /csp-report;
    report-to csp-endpoint
">
```

## Authentication and Authorization (Future)

### User Roles and Permissions
```javascript
// Proposed role-based access control
const roles = {
    ADMIN: {
        permissions: ['create', 'read', 'update', 'delete', 'manage_users']
    },
    EDITOR: {
        permissions: ['create', 'read', 'update']
    },
    VIEWER: {
        permissions: ['read']
    },
    CONTRIBUTOR: {
        permissions: ['create', 'read']  // Can submit for approval
    }
};

// Permission middleware
function requirePermission(permission) {
    return (req, res, next) => {
        const userRole = req.user?.role;
        const rolePermissions = roles[userRole]?.permissions || [];
        
        if (!rolePermissions.includes(permission)) {
            return res.status(403).json({ error: 'Insufficient permissions' });
        }
        
        next();
    };
}
```

### Multi-Factor Authentication
```javascript
// MFA implementation for admin functions
const speakeasy = require('speakeasy');

function generateMFASecret(userId) {
    return speakeasy.generateSecret({
        name: `Dyatlov Map (${userId})`,
        issuer: 'Dyatlov Map Maker'
    });
}

function verifyMFAToken(secret, token) {
    return speakeasy.totp.verify({
        secret: secret,
        encoding: 'base32',
        token: token,
        window: 2  // Allow 2 time steps of variance
    });
}
```

## Data Protection and Privacy

### Personal Data Handling
**Current Data Types**:
- Receiver operator names (potentially personal)
- Geographic coordinates (potentially sensitive)
- Technical configurations (operational security)

**GDPR Compliance Requirements**:
1. **Data Minimization**: Only collect necessary receiver information
2. **Consent**: Obtain explicit consent for data publication
3. **Right to Erasure**: Ability to remove receiver listings
4. **Data Portability**: Export receiver data in standard formats
5. **Breach Notification**: Procedures for security incident response

### Data Anonymization
```javascript
// Optional data anonymization for sensitive receivers
function anonymizeReceiver(receiver, level = 'partial') {
    const anonymized = { ...receiver };
    
    switch (level) {
        case 'full':
            anonymized.name = `Anonymous Receiver (${receiver.country})`;
            anonymized.coords = fuzzyCoordinates(receiver.coords, 10000); // 10km radius
            delete anonymized.sdr_hw;
            delete anonymized.antenna;
            break;
            
        case 'partial':
            anonymized.coords = fuzzyCoordinates(receiver.coords, 1000); // 1km radius
            break;
    }
    
    return anonymized;
}

function fuzzyCoordinates(coords, radiusMeters) {
    const earthRadius = 6371000; // Earth radius in meters
    const deltaLat = (radiusMeters / earthRadius) * (180 / Math.PI);
    const deltaLng = deltaLat / Math.cos(coords.lat * Math.PI / 180);
    
    return {
        lat: coords.lat + (Math.random() - 0.5) * deltaLat,
        lng: coords.lng + (Math.random() - 0.5) * deltaLng
    };
}
```

## Security Monitoring and Logging

### Security Event Logging
```javascript
// Security event logging framework
class SecurityLogger {
    static logSecurityEvent(event, details = {}) {
        const logEntry = {
            timestamp: new Date().toISOString(),
            event: event,
            details: details,
            userAgent: details.userAgent || 'unknown',
            ipAddress: details.ipAddress || 'unknown',
            severity: this.getSeverity(event)
        };
        
        // Send to security monitoring system
        console.log('[SECURITY]', JSON.stringify(logEntry));
        
        // Alert on high-severity events
        if (logEntry.severity === 'HIGH') {
            this.sendAlert(logEntry);
        }
    }
    
    static getSeverity(event) {
        const highSeverityEvents = [
            'xss_attempt', 'malicious_url', 'brute_force', 
            'privilege_escalation', 'data_breach'
        ];
        
        return highSeverityEvents.includes(event) ? 'HIGH' : 'MEDIUM';
    }
    
    static sendAlert(logEntry) {
        // Implementation for alerting system
        // Could send email, Slack message, etc.
    }
}

// Usage examples
SecurityLogger.logSecurityEvent('url_validation_failed', {
    url: suspiciousUrl,
    reason: 'malicious_protocol',
    ipAddress: req.ip
});

SecurityLogger.logSecurityEvent('xss_attempt', {
    payload: sanitizedInput,
    originalInput: rawInput,
    blocked: true
});
```

## Incident Response Plan

### Security Incident Categories
1. **Code Injection**: XSS, script injection, HTML injection
2. **Malicious Content**: Malware URLs, phishing sites
3. **Data Breach**: Unauthorized access to receiver data
4. **Service Disruption**: DDoS, resource exhaustion
5. **Supply Chain**: Compromised dependencies

### Response Procedures
```markdown
## Incident Response Checklist

### Immediate Response (0-1 hour)
- [ ] Identify and classify the incident
- [ ] Isolate affected systems
- [ ] Preserve evidence
- [ ] Notify incident response team
- [ ] Document initial findings

### Investigation (1-24 hours)
- [ ] Analyze attack vectors
- [ ] Assess scope of compromise
- [ ] Identify affected data/users
- [ ] Determine root cause
- [ ] Implement containment measures

### Recovery (24-72 hours)
- [ ] Remove malicious content
- [ ] Apply security patches
- [ ] Restore from clean backups
- [ ] Verify system integrity
- [ ] Monitor for recurring issues

### Post-Incident (1-2 weeks)
- [ ] Conduct lessons learned review
- [ ] Update security procedures
- [ ] Implement preventive controls
- [ ] Report to stakeholders
- [ ] Document recommendations
```

## Security Testing Recommendations

### Automated Security Testing
```javascript
// Example security test cases
describe('URL Validation Security', () => {
    const maliciousURLs = [
        'javascript:alert(1)',
        'data:text/html,<script>alert(1)</script>',
        'file:///etc/passwd',
        'ftp://malicious.com/file.exe',
        'http://127.0.0.1:22/',
        'https://phishing-site.com'
    ];
    
    maliciousURLs.forEach(url => {
        test(`should reject malicious URL: ${url}`, () => {
            expect(() => validateReceiverURL(url)).toThrow();
        });
    });
});

describe('Input Sanitization', () => {
    test('should sanitize HTML in receiver names', () => {
        const maliciousName = 'Receiver<script>alert(1)</script>';
        const sanitized = sanitizeHTML(maliciousName);
        expect(sanitized).not.toContain('<script>');
        expect(sanitized).toBe('Receiver');
    });
});
```

### Manual Security Testing
1. **XSS Testing**: Inject various XSS payloads in receiver data
2. **URL Manipulation**: Test various malicious URL formats
3. **CORS Testing**: Verify cross-origin request handling
4. **Rate Limiting**: Test rate limiting effectiveness
5. **CSP Testing**: Verify Content Security Policy enforcement

## Compliance and Standards

### Security Standards Alignment
- **OWASP Top 10**: Address common web application vulnerabilities
- **NIST Cybersecurity Framework**: Implement security controls framework
- **ISO 27001**: Information security management best practices

### Regular Security Assessments
1. **Quarterly Code Reviews**: Manual security code review
2. **Annual Penetration Testing**: External security assessment
3. **Monthly Dependency Audits**: Check for vulnerable dependencies
4. **Continuous Monitoring**: Automated security scanning

## Recommendations

### Immediate Actions (High Priority)
1. **Implement URL Validation**: Strict protocol and domain validation
2. **Add CSP Headers**: Implement Content Security Policy
3. **Enhance Input Sanitization**: Strengthen HTML encoding
4. **Security Testing**: Implement automated security tests

### Short-term Improvements (Medium Priority)
1. **Rate Limiting**: Implement client-side rate limiting
2. **Security Logging**: Add comprehensive security event logging
3. **Dependency Management**: Use package-lock.json and audit regularly
4. **Incident Response**: Document security incident procedures

### Long-term Enhancements (Future Planning)
1. **Authentication System**: Implement proper user authentication
2. **API Security**: Secure REST API with proper authorization
3. **Database Security**: Encrypted storage and secure access controls
4. **Monitoring System**: Real-time security monitoring and alerting

## Conclusion

The current Dyatlov Map Maker has moderate security risks primarily related to XSS and malicious URL injection. While the static nature of the application limits the attack surface, adding new receiver URLs introduces additional security considerations that must be addressed through proper validation, sanitization, and monitoring.

Implementation of the recommended security controls will significantly improve the security posture and enable safe expansion of the receiver network while protecting users from malicious content and maintaining system integrity.