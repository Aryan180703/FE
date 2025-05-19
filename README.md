**It’s 5:44 PM IST on Monday, May 19, 2025.**

I understand the issues with the `HeaderComponent` and the homepage:

1. **Navbar Issue**: When logged in as a Delivery Agent, the navbar shows the same options as a Customer (e.g., Products, Deals, Cart), which shouldn’t happen. The navbar should reflect Delivery Agent-specific options, and the search bar and cart should be hidden.
2. **Homepage Banner Missing**: The homepage banner is not displaying, likely because the homepage component or its template is incomplete or not properly set up.

Let’s address these issues step-by-step.

---

### Step 1: Fix the Navbar for Delivery Agent Role
The `HeaderComponent` currently uses `isMerchant` and `isCustomer` to conditionally show navbar items, but it doesn’t account for the Delivery Agent role properly. We need to:

- Add a check for the Delivery Agent role.
- Show Delivery Agent-specific navbar links (e.g., Delivery Agent Home, Delivery Agent Dashboard).
- Hide the search bar and cart for Delivery Agents.

#### Update `header.component.ts`
We’ll add an `isDeliveryAgent` flag and adjust the logic to show appropriate navbar items.

**`header.component.ts` (Updated)**:
```typescript
import { Component, OnInit, OnDestroy } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterLink, RouterLinkActive, Router, NavigationEnd } from '@angular/router';
import { AuthService } from '../../services/auth.service';
import { CartService } from '../../services/cart.service';
import { RoleService } from '../../services/role.service';
import { Subscription } from 'rxjs';

interface RoleRequest {
  id: number;
  userId: number;
  requestedRole: string;
  status: string;
  requestedAt: string;
  reviewedAt: string | null;
  user: {
    id: number;
    userName: string;
    email: string;
    phoneNumber: string;
  } | null;
}

interface ResponseDTO<T> {
  success: boolean;
  message: string;
  data: T;
}

@Component({
  selector: 'app-header',
  standalone: true,
  imports: [CommonModule, RouterLink, RouterLinkActive],
  templateUrl: './header.component.html',
  styleUrls: ['./header.component.css']
})
export class HeaderComponent implements OnInit, OnDestroy {
  searchQuery: string = '';
  isDropdownOpen: boolean = false;
  cartItemCount: number = 0;
  firstName: string | null = null;
  isMerchant: boolean = false;
  isCustomer: boolean = false;
  isDeliveryAgent: boolean = false;
  selectedRole: string | null = null;
  roleRequest: RoleRequest | null = null;
  roleRequestMessage: string | null = null;
  errorMessage: string | null = null;
  private authSubscription!: Subscription;

  constructor(
    public authService: AuthService,
    private router: Router,
    private cartService: CartService,
    private roleService: RoleService
  ) {
    this.updateCartCount();
  }

  ngOnInit() {
    this.authSubscription = this.authService.authState$.subscribe(isLoggedIn => {
      this.updateHeaderState(isLoggedIn);
    });

    this.router.events.subscribe(event => {
      if (event instanceof NavigationEnd) {
        this.closeDropdown();
      }
    });
  }

  ngOnDestroy() {
    if (this.authSubscription) {
      this.authSubscription.unsubscribe();
    }
  }

  updateHeaderState(isLoggedIn: boolean) {
    if (isLoggedIn) {
      this.authService.viewProfile().subscribe({
        next: (response) => {
          if (response.success && response.data) {
            this.firstName = response.data.fullName.split(' ')[0];
          }
        },
        error: (err) => {
          console.error('Error fetching profile:', err);
          this.firstName = null;
        }
      });

      const userRole = this.authService.getUserRole();
      this.isMerchant = userRole === 'Merchant';
      this.isCustomer = userRole === 'Customer';
      this.isDeliveryAgent = userRole === 'Delivery Agent';

      if (this.isCustomer) {
        this.cartService.cartUpdate$.subscribe(count => {
          this.cartItemCount = count;
        });
        this.updateCartCount();
      } else {
        this.cartItemCount = 0;
      }
    } else {
      this.firstName = null;
      this.isMerchant = false;
      this.isCustomer = false;
      this.isDeliveryAgent = false;
      this.cartItemCount = 0;
    }
  }

  search(event: Event) {
    const query = (event.target as HTMLInputElement).value;
    this.router.navigate(['/products'], { queryParams: { search: query } });
  }

  toggleDropdown(event?: Event) {
    if (event) {
      event.preventDefault();
    }
    this.isDropdownOpen = !this.isDropdownOpen;
  }

  closeDropdown() {
    this.isDropdownOpen = false;
  }

  logout() {
    this.authService.logout();
    this.isDropdownOpen = false;
  }

  updateCartCount() {
    if (this.authService.isLoggedIn() && this.authService.getUserRole() === 'Customer') {
      this.cartService.getCart().subscribe({
        next: (response) => {
          if (response.success && response.data) {
            this.cartItemCount = response.data.items.reduce((sum, item) => sum + item.quantity, 0);
          }
        },
        error: (err) => {
          console.error('Error fetching cart count:', err);
        }
      });
    }
  }

  navigateTo(route: string) {
    this.router.navigate([route]);
    this.closeDropdown();
  }

  openRoleRequestModal(role: string) {
    this.selectedRole = role;
    this.roleService.getMyRoleRequest().subscribe({
      next: (response) => {
        if (response.success && response.data) {
          this.roleRequest = response.data;
          const pendingModal = new (window as any).bootstrap.Modal(document.getElementById('pendingRequestModal'));
          pendingModal.show();
        } else {
          const confirmModal = new (window as any).bootstrap.Modal(document.getElementById('confirmRoleModal'));
          confirmModal.show();
        }
      },
      error: (err) => {
        this.errorMessage = err.error?.message || 'Failed to check role requests.';
        console.error('Error checking role requests:', err);
      }
    });
  }

  confirmRoleRequest() {
    if (!this.selectedRole) return;

    this.roleService.submitRoleRequest({ requestedRole: this.selectedRole }).subscribe({
      next: (response) => {
        const confirmModal = (window as any).bootstrap.Modal.getInstance(document.getElementById('confirmRoleModal'));
        confirmModal.hide();

        const messageModal = new (window as any).bootstrap.Modal(document.getElementById('messageModal'));
        this.errorMessage = response.success ? null : response.message;
        this.roleRequestMessage = response.success ? response.message : null;
        messageModal.show();
      },
      error: (err) => {
        const confirmModal = (window as any).bootstrap.Modal.getInstance(document.getElementById('confirmRoleModal'));
        confirmModal.hide();

        const messageModal = new (window as any).bootstrap.Modal(document.getElementById('messageModal'));
        this.errorMessage = err.error?.message || 'Failed to submit role request.';
        this.roleRequestMessage = null;
        messageModal.show();
      }
    });
  }

  openCheckRequestModal() {
    this.roleService.getMyRoleRequest().subscribe({
      next: (response) => {
        this.roleRequest = response.success ? response.data : null;
        this.roleRequestMessage = response.message;
        const statusModal = new (window as any).bootstrap.Modal(document.getElementById('roleRequestStatusModal'));
        statusModal.show();
      },
      error: (err) => {
        this.errorMessage = err.error?.message || 'Failed to check role requests.';
        console.error('Error checking role requests:', err);
      }
    });
  }
}
```

