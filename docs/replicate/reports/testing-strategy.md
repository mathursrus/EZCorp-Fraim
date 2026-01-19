# EZPawn Testing Strategy

## Overview
This document outlines the comprehensive testing strategy for the EZPawn e-commerce platform replication, covering all aspects from unit tests to end-to-end testing, performance testing, and accessibility validation.

## Testing Philosophy

### Testing Pyramid
```
                    /\
                   /  \
                  / E2E \
                 /______\
                /        \
               /Integration\
              /__________\
             /            \
            /     Unit     \
           /________________\
```

**Distribution**:
- **Unit Tests**: 70% - Fast, isolated, comprehensive coverage
- **Integration Tests**: 20% - API endpoints, database interactions
- **End-to-End Tests**: 10% - Critical user journeys, cross-browser

### Quality Gates
- **Code Coverage**: Minimum 90% for unit tests
- **Performance**: All pages load under 3 seconds
- **Accessibility**: WCAG 2.1 AA compliance
- **Cross-browser**: Support for Chrome, Firefox, Safari, Edge
- **Mobile**: Responsive design works on iOS and Android

## Unit Testing Strategy

### Frontend Unit Tests (Jest + React Testing Library)

#### Component Testing Framework
```typescript
// Example: LocationFilter component test
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { LocationFilter } from '../LocationFilter';

describe('LocationFilter Component', () => {
  const mockProps = {
    onLocationChange: jest.fn(),
    stores: mockStores,
    initialZipCode: '',
    initialRadius: 35
  };

  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('ZIP Code Input', () => {
    it('validates ZIP code format', async () => {
      render(<LocationFilter {...mockProps} />);
      
      const zipInput = screen.getByLabelText(/zip code/i);
      
      // Test valid ZIP code
      fireEvent.change(zipInput, { target: { value: '75201' } });
      expect(screen.queryByText(/invalid zip code/i)).not.toBeInTheDocument();
      
      // Test invalid ZIP code
      fireEvent.change(zipInput, { target: { value: 'invalid' } });
      await waitFor(() => {
        expect(screen.getByText(/invalid zip code/i)).toBeInTheDocument();
      });
    });

    it('calls onLocationChange with valid ZIP code', () => {
      render(<LocationFilter {...mockProps} />);
      
      const zipInput = screen.getByLabelText(/zip code/i);
      fireEvent.change(zipInput, { target: { value: '75201' } });
      fireEvent.blur(zipInput);
      
      expect(mockProps.onLocationChange).toHaveBeenCalledWith(
        expect.objectContaining({
          zipCode: '75201',
          radius: 35
        })
      );
    });
  });

  describe('Radius Slider', () => {
    it('updates radius value', () => {
      render(<LocationFilter {...mockProps} />);
      
      const slider = screen.getByRole('slider');
      fireEvent.change(slider, { target: { value: '25' } });
      
      expect(mockProps.onLocationChange).toHaveBeenCalledWith(
        expect.objectContaining({
          radius: 25
        })
      );
    });

    it('enforces radius limits', () => {
      render(<LocationFilter {...mockProps} />);
      
      const slider = screen.getByRole('slider');
      
      // Test minimum
      fireEvent.change(slider, { target: { value: '5' } });
      expect(slider).toHaveValue('10');
      
      // Test maximum
      fireEvent.change(slider, { target: { value: '100' } });
      expect(slider).toHaveValue('60');
    });
  });

  describe('Geolocation', () => {
    it('requests geolocation permission', async () => {
      const mockGeolocation = {
        getCurrentPosition: jest.fn()
      };
      global.navigator.geolocation = mockGeolocation;
      
      render(<LocationFilter {...mockProps} />);
      
      const geoButton = screen.getByText(/use my current location/i);
      fireEvent.click(geoButton);
      
      expect(mockGeolocation.getCurrentPosition).toHaveBeenCalled();
    });

    it('handles geolocation errors gracefully', async () => {
      const mockGeolocation = {
        getCurrentPosition: jest.fn((success, error) => {
          error({ code: 1, message: 'Permission denied' });
        })
      };
      global.navigator.geolocation = mockGeolocation;
      
      render(<LocationFilter {...mockProps} />);
      
      const geoButton = screen.getByText(/use my current location/i);
      fireEvent.click(geoButton);
      
      await waitFor(() => {
        expect(screen.getByText(/location access denied/i)).toBeInTheDocument();
      });
    });
  });
});
```

