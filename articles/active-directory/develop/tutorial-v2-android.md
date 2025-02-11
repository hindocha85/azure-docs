---
title: Getting started with Android - Microsoft identity platform | Azure 
description: How an Android app can get an access token and call Microsoft Graph API or APIs that require access tokens from Microsoft identity platform.
services: active-directory
documentationcenter: dev-center-name
author: tylermsft
manager: CelesteDG

ms.service: active-directory
ms.subservice: develop
ms.devlang: na
ms.topic: tutorial
ms.tgt_pltfrm: na
ms.workload: identity
ms.date: 07/09/2019
ms.author: jmprieur
ms.reviwer: brandwe
ms.custom: aaddev 
ms.collection: M365-identity-device-management
---

# Sign in users and call the Microsoft Graph from an Android app

In this tutorial, you'll learn how to integrate an Android app with the Microsoft identity platform. Your app will sign in a user, get an access token to call the Microsoft Graph API, and make a request to the Microsoft Graph API.  

When you've completed the guide, your application will accept sign-ins of personal Microsoft accounts (including outlook.com, live.com, and others) and work or school accounts from any company or organization that uses Azure Active Directory.

## How this tutorial works

![Shows how the sample app generated by this tutorial works](../../../includes/media/active-directory-develop-guidedsetup-android-intro/android-intro.svg)

The app in this tutorial will sign in users and get data on their behalf.  This data will be accessed through a protected API (Microsoft Graph API) that requires authorization and is protected by the Microsoft identity platform.

More specifically:

* Your app will sign in the user either through a browser or the Microsoft Authenticator and Intune Company Portal.
* The end user will accept the permissions your application has requested.
* Your app will be issued an access token for the Microsoft Graph API.
* The access token will be included in the HTTP request to the web API.
* Process the Microsoft Graph response.

