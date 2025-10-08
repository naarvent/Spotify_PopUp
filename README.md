# Spotify Now Playing Widget

A stylish and interactive floating widget for displaying your current Spotify track.  
Built with **Python** and **PyQt5**, this widget mimics the Spotify design language and includes real-time updates, draggable positioning, fade animations, and full Spotify API integration using `spotipy`.

## 🎧 Features

- 📡 Real-time playback tracking via Spotify API
- 🎛️ Modern and clean design inspired by the official Spotify overlay
- ⏱️ Shows current time and total track duration
- 🎵 Displays album artwork, track title, and artist (supports podcasts too)
- 🎚️ Interactive progress bar with hover effects and click-to-seek
- 🔘 Handle appears when hovering and smoothly protrudes above the bar
- 🖱️ Right-click to drag the widget anywhere on screen
- 🖱️ Triple right-click to snap it back to bottom-right corner
- 💨 Smooth fade-in/out animation with auto-hide after inactivity

## 📷 Example UI

<img width="1920" height="1080" alt="imagen" src="https://github.com/user-attachments/assets/1e2a3b6b-93c8-4ca1-bbe0-06f1bf234e4e" />

## 🛠 Requirements

Install the following Python libraries:

      pip install spotipy PyQt5 pillow requests keyboard


## 🚀 How to Use

1. Make sure a track is currently playing in the Spotify **Desktop App**.
2. Run the script:
   
      python spotify_widget.py

4. Press `F10` (or `+` key) to manually toggle the visibility of the widget.

⚠️ **Important:** Spotify playback must be active. If you only have the app open without music playing, the widget won’t show any info.


## ⚙️ Configuration

You’ll need to create a Spotify developer application at [Spotify Developer Dashboard](https://developer.spotify.com/dashboard/) and copy your:

- `Client ID`
- `Client Secret`
- Add `http://127.0.0.1:8000/callback` as a redirect URI

Then, update these values in your Python script:

      client_id="YOUR_CLIENT_ID"
      client_secret="YOUR_CLIENT_SECRET"
      redirect_uri="http://127.0.0.1:8000/callback"


## 🔁 Automatically Run on Windows Startup (Optional)

If you want this widget to run when your PC starts:

1. Convert the script into a .exe using PyInstaller:

        pip install pyinstaller
        pyinstaller --onefile spotify_widget.py

2. Press Win + R, type:

        shell:startup

3. Place a shortcut of the .exe inside that folder.

## 📁 File Structure and Safety

All generated files such as `.cache` (used for Spotify API credentials) will be stored inside:
Documents/
└── naarvents projects/
└── Spotify Widget/
This ensures that the widget does not interfere with other Spotify-based scripts or tools you might be using. The folder is created automatically on first run.


## 📄 License
This widget is free to use and modify. If you share it, consider crediting this repo.

## 🧠 Credits
- Developed with Python by [@naarvent]
- Spotify API wrapper: spotipy
- GUI: PyQt5