#### Hook Testing
```typescript
// Example: useProductSearch hook test
import { renderHook, act } from '@testing-library/react';
import { useProductSearch } from '../hooks/useProductSearch';

describe('useProductSearch Hook', () => {
  it('initializes with default state', () => {
    const { result } = renderHook(() => useProductSearch());
    
    expect(result.current.products).toEqual([]);
    expect(result.current.isLoading).toBe(false);
    expect(result.current.error).toBeNull();
    expect(result.current.totalCount).toBe(0);
  });

  it('handles search with filters', async () => {
    const { result } = renderHook(() => useProductSearch());
    
    await act(async () => {
      result.current.search({
        query: 'laptop',
        filters: {
          location: { zipCode: '75201', radius: 25 },
          categories: ['electronics']
        }
      });
    });
    
    expect(result.current.isLoading).toBe(false);
    expect(result.current.products.length).toBeGreaterThan(0);
    expect(result.current.error).toBeNull();
  });

  it('handles search errors', async () => {
    // Mock API error
    jest.spyOn(api, 'searchProducts').mockRejectedValue(new Error('API Error'));
    
    const { result } = renderHook(() => useProductSearch());
    
    await act(async () => {
      result.current.search({ query: 'test' });
    });
    
    expect(result.current.isLoading).toBe(false);
    expect(result.current.error).toBe('API Error');
    expect(result.current.products).toEqual([]);
  });
});
```

### Backend Unit Tests (Jest + Supertest)

#### API Endpoint Testing
```typescript
// Example: Product API tests
import request from 'supertest';
import { app } from '../app';
import { Product } from '../models/Product';

describe('Product API', () => {
  describe('GET /api/v1/products', () => {
    it('returns products with default pagination', async () => {
      const response = await request(app)
        .get('/api/v1/products')
        .expect(200);
      
      expect(response.body.success).toBe(true);
      expect(response.body.data.products).toBeInstanceOf(Array);
      expect(response.body.meta.pagination).toMatchObject({
        page: 1,
        limit: 20,
        hasNext: expect.any(Boolean),
        hasPrev: false
      });
    });

    it('filters products by location', async () => {
      const response = await request(app)
        .get('/api/v1/products')
        .query({
          zipCode: '75201',
          radius: 25
        })
        .expect(200);
      
      expect(response.body.success).toBe(true);
      expect(response.body.data.products).toBeInstanceOf(Array);
      
      // Verify all products are within radius
      response.body.data.products.forEach(product => {
        expect(product.location.distance).toBeLessThanOrEqual(25);
      });
    });

    it('validates ZIP code format', async () => {
      const response = await request(app)
        .get('/api/v1/products')
        .query({
          zipCode: 'invalid',
          radius: 25
        })
        .expect(400);
      
      expect(response.body.success).toBe(false);
      expect(response.body.error.code).toBe('VALIDATION_ERROR');
      expect(response.body.error.message).toContain('ZIP code');
    });

    it('handles price range filtering', async () => {
      const response = await request(app)
        .get('/api/v1/products')
        .query({
          minPrice: 100,
          maxPrice: 500
        })
        .expect(200);
      
      response.body.data.products.forEach(product => {
        expect(product.price).toBeGreaterThanOrEqual(100);
        expect(product.price).toBeLessThanOrEqual(500);
      });
    });
  });

  describe('GET /api/v1/products/:id', () => {
    it('returns product details', async () => {
      const product = await Product.create(mockProductData);
      
      const response = await request(app)
        .get(`/api/v1/products/${product.id}`)
        .expect(200);
      
      expect(response.body.success).toBe(true);
      expect(response.body.data.product.id).toBe(product.id);
      expect(response.body.data.product.name).toBe(product.name);
    });

    it('returns 404 for non-existent product', async () => {
      const response = await request(app)
        .get('/api/v1/products/non-existent-id')
        .expect(404);
      
      expect(response.body.success).toBe(false);
      expect(response.body.error.code).toBe('PRODUCT_NOT_FOUND');
    });
  });
});
```

