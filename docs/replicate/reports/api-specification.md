# EZPawn API Specification

## Overview
This document defines the complete API specification for the EZPawn e-commerce platform replication, including all endpoints, request/response formats, authentication, and integration patterns.

## Base Configuration

### API Base URLs
```
Development: http://localhost:3001/api/v1
Staging:     https://staging-api.ezpawn-replica.com/api/v1
Production:  https://api.ezpawn-replica.com/api/v1
```

### Authentication
```typescript
// JWT Token Authentication
interface AuthHeaders {
  'Authorization': 'Bearer <jwt_token>';
  'Content-Type': 'application/json';
  'X-API-Version': 'v1';
}

// API Key for external integrations
interface ExternalAuthHeaders {
  'X-API-Key': '<api_key>';
  'Content-Type': 'application/json';
}
```

### Standard Response Format
```typescript
interface APIResponse<T> {
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
    details?: any;
  };
  meta?: {
    pagination?: PaginationMeta;
    filters?: FilterMeta;
    timestamp: string;
  };
}

interface PaginationMeta {
  page: number;
  limit: number;
  total: number;
  totalPages: number;
  hasNext: boolean;
  hasPrev: boolean;
}
```

## Product API Endpoints

### GET /products
**Purpose**: Retrieve products with filtering and pagination

**Query Parameters**:
```typescript
interface ProductSearchParams {
  // Pagination
  page?: number;           // Default: 1
  limit?: number;          // Default: 20, Max: 100
  
  // Search
  q?: string;              // Search query
  
  // Location filtering
  zipCode?: string;        // 5-digit ZIP code
  radius?: number;         // Miles (10-60)
  storeIds?: string[];     // Specific store IDs
  
  // Category filtering
  categoryIds?: string[];  // Category IDs
  
  // Price filtering
  minPrice?: number;       // Minimum price
  maxPrice?: number;       // Maximum price
  
  // Size filtering (for shoes)
  shoeSizes?: string[];    // Shoe size IDs
  
  // State filtering
  states?: string[];       // State codes
  
  // Sorting
  sortBy?: 'price' | 'distance' | 'name' | 'date';
  sortOrder?: 'asc' | 'desc';
  
  // Additional filters
  condition?: 'excellent' | 'good' | 'fair';
  availability?: 'available' | 'reserved';
  brand?: string;
}
```

**Response**:
```typescript
interface ProductSearchResponse {
  products: Product[];
  facets: {
    categories: CategoryFacet[];
    priceRange: [number, number];
    locations: LocationFacet[];
    brands: BrandFacet[];
    conditions: ConditionFacet[];
  };
  totalCount: number;
  appliedFilters: AppliedFilters;
}

interface Product {
  id: string;
  name: string;
  description: string;
  price: number;
  originalPrice?: number;
  images: ProductImage[];
  category: Category;
  brand?: string;
  condition: 'excellent' | 'good' | 'fair';
  availability: 'available' | 'reserved' | 'sold';
  location: Store;
  size?: string;
  specifications?: Record<string, any>;
  createdAt: string;
  updatedAt: string;
}
```

**Example Request**:
```bash
GET /api/v1/products?zipCode=75201&radius=25&categoryIds=electronics&minPrice=50&maxPrice=500&page=1&limit=20
```

**Example Response**:
```json
{
  "success": true,
  "data": {
    "products": [
      {
        "id": "prod_123",
        "name": "iPhone 12 Pro",
        "description": "Excellent condition iPhone 12 Pro",
        "price": 599.99,
        "originalPrice": 999.99,
        "images": [
          {
            "id": "img_1",
            "url": "https://cdn.ezpawn.com/products/prod_123/main.jpg",
            "alt": "iPhone 12 Pro front view",
            "isPrimary": true
          }
        ],
        "category": {
          "id": "cat_electronics",
          "name": "Electronics",
          "path": "Electronics > Mobile Phones"
        },
        "brand": "Apple",
        "condition": "excellent",
        "availability": "available",
        "location": {
          "id": "store_dallas_1",
          "name": "EZPawn Dallas Central",
          "address": "123 Main St, Dallas, TX 75201",
          "distance": 2.3
        },
        "createdAt": "2025-01-15T10:00:00Z",
        "updatedAt": "2025-01-18T14:30:00Z"
      }
    ],
    "facets": {
      "categories": [
        {
          "id": "cat_electronics",
          "name": "Electronics",
          "count": 156,
          "children": [
            {
              "id": "cat_phones",
              "name": "Mobile Phones",
              "count": 45
            }
          ]
        }
      ],
      "priceRange": [10.00, 2500.00],
      "locations": [
        {
          "id": "store_dallas_1",
          "name": "EZPawn Dallas Central",
          "count": 89,
          "distance": 2.3
        }
      ]
    },
    "totalCount": 156
  },
  "meta": {
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 156,
      "totalPages": 8,
      "hasNext": true,
      "hasPrev": false
    },
    "timestamp": "2025-01-18T15:00:00Z"
  }
}
```

