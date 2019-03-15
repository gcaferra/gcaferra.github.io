## Mixing Web Forms and Web Api2 ##

I need to update a Legacy Web Forms application and I don't wnat to use anymore Update Panels.
I've searched different options and my favourite choice was mix Web Api and Web form following this guide [Hands On Lab: One ASP.NET: Integrating ASP.NET Web Forms, MVC and Web API](https://docs.microsoft.com/en-us/aspnet/visual-studio/overview/2013/one-aspnet-integrating-aspnet-web-forms-mvc-and-web-api#Exercise1).

I wrote this code:


> Code from Startup.cs
```C#
 app.UseCookieAuthentication(new CookieAuthenticationOptions
          {
            AuthenticationType = DefaultAuthenticationTypes.ApplicationCookie,
            CookieName = "myapplication.cookiename",
            LoginPath = new PathString("/login.aspx"),
            CookieSecure =  CookieSecureOption.SameAsRequest,
            ExpireTimeSpan = TimeSpan.FromMinutes(30),
            SlidingExpiration = true
          });
```
> Loginpage.aspx

```C#
    var authenticationManager = HttpContext.Current.GetOwinContext().Authentication;
    var claimsIdentity = new ClaimsIdentity(DefaultAuthenticationTypes.ApplicationCookie,
    ClaimsIdentity.DefaultNameClaimType, ClaimsIdentity.DefaultRoleClaimType);
 ... // Adding Claims

authenticationManager.SignIn(new AuthenticationProperties {IsPersistent = false}, claimsIdentity);

```
> Logout.aspx
```C#
    var authenticationManager = HttpContext.Current.GetOwinContext().Authentication;
      authenticationManager.SignOut();
    Session.Abandon();
    Response.Redirect("~/");
```

Initialy all was working well but I found a strange behaviour after IE and Chrome won't log in.

I start to serch and I find several results and also this [StackOverflow Answer](https://stackoverflow.com/questions/20737578/asp-net-sessionid-owin-cookies-do-not-send-to-browser/35578625) that talk about my issue but unfortunately anything help my problem.

The problem is randomly happening and some time once I deploy e new version seems work for a while and after few hours the problems come back.

Seems to be a problem with the session.

Options tried whitout success until now:
 
 * Update all projects to Framework 4.7.2
 * Setting unique name to the cookie
 * changing the usage of Session

Tht Only workaround found is be sure the session is started before login with a simple call to the session :

```C#
      Session["WorkaroundForOwinLogin"] = true;
```
then the previous code become like this:

```C#
    var authenticationManager = HttpContext.Current.GetOwinContext().Authentication;
    var claimsIdentity = new ClaimsIdentity(DefaultAuthenticationTypes.ApplicationCookie,
    ClaimsIdentity.DefaultNameClaimType, ClaimsIdentity.DefaultRoleClaimType);
 ... // Adding Claims
  Session["WorkaroundForOwinLogin"] = true;
  authenticationManager.SignIn(new AuthenticationProperties {IsPersistent = false}, claimsIdentity);

```


and reconfigure the OWIN CookieManager adding this in CookieAuthenticationOptions:
```C#
     CookieManager = new SystemWebCookieManager()
```
then the startup file will look like this:

```C#
 app.UseCookieAuthentication(new CookieAuthenticationOptions
          {
            AuthenticationType = DefaultAuthenticationTypes.ApplicationCookie,
            CookieName = "myapplication.cookiename",
            LoginPath = new PathString("/login.aspx"),
            CookieSecure =  CookieSecureOption.SameAsRequest,
            ExpireTimeSpan = TimeSpan.FromMinutes(30),
            SlidingExpiration = true,
            CookieManager = new SystemWebCookieManager()
          });
```

this combination of two line of code seems to work.

I hate the session but this legacy project rely on that and I cant disable it completely. If possibile disable the session as first option should solve this tedious problem that took me 3 days before I've found a solution.
