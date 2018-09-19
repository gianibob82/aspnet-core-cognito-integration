# aspnet-core-cognito-integration
Cognito OpenId Connect integration in aspnet core razor project. This is all you need to integrate Cognito in your application as OpenId Connect so that you'll end up using Cognito UI pages. 

Use AspNetCore.DataProtection.Aws.S3 to store the machine keys.

## StartUp.cs
### In ConfigureServices
```
services.AddDataProtection()
.PersistKeysToAwsS3(
    new AmazonS3Client(Configuration.GetValue<string>("AWS:AccessKey"), 
        Configuration.GetValue<string>("AWS:SecretKey"),
        awsOptions.Region),
        
    new S3XmlRepositoryConfig("bucketname")
    // Configuration has defaults; all below are OPTIONAL
    {
        // How many concurrent connections will be made to S3 to retrieve key data
        MaxS3QueryConcurrency = 10,
        // Custom prefix in the S3 bucket enabling use of folders
        KeyPrefix = "MyKeys/",
        // Customise storage class for key storage
        StorageClass = S3StorageClass.OneZoneInfrequentAccess,
        //// Customise encryption options (these can be mutually exclusive - don't just copy & paste!)
        //ServerSideEncryptionMethod = ServerSideEncryptionMethod.AES256,
        //ServerSideEncryptionCustomerMethod = ServerSideEncryptionCustomerMethod.AES256,
        //ServerSideEncryptionCustomerProvidedKey = "MyBase64Key",
        // ServerSideEncryptionCustomerProvidedKeyMd5 = "MD5OfMyBase64Key",
        //ServerSideEncryptionKeyManagementServiceKeyId = "AwsKeyManagementServiceId",
        //// Compress stored XML before write to S3
        //ClientSideCompression = true,
        //// Validate downloads
        //ValidateETag = false,
        //ValidateMd5Metadata = true
    }
);
    
services.AddAuthentication(options =>
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
});
```
Here's the callback method:
```
private Task OnRedirectToIdentityProviderForSignOut(RedirectContext context)
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
}
```

### Signout controller
Add the following code to a controller, for example: "/account/signout".
```[HttpGet]
public IActionResult SignOut()
{
    var callbackUrl = Url.Page("/SignedOut", pageHandler: null, values: null, protocol: Request.Scheme);
    return SignOut(
        new AuthenticationProperties { RedirectUri = callbackUrl },
        CookieAuthenticationDefaults.AuthenticationScheme, OpenIdConnectDefaults.AuthenticationScheme
    );
}
```

### Protected page
Below is a protected page with an example on how to get the users claims.
```
[Authorize]
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
}
```

### appsettings.json
Finally, add this to your `appsettings.json`.
```
"ICognitoConfig":
{
    "ClientId": "",
    "ClientSecret": "",
    "LogoutUrl": "https://{your_client_url}/logout",
    "LogoutRelPath": "/signedout"
}
```
