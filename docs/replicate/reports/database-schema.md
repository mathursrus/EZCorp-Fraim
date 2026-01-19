# EZPawn Database Schema Design

## Overview
This document defines the complete database schema for the EZPawn e-commerce platform replication, including all tables, relationships, indexes, and constraints necessary to support the application's functionality.

## Database Technology Stack

### Primary Database: PostgreSQL 14+
**Rationale**: 
- ACID compliance for transactional integrity
- Advanced indexing capabilities (GIN, GiST for full-text search)
- JSON/JSONB support for flexible product attributes
- Geospatial support with PostGIS extension
- Excellent performance for complex queries

### Caching Layer: Redis 6+
**Usage**:
- Session storage
- API response caching
- Real-time inventory counts
- Search result caching
- Rate limiting counters

### Search Engine: Elasticsearch 8+
**Usage**:
- Full-text product search
- Faceted search and filtering
- Search analytics and suggestions
- Real-time search indexing

## Core Tables

### Users Table
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    date_of_birth DATE,
    is_active BOOLEAN DEFAULT true,
    is_verified BOOLEAN DEFAULT false,
    email_verified_at TIMESTAMP,
    last_login_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    -- Indexes
    CONSTRAINT users_email_check CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    CONSTRAINT users_phone_check CHECK (phone IS NULL OR phone ~* '^\+?[1-9]\d{1,14}$')
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_active ON users(is_active) WHERE is_active = true;
CREATE INDEX idx_users_created_at ON users(created_at);
```

### User Profiles Table
```sql
CREATE TABLE user_profiles (
    user_id UUID PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    preferred_zip_code VARCHAR(10),
    preferred_radius INTEGER DEFAULT 35,
    preferred_categories TEXT[],
    marketing_opt_in BOOLEAN DEFAULT false,
    notification_preferences JSONB DEFAULT '{}',
    address JSONB,
    preferences JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT user_profiles_radius_check CHECK (preferred_radius BETWEEN 10 AND 60)
);

CREATE INDEX idx_user_profiles_zip ON user_profiles(preferred_zip_code);
```

### EZ+ Rewards Table
```sql
CREATE TABLE ez_plus_memberships (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    member_since DATE NOT NULL,
    current_points INTEGER DEFAULT 0,
    lifetime_points INTEGER DEFAULT 0,
    tier VARCHAR(20) DEFAULT 'bronze',
    tier_expires_at DATE,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT ez_plus_tier_check CHECK (tier IN ('bronze', 'silver', 'gold', 'platinum')),
    CONSTRAINT ez_plus_points_check CHECK (current_points >= 0 AND lifetime_points >= 0)
);

CREATE INDEX idx_ez_plus_user ON ez_plus_memberships(user_id);
CREATE INDEX idx_ez_plus_tier ON ez_plus_memberships(tier);
```

### Stores Table
```sql
CREATE TABLE stores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    store_code VARCHAR(20) UNIQUE NOT NULL,
    brand VARCHAR(50) NOT NULL, -- 'EZPAWN', 'Value Pawn', etc.
    address TEXT NOT NULL,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(2) NOT NULL,
    zip_code VARCHAR(10) NOT NULL,
    phone VARCHAR(20),
    email VARCHAR(255),
    coordinates POINT, -- PostGIS point type
    manager_name VARCHAR(255),
    store_hours JSONB NOT NULL,
    services TEXT[],
    is_active BOOLEAN DEFAULT true,
    supports_online_orders BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT stores_state_check CHECK (LENGTH(state) = 2),
    CONSTRAINT stores_brand_check CHECK (brand IN ('EZPAWN', 'Value Pawn', 'USA Pawn', 'Cash America'))
);

