**It’s 7:49 PM IST on Monday, May 19, 2025.**

I understand the issue: the homepage (`/home`) is showing the same content for all users (Customer, Merchant, Delivery Agent, Admin, and guest users), but you want it to display role-specific content for each user type. Let’s fix this by modifying the `HomeComponent` to dynamically render different content based on the user’s role.

---

### Step 1: Analyze the Current Homepage
The current `HomeComponent` displays a static banner with a “Shop Now” button that links to `/products`. This is suitable for Customers and guest users, but for other roles (Merchant, Delivery Agent, Admin), the homepage should reflect their specific roles and actions. For example:
- **Customer/Guest**: Show the shopping banner with a “Shop Now” button.
- **Merchant**: Show a message like “Welcome to your Merchant Dashboard” with a link to `/merchant-dashboard`.
- **Delivery Agent**: Show a message like “Welcome to your Delivery Dashboard” with a link to `/delivery-agent-home`.
- **Admin**: Show a message like “Welcome to Admin Dashboard” with a link to `/admin-home`.

#### Current `HomeComponent` Files
**`home.component.ts`**:
```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterLink, ActivatedRoute } from '@angular/router';

@Component({
  selector: 'app-home',
  standalone: true,
  imports: [CommonModule, RouterLink],
  templateUrl: './home.component.html',
  styleUrls: ['./home.component.css']
})
export class HomeComponent implements OnInit {
  errorMessage: string | null = null;

  constructor(private route: ActivatedRoute) {}

  ngOnInit() {
    this.route.queryParams.subscribe(params => {
      if (params['error'] === 'access-denied') {
        this.errorMessage = 'Access Denied: You do not have permission to access that page.';
      }
    });
  }
}
```

**`home.component.html`**:
```html
<!-- Error Message -->
<div *ngIf="errorMessage" class="alert alert-danger mt-4" role="alert">
  {{ errorMessage }}
</div>

<!-- Banner Section -->
<div class="banner-section">
  <div class="container">
    <div class="row align-items-center">
      <div class="col-md-6 text-center text-md-start">
        <h1 class="display-4 fw-bold text-white">Welcome to EShoppingZone</h1>
        <p class="lead text-white">Discover amazing deals and shop your favorite products today!</p>
        <a class="btn btn-primary btn-lg" [routerLink]="['/products']">Shop Now</a>
      </div>
      <div class="col-md-6 text-center">
        <img src="https://via.placeholder.com/500x300?text=Shopping+Banner" alt="Shopping Banner" class="img-fluid banner-image">
      </div>
    </div>
  </div>
</div>
```

**`home.component.css`**:
```css
.banner-section {
  background: linear-gradient(rgba(0, 0, 0, 0.5), rgba(0, 0, 0, 0.5)), url('https://via.placeholder.com/1500x500?text=Background+Image');
  background-size: cover;
  background-position: center;
  padding: 100px 0;
  color: white;
}

.banner-image {
  max-width: 100%;
  border-radius: 10px;
}
```

---

### Step 2: Modify `HomeComponent` to Show Role-Specific Content
We’ll update the `HomeComponent` to:
1. Use `AuthService` to check the user’s role.
2. Conditionally render different banner content based on the role using `*ngIf`.

#### Update `home.component.ts`
We’ll add `AuthService` to check the user’s role and set a `userRole` property to determine which content to display.

**`home.component.ts` (Updated)**:
```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterLink, ActivatedRoute } from '@angular/router';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'app-home',
  standalone: true,
  imports: [CommonModule, RouterLink],
  templateUrl: './home.component.html',
  styleUrls: ['./home.component.css']
})
export class HomeComponent implements OnInit {
  errorMessage: string | null = null;
  userRole: string | null = null;

  constructor(private route: ActivatedRoute, private authService: AuthService) {}

  ngOnInit() {
    // Check for error query params
    this.route.queryParams.subscribe(params => {
      if (params['error'] === 'access-denied') {
        this.errorMessage = 'Access Denied: You do not have permission to access that page.';
      }
    });

    // Determine user role
    if (this.authService.isLoggedIn()) {
      this.userRole = this.authService.getUserRole();
    } else {
      this.userRole = 'Guest'; // Treat unauthenticated users as Guests
    }
  }
}
```

