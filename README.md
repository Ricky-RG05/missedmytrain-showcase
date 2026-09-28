# missedmytrain-showcase 🚉
***The full source is maintained privately during active development. This repository documents the project's design, architecture, and a working demo.***

An automatic route-planner for all travelers within the German territory that provides the best route, based on a custom-made algorithm from scratch.

https://github.com/user-attachments/assets/b98e799b-3168-4346-9a84-f901eaac5d27

## The Problem
According to news articles published by DW and Der Spiegel, published respectively in 2025 and 2026, approximately only 62.5% of DB ICE and IC long-distance trains arrived on time. 

DB's 2025 Integrated Report states approximately 1.22 billion total passenger journeys across the network. Separately, Der Spiegel reported that in the first half of 2026, roughly 1 in 12 long-distance (ICE/IC) trains were fully or partially cancelled — a figure specific to long-distance service, not the network as a whole.

Even without combining these into a single derived statistic, both point at the same underlying reality: at national scale, even a small percentage of disruption translates into a large absolute number of affected travelers every single day — travelers with no guidance from the system itself on what to do next.

**There's a significant gap worth addressing**. 
As an international student having to make use of the German public transport, I've felt dissatisfied with the lack of guidance offered by the German Railway System on to ***what to do*** when a train is delayed, separated, new track, or worst of all, **it's cancelled**.

## Product's Aggregated Value
Based on personal experience and passenger-observation in the train stations, I've seen a major problem: All the information may be available (some easier accessible than other), but there is no guidance as to what to do if a problem happens (e.g. a delayed/cancelled train).

The market-differenciator of my product provides a route-calculator and an AI-assistant that has access to that information, in real time, with the most relevant details for disruptions that can solve any possible questions the user may have in his own language (currently I offer English, German, Spanish, French and Italian for conversation-capabilities; nonetheless, the project is on the works and languages with high demand are to be considered).

## What does it do
1. The user writes his Departing Station, Destination Station, Departing Time, Preference for regional trains only and Desired Buffer Time between stops.
2. The system runs a custom-made algorithm in Python that returns a list of maximum 3 distinct alternatives to the user's destination.
3. The user obtains the possible routes to his destination, with the option of opening a retractable list of all the stops each specific one has, so the path is entirely known with high attention to detail. If no routes are found for that specific destination or time, the user is given feedback and prompted to try again with new values.
4. If the user is satisfied with the obtained routes, the user has the option to request, once per minute, the current status of his desired train route to view any possible delays, changes of track, or possible cancellations.

## Under the Hood (How the most relevant parts work internally)
- **Custom-made algorithm:**
The system utilizes a beam-search principle that works on a series of depth-steps, instead of brute-force exploration, to make the most out of the least calls possible. 

First, the system obtains the latitude and longitude of the base train station and the destination station; then, the system queries Deutsche Bahn's live Timetable API for real departures from the base station at the desired time; based on a harvesine distance-calculation, taking into account the earth radius in kilometers (6371), the routes that take the closest to the destination are selected to then run another beam-depth search if the route hasn't yet been achieved.

At each step, either the object of candidate routes expands (querying further stations along the most promising branches) or collapses (narrowing several sibling candidates back down to the single closest-scoring one), before the next expansion begins.

My current system makes use of the following fetch-system:
[("expand", 3), ("expand", 3), ("expand", 2), ("collapse", 1), ("expand", 3)]

The candidates being expanded/collapsed and the children produced look the following way:
<img width="600" height="300" alt="image" src="https://github.com/user-attachments/assets/8166a2a9-23a4-448a-98fa-1186f6e4c41e" />

**In the worst case scenario, we get 32 departure fetches**

The system makes API-calls from the Deutsche Bahn Timetable to retrieve all the departures from a given train station. Since the algorithm caches all the departures, it means that trains getting stops at repeated stations don't get further API-calls.

The current system uses on average 4 API-calls per search. Nonetheless, the algorithm has a local-minimum constraint, namely: For each route-candidate we calculate a score (the remaining distance between the endpoint-coordinates and the destination-coordinates), which is a straight-line. 

