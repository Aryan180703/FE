Let’s implement the delivery agent’s homepage and dashboard for your EShoppingZone project. We’ll start by analyzing the backend implementations to understand the data and functionality available, then proceed to create the frontend components as requested. The delivery agent should only see the order details (like order ID, items, and address) and the status, with the ability to update the status for non-delivered orders. We’ll ensure orders are displayed from most recent to oldest and include date and time at each step.

---

### Step 1: Analyze Backend Implementations
Let’s break down the backend to understand the data and endpoints available for the delivery agent:

#### **Key Endpoints and DTOs**
1. **`GetAllOrdersForDeliveryAgent` Endpoint**:
   - **Path**: `GET /api/OrderController/GetAllOrdersForDeliveryAgent`
   - **Role**: `Delivery Agent`
   - **Response**: `ResponseDTO<List<DeliveryAgentOrderResponse>>`
     - `DeliveryAgentOrderResponse`:
       - `Id`: Order ID
       - `Address`: Address of the customer (via `AddressResponse`)
       - `Status`: Current status of the order (`Placed`, `Shipped`, `Delivered`, `Cancelled`)
   - **Note**: The service filters out `Delivered` orders (`orders = orders.Where(o => o.Status != "Delivered").ToList()`), so the delivery agent only sees orders that are `Placed`, `Shipped`, or `Cancelled`.

2. **`UpdateOrderStatus` Endpoint**:
   - **Path**: `PUT /api/OrderController/UpdateOrderStatus/{orderId}`
   - **Role**: `Delivery Agent`
   - **Request**: `UpdateOrderStatusRequest`
     - `Status`: New status (`Placed`, `Shipped`, `Delivered`, `Cancelled`)
   - **Response**: `ResponseDTO<OrderResponse>`
     - `OrderResponse`:
       - `Id`, `CustomerId`, `AddressId`, `TotalPrice`, `Status`, `OrderDate`, `Items` (list of `OrderItemResponse`)
     - `OrderItemResponse`:
       - `ProductId`, `ProductName`, `Price`, `Quantity`
   - **Note**: The delivery agent can update the status of any order, but we’ll restrict status changes for `Delivered` orders on the frontend as per your requirement.

#### **Data Available to Delivery Agent**
- The delivery agent can see:
  - **Order Details**: Order ID, items (product name, quantity), address, status, and order date.
  - **Address Details**: Via `AddressResponse` (likely includes fields like street, city, state, etc., based on typical address DTOs).
- The delivery agent cannot see:
  - Customer personal details (e.g., name, email), total price, or other sensitive information.

#### **Sorting Requirement**
- Orders must be displayed from most recent to oldest. The `OrderDate` field in `OrderResponse` (returned after status updates) can be used for sorting, but `DeliveryAgentOrderResponse` doesn’t include `OrderDate`. We’ll need to fetch additional details via the `GetOrder` endpoint or modify the backend to include `OrderDate` in `DeliveryAgentOrderResponse`.

#### **Date and Time Display**
- We’ll display the `OrderDate` for each order and the timestamp of status updates (if available). Since `DeliveryAgentOrderResponse` doesn’t include `OrderDate`, we’ll need to adjust our approach.

---

### Step 2: Plan the Frontend Implementation
#### **Delivery Agent Home Page**
- **Purpose**: Welcome the delivery agent and provide quick actions.
- **Quick Actions** (Ideas):
  - View Dashboard (navigate to `/delivery-agent-dashboard`)
  - View Profile (navigate to `/delivery-agent-profile`)
  - Contact Support (placeholder link or modal)
- **Layout**: A simple banner with a welcome message and a section for quick actions.

#### **Delivery Agent Dashboard**
- **Purpose**: Display orders in three categories: Placed, Shipped, and Cancelled.
- **Details to Show**:
  - For each order: Order ID, items (product name and quantity), address, status, and order date (we’ll need to fetch this).
  - Option to change status (dropdown with `Placed`, `Shipped`, `Delivered`, `Cancelled`), disabled for `Delivered` orders.
- **Sorting**: Most recent to oldest (based on `OrderDate`).
- **Date and Time**: Show `OrderDate` for each order and the timestamp of the last status update (if available).

