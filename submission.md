The routes has 4 files: feed.py, playlists.py, songs.py, and users.py.  
- in feed.py, it takes in get_friends_listening_now() and get_activity_feed() from feed_service.py  it returns a json object of feed and count from listening now, it also returns their activity by feed and count 
- in playlists.py, it takes in create_playlist, get_playlist_songs, and get_
playlist 
    - when someone "posts" it creates a playlist with the data, name, and created by and returns that as a dictionary 
    - when someone "visits" the playlist id are of flask, they get the playlist
    - when someone "visits" the songs, they get a songs a count dictionary
    - when some "posts" songs, the song gets added, 
-in songs.py, when someone searches, they get their query back
    - when someone "vists" by song_id, they get a json object of that song ID
    - when some "posts" a rate, it gets added or an error
    - when someone "posts" a listen, the listening event gets triggered
-in users.py, it gets the user by id, 
    - the streak returns the streak
    - the nofications returns the notifications and marks them as read