-- Spatial index for location-based queries
CREATE INDEX idx_stores_coordinates ON stores USING GIST(coordinates);
CREATE INDEX idx_stores_state ON stores(state);
CREATE INDEX idx_stores_zip ON stores(zip_code);
CREATE INDEX idx_stores_active ON stores(is_active) WHERE is_active = true;
CREATE INDEX idx_stores_online_orders ON stores(supports_online_orders) WHERE supports_online_orders = true;
```

### Categories Table
```sql
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) UNIQUE NOT NULL,
    description TEXT,
    parent_id UUID REFERENCES categories(id) ON DELETE SET NULL,
    level INTEGER NOT NULL DEFAULT 0,
    sort_order INTEGER DEFAULT 0,
    image_url TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT categories_level_check CHECK (level >= 0 AND level <= 5)
);

CREATE INDEX idx_categories_parent ON categories(parent_id);
CREATE INDEX idx_categories_slug ON categories(slug);
CREATE INDEX idx_categories_active ON categories(is_active) WHERE is_active = true;
CREATE INDEX idx_categories_level ON categories(level);

-- Materialized path for efficient hierarchy queries
ALTER TABLE categories ADD COLUMN path TEXT;
CREATE INDEX idx_categories_path ON categories USING GIN(string_to_array(path, '.'));
```

### Products Table
```sql
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(500) NOT NULL,
    description TEXT,
    detailed_description TEXT,
    sku VARCHAR(100) UNIQUE,
    brand VARCHAR(255),
    model VARCHAR(255),
    category_id UUID NOT NULL REFERENCES categories(id),
    condition VARCHAR(20) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    original_price DECIMAL(10,2),
    cost DECIMAL(10,2), -- Internal cost for margin calculation
    specifications JSONB DEFAULT '{}',
    dimensions JSONB, -- width, height, depth, weight
    is_active BOOLEAN DEFAULT true,
    requires_pickup BOOLEAN DEFAULT true,
    warranty_info JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT products_condition_check CHECK (condition IN ('excellent', 'good', 'fair', 'poor')),
    CONSTRAINT products_price_check CHECK (price > 0),
    CONSTRAINT products_original_price_check CHECK (original_price IS NULL OR original_price >= price)
);

CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_brand ON products(brand);
CREATE INDEX idx_products_condition ON products(condition);
CREATE INDEX idx_products_price ON products(price);
CREATE INDEX idx_products_active ON products(is_active) WHERE is_active = true;
CREATE INDEX idx_products_name_gin ON products USING GIN(to_tsvector('english', name));
CREATE INDEX idx_products_description_gin ON products USING GIN(to_tsvector('english', description));
```

### Product Images Table
```sql
CREATE TABLE product_images (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    url TEXT NOT NULL,
    alt_text TEXT,
    is_primary BOOLEAN DEFAULT false,
    sort_order INTEGER DEFAULT 0,
    width INTEGER,
    height INTEGER,
    file_size INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT product_images_dimensions_check CHECK (
        (width IS NULL AND height IS NULL) OR (width > 0 AND height > 0)
    )
);

CREATE INDEX idx_product_images_product ON product_images(product_id);
CREATE INDEX idx_product_images_primary ON product_images(product_id, is_primary) WHERE is_primary = true;
CREATE UNIQUE INDEX idx_product_images_one_primary ON product_images(product_id) WHERE is_primary = true;
```

### Inventory Table
```sql
CREATE TABLE inventory (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    store_id UUID NOT NULL REFERENCES stores(id) ON DELETE CASCADE,
    quantity INTEGER NOT NULL DEFAULT 1,
    availability VARCHAR(20) NOT NULL DEFAULT 'available',
    reserved_until TIMESTAMP,
    reserved_by UUID REFERENCES users(id) ON DELETE SET NULL,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT inventory_quantity_check CHECK (quantity >= 0),
    CONSTRAINT inventory_availability_check CHECK (
        availability IN ('available', 'reserved', 'sold', 'damaged', 'returned')
    ),
    UNIQUE(product_id, store_id)
);

