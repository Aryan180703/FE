**It’s 5:10 PM IST on Monday, May 19, 2025.**

Thank you for sharing the header component files. I understand that the profile dropdown is in the `HeaderComponent`, and the new options ("Become a Merchant", "Become a Delivery Agent", and "Check My Role Requests") should only be available for users with the "Customer" role (since you mentioned "for users only" and the context of the project indicates users start as customers by default). We’ll implement the dialog boxes using Bootstrap modals, as you’ve requested to continue using Bootstrap for consistency with the rest of the project. Let’s implement the required functionality step-by-step.

---

### Step 1: Plan the Implementation
#### HeaderComponent Updates
- **Add Options to Dropdown**:
  - Add "Become a Merchant", "Become a Delivery Agent", and "Check My Role Requests" to the profile dropdown in `HeaderComponent`.
  - These options should only appear for users with the "Customer" role (checked via `authService.getUserRole() === 'Customer'`).
- **Dialog Boxes (Bootstrap Modals)**:
  - **Confirmation Modal**: On clicking "Become a Merchant" or "Become a Delivery Agent", show a modal asking, "Are you sure you want to become a [Merchant/Delivery Agent]?" with "Yes, I Confirm" and "No" buttons, and a close icon.
  - **Pending Request Modal**: If a pending request exists, show a modal saying, "You already have a pending request."
  - **Role Request Status Modal**: On clicking "Check My Role Requests", show a modal with the current role request status (if any).
- **API Integration**:
  - Use `SubmitRoleRequest` (`POST /api/RoleRequestController/SubmitRoleRequest`) when confirming a role request.
  - Use `GetMyRoleRequest` (`GET /api/RoleRequestController/MyRequest`) to check for existing requests and display request status.

#### Admin Home Page
- **Create `AdminHomeComponent`**:
  - Display a table of pending role requests using `GetPendingRequests` (`GET /api/RoleRequestController/PendingRequests`).
  - Columns: Request ID, User ID, Username, Email, Requested Role, Requested At, Actions (Approve/Reject buttons).
  - Use `ReviewRequest` (`POST /api/RoleRequestController/ReviewRequest`) to approve or reject requests.
  - Style with Bootstrap for consistency.

#### Role Service
- **Create `RoleService`**:
  - Handle all role-related API calls (`SubmitRoleRequest`, `GetMyRoleRequest`, `GetPendingRequests`, `ReviewRequest`).
  - Move existing order-related API calls (`GetAllOrdersForDeliveryAgent`, `UpdateOrderStatus`) from `DeliveryAgentHomeComponent` and `DeliveryAgentDashboardComponent` to this service.
  - Use the base URL `http://localhost:5287`.

#### Routing
- Add a route for the Admin Home Page (e.g., `/admin-home`).

---

### Step 2: Create the Role Service
Let’s create a `RoleService` to handle all role-related and order-related API calls, ensuring the code remains modular.

**`src/app/services/role.service.ts` (New)**:
```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

interface Address {
  id: number;
  houseNumber: number;
  streetName: string;
  colonyName?: string;
  city: string;
  state: string;
  pincode: number;
}

interface DeliveryAgentOrder {
  id: number;
  address: Address;
  status: string;
  orderDate: string;
  placedAt: string | null;
  shippedAt: string | null;
  deliveredAt: string | null;
  cancelledAt: string | null;
}

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

interface RoleRequestDTO {
  requestedRole: string;
}

interface RoleRequestResponseDTO {
  requestId: number;
  status: string;
}

interface ResponseDTO<T> {
  success: boolean;
  message: string;
  data: T;
}

@Injectable({
  providedIn: 'root'
})
export class RoleService {
  private baseUrl = 'http://localhost:5287/api/RoleRequestController';

  constructor(private http: HttpClient) {}

  submitRoleRequest(request: RoleRequestDTO): Observable<ResponseDTO<string>> {
    return this.http.post<ResponseDTO<string>>(`${this.baseUrl}/SubmitRoleRequest`, request);
  }

  getMyRoleRequest(): Observable<ResponseDTO<RoleRequest | null>> {
    return this.http.get<ResponseDTO<RoleRequest | null>>(`${this.baseUrl}/MyRequest`);
  }

  getPendingRequests(): Observable<ResponseDTO<RoleRequest[]>> {
    return this.http.get<ResponseDTO<RoleRequest[]>>(`${this.baseUrl}/PendingRequests`);
  }

  reviewRequest(request: RoleRequestResponseDTO): Observable<ResponseDTO<string>> {
    return this.http.post<ResponseDTO<string>>(`${this.baseUrl}/ReviewRequest`, request);
  }

  getOrdersForDeliveryAgent(): Observable<ResponseDTO<DeliveryAgentOrder[]>> {
    return this.http.get<ResponseDTO<DeliveryAgentOrder[]>>(`http://localhost:5287/api/OrderController/GetAllOrdersForDeliveryAgent`);
  }

  updateOrderStatus(orderId: number, newStatus: string): Observable<ResponseDTO<DeliveryAgentOrder>> {
    return this.http.put<ResponseDTO<DeliveryAgentOrder>>(
      `http://localhost:5287/api/OrderController/UpdateOrderStatus/${orderId}`,
      { status: newStatus }
    );
  }
}
```

- **Details**:
  - Handles role-related APIs (`SubmitRoleRequest`, `GetMyRoleRequest`, `GetPendingRequests`, `ReviewRequest`).
  - Includes order-related APIs (`GetOrdersForDeliveryAgent`, `UpdateOrderStatus`) moved from previous components.
  - Uses lowercase properties to match the backend response (e.g., `success`, `message`, `data`).
  - Base URL updated to `http://localhost:5287`.

