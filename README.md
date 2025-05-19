**It’s 8:10 PM IST on Monday, May 19, 2025.**

I understand that the homepage (`/home`) currently shows the same content (a banner, top picks, and categories) for all users, but you want to display role-specific content for each user type (Customer, Guest, Merchant, Delivery Agent, Admin). The current `HomeComponent` is designed with a customer-focused layout, including product listings and a cart feature, which isn’t suitable for other roles like Merchants, Delivery Agents, or Admins. We’ll modify the `HomeComponent` to show role-specific content while preserving the existing functionality for Customers and Guests.

---

### Step 1: Plan the Changes
We need to:
1. **Determine the User’s Role**: Use `AuthService` to check the user’s role.
2. **Show Role-Specific Content**:
   - **Customer/Guest**: Keep the current layout (banner, top picks, categories, and cart functionality).
   - **Merchant**: Show a welcome message with a link to `/merchant-dashboard`.
   - **Delivery Agent**: Show a welcome message with a link to `/delivery-agent-home`.
   - **Admin**: Show a welcome message with a link to `/admin-home`.
3. **Preserve Existing Functionality**: Ensure the product listings, cart modals, and other features remain functional for Customers and Guests.

---

### Step 2: Update `HomeComponent` Files
#### Update `home.component.ts`
We’ll add logic to determine the user’s role and conditionally load the product data only for Customers and Guests.

**`home.component.ts` (Updated)**:
```typescript
import { Component, OnInit } from '@angular/core';
import { RouterLink } from '@angular/router';
import { CommonModule } from '@angular/common';
import { ProductService, Product } from '../../services/product.service';
import { CartRequest, CartService } from '../../services/cart.service';
import { AuthService } from '../../services/auth.service';
import { FormsModule } from '@angular/forms';

@Component({
  selector: 'app-home',
  standalone: true,
  imports: [RouterLink, CommonModule, FormsModule],
  templateUrl: './home.component.html',
  styleUrls: ['./home.component.css']
})
export class HomeComponent implements OnInit {
  products: Product[] = [];
  topPicks: Product[] = [];
  categories: string[] = [];
  selectedProductId: number | null = null;
  quantity: number = 1;
  showQuantityModal: boolean = false;
  showLoginModal: boolean = false;
  userRole: string | null = null;

  constructor(
    private productService: ProductService,
    private cartService: CartService,
    private authService: AuthService
  ) {}

  ngOnInit() {
    // Determine user role
    if (this.authService.isLoggedIn()) {
      this.userRole = this.authService.getUserRole();
    } else {
      this.userRole = 'Guest';
    }

    // Load products only for Customer and Guest users
    if (this.userRole === 'Customer' || this.userRole === 'Guest') {
      this.productService.getProducts().subscribe({
        next: (response) => {
          if (response.success && response.data && response.data.length > 0) {
            this.products = response.data;
            this.topPicks = response.data.slice(0, 4);
            this.categories = [...new Set(response.data.map(p => p.category).filter(c => c))];
          } else {
            console.error('No products received:', response.message);
            this.products = [];
            this.topPicks = [];
            this.categories = [];
          }
        },
        error: (err) => {
          console.error('Error fetching products:', err);
          this.products = [];
          this.topPicks = [];
          this.categories = [];
        }
      });
    }
  }

  getFirstImage(images?: string): string {
    return images ? images.split(',')[0] : 'https://via.placeholder.com/300x200?text=No+Image';
  }

  handleImageError(event: Event) {
    const imgElement = event.target as HTMLImageElement;
    if (imgElement.src !== 'https://via.placeholder.com/300x200?text=No+Image') {
      imgElement.src = 'https://via.placeholder.com/300x200?text=No+Image';
      imgElement.onerror = null;
    }
  }

  showLoginDialog() {
    this.showLoginModal = true;
  }

  closeLoginModal() {
    this.showLoginModal = false;
  }

  addToCart(productId: number) {
    const isLoggedIn = this.authService.isLoggedIn();
    console.log('Is user logged in:', isLoggedIn);
    if (!isLoggedIn) {
      this.showLoginDialog();
    } else {
      this.selectedProductId = productId;
      this.quantity = 1;
      this.showQuantityModal = true;
      console.log('Show quantity modal:', this.showQuantityModal);
    }
  }

  confirmAddToCart() {
    if (this.selectedProductId) {
      const cartRequest: CartRequest = {
        productId: this.selectedProductId,
        quantity: this.quantity
      };
      this.cartService.addToCart(cartRequest).subscribe({
        next: (response) => {
          if (response.success) {
            console.log('Added to cart:', response.data);
            this.closeQuantityModal();
          } else {
            console.error('Failed to add to cart:', response.message);
          }
        },
        error: (err) => {
          console.error('Error adding to cart:', err);
        }
      });
    }
  }

  closeQuantityModal() {
    this.showQuantityModal = false;
    this.selectedProductId = null;
    this.quantity = 1;
  }
}
```

