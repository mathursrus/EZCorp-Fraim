# EZPawn Component Specifications

## Overview
This document provides detailed technical specifications for each component identified in the EZPawn replication analysis. Each component includes implementation details, props/parameters, state management, and integration requirements.

## Navigation Components

### Header Component (NAV-001)
**Purpose**: Primary site navigation with search, account, and cart functionality

**Technical Specifications**:
```typescript
interface HeaderProps {
  user?: User;
  cartItemCount: number;
  searchQuery?: string;
  onSearch: (query: string) => void;
  onCartClick: () => void;
  onAccountClick: () => void;
}

interface HeaderState {
  isSearchFocused: boolean;
  isMobileMenuOpen: boolean;
  searchSuggestions: SearchSuggestion[];
}
```

**Key Features**:
- Responsive design with mobile hamburger menu
- Real-time search with autocomplete
- Cart item count badge
- Brand family logo display
- Promotional banner integration

**Implementation Notes**:
- Search integrates with external API (miranda-api.wizsearch.in)
- Cart count updates via global state management
- Mobile menu collapses on scroll
- Sticky header behavior on desktop

**Dependencies**:
- Search API service
- Cart state management
- User authentication service
- Responsive breakpoint system

---

### Search Bar Component (NAV-002)
**Purpose**: Product search with autocomplete and suggestions

**Technical Specifications**:
```typescript
interface SearchBarProps {
  placeholder: string;
  value: string;
  onSearch: (query: string) => void;
  onSuggestionSelect: (suggestion: SearchSuggestion) => void;
  isLoading?: boolean;
  suggestions?: SearchSuggestion[];
}

interface SearchSuggestion {
  id: string;
  text: string;
  category?: string;
  resultCount: number;
}
```

**Key Features**:
- Debounced search input (300ms delay)
- Keyboard navigation for suggestions
- Search history integration
- Category-based suggestions
- Mobile-optimized input

**Implementation Notes**:
- Uses external search API for suggestions
- Implements proper accessibility (ARIA labels)
- Handles empty states and error conditions
- Integrates with analytics for search tracking

---

## Filter Components

### Location Filter Component (FILTER-001)
**Purpose**: ZIP code and radius-based location filtering

**Technical Specifications**:
```typescript
interface LocationFilterProps {
  onLocationChange: (location: LocationFilter) => void;
  initialZipCode?: string;
  initialRadius?: number;
  stores: Store[];
}

interface LocationFilter {
  zipCode: string;
  radius: number; // 10-60 miles
  selectedStores: string[]; // store IDs
  useCurrentLocation: boolean;
}

interface Store {
  id: string;
  name: string;
  address: string;
  city: string;
  state: string;
  zipCode: string;
  distance?: number;
  itemCount: number;
}
```

**Key Features**:
- ZIP code validation (5-digit US format)
- Radius slider with visual feedback
- Geolocation API integration
- Store list with distances and item counts
- Multi-store selection capability

**Implementation Notes**:
- Validates ZIP codes against US postal service
- Calculates distances using haversine formula
- Stores user location preferences in localStorage
- Updates product results in real-time
- Handles geolocation permission states

**API Integration**:
```typescript
// Store lookup API call
const getStoresByLocation = async (zipCode: string, radius: number): Promise<Store[]> => {
  const response = await fetch(`/api/stores?zip=${zipCode}&radius=${radius}`);
  return response.json();
};
```

---

### Category Filter Component (FILTER-002)
**Purpose**: Hierarchical product category filtering with item counts

**Technical Specifications**:
```typescript
interface CategoryFilterProps {
  categories: Category[];
  selectedCategories: string[];
  onCategoryChange: (categoryIds: string[]) => void;
  showItemCounts: boolean;
}

interface Category {
  id: string;
  name: string;
  parentId?: string;
  itemCount: number;
  children?: Category[];
  isExpanded?: boolean;
}
```

**Key Features**:
- Hierarchical category structure
- Real-time item count updates
- Multiple category selection
- Expand/collapse functionality
- Search within categories

**Implementation Notes**:
- Categories load from API with current filter context
- Item counts update based on location and other filters
- Supports nested category structures
- Implements checkbox indeterminate states for parent categories
- Optimized for large category lists with virtualization

