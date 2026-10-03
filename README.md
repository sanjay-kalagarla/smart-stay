# Smart Stay

A hostel management platform where residents book shared facilities, raise maintenance requests, and recover lost belongings, while admins run everything from one dashboard.

Smart Stay goes beyond a plain CRUD app. Facility bookings are policed by a reliability score, maintenance tickets escalate on their own when ignored, and lost-and-found reports are matched by comparing item photos with a computer-vision model.

---

## How it works

**Accounts and access.** Residents register with an email and password (hashed with bcrypt) and log in with a six-digit OTP sent to their inbox, valid for five minutes. Sessions live in MongoDB for 24 hours. The admin account is seeded on first start and logs in directly.

**Facility booking.** A resident picks a facility (gym, table tennis, study hall, badminton, TV room), a date and a time slot. Past dates and already-taken slots are rejected. Every resident starts with a reliability rating of 5.0, and the system adjusts it automatically:

| Event | Effect on rating |
| --- | --- |
| Admin verifies the slot was used properly | +0.2 (capped at 5.0) |
| Booking cancelled by the resident | -0.5 |
| No check-in within 15 minutes of start (auto-cancelled) | -0.5 |
| Booking auto-cancelled after two "facility is empty" reports | -0.5 |
| Filing a report against a booking that was actually in use | -0.7 |

Below 3.0 a resident is blocked from booking until an admin unblocks them. A background job runs every five minutes to enforce the no-show rule.

**Service requests.** Residents file a request with a category, location and severity, which sets its priority. Repeat issues are flagged as recurring, and each request is assigned to the staff member with the best balance of rating and current workload. A request left untouched for 24 hours is escalated to high priority and the admin is emailed. After completion, the resident rates the work, which feeds back into the staff member's rating. An analytics endpoint reports average resolution time, the most frequent issue categories and top-rated staff.

**Lost and found.** Residents report a lost or found item with a photo. A separate Python service turns each photo into an embedding using a pretrained MobileNetV2, and the backend compares a report against all open reports of the opposite type using cosine similarity. Matches scoring 0.60 or higher are returned, best three first. Confirming a match closes both reports. If the AI service is offline, matching is skipped and the rest of the app keeps working.

**Notifications.** Registration, OTPs, bookings, status changes, penalties and escalations all trigger HTML emails through SendGrid. Without a SendGrid key, emails are logged to the console instead of sent, which makes local development painless.

---

## Architecture

```
 Browser (HTML / CSS / JS)
          │  fetch + session cookie
          ▼
 Express API  ──────────────►  SendGrid (email)
  │   │   │                    Cloudinary (images)
  │   │   └─ node-cron: no-show cancellation, ticket escalation
  │   │
  │   └──► FastAPI service (MobileNetV2 embeddings + similarity)
  ▼
 MongoDB (users, bookings, requests, lost & found, sessions)
```

---

## Getting started

**Prerequisites:** Node.js 18+, a MongoDB instance (local or Atlas), and Python 3.10 if you want image matching.

```bash
git clone https://github.com/WebDoveleprrr/Smart_Stay.git
cd Smart_Stay
npm install
```

Create a `.env` file in the project root:

| Variable | Purpose |
| --- | --- |
| `MONGO_URI` | MongoDB connection string. **Required.** |
| `SESSION_SECRET` | Secret used to sign session cookies. |
| `ADMIN_EMAIL`, `ADMIN_PASSWORD` | Credentials for the seeded admin account. Set your own. |
| `SENDGRID_API_KEY`, `EMAIL_USER` | SendGrid key and the verified sender address. |
| `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` | Image hosting for lost-and-found photos. |
| `AI_URL` | Base URL of the AI service, for example `http://localhost:8000`. |
| `PORT` | Server port. Defaults to 3000. |

Start the app:

```bash
npm run dev      # development, with nodemon
npm start        # production
```

To enable image matching, run the AI service in a second terminal:

```bash
cd ai_service
pip install -r requirements.txt
uvicorn main:app --port 8000
```

The app is then available at `http://localhost:3000`.

---

## API overview

| Area | Endpoints |
| --- | --- |
| Auth | `POST /api/register`, `/api/login`, `/api/verify-otp`, `/api/resend-otp`, `/api/logout`; `GET /api/session`, `/api/profile` |
| Bookings | `POST /api/bookings`; `GET /api/bookings`, `/api/bookings/all-active`; `DELETE /api/bookings/:id`; `POST /api/bookings/checkin/:id`, `/api/bookings/report/:id`; `PATCH /api/bookings/:id/usage` |
| Service requests | `POST /api/services`; `GET /api/services`, `/api/services/analytics`; `PATCH /api/services/:id`; `POST /api/services/rate/:id` |
| Lost and found | `POST /api/lost-found`, `/api/match-image`, `/api/confirm-match`; `GET /api/lost-found`; `PATCH /api/lost-found/:id` |
| Admin | `GET /api/admin/stats`; `PATCH /api/admin/users/:id/unblock` |

---

## Project structure

```
Smart_Stay/
├── server.js          App setup, auth, lost & found, admin routes
├── controllers/       Booking and service request logic
├── routes/            Express routers for bookings and services
├── models/            Mongoose schemas (Booking, ServiceRequest)
├── utils/             Cron jobs and the SendGrid mailer
├── ai_service/        FastAPI service for image embeddings and matching
├── public/images/     Static assets
└── *.html             Login, resident dashboard, admin dashboard
```

---

## Tech stack

| Layer | Technologies |
| --- | --- |
| Frontend | HTML5, CSS3, vanilla JavaScript (Fetch API), Google Fonts |
| Backend | Node.js 18+, Express.js 4, express-session, connect-mongo, bcryptjs, Multer, node-cron, CORS, uuid, dotenv |
| Database | MongoDB, Mongoose |
| AI service | Python 3.10, FastAPI, Uvicorn, PyTorch and TorchVision (MobileNetV2), NumPy, Pillow, Pydantic |
| Third-party services | SendGrid (email), Cloudinary (image hosting) |
| Tooling and hosting | Nodemon, Git, Render |
