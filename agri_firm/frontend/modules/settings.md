# User & Farm Settings Module

**Frontend Route:** `/settings`
**API Base Path:** `/settings`
**Auth Required:** Yes (Bearer token on all endpoints)

---

## TypeScript Interfaces

```typescript
interface UserProfile {
  fullName: string;
  email: string;
  phone: string;
  farmName: string;
  photoUrl?: string;      // Full URL to the uploaded profile photo, or null
}

interface NotificationSettings {
  emailNotifications: boolean;
  pushNotifications: boolean;
  weatherAlerts: boolean;
  lowInventoryAlerts: boolean;
  taskReminders: boolean;
}

interface AppearanceSettings {
  theme: "light" | "dark" | "system";
  language: "en" | "es" | "fr";
}

interface ChangePasswordPayload {
  currentPassword: string;
  newPassword: string;
  confirmPassword: string;
}
```

---

## Endpoints

---

### GET `/settings/profile`

Retrieve the current user's profile information and associated farm name.

**Auth Required:** Yes

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "fullName": "John Doe",
    "email": "john.doe@farm.com",
    "phone": "+254712345678",
    "farmName": "Green Valley Farm",
    "photoUrl": "https://cdn.yourdomain.com/photos/user-uuid-001.jpg"
  }
}
```

**Note:** `photoUrl` is `null` or omitted if no profile photo has been uploaded. The frontend renders a default avatar placeholder when `photoUrl` is absent or null.

---

### PUT `/settings/profile`

Update the user's profile information.

**Auth Required:** Yes

**Request Body:**

```json
{
  "fullName": "John Doe",
  "email": "john.doe@farm.com",
  "phone": "+254712345678",
  "farmName": "Green Valley Farm"
}
```

**Note:** `photoUrl` is NOT updated via this endpoint. Use `POST /settings/profile/photo` for photo uploads.

**Success Response — `200 OK`:**

```json
{
  "message": "Profile updated successfully",
  "data": {
    "fullName": "John Doe",
    "email": "john.doe@farm.com",
    "phone": "+254712345678",
    "farmName": "Green Valley Farm",
    "photoUrl": "https://cdn.yourdomain.com/photos/user-uuid-001.jpg"
  }
}
```

**Validation Rules:**
- `fullName`: required, 2–100 characters
- `email`: required, valid email format; if changed, must be unique across all users
- `phone`: optional, valid phone format
- `farmName`: required, 2–100 characters

**Side Effects:** If `farmName` is updated here, it should propagate to the farm record associated with the user. The `farmName` returned from `GET /auth/me` should reflect this change.

---

### POST `/settings/profile/photo`

Upload or replace the user's profile photo.

**Auth Required:** Yes

**Content-Type:** `multipart/form-data` (NOT `application/json`)

**Form Fields:**

| Field   | Type | Required | Description                                   |
|---------|------|----------|-----------------------------------------------|
| `photo` | file | Yes      | Image file: JPEG, PNG, or WebP. Max 5MB.      |

**Request Example (multipart):**

```
POST /settings/profile/photo
Authorization: Bearer <token>
Content-Type: multipart/form-data; boundary=----FormBoundary

------FormBoundary
Content-Disposition: form-data; name="photo"; filename="avatar.jpg"
Content-Type: image/jpeg

[binary image data]
------FormBoundary--
```

**Success Response — `200 OK`:**

```json
{
  "message": "Profile photo updated successfully",
  "data": {
    "photoUrl": "https://cdn.yourdomain.com/photos/user-uuid-001-v2.jpg"
  }
}
```

**Validation Rules:**
- File type must be one of: `image/jpeg`, `image/png`, `image/webp`
- Maximum file size: 5MB
- Return `400` with descriptive error if file type or size is invalid

**Backend Storage:** The backend should store the file in object storage (e.g. AWS S3, Cloudinary, or local static server) and return a publicly accessible URL. Do not return a relative path.

**Frontend Impact:** After a successful upload, the frontend updates the profile avatar immediately using the returned `photoUrl`. If the URL changes format (e.g. adds a CDN prefix), the frontend renders it correctly as long as it is a valid HTTP/HTTPS URL.

---

### GET `/settings/notifications`

Retrieve the current user's notification toggle preferences.

**Auth Required:** Yes

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "emailNotifications": true,
    "pushNotifications": false,
    "weatherAlerts": true,
    "lowInventoryAlerts": true,
    "taskReminders": true
  }
}
```