---

### Step 3: Update `HeaderComponent` to Add Role Options and Modals
#### Update TypeScript File
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
Update CSS Fileheader.component.css (Updated):.navbar-brand {
  font-size: 1.5rem;
}
.nav-link {
  font-weight: 500;
}
.input-group {
  max-width: 400px;
}
.login-signup-btn:hover {
  background-color: #ffffff;
  color: #0d6efd;
  border-color: #0d6efd;
}
.dropdown-menu {
  min-width: 150px;
}
.dropdown-toggle {
  cursor: pointer;
}
.badge {
  font-size: 0.7rem;
  padding: 0.25em 0.5em;
}
.dropdown-item {
  cursor: pointer;
}Changes:Added cursor: pointer to .dropdown-item for better UX.Step 4: Update DeliveryAgentHomeComponent to Use RoleServicedelivery-agent-home.component.ts (Updated):import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterLink } from '@angular/router';
import { AuthService } from '../../services/auth.service';
import { RoleService } from '../../services/role.service';

interface Address {
  id: number;
  houseNumber: number;
  streetName: string;
  colonyName?: string;
  city: string;
  state: string;
  pincode: number;
}

interface DeliveryAgentOrder {
  id: number;
  address: Address;
  status: string;
  orderDate: string;
  placedAt: string | null;
  shippedAt: string | null;
  deliveredAt: string | null;
  cancelledAt: string | null;
}

@Component({
  selector: 'app-delivery-agent-home',
  standalone: true,
  imports: [CommonModule, RouterLink],
  templateUrl: './delivery-agent-home.component.html',
  styleUrls: ['./delivery-agent-home.component.css']
})
export class DeliveryAgentHomeComponent implements OnInit {
  orders: DeliveryAgentOrder[] = [];
  errorMessage: string | null = null;
  isLoading: boolean = false;

  constructor(private roleService: RoleService, private authService: AuthService) {}

  ngOnInit() {
    this.fetchOrders();
  }

  fetchOrders() {
    if (!this.authService.isLoggedIn() || this.authService.getUserRole() !== 'Delivery Agent') {
      this.errorMessage = 'Unauthorized access. Please log in as a Delivery Agent.';
      return;
    }

    this.isLoading = true;
    this.errorMessage = null;
    this.orders = [];

    this.roleService.getOrdersForDeliveryAgent().subscribe({
      next: (response) => {
        console.log('Full response:', response);
        if (response.success) {
          this.orders = response.data;
          console.log('Assigned orders:', this.orders);
        } else {
          this.errorMessage = response.message;
        }
        this.isLoading = false;
      },
      error: (err) => {
        this.errorMessage = err.error?.message || 'Failed to fetch orders.';
        console.error('Error fetching orders:', err);
        this.isLoading = false;
      }
    });
  }