### GET /products/:id
**Purpose**: Get detailed product information

**Response**:
```typescript
interface ProductDetailResponse {
  product: ProductDetail;
  relatedProducts: Product[];
  availableLocations: Store[];
}

interface ProductDetail extends Product {
  detailedDescription: string;
  specifications: Record<string, any>;
  images: ProductImage[];
  reviews?: ProductReview[];
  warranty?: WarrantyInfo;
  pickupInstructions: string;
}
```

## Store/Location API Endpoints

### GET /stores
**Purpose**: Get list of stores with optional location filtering

**Query Parameters**:
```typescript
interface StoreSearchParams {
  zipCode?: string;
  radius?: number;
  state?: string;
  city?: string;
  hasInventory?: boolean;
  page?: number;
  limit?: number;
}
```

**Response**:
```typescript
interface StoreSearchResponse {
  stores: Store[];
  totalCount: number;
}

interface Store {
  id: string;
  name: string;
  address: string;
  city: string;
  state: string;
  zipCode: string;
  phone: string;
  email?: string;
  coordinates: {
    latitude: number;
    longitude: number;
  };
  hours: StoreHours;
  services: string[];
  inventoryCount: number;
  distance?: number; // Only when location filtering is applied
}

interface StoreHours {
  monday: { open: string; close: string; closed?: boolean };
  tuesday: { open: string; close: string; closed?: boolean };
  wednesday: { open: string; close: string; closed?: boolean };
  thursday: { open: string; close: string; closed?: boolean };
  friday: { open: string; close: string; closed?: boolean };
  saturday: { open: string; close: string; closed?: boolean };
  sunday: { open: string; close: string; closed?: boolean };
}
```

### GET /stores/:id
**Purpose**: Get detailed store information

**Response**:
```typescript
interface StoreDetailResponse {
  store: StoreDetail;
  recentInventory: Product[];
  popularCategories: CategoryFacet[];
}

interface StoreDetail extends Store {
  description: string;
  manager: string;
  specialties: string[];
  images: string[];
  directions: string;
  parkingInfo: string;
}
```

## Category API Endpoints

### GET /categories
**Purpose**: Get hierarchical category structure with item counts

**Query Parameters**:
```typescript
interface CategoryParams {
  includeEmpty?: boolean;  // Include categories with 0 items
  locationContext?: {      // Filter counts by location
    zipCode: string;
    radius: number;
  };
  maxDepth?: number;       // Limit hierarchy depth
}
```

**Response**:
```typescript
interface CategoryResponse {
  categories: Category[];
  totalCount: number;
}

interface Category {
  id: string;
  name: string;
  slug: string;
  description?: string;
  parentId?: string;
  level: number;
  itemCount: number;
  children?: Category[];
  image?: string;
  isActive: boolean;
  sortOrder: number;
}
```

## Search API Endpoints

### POST /search/products
**Purpose**: Advanced product search with complex filtering

**Request Body**:
```typescript
interface AdvancedSearchRequest {
  query?: string;
  filters: {
    location?: LocationFilter;
    categories?: string[];
    priceRange?: [number, number];
    shoeSizes?: string[];
    states?: string[];
    brands?: string[];
    conditions?: string[];
    availability?: string[];
  };
  sort?: {
    field: string;
    direction: 'asc' | 'desc';
  };
  pagination: {
    page: number;
    limit: number;
  };
  facets?: string[]; // Which facets to return
}

interface LocationFilter {
  zipCode?: string;
  radius?: number;
  storeIds?: string[];
  coordinates?: {
    latitude: number;
    longitude: number;
    radius: number;
  };
}
```

### GET /search/suggestions
**Purpose**: Get search autocomplete suggestions

**Query Parameters**:
```typescript
interface SuggestionParams {
  q: string;           // Search query (minimum 2 characters)
  limit?: number;      // Default: 10, Max: 20
  types?: string[];    // ['products', 'categories', 'brands']
  locationContext?: {
    zipCode: string;
    radius: number;
  };
}
```

