# Salary Slip Maker

A web-based salary slip management application using Firebase
Authentication, Cloud Firestore, and Firebase Hosting.

> **Project ID:** `mtsalarysilip-f64ba`

## Features

-   Employee sign-in using Firebase Authentication (Email/Password).
-   Password reset through Firebase Authentication, when implemented in
    the website UI.
-   Admin and employee roles.
-   Admin can manage salary-slip records and view employee/month data.
-   Employees can view salary slips associated with their login email.
-   Admin delete actions:
    -   Delete salary slips belonging to one employee.
    -   Delete salary slips for a selected month.
-   Firebase Hosting deployment.

## Technology

-   HTML, CSS, and JavaScript
-   Firebase Authentication
-   Cloud Firestore
-   Firebase Hosting
-   Firebase JavaScript SDK (Compat, version 10.12.2 in the current
    project file)

## Firebase project configuration

The current website is configured for Firebase project:

``` text
mtsalarysilip-f64ba
```

Keep Firebase web configuration in the website's configuration section.
The Firebase web API key is not a server secret; access must be
protected with Authentication and Firestore Security Rules. Never put
service-account private keys or Admin SDK credentials in browser code.

## Firestore data structure

### `users` collection

Use the Firebase Authentication UID as the document ID.

Example employee document:

``` json
{
  "role": "employee",
  "loginEmail": "employee@example.com",
  "employeeId": "EMP001"
}
```

Example admin document:

``` json
{
  "role": "admin",
  "loginEmail": "admin@example.com"
}
```

Create or promote the trusted administrator's user document through a
trusted administrative process (for example, Firebase Console after
verifying the account). Do not provide an employee-facing feature that
lets users set their own role.

### `slips` collection

Each salary slip should include at least:

``` json
{
  "month": "2026-11",
  "empId": "EMP001",
  "loginEmail": "employee@example.com"
}
```

Add the salary fields required by your application, such as employee
name, earnings, deductions, and net salary. Use a consistent month
format throughout the application. The current application may also use
a document ID convention such as `month_employeeId`; verify the code
before changing existing document IDs.

**Important:** `loginEmail` must exactly match the employee's Firebase
Authentication email for the employee read rule below to work. Existing
records with missing or incorrect email values need to be corrected.

## Firestore Security Rules

Open **Firebase Console → Firestore Database → Rules** and review the
rules before publishing.

The following rules implement the role model described here:

