# Smart Gateway Configuration Manual

## Grant SAP Link Access to a User

This manual describes how to grant access to an existing SAP Link to another user in the OneConnect platform. This is useful when multiple team members need to collaborate on the same SAP Link, such as when a developer creates the SAP Link and a data engineer or analyst needs read access to monitor its activity.

> 📋 **Prerequisites:**
>
> - **Administrator account credentials.** Only administrators can modify SAP Link access permissions.
> - The **email address** of the user who will receive access. This user must already exist in the OneConnect platform. If not, see the [Add a New User manual](./01-Adding_New_Users.md).

---

## Step 1: Log In as Administrator

Log in to the OneConnect platform with your **administrator account**.

---

## Step 2: Locate the User Who Owns the SAP Link

In the **User Dashboard**, locate the user who originally created the SAP Link.

Click the **arrow button** next to their name to open their profile.

---

## Step 3: Navigate to the SAP Links Tab

Inside the user's profile, click the **"SAP Links"** tab to see all SAP Links owned by this user.

---

## Step 4: Access the Target SAP Link

Locate the SAP Link to which you want to grant access, and click the **"GO"** button.

---

## Step 5: Open the Configuration Panel

In the SAP Link view, click the **"Configuration"** button in the upper-right corner.

---

## Step 6: Open the SAP Link Access Tab

In the Configuration panel, select the **"SAP Link Access"** tab.

---

## Step 7: Add the New User

In the text field, enter the **email address** of the user you want to grant access to, then click **"Add user"**.

> ⚠️ **Important:** Make sure you enter the **email address exactly** as it is registered in the system. Any typo, missing character, or case mismatch may prevent the user from being found.

---

## Step 8: Verify the User Was Added

Confirm that the new user appears under the **"Users with access"** section.

---

## Step 9: Confirm Access from the User's Side

Log out of the administrator account and **log in with the newly added user's account** to confirm that the SAP Link is now visible and accessible from their session.

---

## Expected Result

> ✅ **Success indicators:**
>
> - The newly added user appears under **"Users with access"**.
> - The user can access the selected **SAP Link** during their next session.

---

## Notes

> 📌 **Note:**
>
> - Make sure you enter the **email address exactly** as it is registered in the system.
> - If the user cannot see the SAP Link immediately, have them **log out and log back in**.

---

## Quick Troubleshooting

| Issue | Cause | Resolution |
|---|---|---|
| **User not listed under "Users with access"** | Email typo or case mismatch. | Verify the email format and click **"Add user"** again. |
| **User cannot see the SAP Link** | User is logging in with a different account. | Ensure they are logging in with the exact same email that was granted access. |
| **Save error** | Insufficient permissions. | Verify your role. Only administrators can modify access permissions. |

---

## Summary

The complete workflow for granting SAP Link access is:

1. Log in with an **administrator account**.
2. Locate the **user who owns the SAP Link** in the User Dashboard.
3. Open their profile and go to the **SAP Links** tab.
4. Click **GO** on the target SAP Link.
5. Click **Configuration → SAP Link Access**.
6. Enter the target user's **email address** and click **Add user**.
7. Verify the user appears under **Users with access**.
8. Have the user log out and log back in to confirm access.
