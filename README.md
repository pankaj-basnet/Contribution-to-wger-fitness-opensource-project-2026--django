## **Technical Internship Report: wger Docker Development Setup & Troubleshooting**

**Date:** March 21, 2026  
**Project:** wger (Workout Manager) - Open Source Fitness Platform  
**Intern:** Sourav  
**Environment:** Windows (MinGW64 / Git Bash) with Docker Desktop

---

### **1. Environment Configuration & Workspace Mapping** 📂

Before starting the server, the workspace must be correctly mapped. The `wger` project usually consists of two separate repositories: the source code and the Docker configuration.

#### **Workspace Structure**
| Step | Task Description | Technical Detail / Command |
| :--- | :--- | :--- |
| **1** | **Locate Source Code** | `D:/src/PY/WGER-2026-PULL-/DJANGO-/CODE-/wger` |
| **2** | **Locate Docker Config** | `D:/src/PY/WGER-2026-PULL-/DJANGO-/wger-docker/CODE-/docker/dev` |
| **3** | **Initialize `.env`** | `cp .env.example .env` (Inside the `docker/dev` folder) |
| **4** | **Link Repositories** | `echo "WGER_CODEPATH=/d/src/PY/WGER-2026-PULL-/DJANGO-/CODE-/wger" > .env` |

> [!TIP]
> **Pro-Tip:** On Windows Git Bash, always use the `/d/path/...` format for the `WGER_CODEPATH` variable to ensure Docker Desktop can resolve the volume mount correctly. 💡

---

### **2. Launching the Development Stack** 🚀

The modern `docker compose` workflow uses the `--watch` flag to improve developer experience by eliminating the need for manual restarts.

#### **The "Watch" Workflow**
| # | Feature | Impact on Intern Workflow |
| :--- | :--- | :--- |
| **1** | **File Syncing** | Automatically pushes local `.py`, `.js`, and `.html` changes into the container. 🔄 |
| **2** | **Auto-Rebuild** | Changes to `pyproject.toml` or `package.json` trigger an image rebuild. 🏗️ |
| **3** | **User Mapping** | Solves Permission Denied errors caused by different User IDs on Host vs. Container. 🛡️ |

**Command to start:**
```bash
docker compose up --watch
```

---

### **3. Application Bootstrapping & Data Loading** ⚡

Once the containers (Web and Redis/Cache) are healthy, the application must be initialized.

#### **Initialization Sequence**
| Order | Command | Purpose |
| :--- | :--- | :--- |
| **1** | `docker compose exec web /bin/bash` | Enter the container's interactive shell. 🐚 |
| **2** | `wger bootstrap` | Install JS/CSS, build Sass, and create initial DB tables. 🛠️ |
| **3** | `python3 manage.py sync-exercises` | Pull exercise data from the official wger.de API. 🏋️ |
| **4** | `wger load-online-fixtures` | Load global nutrition and food data. 🍎 |
| **5** | `python3 manage.py runserver 0.0.0.0:8000` | Start the development server for browser access. 🌐 |

**Access Credentials:**
* **URL:** `http://localhost:8000`
* **User:** `admin` | **Pass:** `adminadmin`

---

### **4. Troubleshooting: Resolving Migration Conflicts** 🛠️

During the setup, an `sqlite3.OperationalError` was encountered regarding an existing index: `exercises_historicalexercise_id_...`. This happens when the DB state and Django migration history fall out of sync.

#### **Resolution Strategy: The "Fake" Migration**
Instead of deleting the database and losing work, we "faked" the migration to tell Django the database already has the required structure.

| Step | Action taken | Success Indicator |
| :--- | :--- | :--- |
| **1** | **Identify Faling Migration** | Detected `exercises.0036` as the conflict point. |
| **2** | **Execute Fake Migration** | `docker compose exec web python3 manage.py migrate exercises 0036 --fake` |
| **3** | **Log Output** | `Applying exercises.0036... FAKED` ✅ |
| **4** | **Resume Migrations** | `docker compose exec web python3 manage.py migrate` |
| **5** | **Final Verification** | Gallery, Manager, and Nutrition migrations applied `OK`. 🎯 |

