# JMeter_project

Performance testing test plan (Smoke Test with 1,000 posts) for **BlogEngine.NET**.

## System Under Test (SUT) & Environment
- **Application:** BlogEngine.NET
- **Virtualization:** VirtualBox (7.0.4)
- **OS & Web Server:** Windows 10 VM with IIS installed
- **Hardware Resources:** 2 CPUs, 4096 MB Base Memory
- **Data Source:** Migrated from the file system data source to a database (DB) data source; configuration testing was performed.
- **Smoke Test Summary:** The smoke test results demonstrate the workability of both the application and the test scripts, confirming that the test scripts and the SUT are ready for further performance testing.

## Testing & Monitoring Stack
- **Apache JMeter (5.4.3):** Load Generator (LG)
- **Telegraf:** Monitoring Agent
- **InfluxDB:** Metrics Collector (`http://localhost:8086/write?db=jmeter`)
- **Grafana:** Real-time Visualization

---

## Test Plan Architecture (`TestPlan_smoke_1000_posts_task7.jmx`)

### 1. Configuration & Test Data
- **CSV Data Set Configs:**
  - `login_admin_user.csv` — Admin credentials (`USERNAME_ADMIN`, `PASSWORD_ADMIN`)
  - `login_editor_user.csv` — Editor credentials (`USERNAME`, `PASSWORD`)
  - `date_created.csv` — Post creation dates (`DATE`)
- **User Defined Variables:**
  - `PROTOCOL`: `http`
  - `DOMAIN`: `192.168.56.101`
  - `PORT`: `80`
  - `USERS`: `${__P(USERS,1)}`
  - `RAMPUP`: `${__P(RAMPUP,1)}`
  - `DURATION`: `${__P(DURATION,1200)}`
  - `MAX_NUMBER_OF_POSTS`: `${__P(MAX_NUMBER_OF_POSTS, 100)}`
  - `CURRENT_MONTH`: `02`
  - `INDEX_OF_USER`: `10`
- **HTTP Config Elements:**
  - `HTTP Request Defaults` pointing to `/blog`
  - `HTTP Cache Manager` & `HTTP Cookie Manager` (configured to clear on each iteration)

### 2. Thread Groups (User Roles & Scenarios)

#### Admin Thread Group (`Admin`)
1. Opens the blog home page (`/blog`).
2. Logs in as an Administrator (`POST /blog/Account/login.aspx`).
3. Executes a loop (10 iterations):
   - Opens the Admin dashboard (`GET /blog/admin/`).
   - Queries the users list (`GET /blog/api/users`) and extracts the total user count (`NUMBER_OF_USERS_matchNr`) and an existing test user (`User_*`).
   - Dynamically manages user count via `IfController`s:
     - **If users < 10:** Calls `Add_User` (`POST /blog/api/users`).
     - **If users > 20:** Calls `Delete_User` (`PUT /blog/api/users/processchecked/delete`).
     - **If 10 <= users <= 20:** Randomly executes either `Add_User` or `Delete_User`.
4. Logs out (`GET /blog/Account/login.aspx?logoff`).

#### Editor Thread Group (`Editor`)
1. Opens the blog home page (`/blog`).
2. Logs in as an Editor (`POST /blog/Account/login.aspx`).
3. Executes a loop (50 iterations):
   - Opens posts for a predefined date (`GET /blog/2023/02/16/default`) and extracts a random post link.
   - Opens the selected post (`GET ${randomPostPath}`) and extracts its post ID.
   - Edits and updates the post (`PUT /blog/api/posts/update/foo`) with randomized title and content.
4. Logs out (`GET /blog/Account/login.aspx?logoff`).

#### Anonymous User Thread Group (`Anonymous`)
1. Opens the blog home page (`/blog`).
2. Executes a loop (50 iterations) using the BlazeMeter **Weighted Switch Controller** to simulate realistic browsing distribution across 6 user paths:
   - **15%** — `OpenHomePageUserPath_15%` (Home page browsing)
   - **10%** — `OpenRandomDateUserPath_10%` (Browsing posts by a randomly generated date in `2023/${CURRENT_MONTH}/${RANDOM_DAY}`)
   - **30%** — `OpenPredefinedDateUserPath_30%` (Browsing posts for `2023/02/10`)
   - **30%** — `SearchByNameUserPath_30%` (Searching posts via `GET /blog/search?q=Welcome+to+BlogEngine.NET`)
   - **10%** — `OpenLargeCalendarUserPath_10%` (Viewing the calendar via `GET /blog/calendar/default.aspx`)
   - **5%** — `OpenContactsUserPath_5%` (Viewing the contact page via `GET /blog/contact`)
3. Within the primary browsing paths, nested **Weighted Switch Controllers** model user behavior:
   - **50% / 50%** chance to visit random navigation pages (`Login`, `Contact`, `Archive` in random order).
   - **80% / 20%** chance to open a post (**35%** first post vs. **65%** random post) or end the flow.
   - **20% / 80%** chance to submit a comment (`POST /blog/post/welcome-to-blogengine-net`) or end the flow.

### 3. Modular Test Fragments & Pacing
- **`User actions` Test Fragment:** Centralizes reusable `TransactionController` modules referenced across Thread Groups via `ModuleController`s.
- **Think Time:** Simulated using `Gaussian Random Timer` (5,000 ms constant delay, 4,000 ms deviation) inside `Flow Control Action` elements.

### 4. Listeners & Reporting
- **InfluxDB Backend Listener:** Streams real-time test metrics (`Blog` application, percentiles `50;90;95;99`) to InfluxDB for Grafana dashboards.
- **JMeter Listeners:** View Results Tree, Aggregate Report, Console Status Logger, Active Threads Over Time, Response Times Over Time, Transactions per Second, and Composite Graph.

---

## Running the Test Plan

### Prerequisites
- **Apache JMeter 5.4.3+**
- **JMeter Plugins:**
  - BlazeMeter Weighted Switch Controller (`bzm - Weighted Switch Controller`)
  - Custom Thread Groups / Graphs (`jp@gc` plugins)

### CLI Execution
```bash
jmeter -n -t TestPlan_smoke_1000_posts_task7.jmx \
  -JUSERS=1 \
  -JRAMPUP=1 \
  -JDURATION=1200 \
  -JMAX_NUMBER_OF_POSTS=100 \
  -l results.jtl
```
