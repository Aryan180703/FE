**It’s 8:05 PM IST on Monday, May 19, 2025.**

Let’s address the issues one by one: the header (navbar) for guest users, the non-functional role request buttons for Customers, the missing quick options for Merchants, and the Delivery Agent header options.

---

### Issue 1: Guest User Not Having Proper Header (Navbar)
The `HeaderComponent` currently shows the same navbar for all users, but guest users (unauthenticated users) should see a simplified navbar without role-specific links like Products, Deals, or Cart, and only display "Login" and "Signup" buttons.

#### Fix: Update `HeaderComponent` to Show a Guest-Specific Navbar
We’ll modify the `HeaderComponent` to conditionally hide role-specific links for guest users and only show the brand logo (linking to `/`), "Login", and "Signup" buttons.

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
        <!-- Show Home link for all users -->
        <li class="nav-item">
          <a class="nav-link" [routerLink]="['/']" routerLinkActive="active">Home</a>
        </li>
        <!-- Show Products and Deals only for authenticated Customers -->
        <li class="nav-item" *ngIf="isCustomer && authService.isLoggedIn()">
          <a class="nav-link" [routerLink]="['/products']" routerLinkActive="active">Products</a>
        </li>
        <li class="nav-item" *ngIf="isCustomer && authService.isLoggedIn()">
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
      <div class="d-flex flex-grow-1 justify-content-center mx-3" *ngIf="isCustomer && authService.isLoggedIn()">
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
  - Added `authService.isLoggedIn()` to the conditions for Products, Deals, and the search bar to ensure they only appear for logged-in Customers.
  - For guest users (`!authService.isLoggedIn()`), the navbar will only show the "Home" link, "Login", and "Signup" buttons.

---

### Issue 2: Nothing Happens When Clicking "Become a Merchant", "Become a Delivery Agent", "Check My Role Request" for Customers
The buttons "Become a Merchant", "Become a Delivery Agent", and "Check My Role Requests" in the profile dropdown are not functioning when clicked by a Customer. This is likely due to an issue with the modal initialization using Bootstrap’s JavaScript, which requires the Bootstrap bundle to be included and properly initialized.

#### Fix: Ensure Bootstrap JavaScript is Included and Modals Work
1. **Check Bootstrap Inclusion**:
   Ensure that the Bootstrap JavaScript bundle is included in your `index.html`. You need the bundle that includes Popper.js for dropdowns and modals to work.

   **`index.html` (Updated)**:
   ```html
   <!DOCTYPE html>
   <html lang="en">
   <head>
     <meta charset="UTF-8">
     <meta name="viewport" content="width=device-width, initial-scale=1.0">
     <title>EShoppingZone</title>
     <!-- Bootstrap CSS -->
     <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet">
     <!-- Bootstrap Icons -->
     <link href="https://cdn.jsdelivr.net/npm/bootstrap-icons/font/bootstrap-icons.css" rel="stylesheet">
   </head>
   <body>
     <app-root></app-root>
     <!-- Bootstrap JS Bundle (includes Popper.js) -->
     <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"></script>
   </body>
   </html>
   ```

   - Ensure the Bootstrap JS bundle is added at the bottom of the `<body>` tag.

