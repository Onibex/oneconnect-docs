# Smart Gateway Configuration Manual

## Add a New User to the OneConnect Cloud Platform

Only **administrator users** with the appropriate permissions can add new users to the OneConnect platform.

> 📌 **Prerequisite:** The [Email Service (SMTP)](10-Configuring_Email_Service_SMTP.md) must be configured before adding new users. The platform sends activation and notification emails through this service.

To access the OneConnect Cloud platform, use the default administrator account that is created automatically. Please contact the Onibex team to obtain the password.

> ℹ️ **Default administrator account**

<img width="582" height="615" alt="image" src="https://github.com/user-attachments/assets/42d0fa05-2bb4-480d-bde9-012afd4df9bb" />

---

## Step 1: Access the Users Section

Once logged into the OneConnect Cloud platform, navigate to the **upper-left corner** and click the **menu button** (the icon with three horizontal lines).
<img width="1296" height="78" alt="image" src="https://github.com/user-attachments/assets/392bc662-6f04-4a21-9cb2-5f37906ae55b" />

When the sidebar is expanded, go to the **"Users"** option and select it.
<img width="1204" height="567" alt="image" src="https://github.com/user-attachments/assets/139fbce5-fa65-42fe-b868-4e6770e234a1" />

---

## Step 2: Add a New User

Click the **"ADD"** button in the upper-right corner.

The process of adding a new user requires the following information:

| Field | Description |
|---|---|
| **First name** | The user's given name. |
| **Last name** | The user's last name. |
| **Email** | The user's email address (used as the login identifier). |
| **Phone number** | The user's contact phone number. |
| **Company** | The organization the user belongs to. |
| **Country** | The user's country of residence. |

Fill in all the fields and click **"ADD"**.
<img width="1310" height="563" alt="image" src="https://github.com/user-attachments/assets/74716f54-6a67-4b5c-b2ef-33945ae7c95f" />

---

## Step 3: Approve the New User

The new user will appear in the **"USERS"** section with a **"PENDING"** status, meaning the user must be approved in order to be activated.
<img width="1299" height="368" alt="image" src="https://github.com/user-attachments/assets/e4d12e2e-3c5e-4b94-8f7c-7d1163b76c0b" />

To activate the user:

1. Locate the user in the list.
2. Click **"ACTIONS"**.
<img width="1299" height="368" alt="image" src="https://github.com/user-attachments/assets/99014e24-c853-4e4a-a276-9802f8cc2768" />

3. Click **"ACCEPT"** on the top right side of the screen to activate the new user. (You must be logged in as admin to make this process)
<img width="1310" height="662" alt="image" src="https://github.com/user-attachments/assets/301ce46d-bbe9-49c0-91ff-fceb93233644" />

---

## Step 4: Verify Activation

Your new user has been successfully approved. You can verify the status change to **"ACTIVE"** in the **"USERS"** section.

| Status | Meaning |
|---|---|
| **PENDING** | The user has been added but not yet approved. Cannot log in. |
| **ACTIVE** | The user has been approved and can log in to the platform. |

---

## Summary

The complete workflow for adding a new user is:

1. Log in as an administrator.
2. Navigate to **Menu → Users**.
3. Click **"ADD"** and complete the required fields.
4. Locate the new user (status: **PENDING**).
5. Click **ACTIONS → ACCEPT** to activate.
6. Confirm the status changes to **ACTIVE**.
