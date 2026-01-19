# EZPawn Use Cases

## Use Case: Location-Based Product Discovery

**ID**: UC-001
**Actor**: Customer (Guest/Registered)
**Category**: Product Discovery
**Priority**: High

**Preconditions**: 
- User has access to the EZPawn website
- User knows their location or ZIP code

**Steps**:
1. User navigates to the EZPawn shop page
2. User enters their ZIP code in the location filter
3. User adjusts search radius using the slider (10-60 miles)
4. System displays available stores within radius
5. User selects specific store locations to view inventory
6. System filters products to show only items available at selected locations

**Postconditions**: 
- Product listings show only items available for pickup at selected locations
- User can see exact item counts per location
- User understands pickup requirements

**Screenshots**: 
- `ezpawn-full-loaded.png` (location filtering interface)

**Technical Notes**:
- Uses geolocation API for "use my current location" feature
- Integrates with store inventory management system
- Real-time inventory counts displayed

**Acceptance Criteria**:
- [ ] ZIP code input validates and accepts 5-digit codes
- [ ] Radius slider updates search area dynamically
- [ ] Store list shows accurate item counts
- [ ] Selected stores filter product results
- [ ] Geolocation button requests user permission appropriately

---

## Use Case: Category-Based Product Browsing

**ID**: UC-002
**Actor**: Customer (Guest/Registered)
**Category**: Product Discovery
**Priority**: High

**Preconditions**: 
- User is on the product listing page
- Product categories are loaded

**Steps**:
1. User views the category filter sidebar
2. User selects one or more product categories (Electronics, Tools, Shoes, etc.)
3. System applies category filters to product results
4. User can see item counts for each category
5. User can combine category filters with other filters

**Postconditions**: 
- Product grid shows only items from selected categories
- Filter state is maintained during session
- Item counts update based on other active filters

**Screenshots**: 
- `ezpawn-full-loaded.png` (category filtering interface)

**Technical Notes**:
- Categories have hierarchical structure
- Real-time count updates based on other active filters
- Multiple category selection supported

**Acceptance Criteria**:
- [ ] Category checkboxes toggle selection state
- [ ] Multiple categories can be selected simultaneously
- [ ] Item counts reflect current filter state
- [ ] Category filters combine with location and price filters
- [ ] Clear visual indication of selected categories

---

## Use Case: Price Range Filtering

**ID**: UC-003
**Actor**: Customer (Guest/Registered)
**Category**: Product Discovery
**Priority**: Medium

**Preconditions**: 
- User is viewing product listings
- Price range data is available

**Steps**:
1. User interacts with the price range slider
2. User sets minimum price by dragging left handle
3. User sets maximum price by dragging right handle
4. System updates price display in real-time
5. System filters products within selected price range

**Postconditions**: 
- Only products within price range are displayed
- Price range values are clearly shown
- Filter integrates with other active filters

**Screenshots**: 
- `ezpawn-full-loaded.png` (price slider interface)

**Technical Notes**:
- Dual-handle range slider implementation
- Price range: $0.00 - $29,000.00
- Real-time filtering without page reload

**Acceptance Criteria**:
- [ ] Slider handles move smoothly and accurately
- [ ] Price values update in real-time during drag
- [ ] Products filter immediately upon slider release
- [ ] Price range respects inventory availability
- [ ] Slider works on both desktop and mobile

---

## Use Case: Shoe Size Specific Search

**ID**: UC-004
**Actor**: Customer looking for footwear
**Category**: Specialized Search
**Priority**: Medium

**Preconditions**: 
- User is interested in purchasing shoes
- Shoe inventory is available

**Steps**:
1. User accesses the shoe size filter
2. User searches for specific size using search input
3. User selects from comprehensive size options (adult, youth, child, width)
4. System shows available shoes in selected sizes
5. User can see exact count of shoes per size

**Postconditions**: 
- Only shoes in selected sizes are displayed
- User understands size availability across locations
- Size filter combines with location and other filters

**Screenshots**: 
- `ezpawn-full-loaded.png` (shoe size filtering interface)

**Technical Notes**:
- Extensive size catalog including specialty sizes
- Search functionality within size options
- Size availability varies by location

**Acceptance Criteria**:
- [ ] Size search finds relevant size options
- [ ] All size formats supported (numeric, letter, width)
- [ ] Size selection filters shoe inventory accurately
- [ ] Size counts reflect location-based availability
- [ ] Multiple size selection supported

---

## Use Case: State-Based Inventory Browsing

**ID**: UC-005
**Actor**: Customer in supported states
**Category**: Geographic Filtering
**Priority**: Medium

**Preconditions**: 
- User is in a state where EZPawn operates
- State-based inventory data is available

**Steps**:
1. User views the states filter section
2. User selects their state or states of interest
3. System filters inventory to show items available in selected states
4. User can see item counts per state
5. User understands geographic limitations of online shopping

**Postconditions**: 
- Product listings reflect state-based availability
- User understands pickup location requirements
- State selection integrates with store location filtering

**Screenshots**: 
- `ezpawn-full-loaded.png` (states filtering and notice)

**Technical Notes**:
- Limited to specific states where online shopping is available
- State selection affects store location options
- Clear messaging about geographic limitations

**Acceptance Criteria**:
- [ ] State selection filters available inventory
- [ ] Item counts accurately reflect state-based availability
- [ ] Geographic limitations are clearly communicated
- [ ] State filter integrates with location-based filtering
- [ ] Unsupported states are handled appropriately

