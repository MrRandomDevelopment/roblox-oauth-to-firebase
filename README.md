# Roblox Oauth 2.0 to Firebase Setup Guide

This guide will help you set up the Roblox OAuth signup flow for Roblox Oauth 2.0 using Firebase.

## Prerequisites

1. **Firebase Project**: Make sure you have a Firebase project set up.
2. **Roblox OAuth Credentials**: Obtain your `client_id` and `client_secret` from Roblox.
3. **Web Server**: Ensure you have a web server to host your HTML file.

## Step-by-Step Setup

### 1. Create the HTML File

Create an HTML file (e.g., `index.html`) with the following content:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Roblox Signup</title>
</head>
<body>

  <div class="signup-container">
    <h2>Sign Up for SkyLox</h2>
    <div class="form-group">
      <button id="oauth-button">Sign Up with Roblox</button>
    </div>
  </div>

  <!-- Firebase SDK -->
  <script src="https://www.gstatic.com/firebasejs/8.0.0/firebase-app.js"></script>
  <script src="https://www.gstatic.com/firebasejs/8.0.0/firebase-auth.js"></script>
  <script src="https://www.gstatic.com/firebasejs/8.0.0/firebase-firestore.js"></script>

  <script>
    document.addEventListener('DOMContentLoaded', function() {
      var firebaseConfig = {
        apiKey: "YOUR_API_KEY",
        authDomain: "yourapp.firebaseapp.com",
        projectId: "yourapp",
        storageBucket: "yourapp.appspot.com",
        messagingSenderId: "your_messaging_sender_id",
        appId: "your_app_id",
      };
      firebase.initializeApp(firebaseConfig);

      const oauthButton = document.getElementById('oauth-button');
      if (oauthButton) {
        oauthButton.addEventListener('click', function(event) {
          event.preventDefault();
          startOAuthFlow();
        });
      }

      function startOAuthFlow() {
        const clientId = 'your_client_id';
        const redirectUri = 'https://www.example.com/callback';
        const state = generateState();

        localStorage.setItem('oauthState', state);

        const authorizationUrl = `https://apis.roblox.com/oauth/v1/authorize?response_type=code&client_id=${clientId}&redirect_uri=${encodeURIComponent(redirectUri)}&state=${state}&scope=openid%20profile`;

        window.location.href = authorizationUrl;
      }

      function generateState() {
        return Math.random().toString(36).substring(2);
      }

      function handleOAuthCallback() {
        const urlParams = new URLSearchParams(window.location.search);
        const code = urlParams.get('code');
        const state = urlParams.get('state');
        const storedState = localStorage.getItem('oauthState');

        if (state !== storedState) {
          alert('Invalid state parameter');
          return;
        }

        exchangeCodeForToken(code);
      }

      function exchangeCodeForToken(code) {
        const tokenUrl = 'https://apis.roblox.com/oauth/v1/token';
        const clientId = 'client_id_here';
        const clientSecret = 'client_secret_here';
        const redirectUri = 'https://www.example.com/callback';

        fetch(tokenUrl, {
          method: 'POST',
          headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
          body: new URLSearchParams({
            grant_type: 'authorization_code',
            code: code,
            redirect_uri: redirectUri,
            client_id: clientId,
            client_secret: clientSecret
          }).toString()
        })
        .then(response => response.json())
        .then(data => {
          if (data.access_token) {
            fetchUserInfo(data.access_token);
          } else {
            console.error('Failed to obtain access token');
            alert('Failed to obtain access token');
          }
        })
        .catch(error => console.error('Error exchanging token:', error));
      }

      function fetchUserInfo(accessToken) {
        fetch('https://apis.roblox.com/oauth/v1/userinfo', {
          headers: { 'Authorization': `Bearer ${accessToken}` }
        })
        .then(response => response.json())
        .then(userInfo => {
          console.log('User Info:', userInfo);
          const additionalUserData = {
            uniqueID: generateSixDigitID(),
            region: userInfo.locale || 'unknown',
            firstName: userInfo.given_name || '',
            lastName: userInfo.family_name || '',
            dob: userInfo.birthdate || '',
            gender: userInfo.gender || '',
            robloxId: userInfo.sub,
            robloxUsername: userInfo.preferred_username
          };
          saveUserToFirebase(userInfo.sub, additionalUserData);
        })
        .catch(error => console.error('Error fetching user info:', error));
      }

      function saveUserToFirebase(uid, additionalUserData) {
        firebase.auth().onAuthStateChanged((user) => {
          if (user) {
            console.log('User authenticated:', user.uid);
            console.log('Saving data to Firestore for UID:', uid);
            firebase.firestore().collection('users').doc(uid).set(additionalUserData)
              .then(() => {
                console.log('User data saved successfully');
                window.location.href = 'https://myportal.skylox.org';
              })
              .catch(error => console.error('Error saving user to Firestore:', error));
          } else {
            console.error('User is not authenticated, redirecting to login...');
            firebase.auth().signInAnonymously().then(() => {
              saveUserToFirebase(uid, additionalUserData);
            }).catch(error => {
              console.error('Error signing in anonymously:', error);
            });
          }
        });
      }

      function generateSixDigitID() {
        return Math.floor(100000 + Math.random() * 900000).toString();
      }

      if (window.location.pathname === '/callback/') {
        handleOAuthCallback();
      }
    });
  </script>
</body>
</html>
```

### 2. Configure Firebase

1. Go to your Firebase project in the Firebase Console.
2. Navigate to **Project Settings** and copy your Firebase configuration details.
3. Replace the placeholder values in the `firebaseConfig` object in the HTML file with your actual Firebase configuration values.

### 3. Enable Firebase Authentication

1. In the Firebase Console, go to **Authentication**.
2. Enable **Anonymous Sign-In** under the Sign-in method tab.

### 4. Set Firestore Security Rules

Set the following security rules for your Firestore database to ensure only authenticated users can read and write data:

```plaintext
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if request.auth != null;
    }
  }
}
```

### 5. Obtain Roblox OAuth Credentials

1. Create an OAuth application on Roblox.
2. Note down your `client_id` and `client_secret`.

### 6. Update OAuth Parameters

Replace the placeholders in the JavaScript code with your actual OAuth application details:

- `your_client_id`
- `your_client_secret`
- `https://www.example.com/callback` (update to your actual callback URL)

### 7. Setup .env File (Optional but Recommended)

Create a `.env` file to store your API keys and other sensitive information. This step requires server-side configuration to load these variables into your frontend code securely.

### 8. Deploy Your HTML File

Deploy your HTML file to your web server.

### 9. Handling OAuth Callback

Ensure that your server can handle the OAuth callback URL (`https://www.example.com/callback`). This URL should match the one configured in your Roblox OAuth application and in the HTML file.

### 10. Testing

1. Navigate to your deployed HTML file.
2. Click the "Sign Up with Roblox" button and complete the OAuth flow.
3. Check the Firebase Firestore to confirm that user data is being saved correctly.

## Troubleshooting

- Ensure all URLs are correctly configured.
- Check the browser console for any error messages.
- Verify your Firebase and Roblox OAuth credentials.

By following these steps, you should have a fully functional Roblox OAuth signup flow integrated with Firebase.