---

### Price Range Filter Component (FILTER-003)
**Purpose**: Dual-handle price range slider for price filtering

**Technical Specifications**:
```typescript
interface PriceRangeProps {
  minPrice: number;
  maxPrice: number;
  currentRange: [number, number];
  onRangeChange: (range: [number, number]) => void;
  step?: number;
  formatValue?: (value: number) => string;
}

interface PriceRangeState {
  isDragging: boolean;
  activeHandle: 'min' | 'max' | null;
  tempRange: [number, number];
}
```

**Key Features**:
- Dual-handle slider implementation
- Real-time value display
- Touch-friendly mobile interaction
- Keyboard accessibility
- Custom price formatting

**Implementation Notes**:
- Range: $0.00 - $29,000.00 based on inventory
- Debounced updates to prevent excessive API calls
- Handles edge cases (min > max prevention)
- Smooth animations and visual feedback
- Works with both mouse and touch events

**Accessibility**:
- ARIA labels for screen readers
- Keyboard navigation support
- Focus indicators
- Value announcements

---

### Shoe Size Filter Component (FILTER-004)
**Purpose**: Comprehensive shoe size filtering with search functionality

**Technical Specifications**:
```typescript
interface ShoeSizeFilterProps {
  availableSizes: ShoeSize[];
  selectedSizes: string[];
  onSizeChange: (sizeIds: string[]) => void;
  showSearch: boolean;
}

interface ShoeSize {
  id: string;
  display: string; // "10.5 W", "7 M", "3C"
  category: 'adult' | 'youth' | 'child';
  width?: 'W' | 'M' | 'B' | 'D' | 'EE';
  numericSize: number;
  itemCount: number;
}
```

**Key Features**:
- Comprehensive size catalog (1C to 18+)
- Width specifications (W, M, B, D, EE)
- Category grouping (adult, youth, child)
- Size search functionality
- Grid layout with clear visual hierarchy

**Implementation Notes**:
- Sizes sorted by category then numeric value
- Search filters by size number or width
- Multiple size selection with clear visual feedback
- Item counts update based on location filters
- Responsive grid layout for different screen sizes

---

## Product Display Components

### Product Grid Component (PRODUCT-001)
**Purpose**: Responsive grid display for product listings

**Technical Specifications**:
```typescript
interface ProductGridProps {
  products: Product[];
  viewMode: 'grid' | 'list';
  onProductClick: (productId: string) => void;
  onAddToCart: (productId: string) => void;
  isLoading?: boolean;
  hasMore?: boolean;
  onLoadMore?: () => void;
}

interface Product {
  id: string;
  name: string;
  price: number;
  originalPrice?: number;
  images: ProductImage[];
  location: Store;
  category: string;
  condition: 'excellent' | 'good' | 'fair';
  availability: 'available' | 'reserved' | 'sold';
  size?: string;
  brand?: string;
}
```

**Key Features**:
- Responsive grid layout (1-4 columns based on screen size)
- Lazy loading for performance
- Infinite scroll or pagination
- Product image carousel
- Quick add to cart functionality
- Condition indicators

**Implementation Notes**:
- Uses CSS Grid with responsive breakpoints
- Implements intersection observer for lazy loading
- Optimized images with multiple sizes
- Skeleton loading states
- Error handling for failed image loads

---

### Product Card Component (PRODUCT-002)
**Purpose**: Individual product display card with key information

**Technical Specifications**:
```typescript
interface ProductCardProps {
  product: Product;
  viewMode: 'grid' | 'list';
  onProductClick: (productId: string) => void;
  onAddToCart: (productId: string) => void;
  showLocation?: boolean;
  showCondition?: boolean;
}
```

**Key Features**:
- Product image with fallback
- Price display with original price strikethrough
- Location and distance information
- Condition badge
- Quick action buttons
- Responsive layout adaptation

**Implementation Notes**:
- Optimized for both grid and list views
- Implements proper image aspect ratios
- Handles missing product data gracefully
- Includes micro-interactions for better UX
- Accessible button and link implementations

---

## Form Components

### Filter Form Container (FORM-001)
**Purpose**: Container component managing all filter states and interactions

