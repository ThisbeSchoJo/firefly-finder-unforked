## Firefly Finder Technical Walkthrough

### Overview

- Full-stack web app for logging and sharing firefly sightings; users authenticate, curate personal observations, and explore community data on an interactive Google Map.
- Frontend: React 18, React Router v6, shared auth context via `<Outlet>`, Google Maps through `@react-google-maps/api`, modular CSS styling.
- Backend: Flask + Flask-RESTful on port 5555, PostgreSQL via SQLAlchemy, session-based auth, Flask-Bcrypt for passwords, static uploads for profile images.
- External data: iNaturalist API enriches the map with public Lampyridae observations; Google Maps API key provided as `REACT_APP_GOOGLE_MAPS_API_KEY`.

### Backend Highlights

- Session-auth routes (`/signup`, `/login`, `/logout`, `/check_session`) manage `session["user_id"]` for stateful authentication.
- File uploads handled during signup, persisted under `server/static/uploads`, served via `send_from_directory`.
- Sighting endpoints support geo-filtering (simple lat/lng bounding box), CRUD actions bound to the signed-in user, and a `/sightings/count` endpoint fueling rank display in the UI.
- Friend graph stored as symmetric rows in `friendships` so lookups are straightforward; friend search filters by case-insensitive username and excludes the requester.

```python
# server/app.py (excerpt)
class Users(Resource):
    def post(self):
        data = request.form
        if not data or 'username' not in data or 'password' not in data:
            return make_response({"error": "Username and password are required"}, 400)
        file = request.files.get('profile_picture')
        # ... store unique filename under /static/uploads ...
        new_user = User(username=username, profile_picture=profile_pic_path)
        new_user.password_hash = password
        session["user_id"] = new_user.id
        return make_response(new_user.to_dict(rules=("-_password_hash",)), 201)
```

```python
# server/models.py (excerpt)
class Sighting(db.Model, SerializerMixin):
    id = db.Column(db.Integer, primary_key=True)
    place_guess = db.Column(db.String)
    observed_on = db.Column(db.DateTime)
    description = db.Column(db.String)
    latitude = db.Column(db.Float)
    longitude = db.Column(db.Float)
    user = db.relationship("User", back_populates="sightings")
    species = db.relationship("Species", back_populates="sightings")
```

### Frontend Flow

- `client/src/components/App.js` performs an initial `/check_session`, stores the user in local state, and passes auth context through `useOutletContext` for child routes.
- `client/src/routes.js` wraps `/profile`, `/profile/:userId`, and `/sightings` in `ProtectedRoute`, which renders a loading splash until the session probe finishes.
- `NavBar` conditionally renders login/logout icons using Lucide React icons.
- Auth screens (`Login`, `Signup`) submit to backend endpoints; signup uses multipart form data to support profile photo upload.
- `Profile` consolidates user details, rank, friend list, and inline friend-management modal.
- `FriendProfile` fetches peers’ data without edit controls, reusing rank calculation.

### Sightings & Map Experience

- `Sighting` resolves browser geolocation, fetches personal sightings, and merges them with iNaturalist observations from `useFireflyInaturalistData`.
- `Map` renders both observation sets; user markers carry full records to drive edit/delete controls in `ObservationPopup`.
- Add/Edit forms fetch species list from `/species`, embed a mini map for coordinate selection, and auto-fill lat/lng in the submission payload.
- CRUD actions optimistically update local `sightings` state on success to keep the interface reactive.

### External Integrations

- iNaturalist API wrapper (`client/src/services/inaturalistApi.js`) discovers the Lampyridae taxon ID dynamically, then reuses it for all observation queries.
- Google Maps script loads lazily; failure states show textual errors instead of breaking the app.
- File storage stays local; UI falls back to `default-profile-pic.png` on broken images.

### Environment & Setup Notes

- Backend expects DB configuration via environment variables or `.flaskenv`; development guide uses Pipenv (`pipenv install`, `flask db upgrade`, `python seed.py`).
- Frontend requires `.env` with `REACT_APP_GOOGLE_MAPS_API_KEY=<key>` before running `npm start`.
- `server/seed.py` pre-populates species, users, and sightings for immediate testing.
- For production, plan on persistent object storage for uploads and HTTPS-aware cookies.

### Likely Questions & Answers

- **How is authentication enforced on protected routes?** Flask stores `session["user_id"]`; every mutating endpoint checks it and returns `401`/`403` if missing. React gates routes with `ProtectedRoute`, redirecting to `/login` after the initial session check.
- **What’s required to deploy the map view successfully?** Provide a Google Maps JavaScript API key in `client/.env` as `REACT_APP_GOOGLE_MAPS_API_KEY` with Maps JavaScript API enabled.
- **How close is the “nearby sightings” filter?** Backend uses a simple bounding-box approximation (`radius/111` degrees). Adequate for small regions but not great-circle precise.
- **Can users upload sighting photos directly?** Not yet; sightings accept a URL. Extending uploads would require multipart handling in the sighting POST/PATCH routes.
- **How do we ensure friend relationships are bidirectional?** `/add-friend` inserts two rows (user→friend and friend→user). `/friends` fetches any row where the requester appears, keeping the list symmetric.
- **Where do ranks come from?** `/sightings/count` returns total sightings for the logged-in user; the frontend maps thresholds to ranks (`Egg`, `Larva`, `Pupa`, `Lightning Bug`) and shows a progress bar.
- **What happens if Google Maps fails to load?** `Map` component renders “Error loading maps” when `useLoadScript` reports failure, preventing crashes while signaling the issue.
- **How are iNaturalist observations fetched without hardcoding taxon IDs?** `searchFireflyObservations` first calls `searchFireflySpecies()` to find the Lampyridae family ID, then includes it as `taxon_id` for observation queries.
- **Is there throttling on friend search?** Not yet; `AddFriendForm` fires a network request on every submit. Debouncing or cancellation would improve UX.

### Suggested Next Steps

- Document `.env` expectations for both tiers in `README`.
- Enhance geo-filtering accuracy (e.g., Haversine) and add pagination to `/sightings`.
- Replace `alert`-based error handling in forms with inline UI messaging.
- Plan for production-ready file storage (S3, cloud bucket) and secure cookies over HTTPS.
