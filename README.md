# Smart Stay

Smart Stay is a web app we built as a team to make college life a bit less chaotic. Students can book shared facilities like the gym or study hall, report problems around campus, and get lost stuff back. The college admin handles everything from one dashboard.

We didn't want it to be just another form-and-database project, so we added a few things that make it smarter: a reliability score that keeps bookings fair, complaints that escalate on their own if nobody looks at them, and a photo-matching model that links lost items with found ones.

---

## Why we built it

Anyone who has lived in a college hostel knows the problems. The table tennis room is "booked" but empty. A broken tap gets reported five times and nothing happens. Someone loses their earphones and the only option is a WhatsApp group message that gets buried in minutes. We tried to fix these three things in one app.

---

## What it does

**Login.** Students sign up with an email and password (passwords are hashed with bcrypt). Every login needs a six-digit OTP sent to their email, which expires in five minutes. Sessions last 24 hours. The admin account is created automatically the first time the server starts.

**Facility booking.** A student picks a facility (gym, table tennis, study hall, badminton or TV room), a date and a time slot. You can't book a past date or a slot someone else already has.

To stop people from blocking slots and not showing up, everyone starts with a reliability rating of 5.0 that changes automatically:

| What happens | Rating change |
| --- | --- |
| Admin confirms you used the slot properly | +0.2 (max 5.0) |
| You cancel your booking | -0.5 |
| You don't check in within 15 minutes of the start time (auto-cancelled) | -0.5 |
| Booking gets auto-cancelled after two "facility is empty" reports | -0.5 |
| You report someone, but the booking was actually being used | -0.7 |

If your rating drops below 3.0, you can't book anything until the admin unblocks you. A background job runs every five minutes to catch no-shows.

**Service requests.** For problems like a broken fan or a leaking pipe, students file a request with a category, location and how serious it is. That decides its priority. If the same issue keeps coming back, it gets flagged as recurring. Each request is given to the staff member with the best mix of good rating and light workload. If a request sits untouched for 24 hours, it automatically becomes high priority and the admin gets an email. Once the work is done, the student rates it, and that rating goes into the staff member's score. There's also an analytics endpoint that shows average fix time, the most common issues and the top-rated staff.

**Lost and found.** Students post a lost or found item with a photo. A separate Python service converts each photo into a set of numbers (an embedding) using a pretrained MobileNetV2 model. We then compare a new post against all open posts of the opposite type using cosine similarity. Anything scoring 0.60 or more is shown as a match, best three first. Once a match is confirmed, both posts close. If the AI service is down, matching is skipped and the rest of the app still works fine.

**Emails.** Registration, OTPs, bookings, status updates, penalties and escalations all send an email through SendGrid. If there's no SendGrid key, the emails just print in the console, which is handy while testing.

---

## How it fits together

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

## Running it yourself

You'll need Node.js 18+, a MongoDB database (local or Atlas), and Python 3.10 if you want the image matching.

```
git clone https://github.com/sanjay-kalagarla/smart-stay.git
cd smart-stay
npm install
```

Make a `.env` file in the project root:

| Variable | What it's for |
| --- | --- |
| `MONGO_URI` | MongoDB connection string. **Required.** |
| `SESSION_SECRET` | Secret for signing session cookies. |
| `ADMIN_EMAIL`, `ADMIN_PASSWORD` | Login for the admin account. Set your own. |
| `SENDGRID_API_KEY`, `EMAIL_USER` | SendGrid key and the verified sender email. |
| `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` | Where lost-and-found photos are stored. |
| `AI_URL` | Address of the AI service, like `http://localhost:8000`. |
| `PORT` | Server port. Defaults to 3000. |

Start the app:

```
npm run dev      # development, auto-restarts with nodemon
npm start        # production
```

For image matching, open a second terminal and run the AI service:

```
cd ai_service
pip install -r requirements.txt
uvicorn main:app --port 8000
```

Then open `http://localhost:3000`.

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
smart-stay/
├── server.js          App setup, auth, lost & found, admin routes
├── controllers/       Booking and service request logic
├── routes/            Express routers for bookings and services
├── models/            Mongoose schemas (Booking, ServiceRequest)
├── utils/             Cron jobs and the SendGrid mailer
├── ai_service/        FastAPI service for image embeddings and matching
├── public/images/     Static assets
└── *.html             Login, student dashboard, admin dashboard
```

---

## Team

This was built by a team of college students.

- Sanjay Kalagarla ([@sanjay-kalagarla](https://github.com/sanjay-kalagarla))
- *Add your teammates here*

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
