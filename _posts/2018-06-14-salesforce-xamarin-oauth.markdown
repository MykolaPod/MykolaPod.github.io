---
layout: post
title:  "Xamarin Authentication as existing user via Salesforce Oauth"
date:   2018-06-14 07:25:54 +0300
categories: [Xamarin, iOS, Android]
tags: [Xamarin, iOS, Android]
---

Hello, here I am going to share high level approach of implementation Oatuh authentication via [https://www.salesforce.com/](Salesforce) (next SF) at xamarin app (iOS / Android).
Interesting in this task is the case when you have user's accounts in your app already and you need to let users login into app through theirs Salesforce account.

SF provided a Xamarin component called [https://components.xamarin.com/view/SalesforceSDK](SalesforceSDK 1.4.4.0) with source code on [https://github.com/xamarin/SalesforceSDK](https://github.com/xamarin/SalesforceSDK) however last published data of component September 29 of 2015 which is quite old and looks like yet another obsolete SDK.


`Official documentation provides an example of usage a component`:
{% highlight C# %}
// Creates our connection to salesforce.
var client = new SalesforceClient(clientId, clientSecret, redirectUrl);

// Get authenticated users from the local keystore
var users = client.LoadUsers();
if (!users.Any())
{
        client.AuthenticationComplete += (sender, e) => OnAuthenticationCompleted(e);
        // Starts the Salesforce login process.
        var loginUI = client.GetLoginInterface();
        DisplayThe(loginUI);
}
else
{
        // We're ready to fetch some data!
        // Let's grab some sales accounts to display.
        IEnumerable<SObject> results =
                    await client.ReadAsync("SELECT Name, AccountNumber FROM Account");
 
        DoSomethingAmazingWith(results);
}
{% endhighlight %}

And guess what? That won't work no metter how I tried.


Main problem is that `OnAuthenticationCompleted` handler was never triggered. Talking a little bit ahead  I may assume that it happens because of SF changed authentication flow on theirs web app and make component not compatible.

So I decided to make some hack.

<strong>Modify login page redirect</strong>

Firs of all lets modify default SF login page with a script that will redirect to our custom address. We need to make it in a way that will prevent redirecting to internal web site page after clicking Login button and go to custom url instead. You can do it at SF portal settings.

When you will do that, keep in mind that you will need to get an authentication code that is sending to SF by default at your app. So I suggest grab it and send as a query string parameter in a way how you will be able to parse it.

Cutom url should be unique, let's use a common url scheme for that:  `com.your.application`

<strong>Modify mobile apps</strong>

`iOS:`

First of all  let's add a custom url scheme. You can do it through IDE or manually at `info.plist`:
{% highlight XML %}
<key>CFBundleURLTypes</key>
<array>
	<dict>
		<key>CFBundleURLName</key>
		<string>oauth2redirect</string>
		<key>CFBundleURLSchemes</key>
		<array>
			<string>com.your.application</string>
		</array>
		<key>CFBundleURLIconFile</key>
		<string>Mobile_App_Icon58x58</string>
		<key>CFBundleURLTypes</key>
		<string>Viewer</string>
	</dict>
</array>
{% endhighlight %}
And add a handler for this scheme at `AppDelegate.cs`:
{% highlight C# %}
public override bool OpenUrl(UIApplication application, NSUrl url, string sourceApplication, NSObject annotation)
        {
            var decodedUrl = Uri.UnescapeDataString(url.AbsoluteString);
            var authenticationCode = SalesForceAuthenticateUrlParser.GetAuthCodeFromUrl(decodedUrl);
            var deviceId = UIDevice.CurrentDevice.IdentifierForVendor.AsString();
 
            var loginService = Mvx.Resolve<ILoginService>();
            loginService.AuthenticateWithSalesForceCode(deviceId, authenticationCode);
 
            return true;
        }
{% endhighlight %}
{% highlight C# %}
public static class SalesForceAuthenticateUrlParser
    {
        public static string GetAuthCodeFromUrl(string url)
        {
            var uriNetfx = new Uri(url);

            // Using regex for parsing auth code            
            var authenticationCode = Regex.Match(uriNetfx.Query, "(?<=code=)(.*)(==)").ToString();
            return authenticationCode;
        }
    }
{% endhighlight %}

As you can see I am using here some `ILoginService`. This is a shared (iOS and Android service that making some business logig related to login / logut):
{% highlight C#%}
 public async Task<LoginResult> GetLoginGuid(string deviceId, string authCode)
        {
                // Make an http call to server. Sending device id and a authtnication code received from SF.
            var result = await _webService.RunAsync(w => w.GetLoginGuid(deviceId, authCode));
            return await HandleLoginResponse(result);
        }

private async Task<LoginResult> HandleLoginResponse(ServiceResponse<UserInfo> result)
        {
            switch (result.ResponseCode)
            {
                case ResponseCode.NoInternetConnectionCode:
                    return LoginResult.NoConnection;
                case ResponseCode.UserDisabledCode:
                    return LoginResult.Disabled;
                case ResponseCode.BadRequestCode:
                case ResponseCode.ServerExceptionCode:
                    return LoginResult.ServerError;
                  default:
                      break;
            }
            var userInfo = result.ResultValue;

            if (userInfo.LoginGuid == default(Guid) || userInfo.ClientId == default(int))
            {
                return LoginResult.Deny;
            }

            /*  Save user information to settings so we can keep the user logged in*/

            _settingsService.LoginGuid = userInfo.LoginGuid;
            _settingsService.ClientId = userInfo.ClientId;

            var profileResult = await _webService.RunAsync(w => w.GetClientProfileAsync(userInfo.ClientId.ToString()));
            var profile = profileResult.ResultValue;

            switch (profileResult.ResponseCode)
            {
                case ResponseCode.UserDisabledCode:
                    return LoginResult.Disabled;
                case ResponseCode.ServerExceptionCode:
                    return LoginResult.ServerError;
                case ResponseCode.OkCode:
                    var clientName = profile.FullName;
                    _settingsService.CurrentUserName = clientName;                    
                    _settingsService.LoggedInEmailAdress = profile.Email;
                    _settingsService.NotificationEmailAddresses = profile.NotificationEmailAddresses;
                    break;
            }
            await _syncService.SyncProfiles();
            await _syncService.CheckCannedNotesAndSyncIfNeeded();
            _syncService.RunSyncPulse(TimeSpan.FromMinutes(1));
            return LoginResult.Success;
        }

public async Task AuthenticateWithSalesForceCode(string deviceId, string authCode)
        {
            var popupService = Mvx.Resolve<IPopupService>();
            var presenter = Mvx.Resolve<IMvxFormsPresenter>();

            var loginResult = await GetLoginGuid(deviceId, authCode);

            switch (loginResult)
            {
                case LoginResult.Success:
                    var azureNotificationHubService = Mvx.Resolve<IAzureNotificationHubService>();
                    azureNotificationHubService.SubscribeDevice(_settingsService.ClientId);
                    _userActivityService.Login(_settingsService.CurrentUserName, _settingsService.LoginGuid.ToString());
                    presenter.ChangePresentation(new MvxChangeRootViewModelPresentationHint(typeof(MainMasterDetailViewModel)));
                    break;
                case LoginResult.Deny:
                    await popupService.DisplayAlert(string.Empty, Resources.LoginViewModel_DoLoginCommand_DidNotRecognizeUserNameAndPassword, Resources.OKButtonText);
                    Logout();
                    break;
                case LoginResult.NoConnection:
                    await popupService.DisplayAlert(string.Empty, Resources.CouldNotReachServerMessage, Resources.OKButtonText);
                    Logout();
                    break;
                case LoginResult.Disabled:
                    await popupService.DisplayAlert(string.Empty, Resources.UserDisabledAlertText, Resources.OKButtonText);
                    Logout();
                    break;
                default:
                    await popupService.DisplayAlert(string.Empty, Resources.ServerError, Resources.OKButtonText);
                    Logout();
                    break;
            }

            _userActivityService.SimpleActionEvent(UserActivityActions.LoginResultHandled);
            _messenger.Publish(new LoginResponseHandledMessage(this));
            if (loginResult != LoginResult.Success)
            {
                presenter.ChangePresentation(new MvxChangeRootViewModelPresentationHint(typeof(SelectViewModel)));
            }

        }
{% endhighlight %}
 You do not need most of this stuff. I just put it here to give you a view what's under the hood.
  There you can find a following :`await _webService.RunAsync(w => w.GetLoginGuid(deviceId, authCode))` line - this will make a http call to our server-side application with parameters created during handling intercepred by our custom url shceme request. (See above `public override bool OpenUrl` and `public static string GetAuthCodeFromUrl`)

Finally we need to navigate at login page somehow. Let's add an `AuthService.cs` for that purpose:
{% highlight C# %}
public override void ShowLoginPage()
{         
            var authenticator = new OAuth2Authenticator(
                clientId: CurrentValues.SalesForceClientKey,
                clientSecret: CurrentValues.SalesForceClientSecret,
                scope: "full",
                authorizeUrl: new Uri(SalesForceAuthUrl), //https://com.your.application/registration/services/oauth2/authorize
                accessTokenUrl: new Uri(SalesForceTokenUrl), //https://com.your.application/registration/services/oauth2/token
                redirectUrl: GetTokenCallbackUri, //com.your.application:/oauth2redirect
                isUsingNativeUI: true
            )
            {
                ShowErrors = false,
                AllowCancel = false,
                ClearCookiesBeforeLogin = true
            };
            
            var uiVewController = authenticator.GetUI();
          
            RootViewController.PresentViewController(new UINavigationController(uiVewController), true, null);
            
}
{% endhighlight %}

`Android:`

For android devices it is doing in a similar way:

First of all let's add new `SalesForceActivity.cs` with IntentFilter which will register custom url handling for our app:
{% highlight C# %}
[Activity(Label = "SalesForceActivity")]
[IntentFilter
      (
            actions: new[] { Intent.ActionView },
            Categories = new[]
            {
                Intent.CategoryDefault,
                Intent.CategoryBrowsable
            },
            DataSchemes = new[]
            {
                "com.your.application",
            },
            DataPaths = new[]
            {
                "/oauth2redirect"
            },
            AutoVerify = false
      )
]
public class SalesForceActivity: Activity
{
        protected override void OnCreate(Bundle savedInstanceState)
        {
            base.OnCreate(savedInstanceState);
            global::Android.Net.Uri uri_android = Intent.Data;
            var decodedUriAndroid = WebUtility.UrlDecode(uri_android.ToString());
 
            var authenticationCode = SalesForceAuthenticateUrlParser.GetAuthCodeFromUrl(decodedUriAndroid);
            var deviceId = Settings.Secure.GetString(ContentResolver, Settings.Secure.AndroidId);
 
            var loginService = Mvx.Resolve<ILoginService>();
            loginService.AuthenticateWithSalesForceCode(deviceId, authenticationCode);
 
            Finish();
        }
}
{% endhighlight %}

<strong>Modify server</strong>


Actually depending of your needs, you can continue authentication with http calls from mobile client and receive a token. But for my case I was needed to match login accounc wiht one of exsisting user accounts and do log it as `exsiting user`. 

Because I need to make a login as one of existing user, I am doing here pretty muth same as for iOS client code - sending deviceId and just received authentication code to server for futher loggin process.

When server will receive a deviceId and a token, continue authorization and receive a token, get user details from SF using that token,  define a user from list of existing ones, create own token and send back to modile client.

<strong>Weakness</strong>

Weak point in this story is loggin out from the mobile client.
Current approach uses browser to get SF login page where make an first step authentication. And it save cookies automatically. After first time when you did a login it remember you and when you will try open login page again to log in as another user, it will not ask you for a password - but just redirect you. Cookies are clearing after some time automatically by a OS, or may be cleaned manually through browser settings. Anyway this is a next point of modernizing current solution. 

It looks like we finally finished this chapter. <u>Contact me if you have an idea how to improve this slution</u>, and 

Have a nice day !