- **Changes**:
  - Added `userRole` property to store the user’s role.
  - Set `userRole` to `'Guest'` for unauthenticated users or fetched it using `authService.getUserRole()` for logged-in users.
  - Wrapped the product fetching logic in an `if` condition to load products only for Customers and Guests.

#### Update `home.component.html`
We’ll modify the template to show role-specific content while keeping the existing layout for Customers and Guests.

**`home.component.html` (Updated)**:
```html
<!-- Role-Specific Content -->
<ng-container *ngIf="userRole === 'Customer' || userRole === 'Guest'; else roleSpecificContent">
  <!-- Banner -->
  <div class="bg-primary text-white py-5 mb-4">
    <div class="container text-center px-5">
      <h1 class="display-4 fw-bold">Welcome to EShoppingZone</h1>
      <p class="lead mb-4">Discover the best deals on your favorite products!</p>
      <a class="btn btn-light btn-lg" [routerLink]="['/products']">Shop Now</a>
    </div>
  </div>

  <!-- Top Picks for You -->
  <div class="container mb-5 px-5" *ngIf="products.length > 0; else loading">
    <h2 class="fw-bold mb-4">Top Picks for You</h2>
    <div class="row row-cols-1 row-cols-sm-2 row-cols-lg-4 g-4">
      <div class="col" *ngFor="let product of topPicks">
        <div class="card h-100 border">
          <a [routerLink]="['/product', product.id]">
            <div class="image-container">
              <img [src]="getFirstImage(product.images)" class="card-img-top" [alt]="product.name" (error)="handleImageError($event)">
            </div>
          </a>
          <div class="card-body">
            <a [routerLink]="['/product', product.id]" class="text-decoration-none">
              <h6 class="card-title">{{ product.name }}</h6>
              <p class="card-text text-muted">{{ product.price | currency : 'INR' }}</p>
              <div class="d-flex align-items-center">
                <ng-container *ngFor="let i of [1,2,3,4,5]; let index = index">
                  <i class="bi bi-star-fill text-warning me-1" *ngIf="product.averageRating >= index + 1"></i>
                  <i class="bi bi-star text-warning me-1" *ngIf="product.averageRating <= index && product.averageRating < index + 1"></i>
                  <i class="bi bi-star-fill text-warning me-1" *ngIf="product.averageRating > index && product.averageRating < index + 1" [style.width]="(product.averageRating - index) * 16 + 'px'" style="overflow: hidden; position: absolute;"></i>
                </ng-container>
                <span class="text-muted small">({{ product.reviewCount }})</span>
              </div>
            </a>
            <button class="btn btn-primary w-100 mt-2" (click)="addToCart(product.id)">Add to Cart</button>
          </div>
        </div>
      </div>
    </div>
  </div>

  <!-- Categories -->
  <div class="container mb-5 px-5" *ngIf="categories.length > 0">
    <h2 class="fw-bold mb-4">Explore Categories</h2>
    <div class="row row-cols-1 row-cols-sm-2 row-cols-lg-4 g-4">
      <div class="col" *ngFor="let category of categories">
        <div class="card text-center border">
          <div class="card-body">
            <h5 class="card-title">{{ category }}</h5>
            <a class="btn btn-outline-primary" [routerLink]="['/products']" [queryParams]="{ category: category }">View Products</a>
          </div>
        </div>
      </div>
    </div>
  </div>

  <ng-template #loading>
    <div class="container text-center py-5 px-5">
      <div class="spinner-border text-primary" role="status">
        <span class="visually-hidden">Loading...</span>
      </div>
    </div>
  </ng-template>
</ng-container>

<!-- Role-Specific Banners for Merchant, Delivery Agent, and Admin -->
<ng-template #roleSpecificContent>
  <div class="bg-primary text-white py-5 mb-4">
    <div class="container text-center px-5">
      <!-- Merchant -->
      <ng-container *ngIf="userRole === 'Merchant'">
        <h1 class="display-4 fw-bold">Welcome, Merchant!</h1>
        <p class="lead mb-4">Manage your products and grow your business with EShoppingZone.</p>
        <a class="btn btn-light btn-lg" [routerLink]="['/merchant-dashboard']">Go to Dashboard</a>
      </ng-container>

      <!-- Delivery Agent -->
      <ng-container *ngIf="userRole === 'Delivery Agent'">
        <h1 class="display-4 fw-bold">Welcome, Delivery Agent!</h1>
        <p class="lead mb-4">Track and manage your deliveries with ease.</p>
        <a class="btn btn-light btn-lg" [routerLink]="['/delivery-agent-home']">Go to Delivery Home</a>
      </ng-container>

      <!-- Admin -->
      <ng-container *ngIf="userRole === 'Admin'">
        <h1 class="display-4 fw-bold">Welcome, Admin!</h1>
        <p class="lead mb-4">Manage users, roles, and more from your Admin Dashboard.</p>
        <a class="btn btn-light btn-lg" [routerLink]="['/admin-home']">Go to Admin Dashboard</a>
      </ng-container>
    </div>
  </div>
</ng-template>

<!-- Modals (unchanged) -->
<div *ngIf="showLoginModal" class="modal d-block" id="loginModal" tabindex="-1" aria-labelledby="loginModalLabel">
  <div class="modal-dialog modal-dialog-centered">
    <div class="modal-content">
      <div class="modal-header bg-primary text-white">
        <h5 class="modal-title" id="loginModalLabel">Login Required 😊</h5>
        <button type="button" class="btn-close btn-close-white" (click)="closeLoginModal()" aria-label="Close"></button>
      </div>
      <div class="modal-body text-center">
        <p class="lead">Login or signup to continue 🛒✨</p>
        <div class="d-flex justify-content-center gap-3 mt-3">
          <a class="btn btn-primary" [routerLink]="['/login']" (click)="closeLoginModal()">Login</a>
          <a class="btn btn-outline-primary" [routerLink]="['/signup']" (click)="closeLoginModal()">Signup</a>
        </div>
      </div>
    </div>
  </div>
</div>
<div *ngIf="showLoginModal" class="modal-backdrop fade show" (click)="closeLoginModal()"></div>

<div *ngIf="showQuantityModal" class="modal d-block" id="quantityModal" tabindex="-1" aria-labelledby="quantityModalLabel">
  <div class="modal-dialog modal-dialog-centered">
    <div class="modal-content">
      <div class="modal-header bg-primary text-white">
        <h5 class="modal-title" id="quantityModalLabel">Select Quantity 🛒</h5>
        <button type="button" class="btn-close btn-close-white" (click)="closeQuantityModal()" aria-label="Close"></button>
      </div>
      <div class="modal-body text-center">
        <div class="mb-3">
          <label for="quantityInput" class="form-label">Quantity</label>
          <div class="input-group w-50 mx-auto">
            <button class="btn btn-outline-primary" (click)="quantity = quantity > 1 ? quantity - 1 : 1">-</button>
            <input type="number" class="form-control text-center" id="quantityInput" [(ngModel)]="quantity" min="1" required>
            <button class="btn btn-outline-primary" (click)="quantity = quantity + 1">+</button>
          </div>
        </div>
        <button class="btn btn-primary w-100" (click)="confirmAddToCart()">Add to Cart</button>
      </div>
    </div>
  </div>
</div>
<div *ngIf="showQuantityModal" class="modal-backdrop fade show" (click)="closeQuantityModal()"></div>
```