#### **Challenges**
- **Missing `OrderDate` in `DeliveryAgentOrderResponse`**: We’ll fetch full order details using the `GetOrder` endpoint for each order to get the `OrderDate` and sort accordingly.
- **Status Update History**: The backend doesn’t track status update timestamps. We’ll only show the `OrderDate` for now, but you might want to add a status history table in the backend later.

---

### Step 3: Create the Delivery Agent Home Page
Let’s create `DeliveryAgentHomeComponent` to welcome the delivery agent and provide quick actions.

#### **`delivery-agent-home.component.ts`**
```typescript
import { Component } from '@angular/core';
import { RouterLink } from '@angular/router';
import { CommonModule } from '@angular/common';

@Component({
  selector: 'app-delivery-agent-home',
  standalone: true,
  imports: [CommonModule, RouterLink],
  templateUrl: './delivery-agent-home.component.html',
  styleUrls: ['./delivery-agent-home.component.css']
})
export class DeliveryAgentHomeComponent {
  // No logic needed for now, just navigation
}
```

#### **`delivery-agent-home.component.html`**
```html
<div class="container py-5 px-5">
  <!-- Banner -->
  <div class="bg-primary text-white text-center py-5 rounded shadow-sm mb-5">
    <h1 class="fw-bold">Welcome, Delivery Agent! 🚚</h1>
    <p class="lead">Let's get those orders delivered on time!</p>
  </div>

  <!-- Quick Actions -->
  <div class="row">
    <div class="col-md-12">
      <h3 class="fw-bold mb-4">Quick Actions</h3>
      <div class="d-flex flex-wrap gap-3">
        <a [routerLink]="['/delivery-agent-dashboard']" class="btn btn-primary btn-lg">
          View Dashboard 📊
        </a>
        <a [routerLink]="['/delivery-agent-profile']" class="btn btn-outline-primary btn-lg">
          View Profile 👤
        </a>
        <button class="btn btn-outline-secondary btn-lg" (click)="contactSupport()">
          Contact Support 📞
        </button>
      </div>
    </div>
  </div>
</div>
```

#### **`delivery-agent-home.component.css`**
```css
.btn-lg {
  padding: 0.75rem 1.5rem;
  font-size: 1.1rem;
}
```

---

### Step 4: Create the Delivery Agent Dashboard
Now, let’s create `DeliveryAgentDashboardComponent` to display orders in three categories: Placed, Shipped, and Cancelled. We’ll fetch orders using the `GetAllOrdersForDeliveryAgent` endpoint, fetch additional details (like `OrderDate`) using the `GetOrder` endpoint, and allow status updates.

#### **Update `OrderService`**
First, let’s update the frontend `OrderService` to include methods for the delivery agent endpoints.

**`order.service.ts`** (Add these methods if not already present):
```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';

export interface AddressResponse {
  id: number;
  street: string;
  city: string;
  state: string;
  postalCode: string;
  country: string;
}

export interface OrderItemResponse {
  productId: number;
  productName: string;
  price: number;
  quantity: number;
}

export interface DeliveryAgentOrderResponse {
  id: number;
  address: AddressResponse;
  status: string;
}

export interface OrderResponse {
  id: number;
  customerId: number;
  addressId: number;
  totalPrice: number;
  status: string;
  orderDate: string; // ISO string
  items: OrderItemResponse[];
}

export interface ResponseDTO<T> {
  success: boolean;
  message: string;
  data: T;
}

@Injectable({
  providedIn: 'root'
})
export class OrderService {
  private baseUrl = 'http://localhost:5287/api/OrderController';

  constructor(private http: HttpClient) {}

  getAllOrdersForDeliveryAgent(): Observable<ResponseDTO<DeliveryAgentOrderResponse[]>> {
    return this.http.get<ResponseDTO<DeliveryAgentOrderResponse[]>>(`${this.baseUrl}/GetAllOrdersForDeliveryAgent`);
  }

  getOrder(profileId: number, orderId: number): Observable<ResponseDTO<OrderResponse>> {
    return this.http.get<ResponseDTO<OrderResponse>>(`${this.baseUrl}/GetOrder/${orderId}`);
  }

  updateOrderStatus(profileId: number, orderId: number, status: string): Observable<ResponseDTO<OrderResponse>> {
    return this.http.put<ResponseDTO<OrderResponse>>(`${this.baseUrl}/UpdateOrderStatus/${orderId}`, { status });
  }
}
```

