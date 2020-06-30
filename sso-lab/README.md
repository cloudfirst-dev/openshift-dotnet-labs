# Red Hat SSO Lab
This lab shows how to use Red Hat SSO to secure your .NET Core applications using OIDC/OAuth2.0 authorization with JWT Barer tokens.

## Red Hat SSO Setup
We will setup an instance of Red Hat SSO on an OpenShift cluster for use by a deployed application.

### Provision SSO Application
1. Login to OpenShift
1. Create a new project `dotnet-sso`
1. Switch to the developer console
1. Click on the +Add band on the left of the screen
1. Choose to create a new app by using "From Catalog"
1. Search for SSO and select 'Red Hat Single Sign-On 7.3 (Ephemeral)'
1. Click on "Instantiate Template"
1. Click on "Create"

### Configure API Users
1. From the terminal execute the following command to get the admin url
    ```
    oc project dotnet-sso
    echo $(oc get route --selector=application=sso --output=jsonpath="https://{ .items[0].spec.host }/auth/admin/")
    ```
1. Open the url outputted during the second command
1. Get the generated username/password
    ```
    echo -e "Username: $(oc get dc sso --output=jsonpath='{ .spec.template.spec.containers[0].env[?(@.name == "SSO_ADMIN_USERNAME")].value}')\n " \
    "Password: $(oc get dc sso --output=jsonpath='{ .spec.template.spec.containers[0].env[?(@.name == "SSO_ADMIN_PASSWORD")].value}')"
    ```
1. Click on Users under Manage on the left of the screen
1. Click the "Add user" button
1. Enter "testuser" in the Username field
1. Click the "Save" button
1. Click on the "Credentials" tab to set a password
1. Enter the password "change@me"
1. Toggle the Temporary selector to off
1. Click the "Reset Password" button

### Create Client for SPA Application

1. Still inside the Red Hat SSO console select "Clients" on the left of the screen
1. Click the create button
1. Enter frontend for the client id and click save
1. Get the redirect url which will be our api route
1. Enter "*" in the "Valid Redirect URIs" field (in a real environment this would be a security risk)
1. Toggle the "Implicit Flow Enabled" to "ON"
1. Click the "Save" button

## Deploy the API and UI
1. Deploy the helm chart
    ```
    cd sso-lab
    helm install dotnet-sso helm
    helm upgrade dotnet-sso helm
    ```

## Test the setup
We will utilize the test ui to login to the SPA using an oauth2 workflow which will then use the response token to make two authenticated requests to the .NET Core web api.  The first will be to retrieve a list of hard coded values.  The second will be to /whoami which will analyze the passed in auth header and extract the user's primary username and return it as a text so the SPA can display a message with "You are successfully authenticated as {username}. You can repeat the steps earlier to add additional users and validate that the message is not hard coded and uses the currently logged in user's JWT token information.  Note the SPA and Red Hat SSO will cache your credentials so you may need to open incognito windows to test different users.

1. Get the web-ui url
    ```
    open http://$(oc get route basic-dotnet-ui --output=jsonpath='{ .spec.host}')
    ```
1. Click the login button
1. Enter the username "testuser" which we created when we configured the SSO realm
1. Enter the password "change@me" which we set when we configured the user in SSO
1. Click the Log In button
1. You should be presented with a message stating the user you are logged in with