# Identityserver4 and 2FA

How to use IdentityServer4 including Asp.net Identity MVC and 2FA (2 factor authentication) to protect the API 


### Step by Step
#### [Add IdentityServer4 Template](https://github.com/IdentityServer/IdentityServer4.Templates)
1. Create an ASP.NET Core empty project. Name it `IdentityServer42FA`
2. Add local git repository
3. Add Visual Studio .gitignore
4. Add IdentityServer4 template 

`IdentityServer42FA> dotnet new is4aspid --force`

5. Do not seed the data yet
6. Change the database provider to SqlServer  
- Change the default connection to SqlServer

```json
{
  "ConnectionStrings": {
    // "DefaultConnection": "Data Source=AspIdUsers.db;",
    "DefaultConnection": "Integrated Security=SSPI;Persist Security Info=False;Initial Catalog=IdentityServer42FA;Data Source=.\\sqlexpress"
  }
}
```

- Install Microsoft.EntityFrameworkCore.SqlServer
- Unistall Microsoft.EntityFrameworkCore.Sqlite
- Change Startup.cs to use SqlServer instead of SqlLite
- Change SeedData.cs to use SqlServer instead of SqlLite 
11. Delete existing Migrations 
12. Build the project
13. Open Package Manager Console
14. Add Initial migration

```PM> Add-Migration -Name InitialCreate```

15. Update the database 

`PM> Update-Database`

16. Open the Developer Command Prompt and run `.\IdentityServer42FA.exe /seed` to seed the data
17. Commit your changes

#### [Scaffold Identity into an MVC project with authorization](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/scaffold-identity?view=aspnetcore-3.1&tabs=visual-studio#scaffold-identity-into-an-mvc-project-with-authorization)

1. Open Visual Studio
2. Create a new Asp.net Core 2.0 MVC web application with Authentication: Individual Accounts. Name it `MVC2FA`.
- Name it with the same name than the previous project and place it in another folder to avoid namespace conflicts when copy/ paste
3. Copy the folowing folders and its files from MVC2FA project to IdentityServer42FA
- Views 
- Model
- Services
- Extensions
4. Copy from MVC2FA Controllers/ManageController to IdentityServer42FA QuickStart/Manage/ManageController 
5. Copy/ Paste the missing methods from MVC2FA Controllers/AccountController to IdentityServer42FA Account/AccountController
6. Rebuild the project and solve the conflicts

#### [Migrating to Bootstrap v4](https://getbootstrap.com/docs/4.0/migration/)

1. Add Twitter Bootstrap client side library
2. Reference bootstrap bundle javascript file in _Layout.cshtml
3. Reference bootstrap css in _Layout.cshtml
3. Migrate markup to bootstrap 4

#### Migrating jQuery
1. Upgrade Client side libraries
- jquery-validate
- jquery-validation-unobtrusive
2. Add reference to jquery in IdentityServer42FA\Views\Shared\_Layout.cshtml
2. Update references to jquery in _ValidationScriptsPartial.cshtml

#### [Adding QR Codes to the 2FA configuration page](https://docs.microsoft.com/en-us/aspnet/core/security/authentication/identity-enable-qrcodes?view=aspnetcore-3.1#adding-qr-codes-to-the-2fa-configuration-page)
1. Download the qrcode.js javascript library to the wwwroot\lib folder in your project using Client Side Library Manager (libman.json)
2. Update IdentityServer42FA/Views/Manage/EnableAuthenticator.cshtml according to the article above.

```html
@section Scripts {
    @await Html.PartialAsync("_ValidationScriptsPartial")

    <script type="text/javascript" src="~/lib/qrcode.js"></script>

    <environment include="Development">
        <script src="~/lib/qrcodejs/qrcode.js"></script>
    </environment>
    <environment exclude="Development">
        <script src="~/lib/qrcodejs/qrcode.min.js"></script>
    </environment>

    <script type="text/javascript">
        new QRCode(document.getElementById("qrCode"),
            {
                text: "@Html.Raw(Model.AuthenticatorUri)",
                width: 150,
                height: 150
            });
    </script>
}
```

#### Adding support for login with 2FA
1. Update the http post Login method in IdentityServer42FA\Quickstart\Account\AccountController.cs class with code similar to the following:

