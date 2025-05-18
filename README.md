Bhai, theek hai! 😊 Abhi tak humne saare issues fix kar liye hain jo tune bataye: `SubmitRoleRequest` ke validation issue ("parameters can't be null"), email sending ka problem, aur `PendingRequests` endpoint ka "an unexpected error occurred" issue. Chuki tu Swagger pe test kar raha hai aur frontend abhi nahi banaya, main ab role request feature ke saare updated files doonga jo backend ke liye relevant hain. Yeh files EShoppingZone project ke backend ke hain (.NET 8 Web API with ASP.NET Core Identity), aur `UserId` `int` type ka hai jo tune migration ke time fix kiya tha.

---

### **Updated Files for Role Request Feature**

#### **DTOs**
##### **File 1: `RoleRequestDTO.cs`**
**Path: `EShoppingZone.Models/RoleRequestDTO.cs`**  
(Validation aur Swagger compatibility ke liye `[Required]` add kiya.)
```csharp
using System.ComponentModel.DataAnnotations;

namespace EShoppingZone.Models
{
    public class RoleRequestDTO
    {
        [Required(ErrorMessage = "RequestedRole is required")]
        public string RequestedRole { get; set; } = string.Empty; // Merchant or DeliveryAgent
    }
}
```

##### **File 2: `RoleRequestResponseDTO.cs`**
**Path: `EShoppingZone.Models/RoleRequestResponseDTO.cs`**  
(Admin ke approve/reject ke liye DTO.)
```csharp
using System.ComponentModel.DataAnnotations;

namespace EShoppingZone.Models
{
    public class RoleRequestResponseDTO
    {
        [Required(ErrorMessage = "RequestId is required")]
        public int RequestId { get; set; }

        [Required(ErrorMessage = "Status is required")]
        public string Status { get; set; } = string.Empty; // Approved or Rejected
    }
}
```

#### **Controller**
##### **File 3: `RoleRequestController.cs`**
**Path: `EShoppingZone.Controllers/RoleRequestController.cs`**  
(Updated validation, error handling, aur email sending ke liye try-catch.)
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
        private readonly IEmailService _emailService;
        private readonly UserManager<UserProfile> _userManager;
        private readonly EShoppingZoneDbContext _context;

        public RoleRequestController(
            IRepository repository,
            IEmailService emailService,
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
            // Validation for allowed roles
            if (request.RequestedRole != "Merchant" && request.RequestedRole != "DeliveryAgent")
            {
                return BadRequest(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "Invalid role. Only Merchant or DeliveryAgent allowed.",
                });
            }

            var userIdString = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            if (string.IsNullOrEmpty(userIdString) || !int.TryParse(userIdString, out int userId))
            {
                return Unauthorized(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "User not authenticated or invalid user ID.",
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

            try
            {
                await _emailService.SendEmailAsync(
                    User.Identity.Name,
                    $"Your application to become a {request.RequestedRole} has been accepted. We will review it and get back to you soon.",
                    "Role Request Submitted"
                );
            }
            catch (Exception ex)
            {
                return Ok(new ResponseDTO<string>
                {
                    Success = true,
                    Message = "Role request submitted successfully, but failed to send email notification: " + ex.Message,
                });
            }

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
            try
            {
                var requests = await _context.RoleRequests
                    .Where(rr => rr.Status == "Pending")
                    .Select(rr => new RoleRequest
                    {
                        Id = rr.Id,
                        UserId = rr.UserId,
                        RequestedRole = rr.RequestedRole,
                        Status = rr.Status,
                        RequestedAt = rr.RequestedAt,
                        ReviewedAt = rr.ReviewedAt,
                        User = rr.User
                    })
                    .ToListAsync();

                return Ok(new ResponseDTO<List<RoleRequest>>
                {
                    Success = true,
                    Message = "Pending role requests retrieved successfully.",
                    Data = requests
                });
            }
            catch (Exception ex)
            {
                return StatusCode(500, new ResponseDTO<string>
                {
                    Success = false,
                    Message = $"An error occurred while retrieving pending requests: {ex.Message}",
                });
            }
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

            try
            {
                if (response.Status == "Approved")
                {
                    var user = roleRequest.User;
                    var existingRoles = await _userManager.GetRolesAsync(user);
                    await _userManager.RemoveFromRolesAsync(user, existingRoles);
                    await _userManager.AddToRoleAsync(user, roleRequest.RequestedRole);

                    await _emailService.SendEmailAsync(
                        user.Email,
                        $"Congratulations! Your request to become a {roleRequest.RequestedRole} has been approved!",
                        "Role Request Approved"
                    );
                }
                else
                {
                    await _emailService.SendEmailAsync(
                        roleRequest.User.Email,
                        "We regret to inform you that your request to become a " +
                        $"{roleRequest.RequestedRole} has been rejected. " +
                        "Currently, we are looking to expand our team in other areas. " +
                        "Thank you for your interest!",
                        "Role Request Rejected"
                    );
                }
            }
            catch (Exception ex)
            {
                return Ok(new ResponseDTO<string>
                {
                    Success = true,
                    Message = $"Role request {response.Status.ToLower()} successfully, but failed to send email notification: {ex.Message}",
                });
            }

            return Ok(new ResponseDTO<string>
            {
                Success = true,
                Message = $"Role request {response.Status.ToLower()} successfully.",
            });
        }

        [HttpGet("MyRequest")]
        [Authorize(Roles = "Customer")]
        public async Task<IActionResult> GetMyRoleRequest()
        {
            var userIdString = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
            if (string.IsNullOrEmpty(userIdString) || !int.TryParse(userIdString, out int userId))
            {
                return Unauthorized(new ResponseDTO<string>
                {
                    Success = false,
                    Message = "User not authenticated or invalid user ID.",
                });
            }

            var roleRequest = await _repository.GetRoleRequestByUserIdAsync(userId);
            if (roleRequest == null)
            {
                return Ok(new ResponseDTO<RoleRequest>
                {
                    Success = true,
                    Message = "No pending role request found.",
                    Data = null
                });
            }

            return Ok(new ResponseDTO<RoleRequest>
            {
                Success = true,
                Message = "Role request retrieved successfully.",
                Data = roleRequest
            });
        }
    }
}
```

#### **Repository**
##### **File 4: `IRepository.cs`**
**Path: `EShoppingZone.Services/IRepository.cs`**
```csharp
using EShoppingZone.Models;