  refreshOrders() {
    this.fetchOrders();
  }
}Changes:Replaced HttpClient with RoleService for API calls.No changes to delivery-agent-home.component.html or .css.Step 5: Update DeliveryAgentDashboardComponent to Use RoleServicedelivery-agent-dashboard.component.ts (Updated):import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { RouterLink } from '@angular/router';
import { AuthService } from '../../services/auth.service';
import { RoleService } from '../../services/role.service';

interface Address {
  id: number;
  houseNumber: number;
  streetName: string;
  colonyName?: string;
  city: string;
  state: string;
  pincode: number;
}

interface DeliveryAgentOrder {
  id: number;
  address: Address;
  status: string;
  orderDate: string;
  placedAt: string | null;
  shippedAt: string | null;
  deliveredAt: string | null;
  cancelledAt: string | null;
}

@Component({
  selector: 'app-delivery-agent-dashboard',
  standalone: true,
  imports: [CommonModule, RouterLink],
  templateUrl: './delivery-agent-dashboard.component.html',
  styleUrls: ['./delivery-agent-dashboard.component.css']
})
export class DeliveryAgentDashboardComponent implements OnInit {
  allOrders: DeliveryAgentOrder[] = [];
  placedOrders: DeliveryAgentOrder[] = [];
  shippedOrders: DeliveryAgentOrder[] = [];
  cancelledOrders: DeliveryAgentOrder[] = [];
  errorMessage: string | null = null;
  isLoading: boolean = false;

  constructor(private roleService: RoleService, private authService: AuthService) {}

  ngOnInit() {
    this.fetchOrders();
  }

  fetchOrders() {
    if (!this.authService.isLoggedIn() || this.authService.getUserRole() !== 'Delivery Agent') {
      this.errorMessage = 'Unauthorized access. Please log in as a Delivery Agent.';
      return;
    }

    this.isLoading = true;
    this.errorMessage = null;
    this.allOrders = [];

    this.roleService.getOrdersForDeliveryAgent().subscribe({
      next: (response) => {
        console.log('Full response:', response);
        if (response.success) {
          this.allOrders = response.data;
          this.placedOrders = this.allOrders.filter(order => order.status === 'Placed');
          this.shippedOrders = this.allOrders.filter(order => order.status === 'Shipped');
          this.cancelledOrders = this.allOrders.filter(order => order.status === 'Cancelled');
          console.log('All orders:', this.allOrders);
        } else {
          this.errorMessage = response.message;
        }
        this.isLoading = false;
      },
      error: (err) => {
        this.errorMessage = err.error?.message || 'Failed to fetch orders.';
        console.error('Error fetching orders:', err);
        this.isLoading = false;
      }
    });
  }

  updateOrderStatus(orderId: number, newStatus: string) {
    this.roleService.updateOrderStatus(orderId, newStatus).subscribe({
      next: (response) => {
        console.log('Update response:', response);
        if (response.success) {
          this.fetchOrders();
        } else {
          this.errorMessage = response.message;
        }
      },
      error: (err) => {
        this.errorMessage = err.error?.message || 'Failed to update order status.';
        console.error('Error updating status:', err);
      }
    });
  }
}Changes:Replaced HttpClient with RoleService for API calls.No changes to delivery-agent-dashboard.component.html or .css.Step 6: Create the Admin Home PageCreate AdminHomeComponentadmin-home.component.ts (New):import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { AuthService } from '../../services/auth.service';
import { RoleService } from '../../services/role.service';

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
  selector: 'app-admin-home',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './admin-home.component.html',
  styleUrls: ['./admin-home.component.css']
})
export class AdminHomeComponent implements OnInit {
  pendingRequests: RoleRequest[] = [];
  errorMessage: string | null = null;
  isLoading: boolean = false;

  constructor(private roleService: RoleService, private authService: AuthService) {}

  ngOnInit() {
    this.fetchPendingRequests();
  }

  fetchPendingRequests() {
    if (!this.authService.isLoggedIn() || this.authService.getUserRole() !== 'Admin') {
      this.errorMessage = 'Unauthorized access. Please log in as an Admin.';
      return;
    }

    this.isLoading = true;
    this.errorMessage = null;
    this.pendingRequests = [];

    this.roleService.getPendingRequests().subscribe({
      next: (response) => {
        if (response.success) {
          this.pendingRequests = response.data;
          console.log('Pending requests:', this.pendingRequests);
        } else {
          this.errorMessage = response.message;
        }
        this.isLoading = false;
      },
      error: (err) => {
        this.errorMessage = err.error?.message || 'Failed to fetch pending requests.';
        console.error('Error fetching requests:', err);
        this.isLoading = false;
      }
    });
  }

