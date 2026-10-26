User IDs



Nova - 0

Darius - 1

Simone - 2

Kenji - 3

Aaliya - 4



Requests - Header or Token?



Repeating songs

* The only song that repeats is "Crown Heights Anthem".



Condition that Triggers Duplicates (Issue #3):

* songs with 3+ tags, expose issue 3 -->
* song\_data\_multi\_tags = \[

&#x20;           ("Crown Heights Anthem", "Borough Kings",     "rap",      \["rap", "hip-hop", "boom bap"]),

&#x20;           ("Harlem Renaissance",   "Uptown Collective", "hip-hop",  \["rap", "hip-hop", "soul"]),

&#x20;           ("After Hours",          "Night City",        "r\&b",      \["r\&b", "soul", "jazz"]),

&#x20;           ("Lagos to London",      "Afrowave",          "afrobeats", \["afrobeats", "electronic", "r\&b"]),

&#x20;           ("Frequencies",          "Static Era",        "electronic", \["electronic", "lo-fi", "indie"]),

&#x20;       ]





Last playlist song missing (Issue #5)

Friday Energy number: 3c829070-8c92-4568-9221-be4e2cd0ee1a (Friday Energy)

With 

&#x09;Invoke-RestMethod "http://127.0.0.1:5000/playlists/3c829070-8c92-4568-9221-be4e2cd0ee1a/songs" | ConvertTo-Json -Depth 5 c

command --> there are 6 songs returned.