```C#
        [HttpPost]
        [ValidateAntiForgeryToken]
        public async Task<IActionResult> Login(LoginInputModel model, string button)
        {
            // check if we are in the context of an authorization request
            var context = await _interaction.GetAuthorizationContextAsync(model.ReturnUrl);

            // the user clicked the "cancel" button
            if (button != "login")
            {
                if (context != null)
                {
                    // if the user cancels, send a result back into IdentityServer as if they
                    // denied the consent (even if this client does not require consent).
                    // this will send back an access denied OIDC error response to the client.
                    await _interaction.DenyAuthorizationAsync(context, AuthorizationError.AccessDenied);

                    // we can trust model.ReturnUrl since GetAuthorizationContextAsync returned non-null
                    if (context.IsNativeClient())
                    {
                        // The client is native, so this change in how to
                        // return the response is for better UX for the end user.
                        return this.LoadingPage("Redirect", model.ReturnUrl);
                    }

                    return Redirect(model.ReturnUrl);
                }
                else
                {
                    // since we don't have a valid context, then we just go back to the home page
                    return Redirect("~/");
                }
            }

            if (ModelState.IsValid)
            {
                var result = await _signInManager.PasswordSignInAsync(model.Username, model.Password, model.RememberLogin, lockoutOnFailure: true);
                if (result.Succeeded)
                {
                    var user = await _userManager.FindByNameAsync(model.Username);
                    await _events.RaiseAsync(new UserLoginSuccessEvent(user.UserName, user.Id, user.UserName, clientId: context?.Client.ClientId));

                    if (context != null)
                    {
                        if (context.IsNativeClient())
                        {
                            // The client is native, so this change in how to
                            // return the response is for better UX for the end user.
                            return this.LoadingPage("Redirect", model.ReturnUrl);
                        }

                        // we can trust model.ReturnUrl since GetAuthorizationContextAsync returned non-null
                        return Redirect(model.ReturnUrl);
                    }

                    // request for a local page
                    if (Url.IsLocalUrl(model.ReturnUrl))
                    {
                        return Redirect(model.ReturnUrl);
                    }
                    else if (string.IsNullOrEmpty(model.ReturnUrl))
                    {
                        return Redirect("~/");
                    }
                    else
                    {
                        // user might have clicked on a malicious link - should be logged
                        throw new Exception("invalid return URL");
                    }
                }

                await _events.RaiseAsync(new UserLoginFailureEvent(model.Username, "invalid credentials", clientId: context?.Client.ClientId));
                ModelState.AddModelError(string.Empty, AccountOptions.InvalidCredentialsErrorMessage);

                if (result.RequiresTwoFactor)
                {
                    return RedirectToAction(nameof(LoginWith2fa), new { model.ReturnUrl, model.RememberLogin });
                }

                if (result.IsLockedOut)
                {
                    _logger.LogWarning("User account locked out.");
                    return RedirectToAction(nameof(Lockout));
                }
            }

            // something went wrong, show form with error
            var vm = await BuildLoginViewModelAsync(model);
            return View(vm);
        }
```


2. Update http post LoginWith2fa method in IdentityServer42FA\Quickstart\Account\AccountController.cs class with code similar to the following:
```C#
        [HttpPost]
        [AllowAnonymous]
        [ValidateAntiForgeryToken]
        public async Task<IActionResult> LoginWith2fa(LoginWith2faViewModel model, string button, bool rememberMe, string returnUrl = null)
        {
            // check if we are in the context of an authorization request
            var context = await _interaction.GetAuthorizationContextAsync(returnUrl);

            // the user clicked the "cancel" button
            if (button != "login")
            {
                if (context != null)
                {
                    // if the user cancels, send a result back into IdentityServer as if they
                    // denied the consent (even if this client does not require consent).
                    // this will send back an access denied OIDC error response to the client.
                    await _interaction.DenyAuthorizationAsync(context, AuthorizationError.AccessDenied);

                    // we can trust model.ReturnUrl since GetAuthorizationContextAsync returned non-null
                    if (context.IsNativeClient())
                    {
                        // The client is native, so this change in how to
                        // return the response is for better UX for the end user.
                        return this.LoadingPage("Redirect", returnUrl);
                    }

                    return Redirect(returnUrl);
                }
                else
                {
                    // since we don't have a valid context, then we just go back to the home page
                    return Redirect("~/");
                }
            }

            if (!ModelState.IsValid)
            {
                return View(model);
            }

            var user = await _signInManager.GetTwoFactorAuthenticationUserAsync();
            if (user == null)
            {
                throw new ApplicationException($"Unable to load user with ID '{_userManager.GetUserId(User)}'.");
            }

            var authenticatorCode = model.TwoFactorCode.Replace(" ", string.Empty).Replace("-", string.Empty);

            var result = await _signInManager.TwoFactorAuthenticatorSignInAsync(authenticatorCode, rememberMe, model.RememberMachine);

            if (result.Succeeded)
            {
                _logger.LogInformation("User with ID {UserId} logged in with 2fa.", user.Id);

                // var user = await _userManager.FindByNameAsync(model.Username);
                await _events.RaiseAsync(new UserLoginSuccessEvent(user.UserName, user.Id, user.UserName, clientId: context?.Client.ClientId));

                if (context != null)
                {
                    if (context.IsNativeClient())
                    {
                        // The client is native, so this change in how to
                        // return the response is for better UX for the end user.
                        return this.LoadingPage("Redirect", returnUrl);
                    }

                    // we can trust returnUrl since GetAuthorizationContextAsync returned non-null
                    return Redirect(returnUrl);
                }

                // request for a local page
                if (Url.IsLocalUrl(returnUrl))
                {
                    return Redirect(returnUrl);
                }
                else if (string.IsNullOrEmpty(returnUrl))
                {
                    return Redirect("~/");
                }
                else
                {
                    // user might have clicked on a malicious link - should be logged
                    throw new Exception("invalid return URL");
                }

                return RedirectToLocal(returnUrl);
            }
            else if (result.IsLockedOut)
            {
                _logger.LogWarning("User with ID {UserId} account locked out.", user.Id);
                return RedirectToAction(nameof(Lockout));
            }
            else
            {
                _logger.LogWarning("Invalid authenticator code entered for user with ID {UserId}.", user.Id);
                ModelState.AddModelError(string.Empty, "Invalid authenticator code.");
                return View();
            }
        }

```

