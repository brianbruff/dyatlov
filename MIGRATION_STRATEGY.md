# Migration Strategy: From Static to Dynamic Device Management

## Overview

This document outlines the migration path from the current static file-based device management to a more scalable, dynamic solution with database storage and REST API endpoints.

## Current State Analysis

### Existing Architecture
- **Data Storage**: Static JavaScript files (`static_rx.js`, `kiwisdr_com.js`)
- **Update Process**: Manual editing and file deployment
- **Data Sources**: Two independent sources with different formats
- **User Interface**: No admin interface for device management
- **Validation**: Client-side only, minimal validation

### Current Limitations
1. **Scalability**: Manual process doesn't scale with growth
2. **Real-time Updates**: No dynamic status updates for static receivers
3. **Collaboration**: Multiple editors create merge conflicts
4. **Validation**: No server-side validation or URL verification
5. **Backup/Recovery**: No versioning or audit trail
6. **Performance**: Large files impact page load times

## Migration Phases

### Phase 1: Database Foundation (Weeks 1-2)

#### Database Schema Design
```sql
-- Core receivers table
CREATE TABLE receivers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    url VARCHAR(500) NOT NULL UNIQUE,
    latitude DECIMAL(10,8) NOT NULL,
    longitude DECIMAL(11,8) NOT NULL,
    frequency_min BIGINT, -- Hz
    frequency_max BIGINT, -- Hz
    max_users INTEGER,
    hardware_info TEXT,
    antenna_info TEXT,
    receiver_type ENUM('static', 'kiwisdr', 'websdr') DEFAULT 'static',
    status ENUM('active', 'inactive', 'maintenance') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_by VARCHAR(100),
    updated_by VARCHAR(100)
);

-- Receiver status tracking (for dynamic updates)
CREATE TABLE receiver_status (
    id SERIAL PRIMARY KEY,
    receiver_id INTEGER REFERENCES receivers(id) ON DELETE CASCADE,
    current_users INTEGER,
    max_users INTEGER,
    is_online BOOLEAN DEFAULT true,
    quality_score DECIMAL(3,2), -- 0.00 to 1.00
    last_checked TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    response_time_ms INTEGER,
    INDEX idx_receiver_status_receiver_id (receiver_id),
    INDEX idx_receiver_status_last_checked (last_checked)
);

-- Audit trail for changes
CREATE TABLE receiver_audit (
    id SERIAL PRIMARY KEY,
    receiver_id INTEGER REFERENCES receivers(id) ON DELETE CASCADE,
    action ENUM('create', 'update', 'delete') NOT NULL,
    old_data JSON,
    new_data JSON,
    changed_by VARCHAR(100),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    change_reason TEXT,
    INDEX idx_audit_receiver_id (receiver_id),
    INDEX idx_audit_changed_at (changed_at)
);
```

#### Data Migration Script
```javascript
// migrate-static-data.js
const fs = require('fs');
const mysql = require('mysql2/promise');

async function migrateStaticData() {
    // Read existing static_rx.js
    const staticData = require('./static_rx.js');
    
    const connection = await mysql.createConnection({
        host: process.env.DB_HOST,
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        database: process.env.DB_NAME
    });

    for (const receiver of static_rx) {
        const coords = parseGPS(receiver.gps);
        const bands = parseBands(receiver.bands);
        
        await connection.execute(
            `INSERT INTO receivers 
             (name, url, latitude, longitude, frequency_min, frequency_max, 
              max_users, hardware_info, antenna_info, receiver_type, created_by) 
             VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, 'static', 'migration')`,
            [
                receiver.name,
                receiver.url,
                coords.lat,
                coords.lng,
                bands.min,
                bands.max,
                receiver.users_max ? parseInt(receiver.users_max) : null,
                receiver.sdr_hw,
                receiver.antenna
            ]
        );
    }
    
    await connection.end();
    console.log(`Migrated ${static_rx.length} receivers to database`);
}

function parseGPS(gpsString) {
    const match = gpsString.match(/\(([\d\.-]+).*?[, ].*?([\d\.-]+)\)/);
    return {
        lat: parseFloat(match[1]),
        lng: parseFloat(match[2])
    };
}

function parseBands(bandsString) {
    const [min, max] = bandsString.split('-').map(s => parseInt(s));
    return { min, max };
}
```

### Phase 2: REST API Development (Weeks 3-4)