---

### **5. Intern Learning Summary & Future Revision** 🧠

* **Data Persistence:** Remember that generated files (like new SQLite DBs or new Migrations) created *inside* the container are **temporary**.
* **Action Required:** If you create a new migration, use `docker cp wger-dev-web-1:/home/wger/src/wger/exercises/migrations/0037_... .` to move it to your host machine. 📂
* **Environment Hygiene:** If the environment becomes too messy, use `docker compose down -v` to wipe volumes and start fresh (only for development!).

Would you like me to generate a **Python script** that automates the `docker cp` process for your migration files so you never lose them?


##### UNTIL 260321-1541


---------------------------------------------------------









---------------------------------------------------------


## Updated Report: Django Server Incident & Resolution
**Date:** March 21, 2026  
**Project:** wger-docker  
**Status:** **Resolved**

---

### 1. Incident Overview
The development server was failing with a `500 Internal Server Error` on the `/en/software/features` route. The traceback identified a `DoesNotExist` exception specifically for the `GymConfig` model with `pk=1`.

### 2. Intern's Command Execution
The intern (Saurav) performed the following sequence to diagnose and restart the environment:

* **Initial Run:** Attempted to start the server via `docker compose exec web python3 manage.py runserver 0.0.0.0:8000`.
* **Debugging:** Entered the Django shell to verify database integrity.
* **Restart:** Terminated the manual run and initiated a managed restart using `docker compose up --watch`.

---

### 3. Database Validation (Django Shell)
To resolve the `DoesNotExist` error, the intern manually verified the existence of the required configuration records:

| Step | Action | Result |
| :--- | :--- | :--- |
| **1** | **Fetch Config** | `config = GymConfig.objects.get(pk=1)` | Object successfully retrieved. |
| **2** | **Link Check** | Verified `config.gym` relationship. | ✅ Success: Linked to Gym: **'wger'**. |
| **3** | **Gym Check** | `gym = Gym.objects.get(pk=1)` | ✅ Success: ID 1 found (**Name: wger**). |

> **Conclusion from Shell:** The data is now correctly present in the database. The previous error was likely due to the database not being fully initialized or a migration/seed script not being finalized before the first request was made.

---

### 4. Resolution & Current State
The intern has successfully synchronized the database state with the application requirements.

* **Fix Applied:** Database records for `GymConfig` and `Gym` (ID 1) were confirmed/created.
* **Final Command:**
    ```bash
    docker compose up --watch
    ```
* **Current Status:** The server is running, and the `DoesNotExist` exception has been cleared. The system is now monitoring for file changes in "watch" mode.

---

### 5. Recommendation for Senior Dev
While the site is now functional, the `context_processor.py` remains a "single point of failure" if the database is empty. I recommend updating the code to handle missing configuration more gracefully:

```python
# Proposed change in wger/utils/context_processor.py
try:
    global_gymconfig = GymConfig.objects.get(pk=1)
except GymConfig.DoesNotExist:
    global_gymconfig = None  # Or a default fallback object
```

Would you like me to generate a full code snippet for this safety check or help with any other intern tasks?


---------------------------------------------------------







---------------------------------------------------------


## Updated Report: Temporary System Patch & Dashboard Verification
**Date:** March 21, 2026  
**Lead Developer:** Pankaj  
**Status:** **Functional (Temporary Fixes Applied)**

---

### 1. Applied Technical Patches
To bypass the immediate `DoesNotExist` crashes and enable dashboard testing, the following temporary modifications were made to the codebase.

#### **A. Context Processor Safety (wger/utils/context_processor.py)**
The hardcoded `get(pk=1)` call was wrapped in a `try...except` block to prevent the global header from crashing the entire application if the database record is missing.
* **Action:** Commented out original logic; implemented a `DoesNotExist` exception handler that sets `global_gymconfig = None`.