- **Changes**:
  - Added `AuthService` to the constructor.
  - Added `userRole` property to store the user’s role.
  - Set `userRole` to `'Guest'` for unauthenticated users, or fetch it from `authService.getUserRole()` for logged-in users.

#### Update `home.component.html`
We’ll use `*ngIf` to render different banner sections based on the `userRole`.

**`home.component.html` (Updated)**:
```html
<!-- Error Message -->
<div *ngIf="errorMessage" class="alert alert-danger mt-4" role="alert">
  {{ errorMessage }}
</div>

<!-- Role-Specific Banner Section -->
<div class="banner-section">
  <div class="container">
    <div class="row align-items-center">
      <!-- Customer or Guest: Shopping Banner -->
      <ng-container *ngIf="userRole === 'Customer' || userRole === 'Guest'">
        <div class="col-md-6 text-center text-md-start">
          <h1 class="display-4 fw-bold text-white">Welcome to EShoppingZone</h1>
          <p class="lead text-white">Discover amazing deals and shop your favorite products today!</p>
          <a class="btn btn-primary btn-lg" [routerLink]="['/products']">Shop Now</a>
        </div>
        <div class="col-md-6 text-center">
          <img src="https://via.placeholder.com/500x300?text=Shopping+Banner" alt="Shopping Banner" class="img-fluid banner-image">
        </div>
      </ng-container>

      <!-- Merchant: Merchant Dashboard Banner -->
      <ng-container *ngIf="userRole === 'Merchant'">
        <div class="col-md-6 text-center text-md-start">
          <h1 class="display-4 fw-bold text-white">Welcome, Merchant!</h1>
          <p class="lead text-white">Manage your products and grow your business with EShoppingZone.</p>
          <a class="btn btn-primary btn-lg" [routerLink]="['/merchant-dashboard']">Go to Dashboard</a>
        </div>
        <div class="col-md-6 text-center">
          <img src="https://via.placeholder.com/500x300?text=Merchant+Dashboard" alt="Merchant Dashboard" class="img-fluid banner-image">
        </div>
      </ng-container>

      <!-- Delivery Agent: Delivery Dashboard Banner -->
      <ng-container *ngIf="userRole === 'Delivery Agent'">
        <div class="col-md-6 text-center text-md-start">
          <h1 class="display-4 fw-bold text-white">Welcome, Delivery Agent!</h1>
          <p class="lead text-white">Track and manage your deliveries with ease.</p>
          <a class="btn btn-primary btn-lg" [routerLink]="['/delivery-agent-home']">Go to Delivery Home</a>
        </div>
        <div class="col-md-6 text-center">
          <img src="https://via.placeholder.com/500x300?text=Delivery+Dashboard" alt="Delivery Dashboard" class="img-fluid banner-image">
        </div>
      </ng-container>

      <!-- Admin: Admin Dashboard Banner -->
      <ng-container *ngIf="userRole === 'Admin'">
        <div class="col-md-6 text-center text-md-start">
          <h1 class="display-4 fw-bold text-white">Welcome, Admin!</h1>
          <p class="lead text-white">Manage users, roles, and more from your Admin Dashboard.</p>
          <a class="btn btn-primary btn-lg" [routerLink]="['/admin-home']">Go to Admin Dashboard</a>
        </div>
        <div class="col-md-6 text-center">
          <img src="https://via.placeholder.com/500x300?text=Admin+Dashboard" alt="Admin Dashboard" class="img-fluid banner-image">
        </div>
      </ng-container>
    </div>
  </div>
</div>
```

- **Changes**:
  - Used `*ngIf` to render different banner content based on `userRole`.
  - Each role (Customer/Guest, Merchant, Delivery Agent, Admin) gets a tailored message, call-to-action button, and image.
  - Placeholder images are used; replace them with actual images relevant to each role.

#### Update `home.component.css`
We’ll keep the CSS mostly the same but ensure the styles apply consistently across all role-specific banners.

