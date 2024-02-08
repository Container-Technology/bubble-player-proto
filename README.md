# Bubble radio

**Bubble radio** is a song archiving and web radio platform that is designed to work in conjunction with Bubble, a Discord bot developed for Trust.

The main components of this project are:

- A **Flask server** which 
	- receives and processes POST requests from Bubble
	- serves as a rudimentary frontend where users can access the radio streams
- A **`downloader.py`** script which downloads songs from links provided via POST request and commits the relevant metadata to a sqlite3 database
- A **`playlists.py`** script which creates `.m3u` files to be used when serving the radio streams
- A **`start_streams.sh`** shell script which utilizes `liquidsoap_script_template` and values from the `.env` file to initialize the radio streams

### Prerequisites

- **Icecast 2** (streaming server)
- **Liquidsoap** (stream generator)
- Python 3.10+

Installing Icecast 2 is fairly straightforward. In case it's helpful, there is a sample `icecast.xml` config file included in this repo. This config is currently working with my instance of Bubble radio; just replace the password values and adjust the listening port if necessary. On unix systems this file typically lives at `/etc/icecast2/icecast.xml`.

Liquidsoap is easily installed and doesn't require any configuration beyond setting the correct values in the `.env` file for this project.

### Setup

**Download / clone this repo**

`git clone https://github.com/stonymf/bubble-radio`

**Install dependencies**

`pip install -r requirements.txt`

**Create a `.env` file from the below template, replacing the values with your own.** Use absolute paths wherever filepaths are required. Dummy values are provided for a user `janedoe` who has cloned this repo into the directory `/home/janedoe/dev/bubble-radio`

```## General config

# maximum length in seconds above which songs will not be downloaded
MAX_LENGTH=4140

# where you want audio files to be downloaded
DOWNLOAD_DIRECTORY=/home/janedoe/dev/bubble-radio/downloads

# where you want log files to live
LOG_DIRECTORY=/home/janedoe/dev/bubble-radio/logs

# where you want playlist files to live
PLAYLIST_DIRECTORY=/home/janedoe/dev/bubble-radio/playlists

# where you want the database to live
DB_PATH=/home/janedoe/dev/bubble-radio/downloads.db

# url where streams are being served by icecast
BASE_STREAM_URL=https://yoururl.com

# secret key to verify POST requests from Bubble
# this can be any arbitrary string of characters
# it just needs to match the "Authorization" value for incoming POST requests
SECRET_KEY=yoursecretkey



## Liquidsoap and Icecast config

# this is where Liquidsoap will look for your Icecast server
ICECAST_HOST=localhost
ICECAST_PORT=8088

# this needs to match with the source-password value in your icecast.xml
ICECAST_PASSWORD=icecastsourcepassword

# set this to the absolute filepath of liquidsoap_script_template.liq
LIQUIDSOAP_SCRIPT_TEMPLATE_PATH=/home/janedoe/dev/bubble-radio/liquidsoap_script_template.liq
```

**Now create a blank `disallow.txt` file in your project's main directory.** This is where you can add sites you would like to ignore, one url on each line. For example, I currently just have one line containing `substack.com` because `yt-dlp` does funny things with substack urls.

**Finally, set up a cron job to run `playlists.py` at regular intervals.** In the terminal, run `crontab -e`, and then add this line at the end of the file, making changes to the filepaths as needed:

`0 */6 * * * /usr/bin/python3 /home/janedoe/dev/bubble-radio/playlists.py`

### Initial use

Open a new `screen` session to run the Flask server:

`screen -R bubble-radio-flask-server`

Make sure you're in the project's main directory, then:

`python app.py`

Now exit the `screen` session (Ctrl-A then Ctrl-D)

With the server now running, try a test POST request. First, edit `test_post_request.py` to reflect your own values, then run `python test_post_request.py`

If a success status is returned, the song should now be in the downloads directory you specified in the `.env` file.

Now, you can manually run `playlists.py` in order to generate a playlists containing the newly downloaded track:

`python playlists.py`

This should generate playlists according to the server and channel that the downloaded songs originated from. These playlists will show up in your specified playlists directory.

With the playlist files created, you can now run the `start_streams.sh` shell script in the main project directory. If your Liquidsoap and Icecast configurations are correct (both in your `.env` file and in `/etc/icecast2/icecast.xml`), this shell script should launch as many streams as there are playlist files and serve them via Icecast. You may need to grant permissions to this file with `chmod +x start_streams.sh` before running it.