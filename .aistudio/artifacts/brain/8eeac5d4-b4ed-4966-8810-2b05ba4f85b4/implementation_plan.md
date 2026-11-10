# Full PPPoE Integration & Multi-OS Expiration Architecture (RouterOS v6 & v7)

Comprehensive, production-grade PPPoE management integration for NextGan WiFi Solution (Mikhmon Server). Mirrors Hotspot workflows with secret management, validity-based expiration (1 day, 5 days, 10 days, and calendar month), active connection monitoring, on-up and scheduled expiry enforcement across RouterOS v6 and v7, and granular reseller access control.

---

### User Review & Critical Decisions

> [!IMPORTANT]
> The following design and architectural choices have been confirmed through Phase 1 clarification:
> - **Expiration Enforcement Action**: When a PPPoE secret expires, the system sets `disabled=yes` on `/ppp/secret` and immediately disconnects any active session from `/ppp/active`.
> - **Navigation Scope**: PPPoE management appears **only after opening/connecting to a router** (`./?session=...`), keeping the top-level global admin clean and router-centric.
> - **Reseller RBAC Structure**: A dedicated **PPPoE** permission group with a master access switch (if disabled, the entire PPPoE module is hidden from that reseller) and granular Read/Write/Full controls including user creation, profile management, enable/disable, and reset/expiry verification.

- **Confirmed Decision 1**: Secrets disabled and kicked from `/ppp/active` on expiration.
- **Confirmed Decision 2**: Router-scoped sidebar navigation (only visible inside active router sessions).
- **Confirmed Decision 3**: Master reseller toggle + granular sub-permissions for Read, Write, and Full modes.
- **Confirmed Decision 4**: Expiry modes supporting 1 Day, 5 Days, 10 Days, and Calendar Month for both MikroTik RouterOS v6 and v7.

---

## 1. Overview & Core Concept

- **What It Does**: Provides a dedicated, full-lifecycle PPPoE management subsystem within NextGan WiFi Solution. Administrators and authorized resellers can create and manage PPPoE secrets (usernames, passwords, profiles, Caller ID/MAC binding, remote addresses), configure PPPoE bandwidth profiles, monitor live active PPPoE connections in real time, and enforce automated expiration with dual RouterOS v6/v7 script compatibility.
- **Target Audience / Persona**: ISP network administrators, broadband operators, and assigned resellers managing client PPPoE connections across multiple MikroTik routers.
- **Key Value**: Delivers the same intuitive, voucher-like simplicity of Mikhmon Hotspot to PPPoE broadband subscribers without requiring complex external billing engines or RADIUS infrastructure.

---

## 2. User Experience & Visual Design

- **Navigation & Layout Flow**:
  - Inside any connected router session (`./?session=<name>`), the sidebar reveals a dedicated **PPPoE** section:
    - **PPPoE Users / Secrets**: Searchable list of all secrets with status badges (Active, Expired, Disabled), profile, Caller ID, IP, expiration countdown, and action menus (`Edit`, `Enable/Disable`, `Disconnect Active`, `Delete`).
    - **Add PPPoE User**: Clean single-user modal/page with auto-generated or manual credentials, profile selection, Caller ID lock, and validity selector.
    - **PPPoE Profiles**: Bandwidth limits (rate-limit rx/tx), local/remote address pools, DNS, and validity configuration.
    - **PPPoE Active**: Real-time connected sessions displaying uptime, caller MAC/IP, interface name, byte counters, and 1-click disconnect.
    - **Reset PPPoE / Expiry Check**: Quick lookup tool (search by username, IP, or Caller ID MAC) to check expiration status, reset MAC binding, or re-enable an expired account.

- **Visual Identity & Theme**:
  - Consistent with NextGan WiFi Solution's theme engine (`dark`, `blue`, `green`, `light`, `pink`).
  - **Zero-Pill Metadata**: Expiry dates, speeds, and timestamps formatted as clean, quiet unboxed text separated by subtle typographic dots (`·`).
  - **Mobile-Compact Layout**: Full viewport adaptation on screens $\le 767\text{px}$ with no horizontal sliders, compact table rows, and floating three-dot action menus.

---

## 3. Key Product Decisions & Trade-Offs

- **Decision 1: Expiration Architecture for MikroTik ROS 6 vs ROS 7**
  - *Chosen Approach*: Hybrid comment timestamp + profile `on-up` script + `/system/scheduler` background script + PHP on-view synchronization.
  - *Why*: RouterOS v6 uses legacy date format (`M/d/yyyy` or `d/M/yyyy`), while RouterOS v7 uses ISO format (`yyyy-mm-dd`). The scheduler script auto-detects the RouterOS major version via `$API->comm('/system/resource/print')` and deploys the appropriate MikroTik syntax. Furthermore, every time an administrator or reseller views PPPoE lists in Mikhmon, the backend checks timestamps and ensures expired users are promptly marked and disconnected even if the router was temporarily offline.
  - *Alternatives Considered*: Pure API-only polling without router scripts (fails if Mikhmon is closed) or RADIUS CoA (unnecessary complexity for standalone MikroTik routers).