#### **`delivery-agent-dashboard.component.ts`**
```typescript
import { Component, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { OrderService, DeliveryAgentOrderResponse, OrderResponse } from '../../services/order.service';
import { FormsModule } from '@angular/forms';
import { AuthService } from '../../services/auth.service';

interface EnhancedOrder extends DeliveryAgentOrderResponse {
  orderDate?: string; // Added for sorting
  items?: { productName: string; quantity: number }[]; // Added for display
}

@Component({
  selector: 'app-delivery-agent-dashboard',
  standalone: true,
  imports: [CommonModule, FormsModule],
  templateUrl: './delivery-agent-dashboard.component.html',
  styleUrls: ['./delivery-agent-dashboard.component.css']
})
export class DeliveryAgentDashboardComponent implements OnInit {
  placedOrders: EnhancedOrder[] = [];
  shippedOrders: EnhancedOrder[] = [];
  cancelledOrders: EnhancedOrder[] = [];
  errorMessage: string | null = null;

  constructor(
    private orderService: OrderService,
    private authService: AuthService
  ) {}

  ngOnInit() {
    this.loadOrders();
  }

  loadOrders() {
    this.orderService.getAllOrdersForDeliveryAgent().subscribe({
      next: (response) => {
        if (response.success && response.data) {
          const orders = response.data;
          // Fetch additional details for each order
          this.enhanceOrders(orders);
        } else {
          this.errorMessage = response.message || 'No orders found';
        }
      },
      error: (err) => {
        this.errorMessage = 'Error loading orders';
        console.error(err);
      }
    });
  }

  enhanceOrders(orders: DeliveryAgentOrderResponse[]) {
    const profileId = this.authService.getProfileId(); // Assumes AuthService provides profileId
    orders.forEach(order => {
      this.orderService.getOrder(profileId, order.id).subscribe({
        next: (orderResponse) => {
          if (orderResponse.success && orderResponse.data) {
            const enhancedOrder: EnhancedOrder = {
              ...order,
              orderDate: orderResponse.data.orderDate,
              items: orderResponse.data.items.map(item => ({
                productName: item.productName,
                quantity: item.quantity
              }))
            };

            // Categorize orders
            if (order.status === 'Placed') {
              this.placedOrders.push(enhancedOrder);
            } else if (order.status === 'Shipped') {
              this.shippedOrders.push(enhancedOrder);
            } else if (order.status === 'Cancelled') {
              this.cancelledOrders.push(enhancedOrder);
            }

            // Sort orders by orderDate (most recent first)
            this.placedOrders.sort((a, b) => new Date(b.orderDate!).getTime() - new Date(a.orderDate!).getTime());
            this.shippedOrders.sort((a, b) => new Date(b.orderDate!).getTime() - new Date(a.orderDate!).getTime());
            this.cancelledOrders.sort((a, b) => new Date(b.orderDate!).getTime() - new Date(a.orderDate!).getTime());
          }
        },
        error: (err) => {
          console.error(`Error fetching details for order ${order.id}`, err);
        }
      });
    });
  }

  updateOrderStatus(order: EnhancedOrder, newStatus: string) {
    if (order.status === 'Delivered') return; // Shouldn't happen due to backend filtering, but just in case

    const profileId = this.authService.getProfileId();
    this.orderService.updateOrderStatus(profileId, order.id, newStatus).subscribe({
      next: (response) => {
        if (response.success && response.data) {
          // Update the order status and recategorize
          const updatedOrder: EnhancedOrder = {
            ...order,
            status: response.data.status,
            orderDate: response.data.orderDate,
            items: response.data.items.map(item => ({
              productName: item.productName,
              quantity: item.quantity
            }))
          };

          // Remove from current category
          this.placedOrders = this.placedOrders.filter(o => o.id !== order.id);
          this.shippedOrders = this.shippedOrders.filter(o => o.id !== order.id);
          this.cancelledOrders = this.cancelledOrders.filter(o => o.id !== order.id);

          // Add to new category (unless Delivered)
          if (updatedOrder.status === 'Placed') {
            this.placedOrders.push(updatedOrder);
            this.placedOrders.sort((a, b) => new Date(b.orderDate!).getTime() - new Date(a.orderDate!).getTime());
          } else if (updatedOrder.status === 'Shipped') {
            this.shippedOrders.push(updatedOrder);
            this.shippedOrders.sort((a, b) => new Date(b.orderDate!).getTime() - new Date(a.orderDate!).getTime());
          } else if (updatedOrder.status === 'Cancelled') {
            this.cancelledOrders.push(updatedOrder);
            this.cancelledOrders.sort((a, b) => new Date(b.orderDate!).getTime() - new Date(a.orderDate!).getTime());
          }
          // If status is Delivered, it will be removed from all lists (as per backend filtering)
        } else {
          this.errorMessage = response.message || 'Failed to update order status';
        }
      },
      error: (err) => {
        this.errorMessage = 'Error updating order status';
        console.error(err);
      }
    });
  }

  formatDate(dateString: string): string {
    const date = new Date(dateString);
    return date.toLocaleString('en-US', { month: 'short', day: 'numeric', year: 'numeric', hour: 'numeric', minute: 'numeric', hour12: true });
  }
}
```