2. **Debug Modal Initialization**:
   The `openRoleRequestModal` and `openCheckRequestModal` methods in `HeaderComponent` use Bootstrap’s JavaScript to show modals (`bootstrap.Modal`). If these methods are not working, it could be due to timing issues with the DOM or errors in the modal logic. Let’s ensure the methods handle errors and log issues for debugging.

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
             const pendingModalElement = document.getElementById('pendingRequestModal');
             if (pendingModalElement) {
               const pendingModal = new (window as any).bootstrap.Modal(pendingModalElement);
               pendingModal.show();
             } else {
               console.error('Pending Request Modal element not found');
             }
           } else {
             const confirmModalElement = document.getElementById('confirmRoleModal');
             if (confirmModalElement) {
               const confirmModal = new (window as any).bootstrap.Modal(confirmModalElement);
               confirmModal.show();
             } else {
               console.error('Confirm Role Modal element not found');
             }
           }
         },
         error: (err) => {
           this.errorMessage = err.error?.message || 'Failed to check role requests.';
           console.error('Error checking role requests:', err);
           const messageModalElement = document.getElementById('messageModal');
           if (messageModalElement) {
             const messageModal = new (window as any).bootstrap.Modal(messageModalElement);
             messageModal.show();
           } else {
             console.error('Message Modal element not found');
           }
         }
       });
     }

     confirmRoleRequest() {
       if (!this.selectedRole) return;

       this.roleService.submitRoleRequest({ requestedRole: this.selectedRole }).subscribe({
         next: (response) => {
           const confirmModalElement = document.getElementById('confirmRoleModal');
           if (confirmModalElement) {
             const confirmModal = (window as any).bootstrap.Modal.getInstance(confirmModalElement);
             confirmModal.hide();
           }

           const messageModalElement = document.getElementById('messageModal');
           if (messageModalElement) {
             const messageModal = new (window as any).bootstrap.Modal(messageModalElement);
             this.errorMessage = response.success ? null : response.message;
             this.roleRequestMessage = response.success ? response.message : null;
             messageModal.show();
           } else {
             console.error('Message Modal element not found');
           }
         },
         error: (err) => {
           const confirmModalElement = document.getId('confirmRoleModal');
           if (confirmModalElement) {
             const confirmModal = (window as any).bootstrap.Modal.getInstance(confirmModalElement);
             confirmModal.hide();
           }

           const messageModalElement = document.getElementById('messageModal');
           if (messageModalElement) {
             const messageModal = new (window as any).bootstrap.Modal(messageModalElement);
             this.errorMessage = err.error?.message || 'Failed to submit role request.';
             this.roleRequestMessage = null;
             messageModal.show();
           } else {
             console.error('Message Modal element not found');
           }
         }
       });
     }

     openCheckRequestModal() {
       this.roleService.getMyRoleRequest().subscribe({
         next: (response) => {
           this.roleRequest = response.success ? response.data : null;
           this.roleRequestMessage = response.message;
           const statusModalElement = document.getElementById('roleRequestStatusModal');
           if (statusModalElement) {
             const statusModal = new (window as any).bootstrap.Modal(statusModalElement);
             statusModal.show();
           } else {
             console.error('Role Request Status Modal element not found');
           }
         },
         error: (err) => {
           this.errorMessage = err.error?.message || 'Failed to check role requests.';
           console.error('Error checking role requests:', err);
           const messageModalElement = document.getElementById('messageModal');
           if (messageModalElement) {
             const messageModal = new (window as any).bootstrap.Modal(messageModalElement);
             messageModal.show();
           } else {
             console.error('Message Modal element not found');
           }
         }
       });
     }
   }
   ```

   - **Changes**:
     - Added checks for modal elements using `document.getElementById` and logged errors if elements are not found.
     - Ensured modals are properly initialized and shown using `bootstrap.Modal`.

3. **Test the Buttons**:
   - Log in as a Customer and click "Become a Merchant", "Become a Delivery Agent", and "Check My Role Requests".
   - The modals should now appear. If there’s a pending request, the `pendingRequestModal` should show; otherwise, the `confirmRoleModal` should appear for role requests, and `roleRequestStatusModal` for checking requests.
   - If the modals still don’t appear, check the browser console for errors (e.g., Bootstrap not loaded, modal elements not found).

---

### Issue 3: Merchant Quick Options Are Missing
The `HeaderComponent` currently shows "Dashboard" and "Manage Products" for Merchants in the navbar, but there are no quick options in the profile dropdown for Merchants, unlike Customers who have options like "Manage Addresses" and "Order History". Let’s add Merchant-specific quick options in the dropdown, such as "Add Product", "View Merchant Products", and "Check Role Requests".

#### Fix: Add Merchant Quick Options to the Profile Dropdown
**`header.component.html` (Updated Dropdown Section)**:
Update the profile dropdown section to include Merchant-specific options:

```html
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

    <!-- Merchant-Specific Options -->
    <li *ngIf="isMerchant">
      <a class="dropdown-item" (click)="navigateTo('/add-product')">Add Product</a>
    </li>
    <li *ngIf="isMerchant">
      <a class="dropdown-item" (click)="navigateTo('/manage-products')">View Merchant Products</a>
    </li>
    <li *ngIf="isMerchant">
      <a class="dropdown-item" (click)="openCheckRequestModal()">Check My Role Requests</a>
    </li>

    <!-- Logout -->
    <li><hr class="dropdown-divider"></li>
    <li>
      <a class="dropdown-item" (click)="logout()">Logout</a>
    </li>
  </ul>