**Response**:
```typescript
interface SuggestionResponse {
  suggestions: SearchSuggestion[];
}

interface SearchSuggestion {
  id: string;
  text: string;
  type: 'product' | 'category' | 'brand';
  category?: string;
  resultCount: number;
  image?: string;
  price?: number;
}
```

## User Account API Endpoints

### POST /auth/register
**Purpose**: User registration

**Request Body**:
```typescript
interface RegisterRequest {
  email: string;
  password: string;
  firstName: string;
  lastName: string;
  phone?: string;
  zipCode?: string;
  acceptTerms: boolean;
  marketingOptIn?: boolean;
}
```

### POST /auth/login
**Purpose**: User authentication

**Request Body**:
```typescript
interface LoginRequest {
  email: string;
  password: string;
  rememberMe?: boolean;
}
```

**Response**:
```typescript
interface AuthResponse {
  user: User;
  token: string;
  refreshToken: string;
  expiresIn: number;
}

interface User {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
  phone?: string;
  preferences: UserPreferences;
  ezPlusStatus: {
    isMember: boolean;
    memberSince?: string;
    points: number;
    tier: 'bronze' | 'silver' | 'gold';
  };
  createdAt: string;
  lastLoginAt: string;
}
```

### GET /users/profile
**Purpose**: Get user profile information

**Headers**: Requires authentication

**Response**:
```typescript
interface ProfileResponse {
  user: UserProfile;
  orderHistory: OrderSummary[];
  savedSearches: SavedSearch[];
  preferences: UserPreferences;
}
```

## Shopping Cart API Endpoints

### GET /cart
**Purpose**: Get current cart contents

**Headers**: Requires authentication or session ID

**Response**:
```typescript
interface CartResponse {
  cart: Cart;
}

interface Cart {
  id: string;
  items: CartItem[];
  subtotal: number;
  tax: number;
  total: number;
  itemCount: number;
  updatedAt: string;
}

interface CartItem {
  id: string;
  product: Product;
  quantity: number;
  price: number;
  pickupLocation: Store;
  reservedUntil: string;
  addedAt: string;
}
```

### POST /cart/items
**Purpose**: Add item to cart

**Request Body**:
```typescript
interface AddToCartRequest {
  productId: string;
  quantity: number;
  pickupLocationId: string;
}
```

### PUT /cart/items/:itemId
**Purpose**: Update cart item

**Request Body**:
```typescript
interface UpdateCartItemRequest {
  quantity?: number;
  pickupLocationId?: string;
}
```

### DELETE /cart/items/:itemId
**Purpose**: Remove item from cart

## Order API Endpoints

### POST /orders
**Purpose**: Create new order

**Request Body**:
```typescript
interface CreateOrderRequest {
  cartId: string;
  customer: {
    firstName: string;
    lastName: string;
    email: string;
    phone: string;
  };
  payment: {
    method: 'card' | 'paypal';
    token: string; // Payment processor token
  };
  pickupInstructions?: string;
}
```

**Response**:
```typescript
interface OrderResponse {
  order: Order;
  paymentStatus: PaymentStatus;
}

interface Order {
  id: string;
  orderNumber: string;
  status: 'pending' | 'confirmed' | 'ready' | 'completed' | 'cancelled';
  items: OrderItem[];
  customer: CustomerInfo;
  subtotal: number;
  tax: number;
  total: number;
  pickupLocations: Store[];
  estimatedReadyTime: string;
  createdAt: string;
  updatedAt: string;
}
```

### GET /orders/:id
**Purpose**: Get order details

**Headers**: Requires authentication

**Response**:
```typescript
interface OrderDetailResponse {
  order: OrderDetail;
  timeline: OrderEvent[];
  pickupInstructions: PickupInstruction[];
}
```

## Inventory API Endpoints

### GET /inventory/availability
**Purpose**: Check real-time product availability

**Query Parameters**:
```typescript
interface AvailabilityParams {
  productIds: string[];
  storeIds?: string[];
}
```

**Response**:
```typescript
interface AvailabilityResponse {
  availability: ProductAvailability[];
}

interface ProductAvailability {
  productId: string;
  storeAvailability: {
    storeId: string;
    available: boolean;
    quantity: number;
    reservedUntil?: string;
    lastUpdated: string;
  }[];
}
```

## Analytics API Endpoints

### POST /analytics/events
**Purpose**: Track user events for analytics