**Technical Specifications**:
```typescript
interface FilterFormProps {
  initialFilters?: FilterState;
  onFiltersChange: (filters: FilterState) => void;
  availableFilters: AvailableFilters;
}

interface FilterState {
  location: LocationFilter;
  categories: string[];
  priceRange: [number, number];
  shoeSizes: string[];
  states: string[];
  searchQuery: string;
}

interface AvailableFilters {
  categories: Category[];
  priceRange: [number, number];
  shoeSizes: ShoeSize[];
  states: State[];
  stores: Store[];
}
```

**Key Features**:
- Centralized filter state management
- Real-time filter application
- Filter persistence across sessions
- Clear all filters functionality
- Filter count indicators

**Implementation Notes**:
- Uses reducer pattern for complex state management
- Debounces filter updates to prevent excessive API calls
- Persists filter state in URL parameters
- Implements optimistic updates for better UX
- Handles filter conflicts and validation

---

## Mobile Components

### Mobile Filter Modal (MOBILE-001)
**Purpose**: Mobile-optimized filter interface in modal format

**Technical Specifications**:
```typescript
interface MobileFilterModalProps {
  isOpen: boolean;
  onClose: () => void;
  filters: FilterState;
  onFiltersChange: (filters: FilterState) => void;
  availableFilters: AvailableFilters;
}
```

**Key Features**:
- Full-screen modal on mobile
- Collapsible filter sections
- Apply/Cancel button pattern
- Touch-optimized controls
- Filter summary display

**Implementation Notes**:
- Uses CSS transforms for smooth animations
- Implements proper focus management
- Prevents body scroll when open
- Optimized for touch interactions
- Includes swipe gestures for closing

---

### Mobile Navigation (MOBILE-002)
**Purpose**: Mobile-optimized navigation with hamburger menu

**Technical Specifications**:
```typescript
interface MobileNavigationProps {
  isOpen: boolean;
  onToggle: () => void;
  user?: User;
  cartItemCount: number;
  onNavigate: (path: string) => void;
}
```

**Key Features**:
- Hamburger menu animation
- Slide-out navigation drawer
- Touch-friendly menu items
- Account and cart quick access
- Search integration

**Implementation Notes**:
- Uses CSS transforms for smooth animations
- Implements proper accessibility (focus trap)
- Optimized for various mobile screen sizes
- Includes gesture support for closing
- Maintains navigation state across pages

---

## State Management Specifications

### Global State Structure
```typescript
interface AppState {
  user: UserState;
  cart: CartState;
  filters: FilterState;
  products: ProductState;
  ui: UIState;
}

interface UserState {
  isAuthenticated: boolean;
  profile?: UserProfile;
  preferences: UserPreferences;
  ezPlusStatus: boolean;
}

interface CartState {
  items: CartItem[];
  total: number;
  itemCount: number;
  isLoading: boolean;
}

interface ProductState {
  items: Product[];
  totalCount: number;
  isLoading: boolean;
  hasMore: boolean;
  error?: string;
}

interface UIState {
  isMobileMenuOpen: boolean;
  isFilterModalOpen: boolean;
  viewMode: 'grid' | 'list';
  notifications: Notification[];
}
```

### API Integration Patterns

#### Product Search API
```typescript
interface ProductSearchRequest {
  query?: string;
  filters: FilterState;
  pagination: {
    page: number;
    limit: number;
  };
  sort?: {
    field: string;
    direction: 'asc' | 'desc';
  };
}

interface ProductSearchResponse {
  products: Product[];
  totalCount: number;
  facets: {
    categories: CategoryFacet[];
    priceRange: [number, number];
    locations: LocationFacet[];
  };
  hasMore: boolean;
}
```

#### Real-time Inventory Updates
```typescript
interface InventoryUpdateEvent {
  productId: string;
  storeId: string;
  availability: 'available' | 'reserved' | 'sold';
  timestamp: string;
}

// WebSocket connection for real-time updates
const inventorySocket = new WebSocket('wss://api.ezpawn.com/inventory');
inventorySocket.onmessage = (event) => {
  const update: InventoryUpdateEvent = JSON.parse(event.data);
  // Update product availability in state
};
```

## Performance Optimization Specifications