#### **`delivery-agent-dashboard.component.html`**
```html
<div class="container py-5 px-5">
  <h2 class="fw-bold mb-4">Delivery Agent Dashboard</h2>
  <div *ngIf="errorMessage" class="alert alert-danger" role="alert">
    {{ errorMessage }}
  </div>

  <!-- Placed Orders -->
  <div class="mb-5">
    <h3 class="fw-bold mb-3">Placed Orders</h3>
    <div *ngIf="placedOrders.length > 0; else noPlacedOrders">
      <div class="card shadow-sm border-0 mb-3" *ngFor="let order of placedOrders">
        <div class="card-body">
          <h5 class="card-title">Order #{{ order.id }}</h5>
          <p class="card-text"><strong>Order Date:</strong> {{ formatDate(order.orderDate!) }}</p>
          <p class="card-text"><strong>Status:</strong> {{ order.status }}</p>
          <p class="card-text"><strong>Items:</strong></p>
          <ul>
            <li *ngFor="let item of order.items">{{ item.productName }} (Qty: {{ item.quantity }})</li>
          </ul>
          <p class="card-text"><strong>Delivery Address:</strong> {{ order.address.street }}, {{ order.address.city }}, {{ order.address.state }}, {{ order.address.postalCode }}, {{ order.address.country }}</p>
          <div class="mt-3">
            <label for="status-{{order.id}}" class="form-label">Update Status:</label>
            <select id="status-{{order.id}}" class="form-select w-auto d-inline-block ms-2" [(ngModel)]="order.status" (change)="updateOrderStatus(order, $event.target.value)">
              <option value="Placed">Placed</option>
              <option value="Shipped">Shipped</option>
              <option value="Delivered">Delivered</option>
              <option value="Cancelled">Cancelled</option>
            </select>
          </div>
        </div>
      </div>
    </div>
    <ng-template #noPlacedOrders>
      <p class="text-muted">No placed orders at the moment.</p>
    </ng-template>
  </div>

  <!-- Shipped Orders -->
  <div class="mb-5">
    <h3 class="fw-bold mb-3">Shipped Orders</h3>
    <div *ngIf="shippedOrders.length > 0; else noShippedOrders">
      <div class="card shadow-sm border-0 mb-3" *ngFor="let order of shippedOrders">
        <div class="card-body">
          <h5 class="card-title">Order #{{ order.id }}</h5>
          <p class="card-text"><strong>Order Date:</strong> {{ formatDate(order.orderDate!) }}</p>
          <p class="card-text"><strong>Status:</strong> {{ order.status }}</p>
          <p class="card-text"><strong>Items:</strong></p>
          <ul>
            <li *ngFor="let item of order.items">{{ item.productName }} (Qty: {{ item.quantity }})</li>
          </ul>
          <p class="card-text"><strong>Delivery Address:</strong> {{ order.address.street }}, {{ order.address.city }}, {{ order.address.state }}, {{ order.address.postalCode }}, {{ order.address.country }}</p>
          <div class="mt-3">
            <label for="status-{{order.id}}" class="form-label">Update Status:</label>
            <select id="status-{{order.id}}" class="form-select w-auto d-inline-block ms-2" [(ngModel)]="order.status" (change)="updateOrderStatus(order, $event.target.value)">
              <option value="Placed">Placed</option>
              <option value="Shipped">Shipped</option>
              <option value="Delivered">Delivered</option>
              <option value="Cancelled">Cancelled</option>
            </select>
          </div>
        </div>
      </div>
    </div>
    <ng-template #noShippedOrders>
      <p class="text-muted">No shipped orders at the moment.</p>
    </ng-template>
  </div>

  <!-- Cancelled Orders -->
  <div class="mb-5">
    <h3 class="fw-bold mb-3">Cancelled Orders</h3>
    <div *ngIf="cancelledOrders.length > 0; else noCancelledOrders">
      <div class="card shadow-sm border-0 mb-3" *ngFor="let order of cancelledOrders">
        <div class="card-body">
          <h5 class="card-title">Order #{{ order.id }}</h5>
          <p class="card-text"><strong>Order Date:</strong> {{ formatDate(order.orderDate!) }}</p>
          <p class="card-text"><strong>Status:</strong> {{ order.status }}</p>
          <p class="card-text"><strong>Items:</strong></p>
          <ul>
            <li *ngFor="let item of order.items">{{ item.productName }} (Qty: {{ item.quantity }})</li>
          </ul>
          <p class="card-text"><strong>Delivery Address:</strong> {{ order.address.street }}, {{ order.address.city }}, {{ order.address.state }}, {{ order.address.postalCode }}, {{ order.address.country }}</p>
          <div class="mt-3">
            <label for="status-{{order.id}}" class="form-label">Update Status:</label>
            <select id="status-{{order.id}}" class="form-select w-auto d-inline-block ms-2" [(ngModel)]="order.status" (change)="updateOrderStatus(order, $event.target.value)">
              <option value="Placed">Placed</option>
              <option value="Shipped">Shipped</option>
              <option value="Delivered">Delivered</option>
              <option value="Cancelled">Cancelled</option>
            </select>
          </div>
        </div>
      </div>
    </div>
    <ng-template #noCancelledOrders>
      <p class="text-muted">No cancelled orders at the moment.</p>
    </ng-template>
  </div>
</div>
```