- **Changes**:
  - Added `isDeliveryAgent` to track the Delivery Agent role.
  - Updated `updateHeaderState` to set `isDeliveryAgent` based on `authService.getUserRole()`.

#### Update `header.component.html`
We’ll modify the navbar to show Delivery Agent-specific links and hide the search bar and cart for Delivery Agents.

**`header.component.html` (Updated)**:
```html
<nav class="navbar navbar-expand-lg navbar-dark bg-primary">
  <div class="container-fluid px-5">
    <!-- Brand -->
    <a class="navbar-brand fw-bold" [routerLink]="['/']">EShoppingZone</a>
    <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav"
      aria-controls="navbarNav" aria-expanded="false" aria-label="Toggle navigation">
      <span class="navbar-toggler-icon"></span>
    </button>

    <!-- Navbar Content -->
    <div class="collapse navbar-collapse" id="navbarNav">
      <!-- Navigation Links -->
      <ul class="navbar-nav me-auto mb-2 mb-lg-0 ms-3">
        <li class="nav-item">
          <a class="nav-link" [routerLink]="['/']" routerLinkActive="active">Home</a>
        </li>
        <!-- Customer Links -->
        <li class="nav-item" *ngIf="isCustomer">
          <a class="nav-link" [routerLink]="['/products']" routerLinkActive="active">Products</a>
        </li>
        <li class="nav-item" *ngIf="isCustomer">
          <a class="nav-link" [routerLink]="['/deals']" routerLinkActive="active">Deals</a>
        </li>
        <!-- Merchant Links -->
        <li class="nav-item" *ngIf="isMerchant">
          <a class="nav-link" [routerLink]="['/merchant-dashboard']" routerLinkActive="active">Dashboard</a>
        </li>
        <li class="nav-item" *ngIf="isMerchant">
          <a class="nav-link" [routerLink]="['/manage-products']" routerLinkActive="active">Manage Products</a>
        </li>
        <!-- Delivery Agent Links -->
        <li class="nav-item" *ngIf="isDeliveryAgent">
          <a class="nav-link" [routerLink]="['/delivery-agent-home']" routerLinkActive="active">Delivery Home</a>
        </li>
        <li class="nav-item" *ngIf="isDeliveryAgent">
          <a class="nav-link" [routerLink]="['/delivery-agent-dashboard']" routerLinkActive="active">Dashboard</a>
        </li>
      </ul>

      <!-- Search Bar (Only for Customers) -->
      <div class="d-flex flex-grow-1 justify-content-center mx-3" *ngIf="isCustomer">
        <div class="input-group w-50">
          <span class="input-group-text bg-light">
            <i class="bi bi-search"></i>
          </span>
          <input type="text" class="form-control" placeholder="Search for products..." (input)="search($event)">
        </div>
      </div>

      <!-- User Actions -->
      <div class="navbar-nav ms-auto">
        <ng-container *ngIf="!authService.isLoggedIn(); else loggedIn">
          <li class="nav-item">
            <a class="nav-link btn btn-outline-light me-2 login-signup-btn" [routerLink]="['/login']">Login</a>
          </li>
          <li class="nav-item">
            <a class="nav-link btn btn-outline-light login-signup-btn" [routerLink]="['/signup']">Signup</a>
          </li>
        </ng-container>

        <ng-template #loggedIn>
          <!-- Cart Link for Customers -->
          <li class="nav-item" *ngIf="isCustomer">
            <a class="nav-link text-white" [routerLink]="['/cart']">
              Cart
              <span *ngIf="cartItemCount > 0" class="badge bg-danger ms-1">{{ cartItemCount }}</span>
            </a>
          </li>

          <!-- Profile Dropdown -->
          <li class="nav-item dropdown" (click)="toggleDropdown(); $event.preventDefault()">
            <a class="nav-link dropdown-toggle text-white" id="profileDropdown" role="button" aria-expanded="false">
              {{ firstName ? firstName : 'Profile' }}
            </a>
            <ul class="dropdown-menu dropdown-menu-end" [ngClass]="{'show': isDropdownOpen}" aria-labelledby="profileDropdown">
              <!-- Common Options for All Users -->
              <li>
                <a class="dropdown-item" (click)="navigateTo('/profile')">Profile</a>
              </li>
              <li>
                <a class="dropdown-item" (click)="navigateTo('/update-profile')">Update Profile</a>
              </li>

              <!-- Customer-Specific Options -->
              <li *ngIf="isCustomer">
                <a class="dropdown-item" (click)="navigateTo('/manage-addresses')">Manage Addresses</a>
              </li>
              <li *ngIf="isCustomer">
                <a class="dropdown-item" (click)="navigateTo('/cart')">Place Order</a>
              </li>
              <li *ngIf="isCustomer">
                <a class="dropdown-item" (click)="navigateTo('/order-history')">Order History</a>
              </li>
              <li *ngIf="isCustomer">
                <a class="dropdown-item" (click)="openRoleRequestModal('Merchant')">Become a Merchant</a>
              </li>
              <li *ngIf="isCustomer">
                <a class="dropdown-item" (click)="openRoleRequestModal('DeliveryAgent')">Become a Delivery Agent</a>
              </li>
              <li *ngIf="isCustomer">
                <a class="dropdown-item" (click)="openCheckRequestModal()">Check My Role Requests</a>
              </li>

              <!-- Logout -->
              <li><hr class="dropdown-divider"></li>
              <li>
                <a class="dropdown-item" (click)="logout()">Logout</a>
              </li>
            </ul>
          </li>
        </ng-template>
      </div>
    </div>
  </div>
</nav>

<!-- Confirmation Modal -->
<div class="modal fade" id="confirmRoleModal" tabindex="-1" aria-labelledby="confirmRoleModalLabel" aria-hidden="true">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title" id="confirmRoleModalLabel">Confirm Role Request</h5>
        <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
      </div>
      <div class="modal-body">
        Are you sure you want to become a {{ selectedRole }}?
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">No</button>
        <button type="button" class="btn btn-primary" (click)="confirmRoleRequest()">Yes, I Confirm</button>
      </div>
    </div>
  </div>
</div>

<!-- Pending Request Modal -->
<div class="modal fade" id="pendingRequestModal" tabindex="-1" aria-labelledby="pendingRequestModalLabel" aria-hidden="true">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title" id="pendingRequestModalLabel">Pending Role Request</h5>
        <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
      </div>
      <div class="modal-body">
        <p>You already have a pending request to become a {{ roleRequest?.requestedRole }}.</p>
        <p>Requested on: {{ roleRequest?.requestedAt | date:'medium' }}</p>
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
      </div>
    </div>
  </div>
</div>

<!-- Role Request Status Modal -->
<div class="modal fade" id="roleRequestStatusModal" tabindex="-1" aria-labelledby="roleRequestStatusModalLabel" aria-hidden="true">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title" id="roleRequestStatusModalLabel">Role Request Status</h5>
        <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
      </div>
      <div class="modal-body">
        <div *ngIf="roleRequest; else noRequest">
          <p><strong>Request ID:</strong> {{ roleRequest.id }}</p>
          <p><strong>Requested Role:</strong> {{ roleRequest.requestedRole }}</p>
          <p><strong>Status:</strong> {{ roleRequest.status }}</p>
          <p><strong>Requested At:</strong> {{ roleRequest.requestedAt | date:'medium' }}</p>
          <p *ngIf="roleRequest.reviewedAt"><strong>Reviewed At:</strong> {{ roleRequest.reviewedAt | date:'medium' }}</p>
        </div>
        <ng-template #noRequest>
          <p>{{ roleRequestMessage || 'No role request found.' }}</p>
        </ng-template>
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
      </div>
    </div>
  </div>
</div>

<!-- Message Modal -->
<div class="modal fade" id="messageModal" tabindex="-1" aria-labelledby="messageModalLabel" aria-hidden="true">
  <div class="modal-dialog">
    <div class="modal-content">
      <div class="modal-header">
        <h5 class="modal-title" id="messageModalLabel">Role Request</h5>
        <button type="button" class="btn-close" data-bs-dismiss="modal" aria-label="Close"></button>
      </div>
      <div class="modal-body">
        <div *ngIf="errorMessage" class="alert alert-danger" role="alert">
          {{ errorMessage }}
        </div>
        <div *ngIf="roleRequestMessage" class="alert alert-success" role="alert">
          {{ roleRequestMessage }}
        </div>
      </div>
      <div class="modal-footer">
        <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
      </div>
    </div>
  </div>
</div>
```