**`home.component.css` (Updated)**:
```css
.banner-section {
  background: linear-gradient(rgba(0, 0, 0, 0.5), rgba(0, 0, 0, 0.5)), url('https://via.placeholder.com/1500x500?text=Background+Image');
  background-size: cover;
  background-position: center;
  padding: 100px 0;
  color: white;
  min-height: 400px; /* Ensure the banner has enough height for all content */
}

.banner-image {
  max-width: 100%;
  border-radius: 10px;
}
```

- **Changes**:
  - Added `min-height: 400px` to ensure the banner section has enough space for all role-specific content.

---

### Step 3: Test and Debug
1. **Guest User**:
   - Log out and visit `/home`. You should see the “Welcome to EShoppingZone” banner with a “Shop Now” button linking to `/products`.
2. **Customer**:
   - Log in as a Customer and visit `/home`. The banner should be the same as for Guest users (since Customers are shoppers).
3. **Merchant**:
   - Log in as a Merchant and visit `/home`. You should see “Welcome, Merchant!” with a “Go to Dashboard” button linking to `/merchant-dashboard`.
4. **Delivery Agent**:
   - Log in as a Delivery Agent and visit `/home`. You should see “Welcome, Delivery Agent!” with a “Go to Delivery Home” button linking to `/delivery-agent-home`.
5. **Admin**:
   - Log in as an Admin and visit `/home`. You should see “Welcome, Admin!” with a “Go to Admin Dashboard” button linking to `/admin-home`.

---

### Step 4: Additional Considerations
- **Images**: The placeholder images (`https://via.placeholder.com/500x300?text=...`) should be replaced with actual images relevant to each role. For example:
  - Customer/Guest: A shopping-related image.
  - Merchant: An image of a dashboard or product management.
  - Delivery Agent: An image of a delivery truck or package.
  - Admin: An image of an admin panel or user management.
- **Styling Consistency**: If you want different background images for each role’s banner, you can add role-specific classes to the `banner-section` div and update the CSS accordingly. For example:
  ```html
  <div class="banner-section" [ngClass]="{
    'banner-customer': userRole === 'Customer' || userRole === 'Guest',
    'banner-merchant': userRole === 'Merchant',
    'banner-delivery-agent': userRole === 'Delivery Agent',
    'banner-admin': userRole === 'Admin'
  }">
  ```
  ```css
  .banner-customer {
    background: linear-gradient(rgba(0, 0, 0, 0.5), rgba(0, 0, 0, 0.5)), url('https://via.placeholder.com/1500x500?text=Customer+Background');
  }
  .banner-merchant {
    background: linear-gradient(rgba(0, 0, 0, 0.5), rgba(0, 0, 0, 0.5)), url('https://via.placeholder.com/1500x500?text=Merchant+Background');
  }
  .banner-delivery-agent {
    background: linear-gradient(rgba(0, 0, 0, 0.5), rgba(0, 0, 0, 0.5)), url('https://via.placeholder.com/1500x500?text=Delivery+Background');
  }
  .banner-admin {
    background: linear-gradient(rgba(0, 0, 0, 0.5), rgba(0, 0, 0, 0.5)), url('https://via.placeholder.com/1500x500?text=Admin+Background');
  }
  ```

- **Dynamic Content**: If you want to add more dynamic content (e.g., recent orders for Customers, pending deliveries for Delivery Agents), we can extend the `HomeComponent` to fetch role-specific data from APIs.

---

### Step 5: Summary
**It’s 8:00 PM IST on Monday, May 19, 2025.**

- **Updated Files**:
  - `home.component.ts`: Added logic to determine the user’s role using `AuthService`.
  - `home.component.html`: Added role-specific banners using `*ngIf`.
  - `home.component.css`: Added `min-height` to ensure proper banner height.
- **Fixes**:
  - The homepage now displays different content for each user role (Customer/Guest, Merchant, Delivery Agent, Admin).
- **Next Steps**:
  - Replace placeholder images with actual role-specific images.
  - Optionally, add role-specific background images for the banner section.
  - Consider adding more dynamic content (e.g., recent orders, pending deliveries) if needed.

The homepage should now be role-specific for each user type. Please test the changes and let me know if you’d like to make further adjustments or proceed with the UI enhancements to make it industry-level! 🚀