---

## Use Case: Product Search

**ID**: UC-006
**Actor**: Customer with specific product in mind
**Category**: Search
**Priority**: High

**Preconditions**: 
- User knows what product they're looking for
- Search functionality is available

**Steps**:
1. User clicks on the search bar in the header
2. User enters search terms for desired product
3. User clicks search button or presses enter
4. System processes search query
5. System displays relevant product results
6. User can combine search with filters

**Postconditions**: 
- Relevant products are displayed based on search terms
- Search results can be further filtered
- Search state is maintained during session

**Screenshots**: 
- `ezpawn-full-loaded.png` (search bar in header)

**Technical Notes**:
- Integrates with external search API (miranda-api.wizsearch.in)
- Search combines with location and category filters
- Real-time search suggestions possible

**Acceptance Criteria**:
- [ ] Search bar accepts text input
- [ ] Search button triggers search functionality
- [ ] Search results are relevant to query
- [ ] Search integrates with existing filters
- [ ] Search state persists during browsing session

---

## Use Case: Account Management

**ID**: UC-007
**Actor**: Registered Customer
**Category**: Account Management
**Priority**: Medium

**Preconditions**: 
- User has an EZPawn account or wants to create one

**Steps**:
1. User clicks "My account" link in header
2. System redirects to login/account page
3. User logs in with credentials or creates new account
4. User accesses account features (order history, preferences, etc.)
5. User can manage EZ+ Rewards membership

**Postconditions**: 
- User is authenticated and can access account features
- Personalized experience based on account data
- EZ+ Rewards benefits are available

**Screenshots**: 
- `ezpawn-full-loaded.png` (account link in header)

**Technical Notes**:
- Account system integrates with EZ+ Rewards program
- Shopify-based account management
- Account affects pricing and rewards eligibility

**Acceptance Criteria**:
- [ ] Account link navigates to login page
- [ ] Login process is secure and user-friendly
- [ ] Account creation is straightforward
- [ ] EZ+ Rewards integration works properly
- [ ] Account preferences affect shopping experience

---

## Use Case: Shopping Cart Management

**ID**: UC-008
**Actor**: Customer ready to purchase
**Category**: E-commerce Transaction
**Priority**: High

**Preconditions**: 
- User has found products they want to purchase
- Products are available for pickup at selected location

**Steps**:
1. User adds products to cart from product listings
2. User clicks cart icon to view cart contents
3. User reviews items, quantities, and pickup locations
4. User proceeds to checkout
5. User completes purchase with pickup arrangement

**Postconditions**: 
- Items are reserved for pickup at specified location
- User receives confirmation and pickup instructions
- Store is notified of pending pickup

**Screenshots**: 
- `ezpawn-full-loaded.png` (cart icon with count)

**Technical Notes**:
- Cart integrates with location-based inventory
- Pickup location must be specified for each item
- Shopify-based cart and checkout system

**Acceptance Criteria**:
- [ ] Cart icon shows accurate item count
- [ ] Cart contents display correctly
- [ ] Pickup locations are clearly specified
- [ ] Checkout process handles pickup requirements
- [ ] Confirmation includes pickup instructions

---

## Use Case: Mobile Shopping Experience

**ID**: UC-009
**Actor**: Customer using mobile device
**Category**: Mobile Experience
**Priority**: High

**Preconditions**: 
- User is accessing site from mobile device
- Mobile-responsive design is implemented

**Steps**:
1. User navigates to EZPawn site on mobile device
2. User interacts with mobile-optimized interface
3. User uses touch-friendly filter controls
4. User browses products in mobile-optimized layout
5. User completes shopping tasks on mobile

**Postconditions**: 
- Full functionality available on mobile device
- User experience is optimized for touch interaction
- Performance is acceptable on mobile networks

**Screenshots**: 
- `ezpawn-full-loaded.png` (responsive design elements visible)

**Technical Notes**:
- Responsive design adapts to various screen sizes
- Touch-friendly controls for sliders and checkboxes
- Mobile-optimized navigation patterns

**Acceptance Criteria**:
- [ ] Site loads quickly on mobile devices
- [ ] All functionality works on touch devices
- [ ] Text and controls are appropriately sized
- [ ] Navigation is intuitive on mobile
- [ ] Performance meets mobile user expectations

---

## Use Case: Brand Family Navigation

**ID**: UC-010
**Actor**: Customer familiar with EZPawn family brands
**Category**: Brand Navigation
**Priority**: Low

**Preconditions**: 
- User recognizes EZPawn family brand names
- Brand navigation is displayed

**Steps**:
1. User views the brand family navigation section
2. User recognizes familiar brand names (Value Pawn, USA Pawn, etc.)
3. User understands these are part of the same network
4. User gains confidence in the platform's legitimacy

**Postconditions**: 
- User has increased trust in the platform
- User understands the scope of the business
- Brand recognition enhances user experience

**Screenshots**: 
- `ezpawn-full-loaded.png` (brand family logos in header)

**Technical Notes**:
- Static brand display for trust building
- Consistent branding across family of companies
- May link to individual brand information

**Acceptance Criteria**:
- [ ] Brand logos display clearly
- [ ] Brand names are recognizable
- [ ] Visual design reinforces brand family connection
- [ ] Brands are presented as unified network
- [ ] Brand information is accessible if needed