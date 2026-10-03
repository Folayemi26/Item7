# Item7 Food Truck

A Flask web app for a campus food truck at Grambling State University. Customers browse the menu, build a cart, and check out with allergy notes, a promo code, a tip, map-based delivery, and card payment. Staff use a separate portal to run the truck: a time clock with shift booking, an order queue, deals, menu management, and sales statistics. All data lives in plain CSV files, so the whole app runs without a database.

![Item7 Food Truck](static/images/logo-hero.svg)

**Live demo:** [https://item7-food-truck-m0db.onrender.com](https://item7-food-truck-m0db.onrender.com)

> The demo runs on Render's free tier. If nobody has visited for a while, the first page load takes 30–60 seconds while the server wakes up.

---

## Try it

| Role     | Email                       | Password     | What to try                                                  |
|----------|-----------------------------|--------------|--------------------------------------------------------------|
| Customer | `customer.demo@example.com` | `Item7demo!` | Add items to the cart, then check out with promo code `HALF50` |
| Staff    | `staff.demo@example.com`    | `Item7demo!` | Staff Portal: claim a shift, check in, manage orders and deals |

Card payments run in Stripe demo mode. Use test card `4242 4242 4242 4242` with any future expiry date and any CVC. No real charges are made.

Removing staff in Staff Management requires a senior manager code, available on request.

The live demo's data resets on every redeploy, so feel free to place orders and claim shifts.

---

## Features

### For customers
- **Menu** organized by category (combos, mains, veg, sides, drinks) in swipeable carousels, with a photo for every item and vegan and allergen tags.
- **Live cart.** Adding items, changing quantities, and removing items update the cart count and totals instantly, without reloading the page.
- **Checkout** with:
  - **Allergy check.** Customers describe their allergies, and each order is flagged as safe or unsafe against the menu's allergen data.
  - **Promo codes.** Codes like `HALF50` apply instantly. The server re-validates the code before the order is placed.
  - **Tip** by percentage or a custom amount, plus tax (7.5% by default).
  - **Map-based delivery.** The customer's location is detected automatically, or they can search for an address or drag the pin on an interactive Leaflet map. The map defaults to the Grambling State campus.
  - **Payment** by card through Stripe, or cash on delivery.
- **Mobile-friendly layout** with a slide-in navigation menu.

### For staff
- **Time clock.** Staff claim shifts without a page reload, then check in, take breaks, check out, and add shift notes. A work history shows hours per week.
- **Order queue.** Orders move through *Pending → Preparation Done → Ready for Delivery*.
- **Deals.** Staff create promotions that appear on the home page. A deal's code becomes a working promo code: the number at the end sets the discount, so `SAVE20` gives 20% off.
- **Menu management.** Staff add, edit, and delete menu items, including image uploads.
- **Statistics:** order counts for today, this month, and this year, estimated sales, and orders per customer.
- **Staff management.** A staff directory, plus staff removal behind a senior-manager code.

### Under the hood
- **Roles.** Customers, staff, and admins see different navigation and pages. Staff sign-up requires a registration code.
- **Security.** Passwords are hashed with Werkzeug, every staff route checks the session's role, and CSV input is sanitized.
- **Progressive enhancement.** Every form still works as a regular form post if JavaScript is off. With JavaScript on, the cart, add-to-cart, and shift booking update in place using `fetch()`.
- **JSON API** for the menu, cart, promo codes, and schedules. See [API endpoints](#api-endpoints).
- **Logging.** Actions are recorded to `ft_management.log`.

---

## Tech stack

- **Backend:** Python 3.11, Flask, Werkzeug, Gunicorn
- **Frontend:** Jinja templates, vanilla JavaScript, CSS
- **Payments:** Stripe (test mode or simulated demo mode)
- **Maps:** Leaflet with OpenStreetMap tiles and geocoding
- **Storage:** CSV files in `data/`
- **Hosting:** Render

---

## Run locally

Requires Python 3.11. A virtual environment is recommended.

```bash
pip install -r requirements.txt
cp .env.example .env      # optional; every setting has a default for local use
python app.py
```

Then open http://localhost:5000.

> **macOS:** AirPlay Receiver uses port 5000. Run `PORT=5001 python app.py` and open http://localhost:5001 instead.

Run the app from the repository root, because it reads and writes `data/*.csv` using relative paths.

---

## Configuration

All settings are environment variables, set in `.env` locally or in your host's dashboard.

| Variable | Default | Purpose |
|----------|---------|---------|
| `SECRET_KEY` | insecure development key | Signs session cookies. **Set this in production.** Generate one with `python -c "import secrets; print(secrets.token_hex(32))"`. |
| `FLASK_ENV` | (unset) | Set to `production` to turn off debug mode. |
| `ADMIN_EMAILS` | (none) | Comma-separated emails that get admin access when they log in. |
| `STAFF_REGISTRATION_CODE` | `1234` | Code required to sign up as staff. **Change this on a public deployment.** |
| `SENIOR_MANAGER_CODE` | `1234` | Unlocks staff removal in the portal. **Change this on a public deployment.** |
| `TAX_RATE` | `0.075` | Sales tax applied at checkout. |
| `STRIPE_SECRET_KEY`, `STRIPE_PUBLISHABLE_KEY` | (none) | Stripe test keys (`sk_test_...`, `pk_test_...`) from [dashboard.stripe.com/test/apikeys](https://dashboard.stripe.com/test/apikeys). Without them, checkout runs in demo mode: the card form is simulated and no request reaches Stripe. |

---

## Deploy to Render

1. In the Render dashboard, create a **New → Web Service** and connect this repository.
2. Use these settings:
   - **Build command:** `pip install -r requirements.txt`
   - **Start command:** `gunicorn app:app`
   - **Instance type:** Free
3. Add these environment variables:
   - `PYTHON_VERSION=3.11.0`
   - `FLASK_ENV=production`
   - A generated `SECRET_KEY`
   - Your own `STAFF_REGISTRATION_CODE` and `SENIOR_MANAGER_CODE`
4. Deploy. Every later push to `main` redeploys automatically.

`render.yaml` describes the same service for Render Blueprints.

Render's free tier has an ephemeral disk: the CSV files return to their committed state on every deploy or restart. That keeps a demo clean, but it isn't permanent storage.

---

## Project structure

```
app.py              Flask routes, auth guards, cart, checkout, Stripe, JSON API
foodtruck.py        FoodTruck class: all CSV reading and writing and business rules
templates/
  base.html         Public layout, nav, shared JS (toasts, live add-to-cart)
  staff_layout.html Staff portal layout with sidebar navigation
  ...               One template per page
static/
  style.css         All styling, including mobile breakpoints
  images/menu/      Menu photos (see CREDITS.md)
data/               CSV storage (users, menu, orders, deals, schedules, shifts)
render.yaml         Render service definition
```

---

## Roles and access

| Role       | How it's assigned | Access |
|------------|-------------------|--------|
| `customer` | Sign up as a customer | Public site: menu, cart, checkout, dashboard |
| `staff`    | Sign up as staff with `STAFF_REGISTRATION_CODE`, or added by an admin | Staff Portal (`/staff/*`) |
| `admin`    | Email listed in `ADMIN_EMAILS` | Staff Portal plus admin pages: `/admin`, `/admin/orders`, `/add_staff`, `/schedules` |

New accounts are verified automatically when they sign up. Customers who try to open a staff page are redirected away.

---

## Data

Each CSV file is created with headers on first run. The app adds new columns to older files automatically, so existing data is never lost.

```
users.csv      Email, Password, First_Name, Last_Name, Mobile_Number, Address, DOB, Sex, Role, Verified
menu.csv       Item_ID, Name, Description, Price, Category, Vegan, Image, Allergens
orders.csv     Order_ID, Customer_Name, Customer_Email, Item, Allergy_Info, Is_Safe, Timestamp, Status
deals.csv      Deal_ID, Title, Description, Discount, Created_By, Created_At, Expires_At, Is_Active
schedules.csv  Manager, Date, Time, staff_Email, staff_Name, work_Time
shifts.csv     Shift_ID, Staff_Email, Date, Scheduled_Start, Scheduled_End, Check_In_Time, Check_Out_Time,
               Break_Start, Break_End, Total_Hours, Status, Notes, Early_Checkout
```

An order's `Item` field records the items plus the delivery address, promo, tax, tip, and payment reference.

---

## API endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/menu` | GET | All menu items |
| `/api/menu/<category>` | GET | Menu items in one category |
| `/api/cart` | GET | Current cart with subtotals |
| `/api/cart` | POST | Add an item: `{"item_name", "price", "qty"}` |
| `/api/cart` | PUT | Set an item's quantity: `{"item_name", "qty"}` |
| `/api/cart` | DELETE | Remove an item: `{"item_name"}` |
| `/api/cart/clear` | POST, DELETE | Empty the cart |
| `/api/promo` | POST | Check a promo code against the cart: `{"code"}`. Returns the discount, tax, and total. |
| `/api/appointments` | GET | All schedules |
| `/get_available_slots/<staff>/<date>` | GET | Open time slots for a staff member on a date |
| `/book_appointment` | POST | Book a schedule slot |

`POST /staff/claim-shift` also returns JSON instead of redirecting when it's called with the header `X-Requested-With: fetch`.

---

## Testing tips

1. Delete `data/*.csv` to start fresh. The app recreates the files on startup.
2. Log in as the demo customer and confirm that `/staff/dashboard` redirects away.
3. At checkout, try an invalid promo code, then `HALF50`, and check that the order in `data/orders.csv` records the discount, tip, and delivery address.
4. Claim a shift as the demo staff user, then try the same date again to see the duplicate warning.
5. Set `ADMIN_EMAILS` and log in with that email to reveal the admin pages.
6. Resize the browser to phone width to check the mobile navigation and menu carousels.

---

## License and credits

MIT License. Built for CS120 coursework, with a staff portal inspired by the KFC Shift Manager experience.

Menu photos come from [Unsplash](https://unsplash.com) under the Unsplash License. See [`static/images/menu/CREDITS.md`](static/images/menu/CREDITS.md) for the photographers.