3. Update IdentityServer42FA\Views\Account\LoginWith2fa.cshtml view with markup similar to the following

```html
@model IdentityServer42FA.Models.AccountViewModels.LoginWith2faViewModel
@{
    ViewData["Title"] = "Two-factor authentication";
}

<h2>@ViewData["Title"]</h2>
<hr />
<p>Your login is protected with an authenticator app. Enter your authenticator code below.</p>
<div class="row">
    <div class="col-md-4">
        <form method="post" asp-route-returnUrl="@ViewData["ReturnUrl"]">
            <input asp-for="RememberMe" type="hidden" />
            <div asp-validation-summary="All" class="text-danger"></div>
            <div class="form-group">
                <label asp-for="TwoFactorCode"></label>
                <input asp-for="TwoFactorCode" class="form-control" autocomplete="off" />
                <span asp-validation-for="TwoFactorCode" class="text-danger"></span>
            </div>
            <div class="form-group">
                <div class="checkbox">
                    <label asp-for="RememberMachine">
                        <input asp-for="RememberMachine" />
                        @Html.DisplayNameFor(m => m.RememberMachine)
                    </label>
                </div>
            </div>
            <div class="form-group">
                <button type="submit" class="btn btn-secondary" name="button" value="login">Log in</button>
            </div>
        </form>
    </div>
</div>
<p>
    Don't have access to your authenticator device? You can
    <a asp-action="LoginWithRecoveryCode" asp-route-returnUrl="@ViewData["ReturnUrl"]">log in with a recovery code</a>.
</p>

@section Scripts {
    @await Html.PartialAsync("_ValidationScriptsPartial")
}
```


#### Check IdentityServer42FA using a JavaScript client.
1. Clone or download jsOidc sample [JsOidc](https://github.com/IdentityServer/IdentityServer4/tree/main/samples/Clients/src/JsOidc) 
2. Add js_oidc client to IdentityServer42FA allowed clients with code similar to the following:

```c#
        public static IEnumerable<Client> Clients =>
            new Client[]
            {

                new Client
                {
                    ClientId = "js_oidc",
                    ClientSecrets = { new Secret("8DBE4132-387F-41FC-9596-3D3BB76CB6A3".Sha256()) },
                    RequireClientSecret = false, // browser based applications can’t be trusted to securely keep the secret

                    AllowedGrantTypes = GrantTypes.Code,

                    RedirectUris = { "https://localhost:44300/callback.html", "https://localhost:44300/popup.html" },
                    PostLogoutRedirectUris = { "https://localhost:44300/index.html" },

                    AllowOfflineAccess = true,
                    AllowedScopes = { "openid", "profile", "email", "resource1.scope1", "resource2.scope1" },
                },
            };

```

3. Add js_oidc uris to IdentityServer42FA default cors policy

```c#
public void ConfigureServices(IServiceCollection services)
        {
            services.AddSingleton<ICorsPolicyService>((container) =>
            {
                var logger = container.GetRequiredService<ILogger<DefaultCorsPolicyService>>();
                var cors = new DefaultCorsPolicyService(logger)
                {
                    AllowedOrigins = { "https://localhost:44300" }
                };
                return cors;
            });

            services.AddControllersWithViews();

            // ...
        }
```


#### TODO: 
- [Create an Anglar Client](https://code-maze.com/angular-oauth2-oidc-configuration-identityserver4/)
- [Angular OAuth2 OIDC Configuration with IdentityServer4](https://offering.solutions/blog/articles/2020/05/18/authentication-and-authorization-with-angular-and-asp.net-core-using-oidc-and-oauth2/)
