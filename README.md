
---

### **Step 1: Discuss the Changes**
Humare paas `EmailService` class hai jo `SmtpClient` ka use karke emails send karti hai. Iska use karne ke liye hume backend mein yeh changes karne honge:

1. **Remove RabbitMQ Dependency**:
   - `IRabbitMQProducer` aur `RabbitMQProducer` ko hata denge, kyunki ab hum directly `EmailService` ka use karenge.
   - `Program.cs` se `IRabbitMQProducer` ki dependency injection registration bhi hata denge.

2. **Inject `IEmailService` in `RoleRequestController`**:
   - `RoleRequestController` mein `IRabbitMQProducer` ke bajaye `IEmailService` inject karenge.
   - Jab bhi email bhejne ki zarurat hogi (application submission, approval, rejection), `IEmailService.SendEmailAsync` call karenge.

3. **Update Email Sending Logic**:
   - Pehle hum `RabbitMQProducer.PublishMessage` call kar rahe the `user-events` queue mein message publish karne ke liye.
   - Ab hum `IEmailService.SendEmailAsync` ka use karenge directly email bhejne ke liye.
   - Email messages ke content same rahenge:
     - Application submission: "Your application to become a [role] has been accepted. We will review it and get back to you soon."
     - Approval: "Congratulations! Your request to become a [role] has been approved!"
     - Rejection: "We regret to inform you that your request to become a [role] has been rejected. Currently, we are looking to expand our team in other areas..."

4. **Remove `EmailMessage` Model** (Optional):
   - Chuki `EmailService` directly `email`, `subject`, aur `body` parameters leti hai, hume `EmailMessage` model ki zarurat nahi hai ab.
   - Ise hata sakte hain, lekin agar future mein use ho sakta hai, toh rakh bhi sakte hain.

5. **Configure SMTP Settings**:
   - `appsettings.json` mein SMTP settings add karne honge (jaise host, port, username, password, etc.) jo `EmailService` use karega.

---

### **Step 2: Implement the Changes**

#### **Step 2.1: Remove RabbitMQ Dependency**
Pehle RabbitMQ related code hata denge.

**Delete `IRabbitMQProducer` and `RabbitMQProducer`**:
- `IRabbitMQProducer.cs` aur `RabbitMQProducer.cs` files delete kar do:
  - Path: `EShoppingZone.Services/IRabbitMQProducer.cs`
  - Path: `EShoppingZone.Services/RabbitMQProducer.cs`

**Delete `EmailMessage` Model** (Optional):
Agar `EmailMessage` model ka aur koi use nahi hai, toh ise bhi hata sakte ho:
- Path: `EShoppingZone.Models/EmailMessage.cs`

**Update `Program.cs`**:
`IRabbitMQProducer` ki registration hata denge.

**File: `EShoppingZone/Program.cs`**
```csharp
using EShoppingZone.Data;
using EShoppingZone.Models;
using EShoppingZone.Services;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllers();
builder.Services.AddDbContext<EShoppingZoneDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
builder.Services.AddIdentity<UserProfile, IdentityRole>()
    .AddEntityFrameworkStores<EShoppingZoneDbContext>()
    .AddDefaultTokenProviders();
builder.Services.AddScoped<IAuthService, AuthService>();
builder.Services.AddScoped<IRepository, Repository>();
builder.Services.AddScoped<IEmailService, EmailService>(); // Add EmailService
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = "Jwt";
    options.DefaultChallengeScheme = "Jwt";
}).AddJwtBearer("Jwt", options =>
{
    options.TokenValidationParameters = new Microsoft.IdentityModel.Tokens.TokenValidationParameters
    {
        ValidateIssuer = false,
        ValidateAudience = false,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(
            System.Text.Encoding.UTF8.GetBytes("YourSuperSecretKey1234567890!@#$%^&*()"))
    };
});
builder.Services.AddAuthorization();

// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

- **Changes**:
  - `IRabbitMQProducer` ki registration hata di.
  - `IEmailService` aur `EmailService` register kiya.

#### **Step 2.2: Update `RoleRequestController`**
Ab `RoleRequestController` mein `IRabbitMQProducer` ko hata ke `IEmailService` inject karenge, aur email sending logic ko update karenge.

**File: `EShoppingZone.Controllers/RoleRequestController.cs`**
```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;
using EShoppingZone.Models;
using EShoppingZone.Services;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