- **Changes**:
  - Added Delivery Agent-specific navbar links (`Delivery Home`, `Dashboard`) using `*ngIf="isDeliveryAgent"`.
  - Updated the search bar condition to `*ngIf="isCustomer"` instead of `!isMerchant`, ensuring it’s hidden for Delivery Agents.
  - Updated the cart link condition to `*ngIf="isCustomer"`, hiding it for Delivery Agents.

---

### Step 2: Add the Homepage Banner
The homepage is likely mapped to the `/` route (as seen in the navbar’s `Home` link). Since you mentioned the banner is missing, it’s possible the homepage component (`HomeComponent` or similar) doesn’t exist or lacks the banner. Let’s create a `HomeComponent` with a banner, assuming it’s not already present.

#### Create `HomeComponent`
**`home.component.ts` (New)**:
```typescript
import { Component } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-home',
  standalone: true,
  imports: [CommonModule, RouterLink],
  templateUrl: './home.component.html',
  styleUrls: ['./home.component.css']
})
export class HomeComponent {}
```

**`home.component.html` (New)**:
```html
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

**`home.component.css` (New)**:
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

- **Details**:
  - Created a simple banner with a gradient overlay, a heading, a description, a "Shop Now" button, and an image.
  - Used Bootstrap classes for responsiveness (`container`, `row`, `col-md-6`).
  - Used placeholder images; replace the URLs with actual images from your assets.

#### Update Routing
Ensure the `/` route points to the `HomeComponent`.

**`app.routes.ts` (Updated)**:
```typescript
import { Routes } from '@angular/router';
import { HomeComponent } from './components/home/home.component';
import { DeliveryAgentHomeComponent } from './components/delivery-agent-home/delivery-agent-home.component';
import { DeliveryAgentDashboardComponent } from './compon