CREATE INDEX idx_inventory_product ON inventory(product_id);
CREATE INDEX idx_inventory_store ON inventory(store_id);
CREATE INDEX idx_inventory_availability ON inventory(availability);
CREATE INDEX idx_inventory_reserved_until ON inventory(reserved_until) WHERE reserved_until IS NOT NULL;
```

### Shoe Sizes Table
```sql
CREATE TABLE shoe_sizes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    display_name VARCHAR(20) NOT NULL, -- "10.5 W", "7 M", "3C"
    numeric_size DECIMAL(4,2) NOT NULL,
    width VARCHAR(5), -- W, M, B, D, EE, etc.
    category VARCHAR(20) NOT NULL, -- adult, youth, child
    gender VARCHAR(10), -- men, women, unisex
    sort_order INTEGER NOT NULL,
    is_active BOOLEAN DEFAULT true,
    
    CONSTRAINT shoe_sizes_category_check CHECK (category IN ('adult', 'youth', 'child')),
    CONSTRAINT shoe_sizes_gender_check CHECK (gender IN ('men', 'women', 'unisex') OR gender IS NULL),
    UNIQUE(numeric_size, width, category, gender)
);

CREATE INDEX idx_shoe_sizes_category ON shoe_sizes(category);
CREATE INDEX idx_shoe_sizes_numeric ON shoe_sizes(numeric_size);
CREATE INDEX idx_shoe_sizes_active ON shoe_sizes(is_active) WHERE is_active = true;
```

### Product Shoe Sizes Table (for footwear products)
```sql
CREATE TABLE product_shoe_sizes (
    product_id UUID REFERENCES products(id) ON DELETE CASCADE,
    shoe_size_id UUID REFERENCES shoe_sizes(id) ON DELETE CASCADE,
    PRIMARY KEY (product_id, shoe_size_id)
);

CREATE INDEX idx_product_shoe_sizes_product ON product_shoe_sizes(product_id);
CREATE INDEX idx_product_shoe_sizes_size ON product_shoe_sizes(shoe_size_id);
```

## Shopping Cart and Orders

### Shopping Carts Table
```sql
CREATE TABLE shopping_carts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    session_id VARCHAR(255), -- For guest users
    status VARCHAR(20) DEFAULT 'active',
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT carts_user_or_session CHECK (
        (user_id IS NOT NULL AND session_id IS NULL) OR 
        (user_id IS NULL AND session_id IS NOT NULL)
    ),
    CONSTRAINT carts_status_check CHECK (status IN ('active', 'abandoned', 'converted', 'expired'))
);

CREATE INDEX idx_carts_user ON shopping_carts(user_id);
CREATE INDEX idx_carts_session ON shopping_carts(session_id);
CREATE INDEX idx_carts_expires ON shopping_carts(expires_at) WHERE expires_at IS NOT NULL;
```

### Cart Items Table
```sql
CREATE TABLE cart_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    cart_id UUID NOT NULL REFERENCES shopping_carts(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    store_id UUID NOT NULL REFERENCES stores(id) ON DELETE CASCADE,
    quantity INTEGER NOT NULL DEFAULT 1,
    price DECIMAL(10,2) NOT NULL, -- Price at time of adding to cart
    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT cart_items_quantity_check CHECK (quantity > 0),
    UNIQUE(cart_id, product_id, store_id)
);