#### Service Layer Testing
```typescript
// Example: ProductService tests
import { ProductService } from '../services/ProductService';
import { Product } from '../models/Product';

describe('ProductService', () => {
  let productService: ProductService;

  beforeEach(() => {
    productService = new ProductService();
  });

  describe('searchProducts', () => {
    it('searches products with location filter', async () => {
      const filters = {
        location: {
          zipCode: '75201',
          radius: 25
        }
      };

      const result = await productService.searchProducts(filters);

      expect(result.products).toBeInstanceOf(Array);
      expect(result.totalCount).toBeGreaterThanOrEqual(0);
      expect(result.facets).toHaveProperty('categories');
      expect(result.facets).toHaveProperty('priceRange');
    });

    it('calculates distances correctly', async () => {
      const filters = {
        location: {
          zipCode: '75201',
          radius: 50
        }
      };

      const result = await productService.searchProducts(filters);

      result.products.forEach(product => {
        expect(product.location.distance).toBeDefined();
        expect(product.location.distance).toBeLessThanOrEqual(50);
      });
    });

    it('handles empty results', async () => {
      const filters = {
        query: 'nonexistentproduct12345',
        location: {
          zipCode: '00000',
          radius: 10
        }
      };

      const result = await productService.searchProducts(filters);

      expect(result.products).toEqual([]);
      expect(result.totalCount).toBe(0);
    });
  });

  describe('getProductById', () => {
    it('returns product with related data', async () => {
      const product = await Product.create(mockProductData);

      const result = await productService.getProductById(product.id);

      expect(result.product.id).toBe(product.id);
      expect(result.relatedProducts).toBeInstanceOf(Array);
      expect(result.availableLocations).toBeInstanceOf(Array);
    });

    it('throws error for non-existent product', async () => {
      await expect(
        productService.getProductById('non-existent-id')
      ).rejects.toThrow('Product not found');
    });
  });
});
```

## Integration Testing

### API Integration Tests
```typescript
// Example: Full API integration test
describe('Product Search Integration', () => {
  beforeAll(async () => {
    await setupTestDatabase();
    await seedTestData();
  });

  afterAll(async () => {
    await cleanupTestDatabase();
  });

  it('performs complete product search workflow', async () => {
    // 1. Search for products
    const searchResponse = await request(app)
      .get('/api/v1/products')
      .query({
        q: 'laptop',
        zipCode: '75201',
        radius: 25,
        categoryIds: 'electronics'
      })
      .expect(200);

    expect(searchResponse.body.data.products.length).toBeGreaterThan(0);
    const product = searchResponse.body.data.products[0];

    // 2. Get product details
    const detailResponse = await request(app)
      .get(`/api/v1/products/${product.id}`)
      .expect(200);

    expect(detailResponse.body.data.product.id).toBe(product.id);

    // 3. Check inventory availability
    const availabilityResponse = await request(app)
      .get('/api/v1/inventory/availability')
      .query({
        productIds: product.id,
        storeIds: product.location.id
      })
      .expect(200);

    expect(availabilityResponse.body.data.availability).toBeInstanceOf(Array);
  });

  it('handles location-based filtering correctly', async () => {
    // Test with Dallas ZIP code
    const dallasResponse = await request(app)
      .get('/api/v1/products')
      .query({ zipCode: '75201', radius: 10 })
      .expect(200);

    // Test with Houston ZIP code
    const houstonResponse = await request(app)
      .get('/api/v1/products')
      .query({ zipCode: '77001', radius: 10 })
      .expect(200);

    // Results should be different based on location
    expect(dallasResponse.body.data.products).not.toEqual(
      houstonResponse.body.data.products
    );
  });
});
```

