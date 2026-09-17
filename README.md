# Part 2: Getting onto MUNDRA

[Part 1](./part-1-backend-fundamentals.md) was the theory. This one is hands on. By the end of today you'll have
MUNDRA running on your own laptop, you'll have written a small FastAPI app from scratch, and
you'll have called every single endpoint in our system yourself.

I'm assuming you've never touched FastAPI before. That's fine. Type everything out rather
than copy pasting. It's slower, and that's the point: your fingers learn the shape of the
code faster than your eyes do.

---

## Contents

1. [What you'll have by the end of today](#1-what-youll-have-by-the-end-of-today)
2. [Set up your machine](#2-set-up-your-machine)
3. [Your first FastAPI app (type this out)](#3-your-first-fastapi-app-type-this-out)
4. [How MUNDRA is laid out](#4-how-mundra-is-laid-out)
5. [Calling the API from Swagger](#5-calling-the-api-from-swagger)
6. [How login works](#6-how-login-works)
7. [Every endpoint, explained](#7-every-endpoint-explained)
8. [One full story: signup to lunch](#8-one-full-story-signup-to-lunch)
9. [Exercises on the real code](#9-exercises-on-the-real-code)
10. [When things break](#10-when-things-break)
11. [Cheat sheet](#11-cheat-sheet)

---

## 1. What you'll have by the end of today

- [ ] MUNDRA running at `http://127.0.0.1:8000`
- [ ] A tiny FastAPI app you wrote yourself, with all four CRUD operations
- [ ] An admin account and a delegate account on your local database
- [ ] Every endpoint in section 7 called at least once
- [ ] One new endpoint added to MUNDRA by you (section 9)

Tick them off as you go.

---

## 2. Set up your machine

Every step has a **macOS** version and a **Windows** version. Where a command is the same
on both, I've only written it once. On Windows, use **PowerShell** for this whole section
(search for it in the Start menu).

Come find me if you get stuck here. Setup is the most annoying part of the whole day and
it's not worth losing hours to.

### 2.1 Install the tools

| Tool | What it's for |
|---|---|
| **git** | Getting the code, and sending your changes back |
| **PostgreSQL** | The database. Runs quietly in the background on your laptop |
| **uv** | Installs Python and all our packages for you, and runs commands inside the project |

**macOS**

```bash
# Homebrew first, if you don't have it: https://brew.sh
brew install git
brew install postgresql@16
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Windows**

```powershell
winget install --id Git.Git -e
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

For Postgres on Windows, download the installer from
[postgresql.org/download/windows](https://www.postgresql.org/download/windows/) (pick
version 16) and run it. While it installs:

- It asks you to set a password for the `postgres` user. **Write this down**, you need it
  in step 2.5.
- Leave the port as `5432`.
- You can untick Stack Builder at the end, we don't need it.

Then tell PowerShell where the Postgres commands live. Search the Start menu for **"Edit
the system environment variables"**, click **Environment Variables**, select **Path** under
your user variables, click **Edit**, then **New**, and add:

```
C:\Program Files\PostgreSQL\16\bin
```

Whichever OS you're on, **close and reopen your terminal** after installing everything so
the new commands show up.

### 2.2 Start Postgres

**macOS**

```bash
brew services start postgresql@16
pg_isready
```

If `brew services` throws a weird Ruby error (it did on my machine), start it directly
instead:

```bash
pg_ctl -D /opt/homebrew/var/postgresql@16 -l /opt/homebrew/var/log/postgresql@16.log start
```

**Windows**

The installer sets Postgres up as a service that starts by itself, so it's usually already
running. Check:

```powershell
pg_isready
```

If it isn't running, start it:

```powershell
Start-Service postgresql-x64-16
```

On both, you want `pg_isready` to say `accepting connections`.

### 2.3 Get the code and install packages

Same on both:

```bash
git clone https://github.com/munsoc-mpstme/mundra
cd mundra
uv sync
```

`uv sync` reads `pyproject.toml`, downloads the right Python version, and installs
everything into a `.venv` folder inside the project. You never have to activate anything,
you just put `uv run` in front of commands.

### 2.4 Create your database

**macOS**

```bash
createdb mundra
psql mundra -c "select current_user;"
```

The second command prints your Postgres username. On a Mac it's usually the same as your
laptop username and there's no password. Remember it, you need it in the next step.

**Windows**

```powershell
createdb -U postgres mundra
psql -U postgres -d mundra -c "select current_user;"
```

Both ask for the password you set during install. Your username is `postgres`.

### 2.5 Make your `.env`

**macOS**

```bash
cp sample.env .env
```

**Windows**

```powershell
Copy-Item sample.env .env
```

Now open `.env` in VS Code and fill it in. The only line that differs between the two is
`DATABASE_URL`.

```ini
SECRET_KEY=paste-a-long-random-string-here
DATABASE_URL=see-below
MAIL_SERVER=localhost
MAIL_PASSWORD=
URL=http://localhost:8000
DOCS_URL=/swagger
REDOC_URL=/docs
```

For `DATABASE_URL`:

| OS | Use |
|---|---|
| macOS | `postgresql+psycopg2://yourname@localhost:5432/mundra` (your username from 2.4, no password) |
| Windows | `postgresql+psycopg2://postgres:yourpassword@localhost:5432/mundra` (the password from the installer) |

To get a random secret key, same on both:

```bash
uv run python -c "import secrets; print(secrets.token_hex(32))"
```

Two things I learned the hard way here:

- **Don't leave lines blank if you don't mean empty.** `DOCS_URL=` does not mean "use the
  default", it means "set this to an empty string", and it breaks the docs page. Either
  fill it in or delete the line.
- **Never commit `.env`.** It's already in `.gitignore`. Keep it that way.

`MAIL_SERVER=localhost` means emails won't actually send on your laptop. That's fine,
section 10 explains what that looks like and how to work around it.

If your Postgres password has symbols like `@`, `#` or `/` in it, the URL breaks. Either
pick a password without them, or ask me how to encode it.

### 2.6 Create the tables

Same on both:

```bash
uv run alembic upgrade head
```

Alembic runs every migration file in `alembic/versions/` and builds the tables. To check
it worked, list them:

**macOS**

```bash
psql mundra -c "\dt"
```

**Windows**

```powershell
psql -U postgres -d mundra -c "\dt"
```

You should see `admins`, `delegates`, `users`, `mun_experiences`, `mm_delegates`,
`mm_mun_experiences`, `refresh_tokens` and `alembic_version`.

### 2.7 Run it

Same on both:

```bash
uv run uvicorn main:app --reload
```

What that command means:

| Part | Meaning |
|---|---|
| `uv run` | Run this inside our project's `.venv` |
| `uvicorn` | The server program that actually handles web requests |
| `main:app` | Look in the file `main.py` for the variable called `app` |
| `--reload` | Restart automatically every time you save a file |

You should see `Uvicorn running on http://127.0.0.1:8000`. Leave that terminal open, and
open [http://127.0.0.1:8000/](http://127.0.0.1:8000/) in your browser. You want:

```json
{"message":"Server is up and running"}
```

If you see that, you're running MUNDRA. Now open
[http://127.0.0.1:8000/swagger](http://127.0.0.1:8000/swagger). That page lists every
endpoint we have, and you can call them from there. We'll use it a lot.

To stop the server, press `Ctrl+C` in that terminal. If port 8000 is already taken by
something else, add `--port 8001` to the end and use that port instead.

### 2.8 A note for Windows about the rest of this guide

From section 6 onwards I use `curl` commands written for a Mac terminal (things like
`export B=...`). These don't work in PowerShell. Git for Windows came with a program called
**Git Bash** that understands them, so for those sections open Git Bash (it's in the Start
menu) and `cd` into the `mundra` folder. Everything else, including running the server,
can stay in PowerShell. Or skip `curl` entirely and use Swagger, which works the same
everywhere.

---

## 3. Your first FastAPI app (type this out)

Before touching MUNDRA, build something tiny yourself. It makes everything in MUNDRA make
sense afterwards. Do this in a separate folder, not inside `mundra`.

```bash
cd ~
mkdir fastapi-playground
cd fastapi-playground
uv init
uv add "fastapi[standard]"
```

### 3.1 Hello world

Open `main.py`, delete whatever is in it, and type this:

```python
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def home():
    return {"message": "hello from my first API"}
```

Run it, the same way as MUNDRA:

```bash
uv run uvicorn main:app --reload
```

(If MUNDRA is still running on port 8000, stop it with `Ctrl+C` first, or add
`--port 8001` here.)

Open `http://127.0.0.1:8000/` and `http://127.0.0.1:8000/docs`. The second one is the docs
page FastAPI built for you from those six lines.

### 3.2 A path parameter

Add this under `home`:

```python
@app.get("/hello/{name}")
def hello(name: str):
    return {"message": f"hello {name}"}
```

Save, then open `http://127.0.0.1:8000/hello/ada`. Whatever you put after `/hello/` ends up
in the `name` variable.

### 3.3 A query parameter

```python
@app.get("/add")
def add(a: int, b: int):
    return {"result": a + b}
```

Try `http://127.0.0.1:8000/add?a=2&b=3`. Now try `?a=2&b=banana`. You get a `422` error
explaining that `b` isn't a number. You didn't write that check, FastAPI did it because
you typed `b: int`.

### 3.4 Full CRUD on something small

Now the real exercise. We'll keep a list of committees in a plain Python dict. No database
yet, that comes later. Replace everything in `main.py` with this:

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel

app = FastAPI()


class Committee(BaseModel):
    name: str
    agenda: str
    seats: int


committees: dict[int, Committee] = {}
next_id = 1


# CREATE
@app.post("/committees", status_code=201)
def create_committee(committee: Committee):
    global next_id
    committees[next_id] = committee
    next_id += 1
    return {"id": next_id - 1, **committee.model_dump()}


# READ all
@app.get("/committees")
def list_committees():
    return [{"id": cid, **c.model_dump()} for cid, c in committees.items()]


# READ one
@app.get("/committees/{committee_id}")
def get_committee(committee_id: int):
    if committee_id not in committees:
        raise HTTPException(status_code=404, detail="Committee not found")
    return {"id": committee_id, **committees[committee_id].model_dump()}


# UPDATE
@app.put("/committees/{committee_id}")
def update_committee(committee_id: int, committee: Committee):
    if committee_id not in committees:
        raise HTTPException(status_code=404, detail="Committee not found")
    committees[committee_id] = committee
    return {"id": committee_id, **committee.model_dump()}


# DELETE
@app.delete("/committees/{committee_id}")
def delete_committee(committee_id: int):
    if committee_id not in committees:
        raise HTTPException(status_code=404, detail="Committee not found")
    del committees[committee_id]
    return {"message": "Committee deleted"}
```

Go to `/docs` and, using only the Swagger page:

1. Create UNSC, DISEC and UNHRC with `POST /committees`
2. List them with `GET /committees`
3. Fetch one with `GET /committees/2`
4. Change DISEC's seats with `PUT /committees/2`
5. Delete one, then try to fetch it again and check you get a `404`

Then restart the server and list them again. They're gone. That's why we need a database,
a dict only lives as long as the program does.

### 3.5 Split it into a router

Last step, and it's exactly how MUNDRA is organised. Make a folder and two files:

```
fastapi-playground/
├── main.py
└── routers/
    ├── __init__.py        (leave this empty)
    └── committees.py
```

Move all five committee endpoints into `routers/committees.py`, with two changes:

```python
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel

router = APIRouter()          # was: app = FastAPI()

# ...the class, the dict and next_id stay the same...


@router.post("", status_code=201)     # was: @app.post("/committees", ...)
def create_committee(committee: Committee):
    ...


@router.get("/{committee_id}")        # was: @app.get("/committees/{committee_id}")
def get_committee(committee_id: int):
    ...
```

Every `@app.` becomes `@router.`, and every path loses the `/committees` at the front.
Now `main.py` becomes:

```python
from fastapi import FastAPI

from routers.committees import router as committees_router

app = FastAPI()
app.include_router(committees_router, prefix="/committees", tags=["Committees"])
```

Check `/docs`. The URLs haven't changed at all, because `prefix="/committees"` puts the
part you removed back on. That's the whole trick behind MUNDRA's `routers/` folder.

---

## 4. How MUNDRA is laid out

Now open the `mundra` folder in VS Code. Here's what everything is:

```
mundra/
├── main.py              creates the app and plugs in every router
├── routers/             the actual endpoints, one file per area
│   ├── auth.py          signup, login, tokens, passwords
│   ├── delegates.py     delegate profiles
│   ├── mumbaimun.py     Mumbai MUN registration
│   ├── qr.py            QR codes and the scanner page
│   ├── food.py          meal check in
│   ├── admin.py         admin tools
│   └── dynamic_data.py  rooms and schedule
├── auth.py              password hashing, tokens, "who is logged in"
├── database.py          every function that reads or writes the database
├── db_models.py         the tables
├── models.py            the shapes of data going in and out of the API
├── config.py            reads your .env
├── mails.py             sends emails
├── utils.py             makes QR code images
├── rate_limiter.py      stops people spamming login
├── templating.py        for the few HTML pages we serve
├── alembic/             migrations
├── templates/           those HTML pages
├── data/                rooms.json and schedule.json
└── static/              images
```

Look at `main.py` first. It's about 60 lines and it's the table of contents for the whole
app. These lines are the important ones:

```python
app.include_router(auth_router, tags=["Auth"])
app.include_router(delegates_router, prefix="/delegates", tags=["Delegates"])
app.include_router(mumbaimun_router, prefix="/mumbaimun", tags=["Mumbai MUN"])
app.include_router(qr_router, tags=["QR"])
app.include_router(food_router, tags=["Food"])
app.include_router(admin_router, tags=["Admin"])
app.include_router(dynamic_data_router, tags=["Dynamic Data"])
```

Same pattern you just built in 3.5. So if you want to know where `GET /delegates/me` lives,
it's `routers/delegates.py`, on a function decorated `@router.get("/me")`.

### The three files that confuse everyone

| File | What's in it | Think of it as |
|---|---|---|
| `models.py` | Pydantic classes | What the app sends us and what we send back |
| `db_models.py` | SQLAlchemy classes | What's actually stored in Postgres |
| `database.py` | Plain functions | The translator between the two |

A route never talks to Postgres directly. It calls a function in `database.py`, and gets a
`models.py` object back. Keep that in your head and the code reads much more easily.

### Who uses MUNDRA

Three kinds of people hit our API, and you'll see all three in section 7:

| Who | How they use it |
|---|---|
| **Delegates** | Through the Delego app. Sign up, log in, edit their profile, see their QR code, rooms and schedule. |
| **Admins (OC)** | Through Swagger or scripts. See every delegate, export CSVs, take backups. |
| **Volunteers on the day** | Through the `/scan` and `/food` web pages, to check people in for meals. |

---

## 5. Calling the API from Swagger

Open [http://127.0.0.1:8000/swagger](http://127.0.0.1:8000/swagger). Every endpoint is
listed, grouped by the tags from `main.py`.

To call one: click it, click **Try it out**, fill in the fields, click **Execute**. Below
that you'll see the exact `curl` command Swagger ran, the status code, and the response.
Reading those `curl` commands is a good way to learn what a request actually looks like.

### 5.1 Logging in inside Swagger

Endpoints with a padlock icon need you to be logged in. At the top right there's an
**Authorize** button. Click it and you get a username and password form.

- **username**: your email (the field is called username for historical reasons, it's an
  OAuth2 standard thing)
- **password**: your password
- leave everything else blank

Click Authorize. Swagger calls `POST /login` for you, keeps the token, and attaches it to
every padlocked request from then on. The padlocks close.

Tokens expire after 10 minutes (section 6), so if padlocked calls suddenly start failing
with `403`, click Authorize again.

### 5.2 Making yourself an admin

There's no endpoint that creates admins, on purpose. You make one straight in the
database. First get a hashed password (we never store real passwords) by opening this in
your browser:

```
http://127.0.0.1:8000/hash_password?password=adminpass123
```

Copy the string it shows, the one starting `$2b$12$`, without the quote marks around it.
Then open Postgres:

**macOS**

```bash
psql mundra
```

**Windows**

```powershell
psql -U postgres -d mundra
```

and at the `mundra=#` prompt type (with your own hash pasted in):

```sql
INSERT INTO admins (email, password)
VALUES ('admin@munsoc.test', '$2b$12$paste-the-rest-of-your-hash-here');
```

Type `\q` to leave. Now you can Authorize in Swagger as `admin@munsoc.test` with
`adminpass123`.

### 5.3 Making yourself a delegate

In Swagger, open `POST /mumbaimun/register`, Try it out, and send:

```json
{
  "firstname": "Ada",
  "lastname": "Lovelace",
  "email": "ada@munsoc.test",
  "password": "password123"
}
```

**You will get a 500 error about port 465.** That's expected on your laptop, it's the email
step failing because there's no real mail server. The account was still created. Section
10 has the full explanation. Log out of the admin in Authorize, and log back in as Ada.

---

## 6. How login works

This is the bit most people find confusing, so here it is slowly.

When you log in, you get **two** tokens back:

| Token | Lives for | What it's for |
|---|---|---|
| **access token** | 10 minutes | Sent with every request to prove who you are |
| **refresh token** | 7 days | Only used to get a new access token when the old one runs out |

Why two? If someone steals your access token, it's useless to them within 10 minutes. The
refresh token is more powerful, but it's only ever sent to one endpoint, and we can cancel
it from the server side when you log out.

```mermaid
sequenceDiagram
    participant App
    participant API as MUNDRA

    App->>API: POST /login (email, password)
    API-->>App: access token (10 min) + refresh token (7 days)

    App->>API: GET /delegates/me<br/>Authorization: Bearer <access>
    API-->>App: 200, your profile

    Note over App,API: ...10 minutes later...

    App->>API: GET /delegates/me<br/>Authorization: Bearer <access>
    API-->>App: 403, token expired

    App->>API: POST /refresh (refresh token)
    API-->>App: a fresh access token

    App->>API: POST /logout (refresh token)
    API-->>App: 200, refresh token cancelled
```

### Sending a token by hand

In `curl`, tokens go in a header:

```bash
curl http://127.0.0.1:8000/delegates/me \
  -H "Authorization: Bearer eyJhbGciOi...your-token..."
```

To save typing, keep the token in a shell variable for the rest of the session:

```bash
export B=http://127.0.0.1:8000

export TOKEN=$(curl -s -X POST $B/login \
  -d "username=ada@munsoc.test&password=password123" \
  | python3 -c "import sys, json; print(json.load(sys.stdin)['access_token'])")

curl $B/delegates/me -H "Authorization: Bearer $TOKEN"
```

Every `curl` example below assumes `$B` and `$TOKEN` are set like this.

### Look inside a token

Copy an access token and paste it into [jwt.io](https://jwt.io). You'll see your email and
the expiry time sitting there in plain text. Tokens are **signed**, meaning nobody can
change them without us noticing, but they are **not encrypted**. Never put anything secret
inside one.

### The code behind it

Open `auth.py` and find `get_current_user`. Any route with this in its parameters:

```python
user: models.Delegate | models.Admin = Depends(get_current_user)
```

runs `get_current_user` first, before the route's own code. That function reads the token,
checks it, looks the email up, and hands the route either an `Admin` or a `Delegate`. If
the token is bad, the route never runs at all. That's what `Depends` means.

---

## 7. Every endpoint, explained

Every endpoint in MUNDRA, grouped the same way Swagger groups them. For each one:

- **Who** can call it: anyone, a logged in delegate, or an admin
- **Send**: what goes in the request
- **Try it**: a `curl` you can run (with `$B` and `$TOKEN` set from section 6)
- **Back**: what a successful response looks like
- **Errors** worth knowing
- **Code**: which file to open

A quick key for **Who**:

| Label | Meaning |
|---|---|
| Anyone | No login needed |
| Delegate | Needs a delegate's access token in the header |
| Admin | Needs an admin's access token |
| Delegate (self) or Admin | A delegate can only do this to their own record, an admin to anyone's |

---

### 7.1 Status

#### `GET /`

The "are you alive?" check.

- **Who:** Anyone
- **Try it:** `curl $B/`
- **Back:** `{"message": "Server is up and running"}`
- **Code:** `main.py`

#### `GET /static/{filename}`

Serves images from the `static/` folder, like the logo and the schedule pictures. The
browser is told to cache them for a day.

- **Who:** Anyone
- **Try it:** open `http://127.0.0.1:8000/static/logo.jpg`
- **Errors:** `404` if the file doesn't exist
- **Code:** `main.py`

---

### 7.2 Auth

#### `POST /register`

Creates a normal (non Mumbai MUN) account and emails a verification link.

- **Who:** Anyone
- **Send:** JSON body. Password must be at least 8 characters.
  ```json
  {"firstname": "Ada", "lastname": "Lovelace", "email": "ada@munsoc.test", "password": "password123"}
  ```
- **Try it:**
  ```bash
  curl -X POST $B/register -H "Content-Type: application/json" \
    -d '{"firstname":"Ada","lastname":"Lovelace","email":"ada@munsoc.test","password":"password123"}'
  ```
- **Back:** `201` with `{"message": "User created successfully. Please verify your email."}`
- **Errors:** `409` if the email already has an account. `422` if a field is missing or the
  password is too short. `500` locally because email can't send (the account is still made).
- **Rate limit:** 10 per minute per IP
- **Code:** `routers/auth.py`

What happens inside: if there's already a delegate record for that email (say an admin
pre registered them) we reuse it, otherwise we create one. Then we create the login and
send the email. The new delegate starts **unverified** and can't use padlocked endpoints
until they click the link.

#### `POST /login`

Swaps an email and password for tokens.

- **Who:** Anyone
- **Send:** a **form**, not JSON. The email goes in a field called `username`.
- **Try it:**
  ```bash
  curl -X POST $B/login -d "username=ada@munsoc.test&password=password123"
  ```
- **Back:**
  ```json
  {
    "access_token": "eyJhbGciOi...",
    "refresh_token": "eyJhbGciOi...",
    "token_type": "bearer",
    "user_type": "user"
  }
  ```
  `user_type` is `"admin"` or `"user"`.
- **Errors:** `401` with `"Invalid email"` or `"Invalid password"`
- **Rate limit:** 10 per minute
- **Code:** `routers/auth.py`

It checks the `admins` table first and the `users` table second, so an email that exists in
both logs in as an admin.

#### `POST /refresh`

Gets a new access token using your refresh token.

- **Who:** Anyone holding a valid refresh token
- **Send:** `{"refresh_token": "eyJhbGciOi..."}`
- **Try it:**
  ```bash
  curl -X POST $B/refresh -H "Content-Type: application/json" \
    -d '{"refresh_token":"PASTE_REFRESH_TOKEN"}'
  ```
- **Back:** the same shape as `/login`. The access token is new, the refresh token is the
  same one you sent.
- **Errors:** `401` if the refresh token is fake, expired, logged out, or is actually an
  access token
- **Rate limit:** 10 per minute
- **Code:** `routers/auth.py`

#### `POST /logout`

Cancels a refresh token so it can never be used again.

- **Who:** Anyone
- **Send:** `{"refresh_token": "eyJhbGciOi..."}`
- **Try it:**
  ```bash
  curl -X POST $B/logout -H "Content-Type: application/json" \
    -d '{"refresh_token":"PASTE_REFRESH_TOKEN"}'
  ```
- **Back:** `{"message": "Logged out successfully"}`
- **Code:** `routers/auth.py`

Try `/refresh` with the same token afterwards and watch it get refused. The access token you
already had keeps working until its 10 minutes are up. That's normal, and it's one reason
access tokens are kept short.

#### `GET /verify_email?token=...`

The link inside the verification email. Clicking it marks the delegate as verified.

- **Who:** Anyone with the link
- **Send:** `token` as a query parameter
- **Back:** `{"message": "Email verified!"}`
- **Errors:** `401` if the link has expired (2 hours by default), `403` if the token is invalid
- **Rate limit:** 10 per minute
- **Code:** `routers/auth.py`

Locally you won't get the email, so use `POST /manual_verify` instead (section 7.6).

#### `GET /resend_verification?email=...`

Sends the verification email again.

- **Who:** Anyone
- **Try it:** `curl "$B/resend_verification?email=ada@munsoc.test"`
- **Back:** `{"message": "Verification email sent!"}`
- **Errors:** `404` if no delegate has that email, `409` if they're already verified,
  `500` locally because email can't send
- **Rate limit:** 10 per minute
- **Code:** `routers/auth.py`

#### `GET /forgot_password?email=...`

Emails a link to reset your password.

- **Who:** Anyone
- **Try it:** `curl "$B/forgot_password?email=ada@munsoc.test"`
- **Back:** `{"message": "Password reset email sent!"}`
- **Errors:** `404` if the email isn't found, `403` if the account isn't verified yet
- **Rate limit:** **1 per minute**, so don't be surprised by a `429` if you try it twice
- **Code:** `routers/auth.py`

#### `GET /reset?token=...`

The page that link opens. It's an HTML page from `templates/reset.html`, not JSON.

- **Who:** Anyone with a valid token in the link
- **Code:** `routers/auth.py`

#### `PATCH /change_pass?password=...`

Changes the logged in delegate's password.

- **Who:** Delegate (admins get a `403`)
- **Send:** the new password as a query parameter
- **Try it:**
  ```bash
  curl -X PATCH "$B/change_pass?password=newpassword123" -H "Authorization: Bearer $TOKEN"
  ```
- **Back:** `{"message": "Password changed!"}`
- **Rate limit:** **1 per minute**
- **Code:** `routers/auth.py`

#### `DELETE /account`

Deletes the logged in delegate's login.

- **Who:** Delegate
- **Try it:** `curl -X DELETE $B/account -H "Authorization: Bearer $TOKEN"`
- **Back:** `{"message": "Account deleted successfully"}`
- **Errors:** `500` with `"You are an admin"` if an admin calls it
- **Code:** `routers/auth.py`

This only deletes the **login**. The delegate record and their conference registration
stay. That's deliberate: someone can delete their account without us losing the fact that
they attended. After this, logging in gives `401 Invalid email`.

---

### 7.3 Delegates

#### `GET /delegates/me`

Your own profile. This is what the Delego app calls right after login.

- **Who:** Delegate
- **Try it:** `curl $B/delegates/me -H "Authorization: Bearer $TOKEN"`
- **Back:**
  ```json
  {
    "id": "896314eac2024436b9d64160e045cbf7",
    "firstname": "Ada",
    "lastname": "Lovelace",
    "email": "ada@munsoc.test",
    "contact": "",
    "dateofbirth": "",
    "gender": "",
    "pastmuns": [],
    "verified": true
  }
  ```
- **Errors:** `401` with `"Please verify your email!"` if unverified. `500` with
  `"You are an admin"` if an admin calls it.
- **Code:** `routers/delegates.py`

Copy your `id` from here, the next few endpoints need it:

```bash
export ID=paste-your-id-here
```

#### `GET /delegates/{id}`

One delegate by id.

- **Who:** Delegate (self) or Admin
- **Try it:** `curl $B/delegates/$ID -H "Authorization: Bearer $TOKEN"`
- **Back:** same shape as `/delegates/me`
- **Errors:** `403` if a delegate asks for someone else, `404` if an admin asks for an id
  that doesn't exist
- **Code:** `routers/delegates.py`

#### `PATCH /delegates/{id}`

Edits a delegate's profile. The Delego profile screen uses this.

- **Who:** Delegate (self) or Admin
- **Send:** any of these as **query parameters**: `firstname`, `lastname`, `contact`,
  `dateofbirth`, `gender`, `verified`. Anything you leave out stays as it is. To replace
  someone's past MUN list, also send a **JSON body**:
  ```json
  [
    {"name": "HarvardMUN", "committee": "UNSC", "delegation": "India", "year": 2024, "award": "Best Delegate"}
  ]
  ```
- **Try it:**
  ```bash
  # just a field
  curl -X PATCH "$B/delegates/$ID?contact=9876543210" -H "Authorization: Bearer $TOKEN"

  # a value with spaces has to be encoded, %20 is a space
  curl -X PATCH "$B/delegates/$ID?gender=Prefer%20not%20to%20say" -H "Authorization: Bearer $TOKEN"

  # past MUNs
  curl -X PATCH "$B/delegates/$ID" -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '[{"name":"HarvardMUN","committee":"UNSC","delegation":"India","year":2024,"award":"Best Delegate"}]'
  ```
- **Back:** the updated profile
- **Errors:** `403` if a delegate edits someone else, `404` if the id doesn't exist
- **Code:** `routers/delegates.py`

Two quirks. An empty value means "don't change it", so there's no way to clear a field back
to blank. And sending `pastmuns` replaces the whole list, it doesn't add to it.

#### `GET /delegates?token=...&format=...`

Every delegate. Add `format=csv` for a spreadsheet.

- **Who:** Admin
- **Send:** the admin token as a **query parameter** called `token`. This is the one route
  that doesn't use the header, so the Swagger padlock won't help you here, paste the token
  into the `token` box yourself.
- **Try it:**
  ```bash
  curl "$B/delegates?token=$ADMIN_TOKEN"
  curl "$B/delegates?token=$ADMIN_TOKEN&format=csv" -o delegates.csv
  ```
- **Back:** a list of delegates, or a CSV file
- **Errors:** `403` if the token isn't an admin's, `404` if there are no delegates yet
- **Code:** `routers/delegates.py`

(Get `$ADMIN_TOKEN` the same way as `$TOKEN` in section 6, but log in as the admin.)

---

### 7.4 Mumbai MUN

#### `POST /mumbaimun/register`

Registers someone for Mumbai MUN. **This is the signup the Delego app actually uses.**

- **Who:** Anyone
- **Send:** the same JSON as `/register`
- **Try it:**
  ```bash
  curl -X POST $B/mumbaimun/register -H "Content-Type: application/json" \
    -d '{"firstname":"Grace","lastname":"Hopper","email":"grace@munsoc.test","password":"password123"}'
  ```
- **Back:** `201`, with one of these messages:
  - brand new person: `"User with id <id> created successfully!"`
  - already had an account: `"Mumbai MUN Delegate registered successfully! ID: <id>"`
- **Errors:** `409` if they're already registered for Mumbai MUN, `400` if a login exists
  with no delegate record, `500` locally from the email step
- **Code:** `routers/mumbaimun.py`

This one is different from `/register` in a few ways. It marks the person as **verified
straight away**, so they can log in without clicking an email. It also creates a row in
`mm_delegates` with the **same id** as their normal delegate record, which is where their
country, committee and meal ticks live.

#### `GET /mumbaimun/delegates?format=...`

Every Mumbai MUN delegate, including country, committee and all nine meal flags.

- **Who:** Admin
- **Try it:**
  ```bash
  curl $B/mumbaimun/delegates -H "Authorization: Bearer $ADMIN_TOKEN"
  curl "$B/mumbaimun/delegates?format=csv" -H "Authorization: Bearer $ADMIN_TOKEN" -o mm.csv
  ```
- **Errors:** `403` if not an admin, `404` if nobody is registered yet
- **Code:** `routers/mumbaimun.py`

---

### 7.5 QR codes and food

These are what volunteers use on conference day. The flow is: a delegate shows the QR code
in their app, a volunteer scans it on `/scan`, which opens `/food` for that delegate, and
the volunteer ticks the meal.

#### `GET /qr?id=...`

A delegate's QR code as a JPEG image. The QR code simply contains their id.

- **Who:** Anyone
- **Try it:** open `http://127.0.0.1:8000/qr?id=YOUR_ID` in the browser
- **Code:** `routers/qr.py`

The first time you ask for an id, the image gets generated and saved in `qrcodes/`. After
that it's served from the saved file.

#### `GET /scan`

The camera scanner page volunteers open on their phones. It's HTML.

- **Who:** Anyone
- **Try it:** open `http://127.0.0.1:8000/scan`
- **Code:** `routers/qr.py`, page in `templates/scan.html`

#### `GET /food?id=...`

The meal checklist page for one Mumbai MUN delegate. HTML.

- **Who:** Anyone
- **Try it:** open `http://127.0.0.1:8000/food?id=YOUR_ID` (must be someone registered
  through `/mumbaimun/register`)
- **Errors:** `404` if that id isn't a Mumbai MUN delegate
- **Code:** `routers/food.py`, page in `templates/food.html`

#### `POST /food`

What the checklist page submits.

- **Who:** Anyone. Yes really, see section 10.
- **Send:** a **form** with `id` plus any of the nine meal fields: `d1_bf`, `d1_lunch`,
  `d1_hitea`, `d2_bf`, `d2_lunch`, `d2_hitea`, `d3_bf`, `d3_lunch`, `d3_hitea`
  (`d1_bf` = day 1 breakfast, `hitea` = high tea)
- **Try it:**
  ```bash
  curl -X POST $B/food -d "id=$ID&d1_bf=true&d1_lunch=true"
  ```
- **Back:** `201` with `{"message": "Food updated successfully"}`
- **Code:** `routers/food.py`

Careful: this sets **all nine** every time. Any meal you don't send goes back to its default
(false, except day 1 breakfast which defaults to true). That matches how HTML checkboxes
work, the page always sends the whole form.

---

### 7.6 Admin and OC

#### `GET /hash_password?password=...`

Turns a password into the scrambled form we store. Only really used to create admins by
hand (section 5.2).

- **Who:** Anyone
- **Try it:** `curl "$B/hash_password?password=adminpass123"`
- **Back:** a string like `"$2b$12$HINBo6Zk..."`. Run it twice and you get two different
  strings. That's normal, each hash gets its own random salt.
- **Code:** `routers/admin.py`

#### `GET /backup`

Downloads a full copy of the database as a zip.

- **Who:** Admin
- **Try it:**
  ```bash
  curl $B/backup -H "Authorization: Bearer $ADMIN_TOKEN" -o backup.zip
  ```
- **Errors:** `403` if not an admin. `500` if `pg_dump` isn't installed on the machine.
- **Code:** `routers/admin.py`

#### `POST /manual_verify?email=...`

Marks a delegate as verified without the email. Your best friend on a laptop with no mail
server.

- **Who:** Anyone (see section 10)
- **Try it:** `curl -X POST "$B/manual_verify?email=ada@munsoc.test"`
- **Back:** `201` with `{"message": "Email verified!"}`
- **Errors:** `404` if no delegate has that email
- **Code:** `routers/admin.py`

---

### 7.7 Dynamic data

These serve JSON files straight from the `data/` folder, so we can change rooms and the
schedule without touching code.

#### `GET /rooms`

Which room each committee is in.

- **Who:** Anyone
- **Try it:** `curl $B/rooms`
- **Back:**
  ```json
  [
    {"id": "jcc_western_1", "committee_name": "JCC - Western Bloc", "room_code": "CR 404", "floor": "Floor 4"}
  ]
  ```
- **Code:** `routers/dynamic_data.py`, data in `data/rooms.json`

#### `GET /schedule`

The conference days and every event.

- **Who:** Anyone
- **Try it:** `curl $B/schedule`
- **Back:**
  ```json
  {
    "conference_days": [{"day_key": "Day 1", "display_date": "November 7, 2025"}],
    "events": [
      {"id": "1", "day": "Day 1", "name": "Registration Desk", "location": "MPSTME Main Gate",
       "time": "9:00 - 10:30 AM", "description": "...", "image": "https://..."}
    ]
  }
  ```
- **Code:** `routers/dynamic_data.py`, data in `data/schedule.json`

Both of these are cached in memory the first time they're read. **If you edit the JSON
file, restart the server** or you'll keep seeing the old version.

---

## 8. One full story: signup to lunch

Now tie it together. This is what really happens across a conference, in order. Do every
step yourself in Swagger or `curl`.

1. **Grace signs up in the app.**
   `POST /mumbaimun/register` with her details. (Locally, ignore the `500` from email.)

2. **Grace logs in.**
   `POST /login` gives her an access token and a refresh token.

3. **The app loads her profile.**
   `GET /delegates/me` with her access token. The app saves her `id`.

4. **She fills in her profile.**
   `PATCH /delegates/{id}?contact=...&gender=...`

5. **The app shows her QR code.**
   `GET /qr?id=<her id>`

6. **She checks where her committee is and what's on.**
   `GET /rooms` and `GET /schedule`

7. **Ten minutes pass and her token expires.**
   Her next `GET /delegates/me` gets a `403`. The app calls `POST /refresh` and retries.

8. **Lunch on day one.**
   A volunteer opens `/scan`, scans Grace's code, lands on `/food?id=<her id>`, ticks
   lunch, and the page sends `POST /food`.

9. **The OC checks numbers.**
   An admin calls `GET /mumbaimun/delegates?format=csv` and sees Grace's `d1_lunch` is `true`.

10. **After the conference, she logs out.**
    `POST /logout` with her refresh token.

If you can do all ten without looking at this list, you understand MUNDRA.

---

## 9. Exercises on the real code

Make a branch before you start so you never break `master`:

```bash
git checkout -b yourname/exercises
```

### Exercise 1: add a ping endpoint (15 minutes)

Add `GET /ping` that returns `{"pong": true}`. Put it in `routers/dynamic_data.py`.

- Does it need a prefix? Look at how `dynamic_data_router` is included in `main.py`.
- Check it shows up in Swagger under Dynamic Data.

### Exercise 2: filter rooms by floor (30 minutes)

Make `GET /rooms?floor=Floor 4` return only the rooms on that floor. With no `floor`, it
should return everything like before.

Hints:
- Add `floor: str = ""` to the function's parameters. That's all it takes to make a query
  parameter.
- `read_rooms_data()` gives you a list of dicts. Filter it with a list comprehension.
- Remember the space in `Floor 4` needs to be `Floor%204` in a `curl` URL. Swagger encodes
  it for you.

### Exercise 3: count delegates, and hit a real bug on purpose (45 minutes)

Add `GET /delegates/count`, admin only, returning `{"count": <number>}`.

1. Add the function to `database.py` first. Look at how `get_delegates` works.
2. Add the route in `routers/delegates.py`, **at the bottom of the file**.
3. Call it as an admin.

It won't work. You'll get a `404 Delegate not found`. Before reading on, try to work out why.

<details>
<summary>Why it breaks</summary>

FastAPI checks routes in the order they're written. `@router.get("/{id}")` is higher up
in the file, and `{id}` matches anything, including the word `count`. So your request is
being handled by `get_delegate_by_id` with `id="count"`, which obviously doesn't exist.

Fix it by moving your route **above** `@router.get("/{id}")`. This is exactly why
`/me` sits above `/{id}` in that file.

</details>

### Exercise 4: trace a request (20 minutes, no code)

Pick `PATCH /delegates/{id}`. On paper, write down every file and every function a
request passes through, from the moment it arrives until the response goes back. Include
`get_current_user`. Then check yourself by reading the code.

### When you're done

```bash
git add -A
git commit -m "Exercises: ping, room filter, delegate count"
git push -u origin yourname/exercises
```

Open a pull request and tag me. I'll review it like any other PR.

---

## 10. When things break

| You see | What's going on | Fix |
|---|---|---|
| `pg_isready` says `no response` | Postgres isn't running | Section 2.2 |
| `connection refused` on port 5432 | Same thing | Section 2.2 |
| `role "user" does not exist` | `DATABASE_URL` still has the example username | Put your real Postgres username in `.env` (section 2.5) |
| `password authentication failed for user "postgres"` (Windows) | Wrong password in `DATABASE_URL` | Use the password you set in the Postgres installer |
| `pg_isready`, `psql` or `createdb` "is not recognized" (Windows) | Postgres isn't on your Path | Add `C:\Program Files\PostgreSQL\16\bin` to Path (section 2.1), then reopen PowerShell |
| `uv` "is not recognized" or "command not found" | Terminal was opened before uv was installed | Close and reopen the terminal |
| `database "mundra" does not exist` | You skipped `createdb` | Section 2.4 |
| `Field required ... mail_server` | `.env` missing, or run from the wrong folder | Run commands from inside `mundra`, check `.env` exists |
| `relation "delegates" does not exist` | Tables were never created | `uv run alembic upgrade head` |
| `Address already in use` / port 8000 busy | Another server is already running | Stop it with `Ctrl+C`, or add `--port 8001` |
| `curl` gives a weird table instead of JSON (Windows) | PowerShell's `curl` isn't the real curl | Use Git Bash (section 2.8), or type `curl.exe` |
| `uv sync` complains about the Python version | Your terminal has `UV_PYTHON` set to something else | Check it with `echo $UV_PYTHON` (Mac) or `echo $env:UV_PYTHON` (Windows) and remove it |
| `500 ... Error connecting to localhost on port 465` | No mail server on your laptop. The account was still created. | Ignore it, then `POST /manual_verify?email=...` |
| `401 Please verify your email!` | Account exists but isn't verified | `POST /manual_verify?email=...` |
| `403 Could not validate credentials` | Token is wrong or has expired (10 min) | Log in again, or Authorize again in Swagger |
| `401 Not authenticated` | You didn't send a token at all | Add the header, or Authorize in Swagger |
| `403 Forbidden` on `/delegates` | Token isn't an admin's, or you sent it as a header | That route wants `?token=` in the URL |
| `422 Unprocessable Entity` | Your request has the wrong shape | Read the response, it names the exact field |
| `429 Too Many Requests` | Rate limit on login, password reset etc. | Wait a minute |
| Swagger page is blank or 404 | `DOCS_URL` is blank or missing | Set `DOCS_URL=/swagger` in `.env` |
| Edited `rooms.json` but nothing changed | It's cached in memory | Restart the server |
| Added a column to `db_models.py` but Postgres didn't change | Models don't change the database by themselves | `uv run alembic revision --autogenerate -m "what changed"` then `uv run alembic upgrade head` |

### Things in MUNDRA that aren't right yet

I'd rather you know about these than trip over them. They're all fair game as future tasks.

- **`POST /food`, `POST /manual_verify` and `GET /hash_password` have no login check.**
  Anyone who can reach the server can use them. Two have comments in the code admitting it.
- **An expired token gives `403`, not `401`.** `401` would be more correct and would make
  the app's life easier.
- **Password reset links now expire in 10 minutes**, because they reuse the access token,
  and access tokens got shorter. That's probably too short for an email.
- **`/change_pass` doesn't check the 8 character minimum** that signup does.
- **Registration isn't all or nothing.** If the email fails, the account is already made
  but the caller sees an error.
- **The Delego app doesn't use refresh tokens yet**, so it'll start failing after 10
  minutes until we update it.
- **There are no automated tests.** "The server started" is not proof that anything works.

---

## 11. Cheat sheet

### Commands

Same on both:

```bash
uv sync                                               # install packages
uv run uvicorn main:app --reload                      # run the server
uv run alembic upgrade head                           # build or update tables
uv run alembic revision --autogenerate -m "message"   # after changing db_models.py
```

Opening the database:

| | macOS | Windows |
|---|---|---|
| Open a prompt | `psql mundra` | `psql -U postgres -d mundra` |
| List tables | `psql mundra -c "\dt"` | `psql -U postgres -d mundra -c "\dt"` |
| Is Postgres up? | `pg_isready` | `pg_isready` |
| Start Postgres | `brew services start postgresql@16` | `Start-Service postgresql-x64-16` |

### Every endpoint on one screen

| Method | Path | Who | What |
|---|---|---|---|
| GET | `/` | Anyone | Health check |
| GET | `/static/{filename}` | Anyone | Images |
| POST | `/register` | Anyone | Normal signup |
| POST | `/login` | Anyone | Email + password to tokens (form) |
| POST | `/refresh` | Refresh token | New access token |
| POST | `/logout` | Refresh token | Cancel refresh token |
| GET | `/verify_email` | Link | Verify email |
| GET | `/resend_verification` | Anyone | Resend verification email |
| GET | `/forgot_password` | Anyone | Send reset email |
| GET | `/reset` | Link | Reset password page |
| PATCH | `/change_pass` | Delegate | Change password |
| DELETE | `/account` | Delegate | Delete login |
| GET | `/delegates/me` | Delegate | My profile |
| GET | `/delegates/{id}` | Self or Admin | One profile |
| PATCH | `/delegates/{id}` | Self or Admin | Edit profile |
| GET | `/delegates` | Admin (`?token=`) | All delegates, JSON or CSV |
| POST | `/mumbaimun/register` | Anyone | Mumbai MUN signup |
| GET | `/mumbaimun/delegates` | Admin | All MM delegates, JSON or CSV |
| GET | `/qr` | Anyone | QR code image |
| GET | `/scan` | Anyone | Scanner page |
| GET | `/food` | Anyone | Meal checklist page |
| POST | `/food` | Anyone | Save meal ticks (form) |
| GET | `/hash_password` | Anyone | Hash a password |
| GET | `/backup` | Admin | Database backup zip |
| POST | `/manual_verify` | Anyone | Verify without email |
| GET | `/rooms` | Anyone | Committee rooms |
| GET | `/schedule` | Anyone | Conference schedule |

### Where to look

| I want to change... | Open |
|---|---|
| An endpoint | `routers/<area>.py` |
| Which routers exist, or their prefix | `main.py` |
| What a request or response looks like | `models.py` |
| A table | `db_models.py`, then make a migration |
| How data is read or saved | `database.py` |
| Login, tokens, passwords | `auth.py` |
| A setting | `.env` and `config.py` |

That's Part 2. Once you've finished section 9, you're ready for real tasks on MUNDRA.
