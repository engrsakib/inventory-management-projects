# Case Study: Dynamic Role and Permission Management (Super Admin Power) — Detailed Analysis

## Context

In a modern e-commerce or business management system, every user may hold the role "admin" for elevated access, but their individual powers (permissions) must differ dynamically.  
A Super Admin is empowered to grant, revoke, or modify any admin's permissions at any time, making the system flexible, secure, and scalable.

---

## The Problem

- If everyone is an admin, unrestricted actions create security risks and loss of control.
- Business requires granular access: some users should be able to delete/publish/edit orders, others only add/edit products, etc.
- Super Admin must be able to instantly update anyone's permissions.
- Permission structure must be dynamic, auditable, and granular.

---

## Solution Overview

### Hybrid RBAC + Permission Array Design

- **RBAC (Role Based Access Control):** Each user has a high-level role (admin, superadmin, etc.).
- **Permissions (Power):** A dynamic array of granular permissions, e.g. `"create_order"`, `"edit_product"`, `"manage_users"`.
- **Super Admin:** Special power to manage permissions of other users in the system.

---

## Schema, Interface, and Database Example

### 1. **TypeScript Interface**

```typescript
// Permission Types
type Permission =
  | "create_order"
  | "delete_order"
  | "edit_product"
  | "update_price"
  | "manage_users"
  | "manage_permissions"
  | "view_reports"
  | "delete_product"
  | "publish_order"
  | "refund_order"
  | "audit_log";

// Role Enum
enum Role {
  ADMIN = "admin",
  SUPERADMIN = "superadmin",
  EDITOR = "editor",
  VIEWER = "viewer"
}

// User Interface
interface IUser {
  _id: string; // MongoDB ObjectId
  name: string;
  email: string;
  password: string; // Hashed
  role: Role;
  permissions: Permission[];
  department?: string;
  attributes?: Record<string, any>;
  createdAt?: Date;
  updatedAt?: Date;
}
```

---

### 2. **Mongoose Schema**

```typescript name=user.model.ts
import { Schema, model } from "mongoose";

const UserSchema = new Schema(
  {
    name: { type: String, required: true },
    email: { type: String, required: true, unique: true },
    password: { type: String, required: true },
    role: {
      type: String,
      enum: ["admin", "superadmin", "editor", "viewer"],
      required: true,
      default: "admin"
    },
    permissions: [{ type: String, required: true }],
    department: { type: String },
    attributes: { type: Schema.Types.Mixed },
  },
  { timestamps: true }
);

export const UserModel = model("User", UserSchema);
```

---

### 3. **MongoDB Collection Example**

```json name=user.document.example.json
{
  "_id": "651e12ea8db2d6e5c15a3c21",
  "name": "Sakib Ahmed",
  "email": "sakib@example.com",
  "password": "$2b$10$xxxx", // hashed password
  "role": "admin",
  "permissions": [
    "create_order",
    "edit_product",
    "view_reports"
  ],
  "department": "sales",
  "attributes": {
    "region": "Dhaka",
    "mobile": "017xxxxxxx"
  },
  "createdAt": "2025-09-27T14:04:11.000Z",
  "updatedAt": "2025-09-27T14:07:21.000Z",
  "__v": 0
}
```

**Super Admin Example:**
```json
{
  "_id": "651e12ea8db2d6e5c15a3c22",
  "name": "Nahid Ahmed",
  "email": "nahid@example.com",
  "password": "$2b$10$xxxx",
  "role": "superadmin",
  "permissions": [
    "create_order",
    "delete_order",
    "edit_product",
    "update_price",
    "manage_users",
    "manage_permissions", // super power
    "view_reports",
    "delete_product",
    "publish_order",
    "refund_order",
    "audit_log"
  ],
  "department": "management"
}
```

---

## Practical Flow

### Permission Check Example

```typescript
function hasPermission(user: IUser, permission: Permission): boolean {
  return user.permissions.includes(permission)
    || user.role === "superadmin"; // Super Admin bypass
}

// Usage:
if (hasPermission(currentUser, "delete_order")) {
  // Allow order deletion
} else {
  // Deny access
}
```

### Super Admin: Grant/Revoke Permissions

```typescript
function updatePermissions(targetUser: IUser, permissions: Permission[], action: "add" | "remove") {
  if (action === "add") {
    targetUser.permissions = Array.from(new Set([
      ...targetUser.permissions,
      ...permissions
    ]));
  } else {
    targetUser.permissions = targetUser.permissions.filter(p => !permissions.includes(p));
  }
  // Save to DB...
}
```

### Real-life Example

- **User A:**  
  - name: "Nahid"
  - permissions: ["create_order", "edit_product"]
- **User B:**  
  - name: "Sakib"
  - permissions: ["create_order", "delete_order", "update_price"]
- **Super Admin:**  
  - name: "SuperAdmin"
  - permissions: ["manage_permissions", ...others]

**Super Admin removes `edit_product` from Nahid:**
```typescript
updatePermissions(nahid, ["edit_product"], "remove");
```
**Super Admin adds `delete_order` for Nahid:**
```typescript
updatePermissions(nahid, ["delete_order"], "add");
```

---

## Common Scenarios

1. **Adding a new admin:**  
   Super Admin creates a user with selected permissions.

2. **Reducing an admin's power:**  
   Super Admin removes permission(s) from their array.

3. **Permission Check for Sensitive Actions:**  
   Every API endpoint or UI action checks permissions before allowing.

4. **Audit log:**  
   Every permission change is logged for traceability and compliance.

5. **Bulk Power Management:**  
   Super Admin can grant/revoke multiple permissions at once.

---

## Security Tips

- Always validate permission changes in the backend.
- Only Super Admin can change others' permissions.
- Strict authentication & authorization for permission management endpoints.
- Keep audit logs for all permission changes.
- Principle of Least Privilege: Grant only what is needed.

---

## Advanced: Google/Microsoft Style (Enterprise Best Practice)

- **Hybrid RBAC + ABAC:** Combine role-based and attribute-based access (e.g., department, location).
- **Policy Engine:** Centralized permission check function.
- **Approval Workflow:** Sensitive permission changes require approval.
- **Audit & Alert:** Automated logging and alerting for suspicious changes.

**Permission Model Example:**
```typescript
interface Permission {
  action: string; // e.g. "delete"
  resource: string; // e.g. "order"
  condition?: (user: IUser, resource: any, context?: any) => boolean; // ABAC
}
```

**Centralized Check:**
```typescript
function canAccess(user: IUser, action: string, resource: string, context?: any, permissionMap?: Record<string, Permission>): boolean {
  if (user.role === "superadmin") return true;
  const key = `${resource}.${action}`;
  if (user.permissions.includes(key)) {
    if (permissionMap && permissionMap[key]?.condition) {
      return permissionMap[key].condition(user, resource, context);
    }
    return true;
  }
  return false;
}
```

---

## Summary

- **Role is fixed** (admin, superadmin, etc.)
- **Permissions are dynamic** (array of granular powers)
- **Super Admin can manage anyone's powers** at any time
- **Permission check for each action**
- **Audit and security** are built-in
- **Scalable, flexible, enterprise-ready design**

With this system, you achieve flexible, secure, and scalable dynamic role-permission management—ready for any modern web or business application!

```typescript
// See user.model.ts and IUser interface above for schema and code.
```