#### **B. Temporary User Provisioning (wger/tasks.py)**
To facilitate immediate login testing without a full user registration flow, a temporary admin user was injected.
* **Details:** Created user `username` with credentials `emailadmin@gmail.com` / `Cfehome199#`.
* **Profile Specs:** Set `is_temporary = True`, age `25`, height `175`.
* **Tag:** Marked with TODO `pbc260321 bpan` for future removal.

---

### 2. Server Runtime & Login Validation
The server was restarted at **13:03:07** and successfully handled high-traffic dashboard requests.

* **13:03:45:** Successful **POST** request to `/en/user/login` (Status 302).
* **13:03:46:** Full dashboard load at `/en/dashboard` (Status 200).
* **API Verification:** Multiple `api/v2/` endpoints (Routine, Nutrition, Weight, Trophies) returned Status 200, confirming the backend is serving data correctly to the frontend.

---

### 3. Dashboard UI/UX Audit
The dashboard is now fully accessible. The following components were verified as visible and active:

| Section | Status | Notes |
| :--- | :--- | :--- |
| **Navigation** | ✅ Active | wger logo, Training, Nutrition, Body weight, About. |
| **Guest Banner** | ⚠️ Warning | Displays: "You are using a guest account, data entered will be deleted after a week." |
| **Routine** | ✅ Verified | API returning status 200. |
| **Nutritional Plan** | ✅ Verified | API returning status 200. |
| **Weight & Calendar** | ✅ Verified | Chart/Log data fetched successfully. |
| **Measurements** | ✅ Verified | Category endpoints active. |

---

### 4. Required Clean-up (Action Items)
These "TODO" items must be addressed before moving to the staging environment:
1.  **Revert Tasks:** Delete the 7-line user creation block in `wger/tasks.py`.
2.  **Restore Context:** Once the database seed/migration is finalized, uncomment the 3 lines in `context_processor.py` to ensure `default_gym` logic is used.
3.  **Account Management:** Transition from the temporary `guest` credentials to a persistent admin account.

Would you like me to create a **Pull Request Summary** for these changes, or should we focus on finalizing the `GymConfig` database record to remove the temporary patch?


---------------------------------------------------------





---------------------------------------------------------


# Pull Request & Issue Resolution Report
**To:** Senior Software Architect / Lead Developer
**From:** Junior Developer / Development Intern
**Date:** March 22, 2026
**Project:** wger Fitness Manager
**Subject:** Comprehensive Technical Report on Issue #2258 - Migration to `django-allauth`

---

### **1. Executive Summary & Architectural Objectives**
* **Primary Objective:** To completely deprecate and remove the legacy `django-email-verification` package and replace it with the robust, industry-standard `django-allauth` package for managing user authentication and email verification workflows.
* **Scope of Implementation:** This task involved routing modifications, view logic refactoring, API endpoint updates, template creation, and serializer pattern adjustments to ensure seamless integration with the new package without disrupting existing client applications.
* **Architectural Justification:** Transitioning to `allauth` centralizes account management (local and social) into a single, cohesive framework, reducing technical debt, improving maintainability, and adhering to cleaner architectural boundaries by offloading authentication domain logic to a dedicated, heavily supported library.

---

### **2. Code Cleanup: Deprecation of Legacy Package (`django-email-verification`)**

#### **2.1. URL Routing Refactoring (`wger/urls.py` & `wger/core/urls.py`)**
* **Global URL Configuration:** The legacy application relied on a dedicated path inclusion for email handling.
    * Removed the import statement: `from django_email_verification import urls as email_urls`.
    * Removed the global route mapping: `path('email/', include(email_urls))`.
    * **Impact:** This cleans up the global namespace and prevents rogue requests to deprecated endpoints. `allauth` handles its own URL routing under its specific account paths, which provides a more encapsulated routing structure.