### Database Integration Tests
```typescript
// Example: Database integration tests
describe('Database Integration', () => {
  describe('Product Queries', () => {
    it('performs complex product search with joins', async () => {
      const result = await db.query(`
        SELECT p.*, c.name as category_name, s.name as store_name
        FROM products p
        JOIN categories c ON p.category_id = c.id
        JOIN inventory i ON p.id = i.product_id
        JOIN stores s ON i.store_id = s.id
        WHERE p.price BETWEEN $1 AND $2
        AND ST_DWithin(s.coordinates, ST_Point($3, $4), $5)
      `, [100, 500, -96.7970, 32.7767, 40000]); // 25 miles in meters

      expect(result.rows.length).toBeGreaterThanOrEqual(0);
      result.rows.forEach(row => {
        expect(row.price).toBeGreaterThanOrEqual(100);
        expect(row.price).toBeLessThanOrEqual(500);
      });
    });

    it('maintains referential integrity', async () => {
      // Create product with category
      const category = await db.query(
        'INSERT INTO categories (name, slug) VALUES ($1, $2) RETURNING id',
        ['Test Category', 'test-category']
      );

      const product = await db.query(
        'INSERT INTO products (name, category_id, price, condition) VALUES ($1, $2, $3, $4) RETURNING id',
        ['Test Product', category.rows[0].id, 99.99, 'excellent']
      );

      // Try to delete category (should fail due to foreign key)
      await expect(
        db.query('DELETE FROM categories WHERE id = $1', [category.rows[0].id])
      ).rejects.toThrow();

      // Delete product first, then category (should succeed)
      await db.query('DELETE FROM products WHERE id = $1', [product.rows[0].id]);
      await db.query('DELETE FROM categories WHERE id = $1', [category.rows[0].id]);
    });
  });
});
```

## End-to-End Testing

### Playwright E2E Tests
```typescript
// Example: E2E test for product search workflow
import { test, expect } from '@playwright/test';

test.describe('Product Search Workflow', () => {
  test('user can search and filter products', async ({ page }) => {
    // Navigate to products page
    await page.goto('/products');
    
    // Enter ZIP code
    await page.fill('[data-testid="zip-code-input"]', '75201');
    await page.click('[data-testid="use-location-button"]');
    
    // Wait for stores to load
    await expect(page.locator('[data-testid="store-list"]')).toBeVisible();
    
    // Select a category
    await page.check('[data-testid="category-electronics"]');
    
    // Set price range
    const priceSlider = page.locator('[data-testid="price-slider"]');
    await priceSlider.fill('500');
    
    // Wait for products to load
    await expect(page.locator('[data-testid="product-grid"]')).toBeVisible();
    
    // Verify products are displayed
    const products = page.locator('[data-testid="product-card"]');
    await expect(products).toHaveCountGreaterThan(0);
    
    // Click on first product
    await products.first().click();
    
    // Verify product detail page
    await expect(page.locator('[data-testid="product-detail"]')).toBeVisible();
    await expect(page.locator('[data-testid="product-name"]')).toBeVisible();
    await expect(page.locator('[data-testid="product-price"]')).toBeVisible();
  });

  test('user can add product to cart', async ({ page }) => {
    await page.goto('/products');
    
    // Search for a specific product
    await page.fill('[data-testid="search-input"]', 'laptop');
    await page.press('[data-testid="search-input"]', 'Enter');
    
    // Wait for search results
    await expect(page.locator('[data-testid="product-grid"]')).toBeVisible();
    
    // Click on first product
    await page.locator('[data-testid="product-card"]').first().click();
    
    // Select pickup location
    await page.selectOption('[data-testid="pickup-location"]', { index: 0 });
    
    // Add to cart
    await page.click('[data-testid="add-to-cart-button"]');
    
    // Verify cart count updated
    await expect(page.locator('[data-testid="cart-count"]')).toHaveText('1');
    
    // Open cart
    await page.click('[data-testid="cart-icon"]');
    
    // Verify product in cart
    await expect(page.locator('[data-testid="cart-item"]')).toBeVisible();
  });

  test('mobile responsive design works correctly', async ({ page }) => {
    // Set mobile viewport
    await page.setViewportSize({ width: 375, height: 667 });
    
    await page.goto('/products');
    
    // Verify mobile header
    await expect(page.locator('[data-testid="mobile-header"]')).toBeVisible();
    
    // Open mobile menu
    await page.click('[data-testid="mobile-menu-button"]');
    await expect(page.locator('[data-testid="mobile-menu"]')).toBeVisible();
    
    // Open mobile filters
    await page.click('[data-testid="mobile-filter-button"]');
    await expect(page.locator('[data-testid="mobile-filter-modal"]')).toBeVisible();
    
    // Apply filters
    await page.fill('[data-testid="zip-code-input"]', '75201');
    await page.check('[data-testid="category-electronics"]');
    await page.click('[data-testid="apply-filters-button"]');
    
    // Verify filters applied
    await expect(page.locator('[data-testid="product-grid"]')).toBeVisible();
  });
});
```