Hence, for routes that have no direct approximation the current system provides a fallback partial route. At the moment, I only use the distance variable to decide whether the route is valid and if the algorithm should query further in a linear-direction. For the next phase, I'm going to obtain the information from the German National Datashare-Database to have more variables to provide a much richer decision-taking process.

I chose the following beam-search principle, considering the fact that the more far away the user is from the destination, the wider the search ought to be; the closer the kilometer distance is, the narrower it must be (i.e. branching factor should shrink as the route approaches the destination).

The worst-case scenario possible is when a user needs to travel from a city that is at the extreme of a cardinal point to the other extreme (e.g. from north to south or from west to east), these are the cities located at the respective extremes:
- East: Görlitz
- West: Aachen
- North: Flensburg
- South: Sonthofen

On a busy Wednesday (the most busy days of the week for the German Rail System are Monday - Friday, within 7:00 - 9:00, and 16:00 - 19:00), these are the required train-connections to take for one extreme cardinal-point to the other (both Regional Trains and Long Distance Trains mixed):
- North -> South: 5
- South -> North: 4
- West -> East: 4
- East -> West: 5

On average, 4 connecting trains required to travel from one extreme of the country to another.

Since most missed trains are for shorter distances, even considering the worst-case scenario proposed above, the steps-excercise suggested above covers a total of 3 expansion depth-steps that most of the times retrieve the most optimal route to get to the destination. Nonetheless, acknowledging the 4 average depth-expansion calls required to cover the most extreme possibilities, I would need to make more API-calls, which with my current project's internal architecture in the prototype-phase, simply isn't possible. 

Evidently the system only applies for routes requiring at least one train-connection to make it to the destination; for the case of trains leading directly to the destination, the beam-search is omitted and the direct route is returned.

It's worth mentioning that my code currently only works for the Deutsche Bahn Rail System, namely, only German trains in German territory.

## Search and Enrichment: A Two-Phase Process
Running the full beam-search with precise arrival-time lookups at every step
would be far too slow and API-expensive to use in practice. Instead, the
system splits the work into two phases:

**Phase 1 — Search.** The algorithm described above explores the beam search
using rough time estimates to keep each hop moving forward, without paying the
cost of a precise arrival-time lookup for every candidate it considers —
including the many branches that get discarded almost immediately. This keeps
the search itself fast and chronologically honest, without wasting real API
calls on routes the search won't ultimately use.

**Phase 2 — Enrichment.** Once a small number of final route candidates (up
to 3) are chosen, the system computes the details that are precise but
expensive: exact arrival times for every stop, the total journey duration,
and whether each leg's data was confirmed or estimated. Because this runs
only on the already-narrowed final routes rather than every candidate
explored, it stays fast even though it's doing more careful, individual work
per route.

Both phases rely on the same core estimation formula — a window of plausible
arrival times, bounded by an optimistic (125 km/h) and pessimistic (55 km/h)
average speed, used to decide how far forward to search for a train's real,
confirmed arrival time before falling back to an estimate:

```python
fast_speed_kmh = 125   # optimistic — direct, fast ICE service
slow_speed_kmh = 55    # pessimistic — slow, indirect, or heavily-stopping service

fastest_arrival = departure_time + timedelta(hours=distance_km / fast_speed_kmh)
slowest_arrival = departure_time + timedelta(hours=distance_km / slow_speed_kmh)

window_start = fastest_arrival - timedelta(minutes=30)
span = slowest_arrival - window_start
search_hours = int(span.total_seconds() // 3600) + 1
```

If no complete route can be found within the search depth, the system does
not simply return "no results." Instead, it returns the closest partial
route it found — one that gets the user as far as possible, even if it
doesn't reach the final destination — along with how far short it falls.

This follows a principle from Don Norman's *The Design of Everyday Things*:
a system should **always** give the user feedback, even when that feedback
isn't the answer they hoped for. A stranded traveler with an honest partial
answer can still act on it; one told nothing at all cannot.

## MCP-Server
The MCP-server currently offers a total of 4 methods, each with a distinct role: 