``` javascript
rules_version = '2';

service cloud.firestore {
  match /databases/{database}/documents {

    function isLoggedIn() {
      return request.auth != null;
    }

    function isAdmin() {
      return isLoggedIn()
        && exists(/databases/$(database)/documents/users/$(request.auth.uid))
        && get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role == "admin";
    }

    function isOwner() {
      return isLoggedIn()
        && resource.data.loginEmail == request.auth.token.email;
    }

    match /users/{userId} {
      allow read: if isLoggedIn()
        && (request.auth.uid == userId || isAdmin());

      allow create: if isLoggedIn()
        && request.auth.uid == userId
        && request.resource.data.role == "employee"
        && request.resource.data.loginEmail == request.auth.token.email;

      allow update, delete: if isAdmin();
    }

    match /slips/{slipId} {
      allow read: if isAdmin() || isOwner();
      allow create, update, delete: if isAdmin();
    }

    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

### Security-rule notes

-   These rules assume each `users` document ID is the corresponding
    Firebase Authentication UID.
-   They assume salary slip documents contain `loginEmail`.
-   The employee query in the website must be constrained to the
    signed-in employee's email (for example,
    `where("loginEmail", "==", currentUser.email)`). Firestore rules are
    not filters; a broad query that could return other employees'
    records can be rejected.
-   The sample `users` create rule permits creation of an employee
    profile only. Admin provisioning must be done through a trusted
    process.
-   The sample rules prevent employees from writing salary slips. If the
    website currently creates slip documents from an employee-facing
    workflow, that workflow will need to be changed or a secure backend
    added.
-   Admin role checks in browser JavaScript are only for UI behaviour.
    Firestore Rules provide the actual database enforcement.
-   Test rules with both admin and employee accounts before using real
    payroll data. Consider Firebase Emulator Suite and Firestore Rules
    unit tests.

## Enable Email/Password Authentication

1.  Open Firebase Console → Authentication → Sign-in method.
2.  Enable **Email/Password**.
3.  Save the setting.
4.  Create or invite user accounts using your intended account-creation
    process.
5.  Verify that each employee's `users/{uid}` profile has the correct
    `role`, `loginEmail`, and `employeeId`.

For password reset, the website should call Firebase Authentication's
password-reset API and show a clear success/error message. Also
configure authorized domains and review the email template in Firebase
Authentication.

## Run locally

This project is a static HTML application using Firebase's hosted SDK.

1.  Download or copy the latest application HTML file.
2.  Rename the file to `index.html`.
3.  Put it in a folder named `public`.
4.  Open the page through a local static web server (recommended) or
    deploy it to Firebase Hosting.
5.  Confirm the Firebase configuration points to `mtsalarysilip-f64ba`.

Do not treat a successful page load as proof that Authentication,
Firestore permissions, deletion, or payroll calculations are working.
Test each workflow.

## Deploy to Firebase Hosting

### Prerequisites

-   Node.js LTS: https://nodejs.org/
-   Access to the Firebase project `mtsalarysilip-f64ba`

### Install Firebase CLI

``` bash
npm install -g firebase-tools
firebase login
```

### Initialize Hosting

From the project folder:

``` bash
firebase init hosting
```

Choose:

-   Use an existing project.
-   Select `mtsalarysilip-f64ba`.
-   Public directory: `public`.
-   Configure as a single-page app: `No` (unless the app has client-side
    routes).
-   Set up automatic builds/deploys: as required.

Ensure the latest application file is saved as:

``` text
public/index.html
```

### Deploy

``` bash
firebase deploy --only hosting
```

Firebase will print the Hosting URL after deployment. Use the URL shown
by the CLI rather than assuming a URL.

## Recommended project layout

``` text
SalarySlipWebsite/
├── public/
│   └── index.html
├── firebase.json
├── .firebaserc
└── README.md
```

The exact layout may differ if the project uses additional CSS,
JavaScript, images, or configuration files.

## Admin deletion features

The updated application version includes admin controls to:

-   Delete all salary-slip records associated with one employee.
-   Delete all salary-slip records for a selected month.

These actions are destructive. Back up Firestore before using them on
live payroll data. Confirm the matching logic against your actual
`slips` documents. The current client-side implementation may scan the
`slips` collection to find matching documents, which can be inefficient
for a large database. For large datasets, use indexed queries or a
trusted backend/Cloud Function with carefully enforced admin
authorization.

Deleting salary slips does **not** necessarily delete the employee's
Firebase Authentication account or the employee's `users` profile.

## Pre-launch checklist

-   [ ] Email/Password sign-in is enabled.
-   [ ] Admin account is verified and has `role: "admin"` in
    `users/{uid}`.
-   [ ] Employee profiles use the correct Authentication UID as document
    ID.
-   [ ] Every employee salary slip has the correct `loginEmail`.
-   [ ] Employee queries filter by the signed-in email.
-   [ ] Firestore rules are published and tested.
-   [ ] Employee cannot read another employee's salary slip.
-   [ ] Employee cannot create, update, or delete salary slips.
-   [ ] Admin can perform only the intended management actions.
-   [ ] Password reset flow is tested.
-   [ ] Backup and restore process is understood.
-   [ ] Hosting deployment is tested using the deployed URL.
-   [ ] Sensitive payroll fields (PAN, UAN, bank details, etc.) are only
    exposed to authorized users.

## Troubleshooting

### Employee cannot see a salary slip

Check:

1.  The employee is signed in with the expected Firebase Authentication
    email.
2.  The slip's `loginEmail` exactly matches that email.
3.  The employee's Firestore query filters on `loginEmail`.
4.  The `users/{uid}` profile exists and has `role: "employee"`.
5.  Firestore Rules have been published.

### Admin gets permission denied

Check:

1.  The signed-in admin's Firebase UID is the document ID in `users`.
2.  That document has `role` set to exactly `admin`.
3.  The browser is signed in to the correct Firebase project.
4.  The latest Firestore Rules are published.

### Hosting shows an old version

Confirm that the updated file is saved as `public/index.html`, then run:

``` bash
firebase deploy --only hosting
```

Refresh the site and clear the browser cache if needed.

## Important production note

This README describes the intended setup and the supplied application
features. It does not certify that the application has been fully
tested, audited, or made production-ready. Before using real salary or
banking data, complete security testing, backup/restore testing,
access-control verification, and a review of applicable privacy and
employment-data requirements.
