# aspnet-core-cognito-integration
Cognito OpenId Connect integration in aspnet core razor project. This is all you need to integrate Cognito in your application as OpenId Connect so that you'll end up using Cognito UI pages. 

Be aware: Aspnet core OpenIdConnect implementation is not working properly behind proxies and when deployed to AWS API Gateway.
As of today 28/05/2018 there's an open bug about this.

##StartUp.cs

### in ConfigureServices
`services.AddAuthentication(options =>
            {
                options.DefaultAuthenticateScheme = CookieAuthenticationDefaults.AuthenticationScheme;
                options.DefaultSignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
                options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
            })
            .AddCookie()
            .AddOpenIdConnect(options =>
            {
                options.ResponseType = "code";
                options.MetadataAddress = "https://cognito-idp.{region}.amazonaws.com/{cognito_pool_id}/.well-known/openid-configuration";
                options.ClientId = Configuration.GetValue<string>("ICognitoConfig:ClientId");
                options.ClientSecret = Configuration.GetValue<string>("ICognitoConfig:ClientSecret");
                // change to your first protected page
                options.CallbackPath = PathString.FromUriComponent("/index");
                options.Events = new OpenIdConnectEvents
                {
                    // this makes signout working
                    OnRedirectToIdentityProviderForSignOut = OnRedirectToIdentityProviderForSignOut
                };
            });`

`private Task OnRedirectToIdentityProviderForSignOut(RedirectContext context)
        {
            context.ProtocolMessage.Scope = "openid";
            context.ProtocolMessage.ResponseType = "code";

            var logoutEndpoint = Configuration.GetValue<string>("ICognitoConfig:LogoutEndpoint");
            var clientId = Configuration.GetValue<string>("ICognitoConfig:ClientId");

            var logoutUrl = $"{context.Request.Scheme}://{context.Request.Host}{Configuration.GetValue<string>("ICognitoConfig:LogoutRelPath")}";

            context.ProtocolMessage.IssuerAddress = $"{logoutEndpoint}?client_id={clientId}&logout_uri={logoutUrl}&redirect_uri={logoutUrl}";

            // delete cookies
            context.Properties.Items.Remove(CookieAuthenticationDefaults.AuthenticationScheme);
            // close openid session
            context.Properties.Items.Remove(OpenIdConnectDefaults.AuthenticationScheme);

            return Task.CompletedTask;
        }`
        
### Signout controller
add it to a controller, ex /account/signout
`[HttpGet]
        public IActionResult SignOut()
        {
                        var callbackUrl = Url.Page("/SignedOut", pageHandler: null, values: null, protocol: Request.Scheme);
            return SignOut(
                new AuthenticationProperties { RedirectUri = callbackUrl },
                CookieAuthenticationDefaults.AuthenticationScheme
            , OpenIdConnectDefaults.AuthenticationScheme
            );
        }`
### Protected page
With example how to get claims
`[Authorize]
    public class SecurePageModel : PageModel
    {
        public string AccountId
        {
            get
            {
                return User?.Claims.FirstOrDefault(c => c.Type.Equals("cognito:username"))?.Value;
            }
        }
        public string Nickname { get { return User?.Claims.FirstOrDefault(c => c.Type.Equals("nickname"))?.Value; } }

    }`
    
    ### appsettings.json
    
    "ICognitoConfig": {
    "ClientId": "",
    "ClientSecret": "",
    "LogoutUrl": "https://{your_client_url}/logout",
    "LogoutRelPath": "/signedout"
  }