- **def find_alternatives(current_station: str, destination: str, date: str, departure_time: str, regional_only: bool = False, buffer: int = 5) -> dict:** 
Returns an object containing a list of routes that get cached in the chat, so as to be later accessed by the AI through other tools, which gets used as the conversation-basis to solve the user's immediate problem to get to his destination based on the required preferences.

- **def find_route_status(route_number: int) -> list[dict]:**
Based on the cached-list of routes in a Python Dictionary, the AI gets the users consentment to which route he would like to know the current status, so the AI only has to return a single number (1 - 3) to obtain the current route status.

- **def time_approximate_calculator(base_station_name: str, final_station_name: str, date: str, departure_time: str) -> dict:**
Gives a rough estimate of how long a train journey between two stations might take based on their coordinates and hour of departure; useful as a conversation helper only.

- **def departures_window_search(base_station: str, departure_date: str, departure_time: str, is_regional: bool = False, direction: str = None, max_results: int = 8) -> dict:**
Shows departures from a station around a given time; useful as a conversation helper only.

Understanding how LLMs work is key to MCP-tool optimization, and to avoiding over-doing prompt-engineering to solve internal method-problems: 
- An LLM has to regenerate every value as text, token by token — even data it already received perfectly formed. Caching route results server-side and letting the model reference them by a simple index removed that fragility entirely (see *find_route_status()* above).

Another hurdle was managing the prompt-engineering to make the AI have the same behaviour consistenly, without using the one-shot approach I've used in prior projects (which in this case appeared to directly influence the way the AI was calling the MCP-tools, which made the server utterly useless).

Consistency, disambiguation statements, a set of clearly-defined rules, and similar techniques were used to provide the most consistent behaviour.

Acknowledging the risks of prompt-injection and data-manipulation, I'm taking consideration into the improvement of security concerns when releasing the service to the public.

## String Station-Matching
Another major hurdle was matching the user's written station name against the
desired station out of 6,511 entries of all train stops at national level.

This problem became a major hurdle once the MCP server was set up — the
model's own imprecision in how it phrased station names (translating to
English, guessing variant names) could otherwise send it into repeated,
unresolved clarification loops (more on how I handled a related case in the
frontend below).

For the backend case, I developed a two-stage matching system:

**Stage 1 — Exact match.** The user's input is lowercased and compared
directly against every station name. If exactly one station matches
perfectly, it's returned immediately — no further ranking needed. This
matters because a naive substring search fails in a specific, common way:
searching "München Hbf" would otherwise match not just the main station, but
also "München Hbf Gl.27-36", "München Hbf (tief)", and other platform-level
variants — all equally valid substring matches, with no way to tell the AI
which one the user actually meant. Checking for a clean exact match first
resolves the common case outright, before ambiguity has a chance to appear.

**Stage 2 — Ranked partial match.** If there's no single exact match, every
station name is scanned for the query string appearing anywhere within it,
and ranked:
1. Query not found anywhere in the name → excluded entirely
2. Query appears at the very start of the name → rank 0 (highest priority)
3. Query starts a later word, preceded by a space (e.g. "Hbf" matching
   "Berlin Hbf") → rank 1
4. Query appears mid-word, regardless of position (e.g. "hof" matching
   "Bahnhofstraße") → rank 2 (lowest priority)

The system only returns a total of 7 maximum matches, and 
they are sorted by rank, so the most relevant stations always surface
first, and the AI is only asked to disambiguate when genuine ambiguity
exists.

*Direct Python implementation:*
```python
# An exact match always wins outright, regardless of what else
# happens to share the same prefix
exact_matches = stations_df[stations_df["NAME"].str.lower() == query_lower]

if len(exact_matches) == 1:
    row = exact_matches.iloc[0]
    return [{"name": row["NAME"], "eva": row["EVA_NR"]}]

# Only reached if there's no clean single exact match —
# fall back to ranked prefix/substring matching
scored_matches = []
for _, row in stations_df.iterrows():
    name_lower = row["NAME"].lower()
    idx = name_lower.find(query_lower)
    if idx == -1:
        continue
    if idx == 0:
        rank = 0
    elif name_lower[idx - 1] == " ":
        rank = 1
    else:
        rank = 2
    scored_matches.append((rank, row["NAME"], row["EVA_NR"]))

scored_matches.sort(key=lambda x: x[0])
return [{"name": name, "eva": eva} for _, name, eva in scored_matches[:max_results]]
```