#### API Endpoints Design
```javascript
// Express.js API routes
const express = require('express');
const router = express.Router();

// GET /api/receivers - List all active receivers
router.get('/receivers', async (req, res) => {
    const { type, bounds, status } = req.query;
    
    let query = `
        SELECT r.*, rs.current_users, rs.is_online, rs.quality_score, rs.last_checked
        FROM receivers r 
        LEFT JOIN receiver_status rs ON r.id = rs.receiver_id 
        WHERE r.status = 'active'
    `;
    
    if (type) query += ` AND r.receiver_type = ?`;
    if (bounds) {
        const [north, south, east, west] = bounds.split(',');
        query += ` AND r.latitude BETWEEN ? AND ? AND r.longitude BETWEEN ? AND ?`;
    }
    
    const receivers = await db.execute(query, params);
    res.json(receivers);
});

// POST /api/receivers - Create new receiver
router.post('/receivers', authenticate, validate, async (req, res) => {
    const {
        name, url, latitude, longitude, frequency_min, frequency_max,
        max_users, hardware_info, antenna_info, receiver_type
    } = req.body;
    
    // Validate URL accessibility
    await validateReceiverURL(url);
    
    const result = await db.execute(
        `INSERT INTO receivers 
         (name, url, latitude, longitude, frequency_min, frequency_max,
          max_users, hardware_info, antenna_info, receiver_type, created_by) 
         VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
        [name, url, latitude, longitude, frequency_min, frequency_max,
         max_users, hardware_info, antenna_info, receiver_type, req.user.id]
    );
    
    res.status(201).json({ id: result.insertId, message: 'Receiver created' });
});

// PUT /api/receivers/:id - Update receiver
router.put('/receivers/:id', authenticate, validate, async (req, res) => {
    const receiverId = req.params.id;
    const updates = req.body;
    
    // Audit trail
    const oldData = await getReceiver(receiverId);
    await logAudit(receiverId, 'update', oldData, updates, req.user.id);
    
    await updateReceiver(receiverId, updates);
    res.json({ message: 'Receiver updated' });
});

// DELETE /api/receivers/:id - Soft delete receiver
router.delete('/receivers/:id', authenticate, async (req, res) => {
    await db.execute(
        'UPDATE receivers SET status = ?, updated_by = ? WHERE id = ?',
        ['inactive', req.user.id, req.params.id]
    );
    res.json({ message: 'Receiver deactivated' });
});
```

#### URL Validation Service
```javascript
// services/url-validator.js
const axios = require('axios');
const { URL } = require('url');

class URLValidator {
    static async validateReceiverURL(url, timeout = 10000) {
        try {
            // Basic URL format validation
            new URL(url);
            
            // Accessibility test
            const response = await axios.get(url, {
                timeout,
                validateStatus: status => status < 500,
                headers: {
                    'User-Agent': 'Dyatlov-MapMaker/1.0 (+https://github.com/priyom/dyatlov)'
                }
            });
            
            // Check for common receiver software signatures
            const body = response.data.toLowerCase();
            const isReceiver = 
                body.includes('websdr') ||
                body.includes('kiwisdr') || 
                body.includes('openwebrx') ||
                body.includes('sdr');
                
            return {
                accessible: response.status < 400,
                isReceiver,
                responseTime: response.headers['x-response-time'],
                status: response.status
            };
            
        } catch (error) {
            return {
                accessible: false,
                isReceiver: false,
                error: error.message
            };
        }
    }
    
    static async batchValidate(receivers) {
        const results = await Promise.allSettled(
            receivers.map(r => this.validateReceiverURL(r.url))
        );
        
        return receivers.map((receiver, index) => ({
            ...receiver,
            validation: results[index].status === 'fulfilled' 
                ? results[index].value 
                : { accessible: false, error: results[index].reason.message }
        }));
    }
}

module.exports = URLValidator;
```

### Phase 3: Admin Interface (Weeks 5-6)

#### React Admin Dashboard
```jsx
// components/ReceiverAdmin.jsx
import React, { useState, useEffect } from 'react';
import { DataGrid } from '@mui/x-data-grid';
import { Button, Dialog, TextField, Select, MenuItem } from '@mui/material';