- **Changes**:
  - Wrapped the existing Customer/Guest content in an `*ngIf` condition to show only for `userRole === 'Customer' || userRole === 'Guest'`.
  - Added a `#roleSpecificContent` template for other roles (Merchant, Delivery Agent, Admin) with tailored welcome messages and links to their respective dashboards.
  - Kept the modals unchanged since they’re only relevant for Customers.

#### Update `home.component.css`
The provided `home.component.css` actually contains styles for the `HeaderComponent` (e.g., `.navbar-brand`, `.nav-link`). Let’s correct this by providing the proper styles for `HomeComponent` and adding any necessary styles for the role-specific banners.

**`home.component.css` (Updated)**:
```css
.image-container {
  width: 100%;
  height: 200px;
  overflow: hidden;
  display: flex;
  align-items: center;
  justify-content: center;
}

.card-img-top {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.card-body {
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.card-title {
  color: #333;
  font-size: 1.1rem;
  margin-bottom: 0.5rem;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.card-text {
  margin-bottom: 0.5rem;
}

.bg-primary {
  background-color: #0d6efd !important;
}

.py-5 {
  padding-top: 3rem !important;
  padding-bottom: 3rem !important;
}

.mb-4 {
  margin-bottom: 1.5rem !important;
}

.display-4 {
  font-size: 2.5rem;
}

.fw-bold {
  font-weight: 700 !important;
}

.lead {
  font-size: 1.25rem;
  font-weight: 300;
}

.btn-light {
  background-color: #fff;
  border-color: #fff;
  color: #0d6efd;
}

.btn-light:hover {
  background-color: #f8f9fa;
  border-color: #f8f9fa;
  color: #0d6efd;
}

.text-center {
  text-align: center;
}

.container {
  max-width: 1200px;
  margin: 0 auto;
}

.px-5 {
  padding-left: 3rem !important;
  padding-right: 3rem !important;
}

/* Ensure consistent styling for role-specific banners */
.bg-primary.text-white {
  min-height: 300px;
  display: flex;
  align-items: center;
  justify-content: center;
}
```