On the frontend, a related but simpler problem exists: a user typing "Munich"
instead of "München", or "Nuremberg" instead of "Nürnberg", since the
underlying dataset only stores German station names. This is handled with a
small alias-lookup step — the input is lowercased, trimmed, and checked
against a set of known English-to-German city name mappings before the
match is attempted, so the correct station is still found even when the
user types the English form.

## Tech Stack
**Backend / Core Algorithm**
- Python — route-search engine (custom beam search + haversine-based heuristic), 
  data parsing, and business logic
- Deutsche Bahn Timetable API — live departure/arrival data and real-time 
  disruption/cancellation status (`/plan` and `/fchg` endpoints)
- pandas — local station dataset (name, coordinates, EVA-code lookup) built 
  from Deutsche Bahn's open station data

**AI / Conversational Layer**
- Model Context Protocol (MCP) — exposes the route-search engine as a set of 
  callable tools to an LLM, rather than hard-coding conversational logic
- LangChain / LangGraph — agent orchestration and tool-calling between the 
  LLM and the MCP server
- Ollama, running Qwen2.5 *(for local development/testing)*
- Cloudflare Workers AI *(for hosted AI model)* — the language model layer 
  itself, swappable behind a standard OpenAI-compatible interface 

**Frontend**
- Vite as framework; standard HTML, CSS and JavaScript

**Infrastructure**
- Not yet publicly hosted, but to be hosted in Railway/Render
- Domain Registrar — missedmytrain.com

## Status
Currently, the system finds itself in prototype-stage.  
The core route-finding algorithm is correct
and reliable — it consistently returns routes that genuinely reach the
requested destination. Its one known limitation is depth: for rare,
long-distance, poorly-connected routes (e.g. Hof to Weimar at an inconvenient
hour), the search may not find a complete route within its current API-call
budget. In that case, the system doesn't fail silently — it returns the
closest reachable partial route instead, along with how far short it falls,
consistent with the feedback-first design principle above.

The next planned step is migrating from live, rate-limited API calls to a
locally-cached copy of Germany's national timetable (updated on a weekly basis), 
sourced from the official National Data Share (DELFI/GTFS). 

This removes the current API-call ceiling entirely, allowing 
deeper search where needed without added cost per query, while 
keeping the same live API for real-time disruption checks.

## A Note on AI-Assisted Development
This project was built with AI assistance used deliberately and transparently,
in the same way a developer might use pair programming, code review, or
Stack Overflow — not as a substitute for understanding the system, but as a
tool within it.

**Backend and algorithm design:** 
Every architectural decision — the beam-search shape, the two-phase search/enrich split, 
the station-matching-ranking system, the MCP tool boundaries — was mine. 

I used AI extensively for debugging: tracing errors back to their root cause, 
catching my own logic mistakes, and working through unfamiliar 
Python constructs and libraries as I learned them. I wrote and understand the 
resulting code; I can explain why every part of it works the way it does.

**Frontend:** 
Built with heavier AI assistance on implementation, under my
own design direction and review.

I see this as representative of how I expect to work going forward: 
using AI as a genuine force multiplier, while staying fully accountable for
understanding, testing, and defending every decision in the system I ship.

## Sources
https://www.dw.com/en/over-a-third-of-deutsche-bahn-long-distance-trains-late/a-71215006

https://www.spiegel.de/wirtschaft/deutsche-bahn-jeder-zwoelfte-fernzug-faellt-im-ersten-halbjahr-aus-a-deae314f-ca15-490d-806d-6741e4ddc1ab

https://zbir.deutschebahn.com/2025/fileadmin/downloads/DB_ZB25_d_web.pdf