### Cross-browser Testing
```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 12'] },
    },
  ],

  webServer: {
    command: 'npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

## Performance Testing

### Load Testing with Artillery
```yaml
# artillery-config.yml
config:
  target: 'http://localhost:3001'
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 120
      arrivalRate: 50
      name: "Ramp up load"
    - duration: 300
      arrivalRate: 100
      name: "Sustained load"

scenarios:
  - name: "Product Search"
    weight: 60
    flow:
      - get:
          url: "/api/v1/products"
          qs:
            zipCode: "75201"
            radius: 25
            page: 1
            limit: 20
      - think: 2
      - get:
          url: "/api/v1/products/{{ $randomString() }}"

  - name: "Category Browsing"
    weight: 30
    flow:
      - get:
          url: "/api/v1/categories"
      - think: 1
      - get:
          url: "/api/v1/products"
          qs:
            categoryIds: "electronics"
            zipCode: "75201"

  - name: "Store Lookup"
    weight: 10
    flow:
      - get:
          url: "/api/v1/stores"
          qs:
            zipCode: "75201"
            radius: 35
```

### Frontend Performance Testing
```typescript
// Performance test with Lighthouse CI
import { test } from '@playwright/test';
import { playAudit } from 'playwright-lighthouse';

test('performance audit', async ({ page }) => {
  await page.goto('/products');
  
  await playAudit({
    page,
    thresholds: {
      performance: 90,
      accessibility: 95,
      'best-practices': 90,
      seo: 90,
    },
    port: 9222,
  });
});
```

## Accessibility Testing

### Automated Accessibility Tests
```typescript
// Example: Accessibility tests with axe-core
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('Accessibility Tests', () => {
  test('product listing page is accessible', async ({ page }) => {
    await page.goto('/products');
    
    const accessibilityScanResults = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21aa'])
      .analyze();
    
    expect(accessibilityScanResults.violations).toEqual([]);
  });

  test('filter components are keyboard accessible', async ({ page }) => {
    await page.goto('/products');
    
    // Test keyboard navigation through filters
    await page.keyboard.press('Tab'); // ZIP code input
    await expect(page.locator('[data-testid="zip-code-input"]')).toBeFocused();
    
    await page.keyboard.press('Tab'); // Radius slider
    await expect(page.locator('[data-testid="radius-slider"]')).toBeFocused();
    
    await page.keyboard.press('Tab'); // First category checkbox
    await expect(page.locator('[data-testid="category-electronics"]')).toBeFocused();
    
    // Test keyboard interaction
    await page.keyboard.press('Space'); // Check category
    await expect(page.locator('[data-testid="category-electronics"]')).toBeChecked();
  });

  test('screen reader compatibility', async ({ page }) => {
    await page.goto('/products');
    
    // Check for proper ARIA labels
    await expect(page.locator('[data-testid="zip-code-input"]')).toHaveAttribute('aria-label');
    await expect(page.locator('[data-testid="radius-slider"]')).toHaveAttribute('aria-label');
    
    // Check for proper heading hierarchy
    const headings = await page.locator('h1, h2, h3, h4, h5, h6').all();
    expect(headings.length).toBeGreaterThan(0);
    
    // Verify first heading is h1
    await expect(page.locator('h1').first()).toBeVisible();
  });
});
```

### Manual Accessibility Testing Checklist
```markdown
## Manual Accessibility Testing Checklist