namespace EShoppingZone.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class RoleRequestController : ControllerBase
    {
        private readonly IRepository _repository;
        private readonly IEmailService _emailService; // Changed to IEmailService
        private readonly UserManager<UserProfile> _userManager;
        private readonly EShoppingZoneDbContext _context;

        public RoleRequestController(
            IRepository repository,
            IEmailService emailService, // Changed to IEmailService
            UserManager<UserProfile> userManager,
            EShoppingZoneDbContext context)
        {
            _repository = repository;
            _emailService = emailService;
            _userManager = userManager;
            _context = context;
        }

        [HttpPost("SubmitRoleRequest")]
        [Authorize(Roles = "Customer")]
        public async Task<IActionResult> SubmitRoleRequest([FromBody] RoleRequestDTO request)
        {
            if (request.RequestedRole != "Merchant" && request.RequestedRole != "DeliveryAgent")
            {
                return BadRequest(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "Invalid role. Only Merchant or DeliveryAgent allowed.",
                });
            }

            var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            if (string.IsNullOrEmpty(userId))
            {
                return Unauthorized(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "User not authenticated.",
                });
            }

            var existingRequest = await _repository.GetRoleRequestByUserIdAsync(userId);
            if (existingRequest != null)
            {
                return BadRequest(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "You already have a pending role request.",
                });
            }

            var roleRequest = new RoleRequest
            {
                UserId = userId,
                RequestedRole = request.RequestedRole,
                Status = "Pending",
                RequestedAt = DateTime.UtcNow,
            };

            await _repository.AddRoleRequestAsync(roleRequest);

            // Send email directly using EmailService
            await _emailService.SendEmailAsync(
                User.Identity.Name,
                $"Your application to become a {request.RequestedRole} has been accepted. We will review it and get back to you soon.",
                "Role Request Submitted"
            );

            return Ok(new ResponseDTO<string>
            {
                Success = true,
                Message = "Role request submitted successfully.",
            });
        }

        [HttpGet("PendingRequests")]
        [Authorize(Roles = "Admin")]
        public async Task<IActionResult> GetPendingRequests()
        {
            var requests = await _repository.GetPendingRoleRequestsAsync();
            return Ok(new ResponseDTO<List<RoleRequest>>
            {
                Success = true,
                Message = "Pending role requests retrieved successfully.",
                Data = requests
            });
        }

        [HttpPost("ReviewRequest")]
        [Authorize(Roles = "Admin")]
        public async Task<IActionResult> ReviewRequest([FromBody] RoleRequestResponseDTO response)
        {
            if (response.Status != "Approved" && response.Status != "Rejected")
            {
                return BadRequest(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "Invalid status. Status must be 'Approved' or 'Rejected'.",
                });
            }

            var roleRequest = await _context.RoleRequests
                .Include(rr => rr.User)
                .FirstOrDefaultAsync(rr => rr.Id == response.RequestId);
            if (roleRequest == null)
            {
                return NotFound(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "Role request not found.",
                });
            }

            roleRequest.Status = response.Status;
            roleRequest.ReviewedAt = DateTime.UtcNow;
            await _repository.UpdateRoleRequestAsync(roleRequest);

            if (response.Status == "Approved")
            {
                // Remove existing roles and add the new role
                var user = roleRequest.User;
                var existingRoles = await _userManager.GetRolesAsync(user);
                await _userManager.RemoveFromRolesAsync(user, existingRoles);
                await _userManager.AddToRoleAsync(user, roleRequest.RequestedRole);

                // Send approval email
                await _emailService.SendEmailAsync(
                    user.Email,
                    $"Congratulations! Your request to become a {roleRequest.RequestedRole} has been approved!",
                    "Role Request Approved"
                );
            }
            else
            {
                // Send rejection email
                await _emailService.SendEmailAsync(
                    roleRequest.User.Email,
                    "We regret to inform you that your request to become a " +
                    $"{roleRequest.RequestedRole} has been rejected. " +
                    "Currently, we are looking to expand our team in other areas. " +
                    "Thank you for your interest!",
                    "Role Request Rejected"
                );
            }

            return Ok(new ResponseDTO<string>
            {
                Success = true,
                Message = $"Role request {response.Status.ToLower()} successfully.",
            });
        }
    }
}
```

- **Changes**:
  - `IRabbitMQProducer` ko hata ke `IEmailService` inject kiya.
  - `RabbitMQProducer.PublishMessage` calls ko hata ke `IEmailService.SendEmailAsync` calls ke saath replace kiya.
  - Email content same rakha jaise mentor ne bola tha.

#### **Step 2.3: Configure SMTP Settings**
`EmailService` ko SMTP settings ki zarurat hai, jo `appsettings.json` se aayengi.

**File: `EShoppingZone/appsettings.json`**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=EShoppingZone;Trusted_Connection=True;"
  },
  "Smtp": {
    "Host": "smtp.gmail.com",
    "Port": "587",
    "Username": "your-email@gmail.com",
    "Password": "your-app-password",
    "FromEmail": "your-email@gmail.com",
    "FromName": "EShoppingZone Team"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

- **How to Get App Password for Gmail**:
  - Agar Gmail use kar rahe ho, toh `Password` mein App Password use karna hoga (normal password nahi chalega due to Gmail's security).
  - Steps:
    1. Gmail account mein jao.
    2. Two-factor authentication enable karo.
    3. Google Account settings mein "App Passwords" search karo.
    4. Ek naya App Password generate karo (e.g., name it "EShoppingZone").
    5. Woh password `appsettings.json` mein `Smtp:Password` mein daal do.

- **Other SMTP Providers**:
  - Agar Gmail nahi use karna chahte, toh koi aur SMTP provider (jaise SendGrid, Outlook) use kar sakte ho. Bas `Host`, `Port`, `Username`, aur `Password` change kar do.

#### **Step 2.4: Update `IEmailService` Interface**
Ensure karte hain ki `IEmailService` interface define hai.

**File: `EShoppingZone.Interfaces/IEmailService.cs`**
```csharp
namespace EShoppingZone.Interfaces
{
    public interface IEmailService
    {
        Task SendEmailAsync(string email, string body, string subject);
    }
}
```

- Yeh already hai, bas confirm kiya.

---

### **Step 3: Test the Changes**

Ab backend mein changes ho gaye hain, toh test karte hain:

1. **Run the Backend**:
   ```bash
   dotnet run
   ```

2. **Test Role Request Submission**:
   - Ek `Customer` user ke saath login karo.
   - `POST /api/RoleRequest/SubmitRoleRequest` call karo:
     ```json
     {
       "RequestedRole": "Merchant"
     }
     ```
   - Check karo ki email aaya ya nahi ("Your application to become a Merchant has been accepted...").

3. **Test Admin Approval/Rejection**:
   - `Admin` user ke saath login karo.
   - `GET /api/RoleRequest/PendingRequests` se pending requests dekho.
   - `POST /api/RoleRequest/ReviewRequest` call karo:
     ```json
     {
       "RequestId": 1,
       "Status": "Approved"
     }
     ```
     - Check karo ki approval email aaya ("Congratulations! Your request to become a Merchant has been approved!").
     - User ka role `Merchant` mein update hua ya nahi, check karo.
   - Doosra test `Status: "Rejected"` ke saath karo aur rejection email check karo.

4. **Error Handling**:
   - Agar email send nahi ho raha, toh check karo:
     - `appsettings.json` mein SMTP settings sahi hain ya nahi.
     - Network issues ya firewall SMTP port (587) block toh nahi kar raha.

---