#### **`delivery-agent-dashboard.component.css`**
```css
.card {
  border-radius: 8px;
}
.card-body {
  padding: 1.5rem;
}
.form-select {
  width: auto;
}
```

---

Step 5: Update RoutingWe need to add routes for the delivery agent’s home page and dashboard, and update RoleGuard to redirect delivery agents to /delivery-agent-home.app.routes.tsimport { Routes } from '@angular/router';
import { DeliveryAgentHomeComponent } from './components/delivery-agent-home/delivery-agent-home.component';
import { DeliveryAgentDashboardComponent } from './components/delivery-agent-dashboard/delivery-agent-dashboard.component';
import { authGuard } from './guards/auth.guard';
import { RoleGuard } from './guards/role.guard';

export const routes: Routes = [
  // Existing routes...
  { path: 'delivery-agent-home', component: DeliveryAgentHomeComponent, canActivate: [authGuard, RoleGuard], data: { expectedRole: 'Delivery Agent' } },
  { path: 'delivery-agent-dashboard', component: DeliveryAgentDashboardComponent, canActivate: [authGuard, RoleGuard], data: { expectedRole: 'Delivery Agent' } },
  // Add delivery-agent-profile route if needed
];role.guard.tsUpdate RoleGuard to redirect delivery agents to /delivery-agent-home:import { Injectable } from '@angular/core';
import { CanActivate, ActivatedRouteSnapshot, RouterStateSnapshot, Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

@Injectable({
  providedIn: 'root'
})
export class RoleGuard implements CanActivate {
  constructor(private authService: AuthService, private router: Router) {}

  canActivate(route: ActivatedRouteSnapshot, state: RouterStateSnapshot): boolean {
    const expectedRole = route.data['expectedRole'];
    const userRole = this.authService.getUserRole();

    if (!userRole || userRole !== expectedRole) {
      // Redirect based on user role
      if (userRole === 'Customer') {
        this.router.navigate(['/home']);
      } else if (userRole === 'Merchant') {
        this.router.navigate(['/merchant-home']);
      } else if (userRole === 'Delivery Agent') {
        this.router.navigate(['/delivery-agent-home']);
      } else {
        this.router.navigate(['/login']);
      }
      return false;
    }
    return true;
  }
}Step 6: Test ScenariosLet’s ensure everything works