namespace EShoppingZone.Services
{
    public interface IRepository
    {
        Task<UserProfile> GetUserByEmailAsync(string email);
        Task AddUserAsync(UserProfile user);
        Task AddRoleRequestAsync(RoleRequest roleRequest);
        Task<RoleRequest> GetRoleRequestByUserIdAsync(int userId);
        Task<List<RoleRequest>> GetPendingRoleRequestsAsync();
        Task UpdateRoleRequestAsync(RoleRequest roleRequest);
    }
}
```

##### **File 5: `Repository.cs`**
**Path: `EShoppingZone.Services/Repository.cs`**  
(`GetPendingRoleRequestsAsync` ko update kiya taaki `User` navigation property optional ho.)
```csharp
using Microsoft.EntityFrameworkCore;
using EShoppingZone.Data;
using EShoppingZone.Models;

namespace EShoppingZone.Services
{
    public class Repository : IRepository
    {
        private readonly EShoppingZoneDbContext _context;

        public Repository(EShoppingZoneDbContext context)
        {
            _context = context;
        }

        public async Task<UserProfile> GetUserByEmailAsync(string email)
        {
            return await _context.Profiles.FirstOrDefaultAsync(u => u.Email == email);
        }

        public async Task AddUserAsync(UserProfile user)
        {
            await _context.Profiles.AddAsync(user);
            await _context.SaveChangesAsync();
        }

        public async Task AddRoleRequestAsync(RoleRequest roleRequest)
        {
            await _context.RoleRequests.AddAsync(roleRequest);
            await _context.SaveChangesAsync();
        }

        public async Task<RoleRequest> GetRoleRequestByUserIdAsync(int userId)
        {
            return await _context.RoleRequests
                .FirstOrDefaultAsync(rr => rr.UserId == userId && rr.Status == "Pending");
        }

        public async Task<List<RoleRequest>> GetPendingRoleRequestsAsync()
        {
            return await _context.RoleRequests
                .Where(rr => rr.Status == "Pending")
                .Select(rr => new RoleRequest
                {
                    Id = rr.Id,
                    UserId = rr.UserId,
                    RequestedRole = rr.RequestedRole,
                    Status = rr.Status,
                    RequestedAt = rr.RequestedAt,
                    ReviewedAt = rr.ReviewedAt,
                    User = rr.User // This will be null if User doesn't exist
                })
                .ToListAsync();
        }

        public async Task UpdateRoleRequestAsync(RoleRequest roleRequest)
        {
            _context.RoleRequests.Update(roleRequest);
            await _context.SaveChangesAsync();
        }
    }
}
```

#### **Email Service**
##### **File 6: `EmailService.cs`**
**Path: `EShoppingZone.Services/EmailService.cs`**  
(Logging aur error handling ke saath updated.)
```csharp
using System;
using System.Net;
using System.Net.Mail;
using System.Threading.Tasks;
using EShoppingZone.Data;
using EShoppingZone.Interfaces;
using EShoppingZone.Models;
using Microsoft.AspNetCore.Identity;
using Microsoft.Extensions.Logging;