**Default Values (first-time user):** All booleans default to `true`.

---

### PUT `/settings/notifications`

Update the notification preference toggles.

**Auth Required:** Yes

**Request Body:**

```json
{
  "emailNotifications": true,
  "pushNotifications": false,
  "weatherAlerts": true,
  "lowInventoryAlerts": false,
  "taskReminders": true
}
```

**Success Response — `200 OK`:**

```json
{
  "message": "Notification preferences updated",
  "data": {
    "emailNotifications": true,
    "pushNotifications": false,
    "weatherAlerts": true,
    "lowInventoryAlerts": false,
    "taskReminders": true
  }
}
```

**Validation Rules:**
- All five fields are required booleans.
- The backend should use these settings when deciding whether to send emails or push notifications.

**Side Effects:** If `lowInventoryAlerts` is set to `false`, the backend should stop creating or delivering inventory-type notifications for this user.

---

### GET `/settings/appearance`

Retrieve the current user's theme and language preferences.

**Auth Required:** Yes

**Success Response — `200 OK`:**

```json
{
  "message": "Success",
  "data": {
    "theme": "system",
    "language": "en"
  }
}
```

**Default Values:** `theme: "system"`, `language: "en"`.

---

### PUT `/settings/appearance`

Update theme and language preferences.

**Auth Required:** Yes

**Request Body:**

```json
{
  "theme": "dark",
  "language": "en"
}
```

**Success Response — `200 OK`:**

```json
{
  "message": "Appearance settings updated",
  "data": {
    "theme": "dark",
    "language": "en"
  }
}
```

**Validation Rules:**
- `theme`: required, must be one of: `light | dark | system`
- `language`: required, must be one of: `en | es | fr`

**Frontend Impact:** The frontend applies theme and language immediately upon receiving the response. `theme: "system"` means the app should follow the OS-level dark/light preference (`prefers-color-scheme`). If this enum changes, the frontend's theme-switching logic must be updated in sync.

---

### POST `/settings/change-password`

Change the current user's password.

**Auth Required:** Yes

**Request Body:**

```json
{
  "currentPassword": "OldPassword123!",
  "newPassword": "NewStrongPass456!",
  "confirmPassword": "NewStrongPass456!"
}
```

**Success Response — `200 OK`:**

```json
{
  "message": "Password changed successfully. Please sign in again.",
  "data": {}
}
```

**Error Responses:**

| Code  | Scenario                                       |
|-------|------------------------------------------------|
| `400` | `newPassword` and `confirmPassword` don't match |
| `401` | `currentPassword` is incorrect                 |
| `422` | New password doesn't meet strength requirements |

**Validation Rules:**
- `currentPassword`: required
- `newPassword`: required, minimum 8 characters, at least one number or special character
- `confirmPassword`: required, must match `newPassword`
- `newPassword` must be different from `currentPassword`

**Frontend Behavior:** After a successful password change, the frontend clears the token from `localStorage` and redirects to `/signin`, forcing the user to re-authenticate with the new password.

---

## Notes

- Settings are stored per-user (not per-farm). A user accessing multiple farms shares the same notification, appearance, and profile settings.
- `farmName` in the profile is linked to the farm record. Changing it here should propagate to the farm table. If the system supports multiple farms per user, this behavior needs to be reconsidered.
- The `photoUrl` is intentionally excluded from the `PUT /settings/profile` body to prevent accidental overwriting of the uploaded photo URL with a null or empty value during profile updates.
- All settings endpoints should return the full settings object in the response, not just a success confirmation. This allows the frontend to sync its local state with the server state in a single round-trip.
- Future consideration: Add a `DELETE /settings/profile/photo` endpoint to remove a profile photo and revert to the default avatar.
- Future consideration: Add a `GET /settings/farm` and `PUT /settings/farm` endpoint for farm-level settings (timezone, currency, units of measurement) separate from user-level settings.