const ReceiverAdmin = () => {
    const [receivers, setReceivers] = useState([]);
    const [editDialog, setEditDialog] = useState({ open: false, receiver: null });
    
    const columns = [
        { field: 'name', headerName: 'Name', width: 300 },
        { field: 'url', headerName: 'URL', width: 200 },
        { field: 'latitude', headerName: 'Latitude', width: 100 },
        { field: 'longitude', headerName: 'Longitude', width: 100 },
        { field: 'receiver_type', headerName: 'Type', width: 100 },
        { field: 'status', headerName: 'Status', width: 100 },
        {
            field: 'actions',
            headerName: 'Actions',
            width: 200,
            renderCell: (params) => (
                <>
                    <Button onClick={() => editReceiver(params.row)}>Edit</Button>
                    <Button onClick={() => deleteReceiver(params.row.id)}>Delete</Button>
                    <Button onClick={() => testReceiver(params.row.url)}>Test</Button>
                </>
            )
        }
    ];
    
    useEffect(() => {
        fetchReceivers();
    }, []);
    
    const fetchReceivers = async () => {
        const response = await fetch('/api/receivers');
        const data = await response.json();
        setReceivers(data);
    };
    
    const editReceiver = (receiver) => {
        setEditDialog({ open: true, receiver });
    };
    
    const saveReceiver = async (receiverData) => {
        const method = receiverData.id ? 'PUT' : 'POST';
        const url = receiverData.id ? `/api/receivers/${receiverData.id}` : '/api/receivers';
        
        await fetch(url, {
            method,
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(receiverData)
        });
        
        setEditDialog({ open: false, receiver: null });
        fetchReceivers();
    };
    
    const testReceiver = async (url) => {
        try {
            const response = await fetch('/api/receivers/validate', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ url })
            });
            const result = await response.json();
            alert(`URL Test: ${result.accessible ? 'Accessible' : 'Not accessible'}`);
        } catch (error) {
            alert(`Test failed: ${error.message}`);
        }
    };
    
    return (
        <div style={{ height: 600, width: '100%' }}>
            <Button onClick={() => setEditDialog({ open: true, receiver: null })}>
                Add Receiver
            </Button>
            <DataGrid
                rows={receivers}
                columns={columns}
                pageSize={25}
                checkboxSelection
                disableSelectionOnClick
            />
            {/* Edit Dialog Component */}
            <ReceiverEditDialog 
                open={editDialog.open}
                receiver={editDialog.receiver}
                onSave={saveReceiver}
                onClose={() => setEditDialog({ open: false, receiver: null })}
            />
        </div>
    );
};
```

### Phase 4: Progressive Migration (Weeks 7-8)

#### Hybrid Data Loading
```javascript
// hybrid-loader.js - Gradually transition from static to dynamic
class HybridDataLoader {
    constructor(config) {
        this.useDatabase = config.useDatabase || false;
        this.fallbackToStatic = config.fallbackToStatic || true;
    }
    
    async loadReceivers() {
        if (this.useDatabase) {
            try {
                return await this.loadFromAPI();
            } catch (error) {
                console.warn('API failed, falling back to static data:', error);
                if (this.fallbackToStatic) {
                    return this.loadFromStatic();
                }
                throw error;
            }
        }
        return this.loadFromStatic();
    }
    
    async loadFromAPI() {
        const response = await fetch('/api/receivers');
        if (!response.ok) throw new Error(`API error: ${response.status}`);
        return response.json();
    }
    
    loadFromStatic() {
        // Merge static sources as before
        return [].concat(
            (typeof static_rx !== "undefined" && static_rx) ? static_rx : [],
            (typeof kiwisdr_com !== "undefined" && kiwisdr_com) ? kiwisdr_com : []
        );
    }
}

// Update dyatlov.js to use hybrid loader
Dyatlov.prototype.receivers = async function() {
    const loader = new HybridDataLoader({
        useDatabase: window.DYATLOV_CONFIG?.useDatabase || false,
        fallbackToStatic: true
    });
    
    const receiverData = await loader.loadReceivers();
    
    return receiverData.map(function(rx) {
        return new this.RX(rx);
    }, this).filter(function(rx) {
        return (rx.bubble_HTML != null);
    });
};
```

## Security Considerations

### Authentication & Authorization
```javascript
// auth/middleware.js
const jwt = require('jsonwebtoken');

const authenticate = (req, res, next) => {
    const token = req.headers.authorization?.split(' ')[1];
    
    if (!token) {
        return res.status(401).json({ error: 'No token provided' });
    }
    
    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user = decoded;
        next();
    } catch (error) {
        res.status(401).json({ error: 'Invalid token' });
    }
};