</li>
```

- **Changes**:
  - Added Merchant-specific options in the dropdown: "Add Product" (`/add-product`), "View Merchant Products" (`/manage-products`), and "Check My Role Requests" (reusing `openCheckRequestModal`).
  - These routes already exist in `app.routes.ts`, so they should work seamlessly.

#### Test Merchant Quick Options
- Log in as a Merchant and check the profile dropdown.
- You should see "Add Product", "View Merchant Products", and "Check My Role Requests" alongside the common options (Profile, Update Profile, Logout).

---

### Issue 4: Delivery Agent Header Contains "Home" and "Delivery Home" - What Should Be Done?
The Delivery Agent navbar currently shows both "Home" and "Delivery Home" links. This is redundant because "Delivery Home" (`/delivery-agent-home`) is the primary landing page for Delivery Agents after login, and "Home" (`/`) takes them to the general homepage, which is more relevant for Customers and Guests. Let’s remove the "Home" link for Delivery Agents to streamline their navigation.

#### Fix: Remove "Home" Link for Delivery Agents
**`header.component.html` (Updated Navigation Links Section)**:
```html
<!-- Navigation Links -->
<ul class="navbar-nav me-auto mb-2 mb-lg-0 ms-3">
  <!-- Show Home link for non-Delivery Agents -->
  <li class="nav-item" *ngIf="!isDeliveryAgent">
    <a class="nav-link" [routerLink]="['/']" routerLinkActive="active">Home</a>
  </li>
  <!-- Show Products and Deals only for authenticated Customers -->
  <li class="nav-item" *ngIf="isCustomer && authService.isLoggedIn()">
    <a class="nav-link" [routerLink]="['/products']" routerLinkActive="active">Products</a>
  </li>
  <li class="nav-item" *ngIf="isCustomer && authService.isLoggedIn()">
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
```

- **Changes**:
  - Added `*ngIf="!isDeliveryAgent"` to the "Home" link so it only appears for non-Delivery Agent users.
  - Delivery Agents will now only see "Delivery Home" and "Dashboard" in their navbar, which is more relevant to their role.

#### Test Delivery Agent Navbar
- Log in as a Delivery Agent and check the navbar.
- You should only see "Delivery Home" and "Dashboard", with no "Home" link.

---

### Summary of Changes
**It’s 8:15 PM IST on Monday, May 19, 2025.**

- **Guest User Navbar**:
  - Updated `HeaderComponent` to show only "Home", "Login", and "Signup" for guest users.
- **Role Request Buttons for Customers**:
  - Ensured Bootstrap JavaScript is included in `index.html` and added debugging for modal initialization in `HeaderComponent`.
- **Merchant Quick Options**:
  - Added "Add Product", "View Merchant Products", and "Check My Role Requests" to the Merchant’s profile dropdown.
- **Delivery Agent Navbar**:
  - Removed the "Home" link for Delivery Agents, keeping only "Delivery Home" and "Dashboard".

#### Test All Scenarios
1. **Guest User**: Log out and check the navbar – it should only show "Home", "Login", and "Signup".
2. **Customer Role Requests**: Log in as a Customer, click "Become a Merchant", "Become a Delivery Agent", and "Check My Role Requests" – modals should appear.
3. **Merchant Quick Options**: Log in as a Merchant, check the profile dropdown for the new options.
4. **Delivery Agent Navbar**: Log in as a Delivery Agent, confirm the navbar only shows "Delivery Home" and "Dashboard".

Let me know if everything works as expected or if you’d like to proceed with further UI enhancements! 🚀