### Keyboard Navigation
- [ ] All interactive elements are reachable via keyboard
- [ ] Tab order is logical and intuitive
- [ ] Focus indicators are clearly visible
- [ ] Escape key closes modals and dropdowns
- [ ] Arrow keys work in sliders and carousels

### Screen Reader Testing
- [ ] All images have appropriate alt text
- [ ] Form inputs have proper labels
- [ ] Error messages are announced
- [ ] Dynamic content changes are announced
- [ ] Page structure is logical with headings

### Visual Accessibility
- [ ] Color contrast meets WCAG AA standards (4.5:1 for normal text)
- [ ] Information is not conveyed by color alone
- [ ] Text can be zoomed to 200% without horizontal scrolling
- [ ] Focus indicators have sufficient contrast
- [ ] Interactive elements are at least 44px touch targets

### Cognitive Accessibility
- [ ] Error messages are clear and helpful
- [ ] Forms provide clear instructions
- [ ] Time limits can be extended or disabled
- [ ] Complex interactions have help text
- [ ] Consistent navigation throughout site
```

## Test Data Management

### Test Data Factory
```typescript
// Test data factory for consistent test data
export class TestDataFactory {
  static createProduct(overrides: Partial<Product> = {}): Product {
    return {
      id: faker.datatype.uuid(),
      name: faker.commerce.productName(),
      description: faker.commerce.productDescription(),
      price: parseFloat(faker.commerce.price()),
      originalPrice: parseFloat(faker.commerce.price()),
      brand: faker.company.name(),
      condition: faker.helpers.arrayElement(['excellent', 'good', 'fair']),
      category: this.createCategory(),
      images: [this.createProductImage()],
      location: this.createStore(),
      availability: 'available',
      createdAt: faker.date.past().toISOString(),
      updatedAt: faker.date.recent().toISOString(),
      ...overrides
    };
  }

  static createStore(overrides: Partial<Store> = {}): Store {
    return {
      id: faker.datatype.uuid(),
      name: `EZPawn ${faker.address.city()}`,
      address: faker.address.streetAddress(),
      city: faker.address.city(),
      state: faker.address.stateAbbr(),
      zipCode: faker.address.zipCode(),
      phone: faker.phone.number(),
      coordinates: {
        latitude: parseFloat(faker.address.latitude()),
        longitude: parseFloat(faker.address.longitude())
      },
      hours: this.createStoreHours(),
      inventoryCount: faker.datatype.number({ min: 0, max: 500 }),
      ...overrides
    };
  }