const authorize = (roles) => (req, res, next) => {
    if (!roles.includes(req.user.role)) {
        return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
};

module.exports = { authenticate, authorize };
```

### Input Validation
```javascript
// validation/schemas.js
const Joi = require('joi');

const receiverSchema = Joi.object({
    name: Joi.string().min(10).max(255).required(),
    url: Joi.string().uri({ scheme: ['http', 'https'] }).required(),
    latitude: Joi.number().min(-90).max(90).required(),
    longitude: Joi.number().min(-180).max(180).required(),
    frequency_min: Joi.number().integer().min(0).max(300000000),
    frequency_max: Joi.number().integer().min(0).max(300000000),
    max_users: Joi.number().integer().min(1).max(10000),
    hardware_info: Joi.string().max(1000),
    antenna_info: Joi.string().max(1000),
    receiver_type: Joi.string().valid('static', 'kiwisdr', 'websdr')
});

const validateReceiver = (req, res, next) => {
    const { error } = receiverSchema.validate(req.body);
    if (error) {
        return res.status(400).json({ 
            error: 'Validation failed', 
            details: error.details 
        });
    }
    next();
};
```

## Performance Optimization

### Caching Strategy
```javascript
// cache/redis-cache.js
const redis = require('redis');
const client = redis.createClient();

class ReceiverCache {
    static async getReceivers(bounds = null) {
        const key = bounds ? `receivers:${bounds}` : 'receivers:all';
        const cached = await client.get(key);
        
        if (cached) {
            return JSON.parse(cached);
        }
        
        const receivers = await this.fetchFromDatabase(bounds);
        await client.setex(key, 300, JSON.stringify(receivers)); // 5 min cache
        return receivers;
    }
    
    static async invalidateCache(receiverId = null) {
        if (receiverId) {
            await client.del(`receiver:${receiverId}`);
        }
        
        // Clear all receiver caches
        const keys = await client.keys('receivers:*');
        if (keys.length > 0) {
            await client.del(...keys);
        }
    }
}
```

### Database Optimization
```sql
-- Indexes for common queries
CREATE INDEX idx_receivers_location ON receivers(latitude, longitude);
CREATE INDEX idx_receivers_type_status ON receivers(receiver_type, status);
CREATE INDEX idx_receivers_frequency ON receivers(frequency_min, frequency_max);

-- Spatial index for geographic queries
ALTER TABLE receivers ADD COLUMN location POINT GENERATED ALWAYS AS (POINT(longitude, latitude));
CREATE SPATIAL INDEX idx_receivers_spatial ON receivers(location);
```

## Deployment Strategy

### Docker Configuration
```dockerfile
# Dockerfile for API service
FROM node:16-alpine

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .
EXPOSE 3000

CMD ["npm", "start"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DB_HOST=db
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
      
  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=${DB_ROOT_PASSWORD}
      - MYSQL_DATABASE=dyatlov
    volumes:
      - db_data:/var/lib/mysql
      
  redis:
    image: redis:alpine
    
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./static:/usr/share/nginx/html
    depends_on:
      - api

volumes:
  db_data:
```

## Migration Timeline

### Week 1-2: Foundation
- [ ] Set up database schema
- [ ] Create migration scripts
- [ ] Implement basic API endpoints
- [ ] Set up authentication

### Week 3-4: API Development
- [ ] Complete CRUD operations
- [ ] Add URL validation service
- [ ] Implement caching
- [ ] Add rate limiting

### Week 5-6: Admin Interface
- [ ] Build React admin dashboard
- [ ] Add bulk operations
- [ ] Implement audit logging
- [ ] Create monitoring dashboard

### Week 7-8: Migration & Testing
- [ ] Deploy hybrid system
- [ ] Test progressive migration
- [ ] Performance optimization
- [ ] Security auditing

### Week 9-10: Production Deployment
- [ ] Full production deployment
- [ ] Monitor performance
- [ ] User training
- [ ] Documentation updates

## Risk Mitigation

### Data Backup Strategy
- Daily database backups
- Version control for static files
- Audit trail for all changes
- Rollback procedures documented

### Compatibility Assurance
- Maintain backward compatibility during transition
- Feature flags for gradual rollout
- Fallback to static files if API fails
- Browser compatibility testing

### Performance Monitoring
- API response time monitoring
- Database query performance tracking
- Real-time error alerting
- User experience metrics

## Success Metrics

### Performance Goals
- API response time < 200ms
- Page load time < 2 seconds
- 99.9% uptime
- Support 1000+ concurrent users

### Operational Goals
- 90% reduction in manual deployment time
- Real-time receiver status updates
- Automated URL validation
- Comprehensive audit trail

### User Experience Goals
- Admin interface reduces receiver addition time by 80%
- Self-service receiver management
- Automated quality monitoring
- Mobile-responsive admin interface