**Request Body**:
```typescript
interface AnalyticsEvent {
  eventType: string;
  userId?: string;
  sessionId: string;
  timestamp: string;
  properties: Record<string, any>;
  context: {
    page: string;
    userAgent: string;
    referrer?: string;
    location?: {
      zipCode: string;
      city: string;
      state: string;
    };
  };
}
```

## WebSocket API for Real-time Updates

### Connection
```typescript
// WebSocket connection for real-time inventory updates
const ws = new WebSocket('wss://api.ezpawn-replica.com/ws');

// Authentication after connection
ws.send(JSON.stringify({
  type: 'auth',
  token: 'jwt_token'
}));

// Subscribe to product updates
ws.send(JSON.stringify({
  type: 'subscribe',
  channel: 'inventory',
  filters: {
    storeIds: ['store_1', 'store_2'],
    categoryIds: ['electronics']
  }
}));
```

### Message Types
```typescript
interface WebSocketMessage {
  type: 'inventory_update' | 'price_change' | 'availability_change';
  data: any;
  timestamp: string;
}

interface InventoryUpdateMessage {
  type: 'inventory_update';
  data: {
    productId: string;
    storeId: string;
    availability: 'available' | 'reserved' | 'sold';
    quantity: number;
  };
}
```

## Error Handling

### Standard Error Codes
```typescript
interface APIError {
  code: string;
  message: string;
  details?: any;
}

// Common error codes
const ERROR_CODES = {
  // Authentication
  'AUTH_REQUIRED': 'Authentication required',
  'AUTH_INVALID': 'Invalid authentication token',
  'AUTH_EXPIRED': 'Authentication token expired',
  
  // Validation
  'VALIDATION_ERROR': 'Request validation failed',
  'INVALID_ZIP_CODE': 'Invalid ZIP code format',
  'INVALID_RADIUS': 'Radius must be between 10 and 60 miles',
  
  // Resources
  'PRODUCT_NOT_FOUND': 'Product not found',
  'STORE_NOT_FOUND': 'Store not found',
  'CATEGORY_NOT_FOUND': 'Category not found',
  
  // Business Logic
  'PRODUCT_UNAVAILABLE': 'Product is no longer available',
  'STORE_CLOSED': 'Store is currently closed',
  'PICKUP_REQUIRED': 'This item requires in-store pickup',
  
  // Rate Limiting
  'RATE_LIMIT_EXCEEDED': 'Too many requests, please try again later',
  
  // Server Errors
  'INTERNAL_ERROR': 'Internal server error',
  'SERVICE_UNAVAILABLE': 'Service temporarily unavailable'
};
```

### HTTP Status Codes
```typescript
// Status code mapping
const STATUS_CODES = {
  200: 'OK',
  201: 'Created',
  400: 'Bad Request',
  401: 'Unauthorized',
  403: 'Forbidden',
  404: 'Not Found',
  409: 'Conflict',
  422: 'Unprocessable Entity',
  429: 'Too Many Requests',
  500: 'Internal Server Error',
  502: 'Bad Gateway',
  503: 'Service Unavailable'
};
```

## Rate Limiting

### Rate Limit Headers
```typescript
interface RateLimitHeaders {
  'X-RateLimit-Limit': string;      // Requests per window
  'X-RateLimit-Remaining': string;  // Remaining requests
  'X-RateLimit-Reset': string;      // Reset timestamp
  'X-RateLimit-Window': string;     // Window duration
}
```

### Rate Limit Rules
```typescript
const RATE_LIMITS = {
  // Per IP address
  anonymous: {
    requests: 100,
    window: '15m'
  },
  
  // Per authenticated user
  authenticated: {
    requests: 1000,
    window: '15m'
  },
  
  // Search endpoints (more restrictive)
  search: {
    requests: 50,
    window: '1m'
  },
  
  // Cart operations
  cart: {
    requests: 20,
    window: '1m'
  }
};
```

## API Versioning

### Version Strategy
- URL-based versioning: `/api/v1/`, `/api/v2/`
- Header-based versioning: `X-API-Version: v1`
- Backward compatibility maintained for at least 12 months
- Deprecation notices provided 6 months in advance

### Version Migration
```typescript
interface VersionMigration {
  from: 'v1';
  to: 'v2';
  changes: {
    breaking: string[];
    deprecated: string[];
    new: string[];
  };
  migrationGuide: string;
}
```

This comprehensive API specification provides the foundation for implementing the EZPawn replication with proper data structures, error handling, and integration patterns.