- **Decision 2: Validity & Calendar Month Expiration Math**
  - *Chosen Approach*:
    - Fixed Day Intervals: `1d` (+24 hours), `5d` (+120 hours), `10d` (+240 hours).
    - Calendar Month (`1m`): Expires at 23:59:59 on the last day of the current billing cycle or 1 calendar month later, respecting differing month lengths (28/29/30/31 days).
  - *Why*: Directly aligns with standard local ISP billing cycles in Bangladesh and worldwide.

- **Decision 3: Reseller Access Control Structure**
  - *Chosen Approach*:
    - Master Switch: `manage_pppoe` (if false, PPPoE completely disappears from the reseller's sidebar and API routes).
    - Sub-permissions: `view_pppoe_users`, `add_pppoe_user`, `edit_pppoe_user`, `delete_pppoe_user`, `enable_disable_pppoe_user`, `view_pppoe_profiles`, `manage_pppoe_profiles`, `view_pppoe_active`, `disconnect_pppoe_active`, and `reset_pppoe`.
  - *Why*: Allows admins to grant full PPPoE capabilities to technical managers, read-only monitoring to support agents, or disable PPPoE entirely for voucher-only resellers.

---

## 4. Technical Architecture & Component Hierarchy

```
┌────────────────────────────────────────────────────────────────────────┐
│                     Client Browser (Desktop & Mobile)                   │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ HTTP / JSON
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│             Top Navbar & Router Session Context (menu.php)              │
│       [Dashboard] [Hotspot] [PPPoE Menu] [Reports] [Settings]          │
└───────────────────┬────────────────────────────────────────────────────┘
                    │
                    ├──► [PPPoE Secrets List & Actions] (pppoe/users.php)
                    ├──► [Add / Edit Secret] (pppoe/user.php)
                    ├──► [PPPoE Profiles & Rates] (pppoe/profiles.php)
                    ├──► [Live Active PPPoE Sessions] (pppoe/active.php)
                    └──► [Reset PPPoE / Expiry Tool] (pppoe/reset_pppoe.php)
                                    │
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                 Mikhmon Backend & Security Services                     │
│  - reseller_acl.php: Permission validator & Master PPPoE switch        │
│  - pppoe_helper.php: Expiry parser, ROS6/ROS7 script generator         │
│  - activity_log.php: PPPoE audit trails (User added, edited, expired)   │
└───────────────────┬────────────────────────────────────────────────────┘
                    │ RouterOS API Protocol (Port 8728)
                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   MikroTik RouterOS (v6.x or v7.x)                      │
│  - /ppp/secret (name, password, profile, caller-id, comment expiry)    │
│  - /ppp/profile (rate-limit rx/tx, on-up script)                       │
│  - /ppp/active (live sessions, disconnect trigger)                     │
│  - /system/script & /system/scheduler (Auto-disable expired secrets)   │
└────────────────────────────────────────────────────────────────────────┘
```

### Module Structure to Implement:
1. **`mikhmon/pppoe/users.php`**: Searchable secrets table, status filters (All, Active, Expired, Disabled), batch selection, pagination, mobile-compact cards.
2. **`mikhmon/pppoe/user.php`**: Add and Edit PPPoE user form with validity presets (`1d`, `5d`, `10d`, `1m` calendar month), Caller ID binding, and comment generator.
3. **`mikhmon/pppoe/profiles.php` & `profile.php`**: PPPoE bandwidth profiles, local/remote address pools, and rate-limits (`5M/10M`, etc.).
4. **`mikhmon/pppoe/active.php`**: Real-time connected sessions with IP, caller MAC, uptime, interface, and kick/disconnect button.
5. **`mikhmon/pppoe/reset_pppoe.php`**: Dedicated PPPoE reset/expiry recovery tool (mirrors Hotspot Reset Voucher).
6. **`mikhmon/include/pppoe_helper.php`**: Helper functions for RouterOS v6 & v7 compatibility, expiry parsing, date math, and scheduler deployment.
7. **`mikhmon/include/reseller_acl.php` & `settings/resellers.php`**: New PPPoE permission category, master toggle, and permission checklist integration.
8. **`mikhmon/include/menu.php`**: Conditional PPPoE sidebar menu item visible only when inside an active router session (`$session`) and permitted by ACL.