  reviewRequest(requestId: number, status: 'Approved' | 'Rejected') {
    this.roleService.reviewRequest({ requestId, status }).subscribe({
      next: (response) => {
        if (response.success) {
          this.fetchPendingRequests();
        } else {
          this.errorMessage = response.message;
        }
      },
      error: (err) => {
        this.errorMessage = err.error?.message || `Failed to ${status.toLowerCase()} request.`;
        console.error('Error reviewing request:', err);
      }
    });
  }
}admin-home.component.html (New):<div class="container py-5">
  <h1 class="text-center mb-4">Admin Dashboard</h1>

  <!-- Error Message -->
  <div *ngIf="errorMessage" class="alert alert-danger mt-4" role="alert">
    {{ errorMessage }}
  </div>

  <!-- Loading State -->
  <div *ngIf="isLoading" class="text-center">
    <div class="spinner-border text-primary" role="status">
      <span class="visually-hidden">Loading...</span>
    </div>
    <p>Loading pending requests...</p>
  </div>

  <!-- Pending Requests -->
  <div *ngIf="!isLoading">
    <h3>Pending Role Requests</h3>
    <div *ngIf="pendingRequests.length === 0" class="alert alert-info">
      No pending role requests at the moment.
    </div>
    <div *ngIf="pendingRequests.length > 0">
      <table class="table table-striped">
        <thead>
          <tr>
            <th scope="col">Request ID</th>
            <th scope="col">User ID</th>
            <th scope="col">Username</th>
            <th scope="col">Email</th>
            <th scope="col">Requested Role</th>
            <th scope="col">Requested At</th>
            <th scope="col">Actions</th>
          </tr>
        </thead>
        <tbody>
          <tr *ngFor="let request of pendingRequests">
            <td>{{ request.id }}</td>
            <td>{{ request.userId }}</td>
            <td>{{ request.user?.userName || 'N/A' }}</td>
            <td>{{ request.user?.email || 'N/A' }}</td>
            <td>{{ request.requestedRole }}</td>
            <td>{{ request.requestedAt | date:'medium' }}</td>
            <td>
              <button class="btn btn-sm btn-success me-1" (click)="reviewRequest(request.id, 'Approved')">Approve</button>
              <button class="btn btn-sm btn-danger" (click)="reviewRequest(request.id, 'Rejected')">Reject</button>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</div>admin-home.component.css (New):.table {
  margin-top: 20px;
}Details:Displays a Bootstrap-styled table with pending role requests.Includes "Approve" and "Reject" buttons for each request.Shows user details (userName, email) if available, otherwise "N/A".Step 7: Update RoutingAdd a route for the AdminHomeComponent.app.routes.ts (Update):import { Routes } from '@angular/router';
import { DeliveryAgentHomeComponent } from './components/delivery-agent-home/delivery-agent-home.component';
import { DeliveryAgentDashboardComponent } from './components/delivery-agent-dashboard/delivery-agent-dashboard.component';
import { AdminHomeComponent } from './components/admin-home/admin-home.component';

export const routes: Routes = [
  { path: 'delivery-agent-home', component: DeliveryAgentHomeComponent },
  { path: 'delivery-agent-dashboard', component: DeliveryAgentDashboardComponent },
  { path: 'admin-home', component: AdminHomeComponent },
  { path: '', redirectTo: '/', pathMatch: 'full' }
];Note: I’ve included the existing routes for DeliveryAgentHomeComponent and DeliveryAgentDashboardComponent. Added /admin-home for the new component. Adjust the redirectTo path ('/') if there’s a different default route in your app.Step 8: Test and DebugHeader Dropdown:Log in as a Customer and verify that "Become a Merchant", "Become a Delivery Agent", and "Check My Role Requests" appear in the profile dropdown.Clicking "Become a Merchant" should open the confirmation modal; clicking "Yes, I Confirm" should submit the request.If a pending request exists, the pending request modal should appear instead.Clicking "Check My Role Requests" should show the status modal.Admin Home Page:Log in as an Admin and navigate to /admin-home.Verify that pending requests are displayed in the table.Test approving and rejecting requests; the list should update accordingly.Error Handling:API errors are displayed in the messageModal for role requests and inline for the admin page.