* **Core URL Configuration:**
    * Removed the sub-pattern specifically tying the `confirm_email` view to the `user` module: `path('confirm-email', user.confirm_email, name='confirm-email')`.

#### **2.2. View Layer Refactoring (`wger/core/views/user.py`)**
* **Removal of Legacy Import:** Deleted `from django_email_verification import send_email`.
* **Account Creation Logic:**
    * During the execution of `User.objects.create_user(username, email, password)`, the hardcoded trigger `send_email(user)` was removed.
    * **Reasoning:** `allauth` utilizes Django signals (specifically `user_signed_up`) to automatically handle the dispatch of verification emails. Manually triggering emails tightly couples the view to the email system, which violates the single responsibility principle.
* **Profile Update Logic:**
    * Previously, when a user updated their email via `UserPersonalInformationForm`, manual logic checked if `user_email != email_form.instance.email` to manually set `request.user.userprofile.email_verified = False`.
    * **Improvement:** `django-allauth` inherently tracks the verification status of *each* email address associated with a user through its `EmailAddress` model. The manual boolean toggle on the `UserProfile` is now redundant and dangerous, as it can fall out of sync with actual email verifications.
* **Confirmation Endpoint Deprecation:**
    * The explicit `confirm_email(request)` function block, which manually checked `if not request.user.userprofile.email_verified` to trigger `send_email(request.user)`, was completely removed.

#### **2.3. API Endpoint Refactoring (`wger/core/api/views.py`)**
* **`verify_email` Action:**
    * The legacy `@action(detail=False, url_name='verify-email', url_path='verify-email')` endpoint was stripped of the `send_email(request.user)` call.
    * **Note for Review:** The API previously returned a manual `{'status': 'sent', 'message': f'A verification email was sent...}`. This endpoint must now rely on `allauth`'s headless API features or standard forms to trigger re-sends.
* **`UserAPIRegistrationViewSet`:**
    * Cleaned up the manual `send_email(user)` invocation post-registration in the API payload, allowing the `allauth` adapter to handle communications.

---

### **3. Dependency Management & Docker Environment Configuration**

#### **3.1. Installation Challenges & Resolutions**
* **The Challenge:** Installing new dependencies (`pip install "django-allauth[socialaccount]"`) locally does not automatically propagate to isolated Docker containers, leading to `ModuleNotFoundError` during runtime. This is a common hurdle for developers transitioning to containerized microservice architectures.
* **The Solution Implemented:**
    * Utilized the `docker exec` command to open an interactive terminal session within the running container: `docker exec -it dbf551cfd450 pip install django-allauth[socialaccount]`.
    * **Architectural Note for Future:** While running `docker exec` works for immediate local development and testing, this state is ephemeral. If the container is spun down and rebuilt, the package will be lost.
    * **Required Follow-up:** The `requirements.txt` or `Pipfile`/`pyproject.toml` (whichever dependency manager wger uses) must be updated with `django-allauth[socialaccount]>=0.x.x` so that the `Dockerfile` builds the image correctly for all team members.

#### **3.2. Global Settings Configuration (`settings/settings_global.py`)**
* **App Registration:**
    * Added `'allauth'` and `'allauth.account'` to the `INSTALLED_APPS` array.
* **Understanding `SITE_ID = 1`:**
    * **Observation:** The codebase already included `SITE_ID = 1`, which is required by `allauth`.
    * **Explanation:** `SITE_ID` is a requirement of Django's built-in "Sites" framework (`django.contrib.sites`). This framework allows a single database and Django project to serve multiple distinct websites/domains. `django-allauth` uses this framework extensively because a social authentication app (like Google or GitHub login) is tied to a specific callback domain. It needs to know which "Site" the user is logging into to provide the correct redirect URIs. It was likely already present in wger for other multi-domain handling or legacy reasons.

---

### **4. Template Integration & UI Adjustments**