CREATE INDEX idx_cart_items_cart ON cart_items(cart_id);
CREATE INDEX idx_cart_items_product ON cart_items(product_id);
```

### Orders Table
```sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_number VARCHAR(50) UNIQUE NOT NULL,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    subtotal DECIMAL(10,2) NOT NULL,
    tax_amount DECIMAL(10,2) NOT NULL DEFAULT 0,
    total_amount DECIMAL(10,2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    
    -- Customer information (for guest orders)
    customer_email VARCHAR(255) NOT NULL,
    customer_first_name VARCHAR(100) NOT NULL,
    customer_last_name VARCHAR(100) NOT NULL,
    customer_phone VARCHAR(20),
    
    -- Order metadata
    pickup_instructions TEXT,
    estimated_ready_time TIMESTAMP,
    completed_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    cancellation_reason TEXT,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT orders_status_check CHECK (
        status IN ('pending', 'confirmed', 'processing', 'ready', 'completed', 'cancelled', 'refunded')
    ),
    CONSTRAINT orders_amounts_check CHECK (
        subtotal > 0 AND tax_amount >= 0 AND total_amount > 0 AND 
        total_amount = subtotal + tax_amount
    )
);

CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at);
CREATE INDEX idx_orders_order_number ON orders(order_number);
```

### Order Items Table
```sql
CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id),
    store_id UUID NOT NULL REFERENCES stores(id),
    product_name VARCHAR(500) NOT NULL, -- Snapshot at time of order
    product_sku VARCHAR(100),
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    total_price DECIMAL(10,2) NOT NULL,
    pickup_status VARCHAR(20) DEFAULT 'pending',
    picked_up_at TIMESTAMP,
    picked_up_by VARCHAR(255),
    
    CONSTRAINT order_items_quantity_check CHECK (quantity > 0),
    CONSTRAINT order_items_prices_check CHECK (
        unit_price > 0 AND total_price > 0 AND 
        total_price = unit_price * quantity
    ),
    CONSTRAINT order_items_pickup_status_check CHECK (
        pickup_status IN ('pending', 'ready', 'picked_up', 'cancelled')
    )
);

CREATE INDEX idx_order_items_order ON order_items(order_id);
CREATE INDEX idx_order_items_product ON order_items(product_id);
CREATE INDEX idx_order_items_store ON order_items(store_id);
CREATE INDEX idx_order_items_pickup_status ON order_items(pickup_status);
```

## Search and Analytics

### Search Queries Table
```sql
CREATE TABLE search_queries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    session_id VARCHAR(255),
    query_text TEXT NOT NULL,
    filters JSONB,
    result_count INTEGER,
    clicked_product_id UUID REFERENCES products(id) ON DELETE SET NULL,
    search_duration_ms INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT search_queries_result_count_check CHECK (result_count >= 0)
);

CREATE INDEX idx_search_queries_user ON search_queries(user_id);
CREATE INDEX idx_search_queries_session ON search_queries(session_id);
CREATE INDEX idx_search_queries_text_gin ON search_queries USING GIN(to_tsvector('english', query_text));
CREATE INDEX idx_search_queries_created_at ON search_queries(created_at);
```

### User Events Table (for analytics)
```sql
CREATE TABLE user_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    session_id VARCHAR(255) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB,
    page_url TEXT,
    referrer TEXT,
    user_agent TEXT,
    ip_address INET,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_user_events_user ON user_events(user_id);
CREATE INDEX idx_user_events_session ON user_events(session_id);
CREATE INDEX idx_user_events_type ON user_events(event_type);
CREATE INDEX idx_user_events_created_at ON user_events(created_at);

-- Partition by month for better performance
CREATE TABLE user_events_y2025m01 PARTITION OF user_events
FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
```

## System Tables

### API Keys Table
```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    key_hash VARCHAR(255) UNIQUE NOT NULL,
    permissions TEXT[] NOT NULL,
    rate_limit INTEGER DEFAULT 1000,
    rate_limit_window INTEGER DEFAULT 3600, -- seconds
    is_active BOOLEAN DEFAULT true,
    expires_at TIMESTAMP,
    last_used_at TIMESTAMP,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT api_keys_rate_limit_check CHECK (rate_limit > 0 AND rate_limit_window > 0)
);

CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
CREATE INDEX idx_api_keys_active ON api_keys(is_active) WHERE is_active = true;
```

### System Settings Table
```sql
CREATE TABLE system_settings (
    key VARCHAR(255) PRIMARY KEY,
    value JSONB NOT NULL,
    description TEXT,
    is_public BOOLEAN DEFAULT false,
    updated_by UUID REFERENCES users(id),
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert default settings
INSERT INTO system_settings (key, value, description, is_public) VALUES
('supported_states', '["TX", "AL", "AR", "AZ", "CO", "GA", "IN", "KS", "KY", "LA", "MS", "MO", "NV", "NM", "OK", "SC", "TN", "VA"]', 'States where online shopping is supported', true),
('default_search_radius', '35', 'Default search radius in miles', true),
('max_search_radius', '60', 'Maximum search radius in miles', true),
('min_search_radius', '10', 'Minimum search radius in miles', true),
('cart_expiry_hours', '24', 'Hours before cart expires', false),
('reservation_minutes', '30', 'Minutes to hold reserved items', false);
```

## Views for Common Queries

### Product Search View
```sql
CREATE VIEW product_search_view AS
SELECT 
    p.id,
    p.name,
    p.description,
    p.brand,
    p.price,
    p.original_price,
    p.condition,
    p.is_active,
    c.name as category_name,
    c.path as category_path,
    array_agg(DISTINCT pi.url) as image_urls,
    array_agg(DISTINCT s.id) as available_store_ids,
    array_agg(DISTINCT s.state) as available_states,
    count(DISTINCT i.store_id) as store_count,
    sum(i.quantity) as total_quantity
FROM products p
JOIN categories c ON p.category_id = c.id
LEFT JOIN product_images pi ON p.id = pi.product_id
LEFT JOIN inventory i ON p.id = i.product_id AND i.availability = 'available'
LEFT JOIN stores s ON i.store_id = s.id AND s.is_active = true
WHERE p.is_active = true
GROUP BY p.id, c.name, c.path;
```

### Store Inventory Summary View
```sql
CREATE VIEW store_inventory_summary AS
SELECT 
    s.id as store_id,
    s.name as store_name,
    s.city,
    s.state,
    s.zip_code,
    count(DISTINCT i.product_id) as product_count,
    sum(i.quantity) as total_items,
    count(DISTINCT p.category_id) as category_count
FROM stores s
LEFT JOIN inventory i ON s.id = i.store_id AND i.availability = 'available'
LEFT JOIN products p ON i.product_id = p.id AND p.is_active = true
WHERE s.is_active = true
GROUP BY s.id, s.name, s.city, s.state, s.zip_code;
```

## Functions and Triggers

### Update Timestamps Function
```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Apply to all tables with updated_at column
CREATE TRIGGER update_users_updated_at BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_products_updated_at BEFORE UPDATE ON products
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ... (apply to other tables)
```

### Inventory Update Function
```sql
CREATE OR REPLACE FUNCTION update_inventory_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.last_updated = CURRENT_TIMESTAMP;
    
    -- Notify real-time systems of inventory change
    PERFORM pg_notify('inventory_update', json_build_object(
        'product_id', NEW.product_id,
        'store_id', NEW.store_id,
        'availability', NEW.availability,
        'quantity', NEW.quantity
    )::text);
    
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER inventory_update_trigger BEFORE UPDATE ON inventory
    FOR EACH ROW EXECUTE FUNCTION update_inventory_timestamp();
```

### Category Path Maintenance Function
```sql
CREATE OR REPLACE FUNCTION update_category_path()
RETURNS TRIGGER AS $$
DECLARE
    parent_path TEXT;
BEGIN
    IF NEW.parent_id IS NULL THEN
        NEW.path = NEW.id::text;
        NEW.level = 0;
    ELSE
        SELECT path, level INTO parent_path, NEW.level 
        FROM categories WHERE id = NEW.parent_id;
        
        NEW.path = parent_path || '.' || NEW.id::text;
        NEW.level = NEW.level + 1;
    END IF;
    
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER category_path_trigger BEFORE INSERT OR UPDATE ON categories
    FOR EACH ROW EXECUTE FUNCTION update_category_path();
```

## Indexes for Performance

### Composite Indexes
```sql
-- Product search with location
CREATE INDEX idx_products_category_price ON products(category_id, price);
CREATE INDEX idx_products_brand_condition ON products(brand, condition);

-- Inventory with availability
CREATE INDEX idx_inventory_store_availability ON inventory(store_id, availability);
CREATE INDEX idx_inventory_product_availability ON inventory(product_id, availability);

-- Orders with status and date
CREATE INDEX idx_orders_status_created ON orders(status, created_at);
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- Search optimization
CREATE INDEX idx_search_queries_text_created ON search_queries(query_text, created_at);
```

### Partial Indexes
```sql
-- Only index active records
CREATE INDEX idx_products_active_category ON products(category_id) WHERE is_active = true;
CREATE INDEX idx_stores_active_state ON stores(state) WHERE is_active = true;
CREATE INDEX idx_inventory_available ON inventory(product_id, store_id) WHERE availability = 'available';
```

## Data Seeding Scripts

### Sample Data Insertion
```sql
-- Insert sample categories
INSERT INTO categories (id, name, slug, level, path) VALUES
('cat_electronics', 'Electronics', 'electronics', 0, 'cat_electronics'),
('cat_phones', 'Mobile Phones', 'mobile-phones', 1, 'cat_electronics.cat_phones'),
('cat_laptops', 'Laptops', 'laptops', 1, 'cat_electronics.cat_laptops'),
('cat_tools', 'Tools & Garden', 'tools-garden', 0, 'cat_tools'),
('cat_shoes', 'Shoes', 'shoes', 0, 'cat_shoes');

-- Insert sample stores
INSERT INTO stores (id, name, store_code, brand, address, city, state, zip_code, phone, coordinates, store_hours, is_active) VALUES
('store_dallas_1', 'EZPawn Dallas Central', 'DAL001', 'EZPAWN', '123 Main St', 'Dallas', 'TX', '75201', '214-555-0001', POINT(-96.7970, 32.7767), '{"monday": {"open": "09:00", "close": "19:00"}, "tuesday": {"open": "09:00", "close": "19:00"}}', true),
('store_houston_1', 'EZPawn Houston West', 'HOU001', 'EZPAWN', '456 West Ave', 'Houston', 'TX', '77001', '713-555-0001', POINT(-95.3698, 29.7604), '{"monday": {"open": "09:00", "close": "19:00"}, "tuesday": {"open": "09:00", "close": "19:00"}}', true);

-- Insert sample shoe sizes
INSERT INTO shoe_sizes (id, display_name, numeric_size, width, category, gender, sort_order) VALUES
('size_10_m', '10 M', 10.0, 'M', 'adult', 'men', 100),
('size_10_w', '10 W', 10.0, 'W', 'adult', 'women', 101),
('size_7_m', '7 M', 7.0, 'M', 'adult', 'women', 70),
('size_3c', '3C', 3.0, NULL, 'child', 'unisex', 30);
```

## Backup and Maintenance

### Backup Strategy
```sql
-- Daily full backup
pg_dump -h localhost -U postgres -d ezpawn_db -f backup_$(date +%Y%m%d).sql

-- Point-in-time recovery setup
archive_mode = on
archive_command = 'cp %p /backup/archive/%f'
wal_level = replica
```

### Maintenance Tasks
```sql
-- Weekly maintenance
VACUUM ANALYZE;
REINDEX DATABASE ezpawn_db;

-- Monthly cleanup
DELETE FROM user_events WHERE created_at < NOW() - INTERVAL '6 months';
DELETE FROM search_queries WHERE created_at < NOW() - INTERVAL '1 year';
```

This comprehensive database schema provides a solid foundation for the EZPawn replication with proper normalization, indexing, and performance considerations.