### Code Splitting Strategy
```typescript
// Lazy load components for better performance
const ProductGrid = lazy(() => import('./components/ProductGrid'));
const FilterModal = lazy(() => import('./components/FilterModal'));
const ProductDetail = lazy(() => import('./pages/ProductDetail'));

// Route-based code splitting
const routes = [
  {
    path: '/products',
    component: lazy(() => import('./pages/ProductListing'))
  },
  {
    path: '/product/:id',
    component: lazy(() => import('./pages/ProductDetail'))
  }
];
```

### Caching Strategy
```typescript
// API response caching
const cacheConfig = {
  products: { ttl: 300000 }, // 5 minutes
  categories: { ttl: 3600000 }, // 1 hour
  stores: { ttl: 1800000 }, // 30 minutes
  user: { ttl: 900000 } // 15 minutes
};

// Image optimization
const imageConfig = {
  formats: ['webp', 'jpg'],
  sizes: [320, 640, 1024, 1920],
  quality: 80,
  lazyLoading: true
};
```

## Testing Specifications

### Component Testing Strategy
```typescript
// Example test for LocationFilter component
describe('LocationFilter', () => {
  it('validates ZIP code input', () => {
    render(<LocationFilter onLocationChange={mockFn} />);
    const zipInput = screen.getByLabelText('ZIP Code');
    
    fireEvent.change(zipInput, { target: { value: '12345' } });
    expect(zipInput).toHaveValue('12345');
    
    fireEvent.change(zipInput, { target: { value: 'invalid' } });
    expect(screen.getByText('Please enter a valid ZIP code')).toBeInTheDocument();
  });
  
  it('updates radius slider', () => {
    const mockOnChange = jest.fn();
    render(<LocationFilter onLocationChange={mockOnChange} />);
    
    const slider = screen.getByRole('slider');
    fireEvent.change(slider, { target: { value: '25' } });
    
    expect(mockOnChange).toHaveBeenCalledWith(
      expect.objectContaining({ radius: 25 })
    );
  });
});
```

### Integration Testing
```typescript
// API integration tests
describe('Product Search Integration', () => {
  it('filters products by location', async () => {
    const mockResponse = { products: [], totalCount: 0 };
    jest.spyOn(api, 'searchProducts').mockResolvedValue(mockResponse);
    
    render(<ProductListing />);
    
    const zipInput = screen.getByLabelText('ZIP Code');
    fireEvent.change(zipInput, { target: { value: '75201' } });
    
    await waitFor(() => {
      expect(api.searchProducts).toHaveBeenCalledWith(
        expect.objectContaining({
          filters: expect.objectContaining({
            location: expect.objectContaining({ zipCode: '75201' })
          })
        })
      );
    });
  });
});
```

## Deployment and Infrastructure

### Build Configuration
```typescript
// Webpack/Vite configuration for optimal builds
const buildConfig = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all'
        }
      }
    }
  },
  performance: {
    maxAssetSize: 250000,
    maxEntrypointSize: 250000
  }
};
```

### Environment Configuration
```typescript
// Environment-specific configurations
interface EnvironmentConfig {
  apiBaseUrl: string;
  searchApiUrl: string;
  analyticsId: string;
  mapApiKey: string;
  features: {
    realTimeInventory: boolean;
    advancedFiltering: boolean;
    socialLogin: boolean;
  };
}

const configs: Record<string, EnvironmentConfig> = {
  development: {
    apiBaseUrl: 'http://localhost:3001',
    searchApiUrl: 'http://localhost:3002',
    analyticsId: 'dev-analytics',
    mapApiKey: 'dev-map-key',
    features: {
      realTimeInventory: false,
      advancedFiltering: true,
      socialLogin: false
    }
  },
  production: {
    apiBaseUrl: 'https://api.ezpawn.com',
    searchApiUrl: 'https://miranda-api.wizsearch.in',
    analyticsId: 'prod-analytics',
    mapApiKey: 'prod-map-key',
    features: {
      realTimeInventory: true,
      advancedFiltering: true,
      socialLogin: true
    }
  }
};
```

This comprehensive component specification provides the technical foundation for implementing the EZPawn replication with proper architecture, state management, and performance considerations.