- **Changes**:
  - Replaced the incorrect styles (which belonged to `HeaderComponent`) with the appropriate styles for `HomeComponent`.
  - Added styles to ensure role-specific banners (Merchant, Delivery Agent, Admin) have consistent height and alignment with the Customer/Guest banner.
  - Kept existing styles for product cards, modals, and other elements.

---

### Step 3: Test and Debug
1. **Guest User**:
   - Log out and visit `/home`. You should see the banner (“Welcome to EShoppingZone”), top picks, and categories, with an “Add to Cart” button that prompts login.
2. **Customer**:
   - Log in as a Customer and visit `/home`. The layout should be the same as for Guests, but “Add to Cart” should show the quantity modal.
3. **Merchant**:
   - Log in as a Merchant and visit `/home`. You should see “Welcome, Merchant!” with a “Go to Dashboard” button linking to `/merchant-dashboard`. No product listings or categories should be shown.
4. **Delivery Agent**:
   - Log in as a Delivery Agent and visit `/home`. You should see “Welcome, Delivery Agent!” with a “Go to Delivery Home” button linking to `/delivery-agent-home`.
5. **Admin**:
   - Log in as an Admin and visit `/home`. You should see “Welcome, Admin!” with a “Go to Admin Dashboard” button linking to `/admin-home`.

---

### Step 4: Additional Considerations
- **Styling Enhancements**:
  - You might want to add role-specific background images or icons to make the banners more visually distinct. For example, add a background image for each role:
    ```html
    <div class="bg-primary text-white py-5 mb-4" [ngClass]="{
      'banner-customer': userRole === 'Customer' || userRole === 'Guest',
      'banner-merchant': userRole === 'Merchant',
      'banner-delivery-agent': userRole === 'Delivery Agent',
      'banner-admin': userRole === 'Admin'
    }">
    ```
    ```css
    .banner-customer {
      background: linear-gradient(rgba(0, 0, 0, 0.5), rgba(0, 0, 0, 0.5)), url('https://via.placeholder.com/1500x500?text=Customer+Background');
      background-size: cover;
      background-position: center;
    }
    .banner-merchant {
      background: linear-gradient(rgba(0, 0, 0, 0.5), rgba(0, 0, 0, 0.5)), url('https://via.placeholder.com/1500x500?text=Merchant+Background');
      background-size: cover;
      background-position: center;
    }
    .banner-delivery-agent {
      background: linear-gradient(rgba(0, 0, 0, 0.5), rgba(0, 0, 0, 0.5)), url('https://via.placeholder.com/1500x500?text=Delivery+Background');
      background-size: cover;
      background-position: center;
    }
    .banner-admin {
      background: linear-gradient(rgba(0, 0, 0, 0.5), rgba(0, 0, 0, 0.5)), url('https://via.placeholder.com/1500x500?text=Admin+Background');
      background-size: cover;
      background-position: center;
    }
    ```
  - Replace the placeholder URLs with actual images.

- **Dynamic Content for Other Roles**:
  - For Merchants, you could fetch and display their recent products.
  - For Delivery Agents, you could show their pending deliveries.
  - For Admins, you could show a summary of pending role requests.
  - If you’d like to add this, let me know, and we can extend the component to fetch role-specific data.

---

### Step 5: Summary
**It’s 8:20 PM IST on Monday, May 19, 2025.**

- **Updated Files**:
  - `home.component.ts`: Added role detection and conditional product loading for Customers and Guests.
  - `home.component.html`: Added role-specific banners for Merchant, Delivery Agent, and Admin while preserving the Customer/Guest layout.
  - `home.component.css`: Corrected styles to match `HomeComponent` and added styling for role-specific banners.
- **Fixes**:
  - The homepage now displays role-specific content for each user type while keeping the full product listing experience for Customers and Guests.

The homepage should now correctly display role-specific content. Please test the changes and let me know if you’d like to add more features (e.g., dynamic data for other roles) or proceed with the UI enhancements to make it industry-level! 🚀