This sample uses the Microsoft Authentication library for Android (MSAL) to implement Authentication: [com.microsoft.identity.client](https://javadoc.io/doc/com.microsoft.identity.client/msal).

 MSAL will automatically renew tokens, deliver single sign-on (SSO) between other apps on the device, and manage the Account(s).

## Prerequisites

* This tutorial requires Android Studio version 16 or later (19+ is recommended).

## Create a project

This tutorial will create a new project. If you want to download the completed tutorial instead, [download the code](https://github.com/Azure-Samples/active-directory-android-native-v2/archive/master.zip).

1. Open Android Studio, and select **Start a new Android Studio project**.
2. Select **Basic Activity** and select **Next**.
3. Name your application.
4. Save the package name. You will enter it later into the Azure portal.
5. Set the **Minimum API level** to **API 19** or higher, and click **Finish**.
6. In the project view, choose **Project** in the dropdown to display source and non-source project files, open **app/build.gradle** and set `targetSdkVersion` to `27`.

## Register your application

1. Go to the [Azure portal](https://aka.ms/MobileAppReg).
2. Open the [App registrations blade](https://ms.portal.azure.com/?feature.broker=true#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/RegisteredAppsPreview) and click **+New registration**.
3. Enter a **Name** for your app and then, without setting a Redirect URI, click **Register**.
4. In the **Manage** section of the pane that appears, select **Authentication** > **+ Add a platform** > **Android**.
5. Enter your project's Package Name. If you downloaded the code, this value is `com.azuresamples.msalandroidapp`.
6. In the **Signature hash** section of the **Configure your Android app** page, click **Generating a development Signature Hash.** and copy the KeyTool command to use for your platform.

   > [!Note]
   > KeyTool.exe is installed as part of the Java Development Kit (JDK). You must also install the OpenSSL tool to execute the KeyTool command.

7. Enter the **Signature hash** generated by KeyTool.
8. Click `Configure` and save the **MSAL Configuration** that appears in the **Android configuration** page so you can enter it when you configure your app later.  Click **Done**.

## Build your app

### Add your app registration

1. In Android Studio's project pane, navigate to **app\src\main\res**.
2. Right-click **res** and choose **New** > **Directory**. Enter `raw` as the new directory name and click **OK**.
3. In **app** > **src** > **res** > **raw**, create a new JSON file called `auth_config.json` and paste the MSAL Configuration that you saved earlier. See [MSAL Configuration for more info](https://github.com/AzureAD/microsoft-authentication-library-for-android/wiki/Configuring-your-app).
4. In **app** > **src** > **main** > **AndroidManifest.xml**, add the `BrowserTabActivity` activity below. This entry allows Microsoft to call back to your application after it completes the authentication:

    ```xml
    <!--Intent filter to capture System Browser or Authenticator calling back to our app after sign-in-->
    <activity
        android:name="com.microsoft.identity.client.BrowserTabActivity">
        <intent-filter>
            <action android:name="android.intent.action.VIEW" />
            <category android:name="android.intent.category.DEFAULT" />
            <category android:name="android.intent.category.BROWSABLE" />
            <data android:scheme="msauth"
                android:host="Enter_the_Package_Name"
                android:path="/Enter_the_Signature_Hash" />
        </intent-filter>
    </activity>
    ```

    Substitute the package name you registered in the Azure portal for the `android:host=` value.
    Substitute the key hash you registered in the Azure portal for the `android:path=` value. The Signature Hash should not be URL encoded.

5. Inside the **AndroidManifest.xml**, just above the `<application>` tag, add the following permissions:

    ```xml
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
    ```

### Create the app's UI

1. In the Android Studio project window, navigate to **app** > **src** > **main** > **res** > **layout** and open **activity_main.xml** and open the **Text** view.
2. Change the activity layout, for example: `<androidx.coordinatorlayout.widget.CoordinatorLayout` to `<androidx.coordinatorlayout.widget.LinearLayout`.
3. Add the `android:orientation="vertical"` property to the `LinearLayout` node.
4. Paste the following code into the `LinearLayout` node, replacing the current content:

    ```xml
    <TextView
        android:text="Welcome, "
        android:textColor="#3f3f3f"
        android:textSize="50px"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginLeft="10dp"
        android:layout_marginTop="15dp"
        android:id="@+id/welcome"
        android:visibility="invisible"/>

    <Button
        android:id="@+id/callGraph"
        android:text="Call Microsoft Graph"
        android:textColor="#FFFFFF"
        android:background="#00a1f1"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginTop="200dp"
        android:textAllCaps="false" />

    <TextView
        android:text="Getting Graph Data..."
        android:textColor="#3f3f3f"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginLeft="5dp"
        android:id="@+id/graphData"
        android:visibility="invisible"/>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="0dip"
        android:layout_weight="1"
        android:gravity="center|bottom"
        android:orientation="vertical" >

        <Button
            android:text="Sign Out"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="15dp"
            android:textColor="#FFFFFF"
            android:background="#00a1f1"
            android:textAllCaps="false"
            android:id="@+id/clearCache"
            android:visibility="invisible" />
    </LinearLayout>
    ```

### Add MSAL to your project

1. In the Android Studio project window, navigate to **app** > **src** > **build.gradle**.
2. Under **Dependencies**, paste the following:

    ```gradle  
    implementation 'com.android.volley:volley:1.1.1'
    implementation 'com.microsoft.identity.client:msal:0.3.+'
    ```

### Use MSAL

Now make changes inside `MainActivity.java` to add and use MSAL in your app.
In the Android Studio project window, navigate to **app** > **src** > **main** > **java** > **com.example.msal**, and open `MainActivity.java`.

#### Required imports

Add the following imports near the top of `MainActivity.java`:

```java
import android.app.Activity;
import android.content.Intent;
import android.util.Log;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;
import android.widget.Toast;
import com.android.volley.*;
import com.android.volley.toolbox.JsonObjectRequest;
import com.android.volley.toolbox.Volley;
import org.json.JSONObject;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import com.microsoft.identity.client.*;
import com.microsoft.identity.client.exception.*;
```

#### Instantiate MSAL

Inside the `MainActivity` class, we'll need to instantiate MSAL along with a few configurations about what are app will do including the scopes and web API we want to access.

Copy the following variables inside the `MainActivity` class:

```java
final static String SCOPES [] = {"https://graph.microsoft.com/User.Read"};
final static String MSGRAPH_URL = "https://graph.microsoft.com/v1.0/me";

/* UI & Debugging Variables */
private static final String TAG = MainActivity.class.getSimpleName();
Button callGraphButton;
Button signOutButton;

/* Azure AD Variables */
private PublicClientApplication sampleApp;
private IAuthenticationResult authResult;
```

Replace the contents of `onCreate()` with the following code to instantiate MSAL:

```java
super.onCreate(savedInstanceState);
setContentView(R.layout.activity_main);

callGraphButton = (Button) findViewById(R.id.callGraph);
signOutButton = (Button) findViewById(R.id.clearCache);

callGraphButton.setOnClickListener(new View.OnClickListener() {
    public void onClick(View v) {
        onCallGraphClicked();
    }
});

signOutButton.setOnClickListener(new View.OnClickListener() {
    public void onClick(View v) {
        onSignOutClicked();
    }
});

/* Configure your sample app and save state for this activity */
sampleApp = new PublicClientApplication(
        this.getApplicationContext(),
        R.raw.auth_config);

/* Attempt to get a user and acquireTokenSilent */
sampleApp.getAccounts(new PublicClientApplication.AccountsLoadedCallback() {
    @Override
    public void onAccountsLoaded(final List<IAccount> accounts) {
        if (!accounts.isEmpty()) {
            /* This sample doesn't support multi-account scenarios, use the first account */
            sampleApp.acquireTokenSilentAsync(SCOPES, accounts.get(0), getAuthSilentCallback());
        } else {
            /* No accounts */
        }
    }
});
```

The code above attempts to sign in users silently when they open your application through `getAccounts()` and, if successful, `acquireTokenSilentAsync()`.  In the next few sections, we'll implement the callback handler for the case there are no signed in accounts.

#### Use MSAL to get Tokens

Now, we can implement the app's UI processing logic and getting tokens interactively through MSAL.

MSAL exposes two primary methods for getting tokens: `acquireTokenSilentAsync()` and `acquireToken()`.  

`acquireTokenSilentAsync()` signs in a user and get tokens without any user interaction if an account is present. If it succeeds, MSAL will handoff the tokens to your app, if it fails it will generate a `MsalUiRequiredException`.  If this exception is generated, or you want the user to have an interactive sign in experience (credentials, mfa, or other Conditional Access policies may or may not be required), then use `acquireToken()`.  

`acquireToken()` displays UI when attempting to sign in the user and get tokens. However, it may  use session cookies in the browser, or an account in the Microsoft authenticator, to provide the interactive-SSO experience.

Create the following three UI methods inside the `MainActivity` class:

```java
/* Set the UI for successful token acquisition data */
private void updateSuccessUI() {
    callGraphButton.setVisibility(View.INVISIBLE);
    signOutButton.setVisibility(View.VISIBLE);
    findViewById(R.id.welcome).setVisibility(View.VISIBLE);
    ((TextView) findViewById(R.id.welcome)).setText("Welcome, " +
            authResult.getAccount().getUsername());
    findViewById(R.id.graphData).setVisibility(View.VISIBLE);
}

/* Set the UI for signed out account */
private void updateSignedOutUI() {
    callGraphButton.setVisibility(View.VISIBLE);
    signOutButton.setVisibility(View.INVISIBLE);
    findViewById(R.id.welcome).setVisibility(View.INVISIBLE);
    findViewById(R.id.graphData).setVisibility(View.INVISIBLE);
    ((TextView) findViewById(R.id.graphData)).setText("No Data");

    Toast.makeText(getBaseContext(), "Signed Out!", Toast.LENGTH_SHORT)
            .show();
}

/* Use MSAL to acquireToken for the end-user
 * Callback will call Graph api w/ access token & update UI
 */
private void onCallGraphClicked() {
    sampleApp.acquireToken(getActivity(), SCOPES, getAuthInteractiveCallback());
}
```

Add the following methods to get the current activity and process silent & interactive callbacks:

```java
public Activity getActivity() {
    return this;
}

/* Callback used in for silent acquireToken calls.
 * Looks if tokens are in the cache (refreshes if necessary and if we don't forceRefresh)
 * else errors that we need to do an interactive request.
 */
private AuthenticationCallback getAuthSilentCallback() {
    return new AuthenticationCallback() {

        @Override
        public void onSuccess(IAuthenticationResult authenticationResult) {
            /* Successfully got a token, call graph now */
            Log.d(TAG, "Successfully authenticated");

            /* Store the authResult */
            authResult = authenticationResult;

            /* call graph */
            callGraphAPI();

            /* update the UI to post call graph state */
            updateSuccessUI();
        }

        @Override
        public void onError(MsalException exception) {
            /* Failed to acquireToken */
            Log.d(TAG, "Authentication failed: " + exception.toString());

            if (exception instanceof MsalClientException) {
                /* Exception inside MSAL, more info inside the exception */
            } else if (exception instanceof MsalServiceException) {
                /* Exception when communicating with the STS, likely config issue */
            } else if (exception instanceof MsalUiRequiredException) {
                /* Tokens expired or no session, retry with interactive */
            }
        }

        @Override
        public void onCancel() {
            /* User cancelled the authentication */
            Log.d(TAG, "User cancelled login.");
        }
    };
}

/* Callback used for interactive request.  If succeeds we use the access
 * token to call the Microsoft Graph. Does not check cache
 */
private AuthenticationCallback getAuthInteractiveCallback() {
    return new AuthenticationCallback() {

        @Override
        public void onSuccess(IAuthenticationResult authenticationResult) {
            /* Successfully got a token, call graph now */
            Log.d(TAG, "Successfully authenticated");
            Log.d(TAG, "ID Token: " + authenticationResult.getIdToken());

            /* Store the auth result */
            authResult = authenticationResult;

            /* call graph */
            callGraphAPI();

            /* update the UI to post call graph state */
            updateSuccessUI();
        }

        @Override
        public void onError(MsalException exception) {
            /* Failed to acquireToken */
            Log.d(TAG, "Authentication failed: " + exception.toString());

            if (exception instanceof MsalClientException) {
                /* Exception inside MSAL, more info inside the exception */
            } else if (exception instanceof MsalServiceException) {
                /* Exception when communicating with the STS, likely config issue */
            }
        }

        @Override
        public void onCancel() {
            /* User cancelled the authentication */
            Log.d(TAG, "User cancelled login.");
        }
    };
}
```

#### Use MSAL for Sign-out

Next, add support for sign-out.

> [!Important]
> Signing out with MSAL removes all known information about a user from the application, but the user will still have an active session on their device. If the user attempts to sign in again they may see sign-in UI, but may not need to reenter their credentials because the device session is still active.

To add sign-out capability, add the following method inside the `MainActivity` class. This method cycles through all accounts and removes them:

```java
/* Clears an account's tokens from the cache.
 * Logically similar to "sign out" but only signs out of this app.
 * User will get interactive SSO if trying to sign back-in.
 */
private void onSignOutClicked() {
    /* Attempt to get a user and acquireTokenSilent
     * If this fails we do an interactive request
     */
    sampleApp.getAccounts(new PublicClientApplication.AccountsLoadedCallback() {
        @Override
        public void onAccountsLoaded(final List<IAccount> accounts) {

            if (accounts.isEmpty()) {
                /* No accounts to remove */

            } else {
                for (final IAccount account : accounts) {
                    sampleApp.removeAccount(
                            account,
                            new PublicClientApplication.AccountsRemovedCallback() {
                        @Override
                        public void onAccountsRemoved(Boolean isSuccess) {
                            if (isSuccess) {
                                /* successfully removed account */
                            } else {
                                /* failed to remove account */
                            }
                        }
                    });
                }
            }

            updateSignedOutUI();
        }
    });
}
```

#### Call the Microsoft Graph API

Once we have received a token, we can make a request to the [Microsoft Graph API](https://graph.microsoft.com) The access token will be inside the `AuthenticationResult` inside the auth callback's `onSuccess()` method. To construct an authorized request, your app will need to add the access token to the HTTP header:

| header key    | value                 |
| ------------- | --------------------- |
| Authorization | Bearer \<access-token> |

Add the following two methods inside the `MainActivity` class to call graph and update the UI:

```java
/* Use Volley to make an HTTP request to the /me endpoint from MS Graph using an access token */
private void callGraphAPI() {
    Log.d(TAG, "Starting volley request to graph");

    /* Make sure we have a token to send to graph */
    if (authResult.getAccessToken() == null) {return;}

    RequestQueue queue = Volley.newRequestQueue(this);
    JSONObject parameters = new JSONObject();

    try {
        parameters.put("key", "value");
    } catch (Exception e) {
        Log.d(TAG, "Failed to put parameters: " + e.toString());
    }
    JsonObjectRequest request = new JsonObjectRequest(Request.Method.GET, MSGRAPH_URL,
            parameters,new Response.Listener<JSONObject>() {
        @Override
        public void onResponse(JSONObject response) {
            /* Successfully called graph, process data and send to UI */
            Log.d(TAG, "Response: " + response.toString());

            updateGraphUI(response);
        }
    }, new Response.ErrorListener() {
        @Override
        public void onErrorResponse(VolleyError error) {
            Log.d(TAG, "Error: " + error.toString());
        }
    }) {
        @Override
        public Map<String, String> getHeaders() {
            Map<String, String> headers = new HashMap<>();
            headers.put("Authorization", "Bearer " + authResult.getAccessToken());
            return headers;
        }
    };

    Log.d(TAG, "Adding HTTP GET to Queue, Request: " + request.toString());

    request.setRetryPolicy(new DefaultRetryPolicy(
            3000,
            DefaultRetryPolicy.DEFAULT_MAX_RETRIES,
            DefaultRetryPolicy.DEFAULT_BACKOFF_MULT));
    queue.add(request);
}

/* Sets the graph response */
private void updateGraphUI(JSONObject graphResponse) {
    TextView graphText = findViewById(R.id.graphData);
    graphText.setText(graphResponse.toString());
}
```

#### Multi-account applications

This app is built for a single account scenario. MSAL also supports multi-account scenarios, but it requires some additional work from apps. You will need to create UI to help user's select which account they want to use for each action that requires tokens. Alternatively, your app can implement a heuristic to select which account to use via the `getAccounts()` method.

## Test your app

### Run locally

Build and deploy the app to a test device or emulator. You should be able to sign in and get tokens for Azure AD or personal Microsoft accounts.

After you sign in, the app will display the data returned from the Microsoft Graph `/me` endpoint.

### Consent

The first time any user signs into your app, they will be prompted by Microsoft identity to consent to the permissions requested.  While most users are capable of consenting, some Azure AD tenants have disabled user consent which requires admins to consent on behalf of all users. To support this scenario, register your app's scopes in the Azure portal.

## Get help

Visit [Help and support](https://docs.microsoft.com/azure/active-directory/develop/developer-support-help-options) if you have trouble with this tutorial or with the Microsoft identity platform.

Help us improve the Microsoft identity platform. Tell us what you think by completing a short two-question survey.

> [!div class="nextstepaction"]
> [Microsoft identity platform survey](https://forms.office.com/Pages/ResponsePage.aspx?id=v4j5cvGGr0GRqy180BHbRyKrNDMV_xBIiPGgSvnbQZdUQjFIUUFGUE1SMEVFTkdaVU5YT0EyOEtJVi4u)