  static createUser(overrides: Partial<User> = {}): User {
    return {
      id: faker.datatype.uuid(),
      email: faker.internet.email(),
      firstName: faker.name.firstName(),
      lastName: faker.name.lastName(),
      phone: faker.phone.number(),
      preferences: {},
      ezPlusStatus: {
        isMember: faker.datatype.boolean(),
        points: faker.datatype.number({ min: 0, max: 10000 }),
        tier: faker.helpers.arrayElement(['bronze', 'silver', 'gold'])
      },
      createdAt: faker.date.past().toISOString(),
      lastLoginAt: faker.date.recent().toISOString(),
      ...overrides
    };
  }
}
```

### Database Seeding for Tests
```typescript
// Test database seeding
export class TestDatabaseSeeder {
  static async seedTestData() {
    // Create categories
    const categories = await Promise.all([
      Category.create({ name: 'Electronics', slug: 'electronics' }),
      Category.create({ name: 'Tools', slug: 'tools' }),
      Category.create({ name: 'Shoes', slug: 'shoes' })
    ]);

    // Create stores
    const stores = await Promise.all([
      Store.create(TestDataFactory.createStore({
        city: 'Dallas',
        state: 'TX',
        zipCode: '75201',
        coordinates: { latitude: 32.7767, longitude: -96.7970 }
      })),
      Store.create(TestDataFactory.createStore({
        city: 'Houston',
        state: 'TX',
        zipCode: '77001',
        coordinates: { latitude: 29.7604, longitude: -95.3698 }
      }))
    ]);

    // Create products with inventory
    for (let i = 0; i < 50; i++) {
      const product = await Product.create(TestDataFactory.createProduct({
        categoryId: faker.helpers.arrayElement(categories).id
      }));

      // Add inventory to random stores
      const randomStores = faker.helpers.arrayElements(stores, faker.datatype.number({ min: 1, max: 2 }));
      for (const store of randomStores) {
        await Inventory.create({
          productId: product.id,
          storeId: store.id,
          quantity: faker.datatype.number({ min: 1, max: 5 }),
          availability: 'available'
        });
      }
    }
  }

  static async cleanupTestData() {
    await Inventory.destroy({ where: {} });
    await Product.destroy({ where: {} });
    await Store.destroy({ where: {} });
    await Category.destroy({ where: {} });
    await User.destroy({ where: {} });
  }
}
```

## Continuous Integration Testing

### GitHub Actions Workflow
```yaml
# .github/workflows/test.yml
name: Test Suite

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: ezpawn_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:6
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run unit tests
      run: npm run test:unit
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/ezpawn_test
        REDIS_URL: redis://localhost:6379
    
    - name: Upload coverage reports
      uses: codecov/codecov-action@v3

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: ezpawn_test
      redis:
        image: redis:6

    steps:
    - uses: actions/checkout@v3
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run integration tests
      run: npm run test:integration
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/ezpawn_test
        REDIS_URL: redis://localhost:6379

  e2e-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    
    steps:
    - uses: actions/checkout@v3
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Install Playwright
      run: npx playwright install --with-deps
    
    - name: Build application
      run: npm run build
    
    - name: Run E2E tests
      run: npm run test:e2e
    
    - name: Upload test results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: playwright-report
        path: playwright-report/
```

## Test Reporting and Metrics

### Coverage Reporting
```json
// jest.config.js coverage configuration
{
  "collectCoverage": true,
  "coverageDirectory": "coverage",
  "coverageReporters": ["text", "lcov", "html"],
  "coverageThreshold": {
    "global": {
      "branches": 90,
      "functions": 90,
      "lines": 90,
      "statements": 90
    }
  },
  "collectCoverageFrom": [
    "src/**/*.{js,ts,tsx}",
    "!src/**/*.d.ts",
    "!src/**/*.stories.{js,ts,tsx}",
    "!src/test-utils/**"
  ]
}
```

### Test Metrics Dashboard
```typescript
// Test metrics collection
interface TestMetrics {
  unitTests: {
    total: number;
    passed: number;
    failed: number;
    coverage: number;
    duration: number;
  };
  integrationTests: {
    total: number;
    passed: number;
    failed: number;
    duration: number;
  };
  e2eTests: {
    total: number;
    passed: number;
    failed: number;
    duration: number;
    browsers: string[];
  };
  performance: {
    averageLoadTime: number;
    lighthouseScore: number;
    coreWebVitals: {
      lcp: number;
      fid: number;
      cls: number;
    };
  };
  accessibility: {
    wcagCompliance: number;
    violations: number;
    warnings: number;
  };
}
```

This comprehensive testing strategy ensures high-quality, reliable, and accessible implementation of the EZPawn replication with proper coverage across all testing levels.