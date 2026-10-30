# Rinder Backend

The backend for **Rinder**, a student housing and roommate-finding
application focused on helping students in Fredericton, New Brunswick
discover compatible roommates and suitable housing listings.

It provides a REST API for roommate matching, housing-listing
recommendations, and an AI-powered housing assistant. Firebase Firestore
is used as the data store.

## Features

-   **Roommate matching:** Scores other user profiles against a selected
    user's preferences and returns candidates from highest to lowest
    compatibility.
-   **Housing recommendations:** Ranks housing listings using the user's
    budget and preferred Fredericton neighbourhood.
-   **AI housing assistant:** Uses Groq tool calling to answer questions
    about available listings, rental statistics, neighbourhoods, and the
    neighbourhood associated with a place.
-   **Listing statistics:** Calculates numeric summaries and category
    counts from the matching Firestore listings.
-   **Place lookup:** Uses OpenStreetMap Nominatim for geocoding and an
    ArcGIS ward service to identify the corresponding Fredericton
    neighbourhood.

## Tech stack

-   Python
-   FastAPI and Uvicorn
-   Firebase Admin SDK and Cloud Firestore
-   Groq API (`openai/gpt-oss-20b`) for the chatbot
-   Pydantic
-   Requests
-   OpenStreetMap Nominatim and ArcGIS REST services

## Repository structure

``` text
rinder-backend/
├── chatbot.py             # Groq chatbot, conversation history, and tool calling
├── firebase_client.py     # Firebase Admin initialization and Firestore client
├── main.py                # FastAPI application and HTTP endpoints
├── matching.py            # Roommate compatibility scoring and ranking
├── recommendations.py     # Housing-listing recommendation and ranking
├── tools.py               # Firestore listing tools and place/neighbourhood lookup
├── requirements.txt       # Python dependencies
└── .gitignore
```

## How the backend works

### 1. Roommate matching

`matching.py` loads user profiles from the Firestore `users` collection
and compares the requested user with every other user.

The compatibility score is out of 100 and uses these weights:

  Compatibility factor          Weight
  --------------------------- --------
  Budget similarity                25%
  Noise-level similarity           20%
  Cleanliness similarity           15%
  Sleep-schedule similarity        15%
  Age similarity                   10%
  Smoking preference match         10%
  Drinking preference match         5%

The resulting candidates are sorted from highest to lowest score. The
API response includes each candidate's ID, score, name, age, program,
and profile picture.

The current implementation expects profile fields such as `id`,
`budget`, `noise_level`, `cleanliness`, `sleep_schedule`, `age`,
`smoking`, and `drinking` to be present in the relevant Firestore user
documents.

### 2. Housing recommendations

`recommendations.py` reads the user's budget and description from the
`users` collection, then reads listings from the `house` collection.

It attempts to find a preferred neighbourhood by looking for one of the
neighbourhood names defined in `KNOWN_AREAS` within the user's profile
description. Listings in that neighbourhood are ranked first. Within
each group, listings are ordered by how close their rent is to the
user's budget.

Listings are ranked rather than filtered out: listings outside the
preferred area can still appear after preferred-area matches. The
response includes the listing ID, title, neighbourhood, rent, bedrooms,
property type, and whether the listing matches the preferred area.

### 3. AI housing assistant

`chatbot.py` connects to Groq and uses the `openai/gpt-oss-20b` model
with function/tool calling. The assistant can call the functions
registered in `tools.py` to retrieve current information from Firestore
or look up a place.

Conversation history is kept separately by `user_id` in the backend
process. The implementation retains the system prompt and up to the six
most recent user turns when trimming conversation history. This history
is in memory, so it is not persistent across server restarts and is not
shared across multiple backend workers.

### 4. Housing and location tools

`tools.py` provides the chatbot with functions to:

-   Count listings overall or within a neighbourhood.
-   List neighbourhoods that currently have listings.
-   Retrieve listing details with optional neighbourhood, rent, and
    property-type filters.
-   Calculate minimum, maximum, and average values for supported numeric
    fields.
-   Calculate counts and percentages for supported categorical fields.
-   Geocode a place in Fredericton and identify its ward/neighbourhood.

The supported numeric fields are `rent`, `bedrooms`, `bathrooms`, and
`available_rooms`. The supported categorical fields are `furnished`,
`utilities_included`, and `property_type`.

### API Endpoints

### `GET /roommates`

Returns roommate suggestions for a user, sorted by compatibility score.

**Query parameter**

  -----------------------------------------------------------------------
  Parameter         Type              Required          Description
  ----------------- ----------------- ----------------- -----------------
  `user_id`         string            Yes               ID of the user
                                                        whose roommate
                                                        matches should be
                                                        calculated

  -----------------------------------------------------------------------

### `GET /recommended-listings`

Returns housing listings ranked for the specified user.

**Query parameter**

  -----------------------------------------------------------------------
  Parameter         Type              Required          Description
  ----------------- ----------------- ----------------- -----------------
  `user_id`         string            Yes               ID of the user
                                                        whose budget and
                                                        preferred area
                                                        are used

  -----------------------------------------------------------------------

The values above are illustrative. The endpoint returns up to 100
listings by default, ordered by preferred-area match and then rent
proximity to the user's budget.

### `POST /chatbot`

Sends a message to the Rinder AI housing assistant.

**Request body**

  -----------------------------------------------------------------------
  Field             Type              Required          Description
  ----------------- ----------------- ----------------- -----------------
  `user_id`         string            Yes               Identifies the
                                                        user's in-memory
                                                        chatbot
                                                        conversation

  `message`         string            Yes               Message to send
                                                        to the assistant
  -----------------------------------------------------------------------