namespace EShoppingZone.Services
{
    public class EmailService : IEmailService 
    {
        private readonly UserManager<UserProfile> _userManager;
        private readonly SignInManager<UserProfile> _signInManager;
        private readonly EShoppingZoneDbContext _context;
        private readonly IConfiguration _configuration;
        private readonly ILogger<EmailService> _logger;

        public EmailService(
            UserManager<UserProfile> userManager,
            SignInManager<UserProfile> signInManager,
            EShoppingZoneDbContext context,
            IConfiguration configuration,
            ILogger<EmailService> logger)
        {
            _userManager = userManager;
            _signInManager = signInManager;
            _context = context;
            _configuration = configuration;
            _logger = logger;
        }

        public async Task SendEmailAsync(string email, string body, string subject)
        {
            try
            {
                var user = await _userManager.FindByEmailAsync(email);
                if (user == null)
                {
                    _logger.LogWarning($"User with email {email} not found.");
                    return;
                }

                var smtpClient = new SmtpClient(_configuration["Smtp:Host"])
                {
                    Port = int.Parse(_configuration["Smtp:Port"]),
                    Credentials = new NetworkCredential(_configuration["Smtp:Username"], _configuration["Smtp:Password"]),
                    EnableSsl = true
                };

                var mailMessage = new MailMessage
                {
                    From = new MailAddress(_configuration["Smtp:FromEmail"], _configuration["Smtp:FromName"]),
                    Subject = subject,
                    Body = body,
                    IsBodyHtml = true
                };
                mailMessage.To.Add(email);

                await smtpClient.SendMailAsync(mailMessage);
                _logger.LogInformation($"Email sent successfully to {email}. Subject: {subject}");
            }
            catch (Exception ex)
            {
                _logger.LogError($"Failed to send email to {email}. Error: {ex.Message}");
                throw;
            }
        }
    }
}
```

#### **Global Exception Handling Middleware**
##### **File 7: `GlobalExceptionMiddleware.cs`**
**Path: `EShoppingZone/Middleware/GlobalExceptionMiddleware.cs`**  
(Unexpected errors ke liye global exception handling.)
```csharp
using Microsoft.AspNetCore.Http;
using System.Text.Json;
using EShoppingZone.Models;

namespace EShoppingZone.Middleware
{
    public class GlobalExceptionMiddleware
    {
        private readonly RequestDelegate _next;

        public GlobalExceptionMiddleware(RequestDelegate next)
        {
            _next = next;
        }

        public async Task InvokeAsync(HttpContext context)
        {
            try
            {
                await _next(context);
            }
            catch (Exception ex)
            {
                await HandleExceptionAsync(context, ex);
            }
        }

        private Task HandleExceptionAsync(HttpContext context, Exception exception)
        {
            context.Response.ContentType = "application/json";
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;

            var response = new ResponseDTO<string>
            {
                Success = false,
                Message = $"An unexpected error occurred: {exception.Message}",
            };

            return context.Response.WriteAsync(JsonSerializer.Serialize(response));
        }
    }

    public static class GlobalExceptionMiddlewareExtensions
    {
        public static IApplicationBuilder UseGlobalExceptionMiddleware(this IApplicationBuilder builder)
        {
            return builder.UseMiddleware<GlobalExceptionMiddleware>();
        }
    }
}
```

#### **Program.cs Update**
##### **File 8: `Program.cs`**
**Path: `EShoppingZone/Program.cs`**  
(Global exception middleware aur logging add kiya.)
```csharp
using EShoppingZone.Data;
using EShoppingZone.Models;
using EShoppingZone.Services;
using EShoppingZone.Middleware;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllers();
builder.Services.AddDbContext<EShoppingZoneDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
builder.Services.AddIdentity<UserProfile, IdentityRole<int>, int>()
    .AddEntityFrameworkStores<EShoppingZoneDbContext>()
    .AddDefaultTokenProviders();
builder.Services.AddScoped<IAuthService, AuthService>();
builder.Services.AddScoped<IRepository, Repository>();
builder.Services.AddScoped<IEmailService, EmailService>();
builder.Services.AddLogging(logging =>
{
    logging.AddConsole();
    logging.SetMinimumLevel(LogLevel.Information);
});
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

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.UseGlobalExceptionMiddleware();
app.MapControllers();

app.Run();
```

#### **Configuration**
##### **File 9: `appsetti