#### **4.1. Account Email Templates**
* **New File Creation:** Created `wger/core/templates/account/email/email_confirmation_signup_message.html`.
* **Purpose:** `allauth` requires specific template paths to generate its emails. By placing this file in `account/email/`, we are overriding the default `allauth` template with wger's custom styling and internationalization (i18n) requirements, ensuring brand consistency.
* **Git Tracking:** Used `git add .` which triggered a warning: `CRLF will be replaced by LF`. This is standard when developing on Windows (MingW64) and committing to a Linux-based repository. Git automatically normalizes line endings to prevent cross-platform formatting issues.

---

### **5. Data Structure Refactoring: The Serializer Modification**

#### **5.1. Deprecating the Hardcoded Profile Flag**
* **Legacy Model Context:** The `UserProfile` model contained `email_verified = models.BooleanField(default=False)`. This is an anti-pattern when using `allauth`, as it creates two sources of truth for a user's verification status (the `UserProfile` table vs. the `allauth_emailaddress` table).

#### **5.2. Implementing `SerializerMethodField` (`UserprofileSerializer`)**
* **The Requirement:** The API payload still needed to return the `email_verified` boolean to the frontend client without breaking existing mobile apps or frontend dashboards, but the data needed to be sourced dynamically from the new package.
* **The Implementation:**
    * Changed `email_verified` from a standard model field to `email_verified = serializers.SerializerMethodField()`.
    * Implemented the `get_email_verified(self, obj)` method to fetch the status programmatically.
* **Code Breakdown & Safety Mechanisms:**
    * ```python
        def get_email_verified(self, obj):
            try:
                # Query the allauth EmailAddress table
                email_obj = EmailAddress.objects.get(
                    user=obj.user,
                    email=obj.user.email
                )
                return email_obj.verified
            except EmailAddress.DoesNotExist:
                # Fail-safe default
                return False
        ```
    * **Why this is robust:** It explicitly queries `EmailAddress.objects.get()` matching both the specific `user` and the `email`. The `try/except` block ensures that if a user exists but does not yet have a corresponding `EmailAddress` record (which can happen during legacy data transitions), the API will safely return `False` rather than throwing a 500 Internal Server Error.

---

### **6. Pending Work: Database Data Migration Strategy**

#### **6.1. The Missing Piece**
* While the code correctly points to the new `allauth` tables, the existing user base's verification status is currently trapped in the `UserProfile.email_verified` column. If deployed as-is, all previously verified users would suddenly appear unverified.

#### **6.2. Proposed Data Migration Plan**
* **Task:** Create a Django Data Migration script.
* **Execution Steps:**
    1.  Generate an empty migration file: `python manage.py makemigrations core --empty -n migrate_legacy_email_verification`.
    2.  Write a Python function inside the migration using `RunPython`.
    3.  Iterate through all existing `UserProfile` objects where `email_verified=True`.
    4.  For each profile, fetch or create the corresponding `EmailAddress` object via `allauth`.
    5.  Set `EmailAddress.verified = True` and `EmailAddress.primary = True`, then save.
    6.  *Only after this data migration is run* can the `email_verified` column be safely dropped from the `UserProfile` model in a subsequent schema migration.

---

### **7. Version Control & Contribution Workflow Log**

#### **7.1. Commit Ledger**
* `[d18a20908]` - "remove django-email-verification package code (github issue=2258)" - Handled the initial deletion of the old dependencies across 10 files.
* `[593bcf56e]` - "remove django-email-verification package code (github issue=2258) --- additional files added" - Handled the staging of the new HTML templates that were initially missed.
* `[b74c47f81]` - "get email_verified status from allauth's EmailAddress model (github issue=2258)" - Handled the `UserprofileSerializer` logic update.

#### **7.2. Best Practices Observed**
* **Issue Tracking:** Commits explicitly linked to `(github issue=2258)` to provide traceability.
* **Branching:** Work was properly isolated on `feature/2258-migrate-to-django-allauth`.

---




---------------------------------------------------------

