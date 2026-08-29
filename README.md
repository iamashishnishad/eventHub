# EventHub

A backend API for a simplified event ticketing platform built with Django and Django REST Framework. Users can browse events, reserve seats, and cancel reservations.

## How to Run the Project

1. **Clone the repository**

   ```bash
   git clone <repo-url>
   cd eventHub
   ```

2. **Create and activate a virtual environment**

   ```bash
   python3 -m venv venv
   source venv/bin/activate   # On Windows: venv\Scripts\activate
   ```

3. **Install dependencies**

   ```bash
   pip install -r requirements.txt
   ```

4. **Run migrations**

   ```bash
   python manage.py migrate
   ```

5. **Start the server**

   ```bash
   python manage.py runserver
   ```

   The API is now available at `http://127.0.0.1:8000/api/`.

   Every request is logged to the terminal by `RequestLoggingMiddleware` in the form:
   `METHOD /path - status_code - duration`

## API Documentation (Swagger)

Interactive API docs are generated automatically with `drf-spectacular`:

| URL | Description |
|---|---|
| `/api/docs/` | Swagger UI — browse and try out every endpoint from the browser. |
| `/api/redoc/` | Redoc — a read-only reference view of the same schema. |
| `/api/schema/` | Raw OpenAPI 3 schema (YAML). |

## Endpoints

### Events

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/events/` | List all events. Supports `?status=` and `?venue=` query params for filtering (venue is a case-insensitive partial match). |
| POST | `/api/events/` | Create a new event. |
| GET | `/api/events/{id}/` | Retrieve a single event, including its live `reservations_count`. |
| PUT/PATCH | `/api/events/{id}/` | Update an event. |
| DELETE | `/api/events/{id}/` | Delete an event. |

### Reservations

| Method | Endpoint | Description |
|---|---|---|
| GET | `/api/reservations/` | List all reservations. Supports `?event_id=` to filter by event. |
| POST | `/api/reservations/` | Reserve seats for an event. Validates that the event is `upcoming`/`ongoing` and that enough seats are available, then deducts the reserved seats from the event's `available_seats`. |
| GET | `/api/reservations/{id}/` | Retrieve a single reservation. |
| POST | `/api/reservations/{id}/cancel/` | Cancel a confirmed reservation. Restores the seats back to the event and marks the reservation `cancelled`. Returns `400` if it's already cancelled. |

## Design Decision

**Seat deduction happens inside `ReservationSerializer.create()` rather than in the view.** Keeping the "validate seats are available → deduct from the event → create the reservation" logic in one place (the serializer) means every entry point that creates a reservation through this serializer gets the same guarantee for free, instead of relying on each view/caller to remember to update the event separately. The trade-off is that these two writes (updating `Event.available_seats` and creating the `Reservation`) aren't wrapped in `transaction.atomic()`, so under concurrent requests for the last few seats there's a small window for a race condition — acceptable for this assignment's scope, but the fix in a production system would be to wrap `create()` in `transaction.atomic()` and use `select_for_update()` when fetching the event.

## Testing Checklist

- [x] Create an event
- [x] List all events
- [x] Filter events by status
- [x] Filter events by venue
- [x] Create a reservation (seats deducted from event)
- [x] Overbooking attempt returns 400
- [x] Cancel a reservation (seats restored to event)
- [x] Filter reservations by event_id
- [x] Middleware logs appear in the terminal for every request
      
1. Successful reservation

<img width="2918" height="1668" alt="image" src="https://github.com/user-attachments/assets/9c608352-4db9-4007-b976-0ddb631adb8e" />

2. Overbooking failure

<img width="2914" height="1660" alt="image" src="https://github.com/user-attachments/assets/1206addf-391e-43ff-9815-4c5512cb4bca" />

3. Successful cancellation

<img width="2922" height="1664" alt="image" src="https://github.com/user-attachments/assets/cd6851f7-cacd-4827-8df1-4b6c92ff2452" />


