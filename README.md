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

*(Insert a screenshot or GIF of the widget in action here)*

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


## 📄 License
This widget is free to use and modify. If you share it, consider crediting this repo.

## 🧠 Credits
- Developed with Python by [@naarvent]
- Spotify API wrapper: spotipy
- GUI: PyQt5


## Code

      import sys
      import threading
      import os
      import time
      import requests
      from io import BytesIO
      from PIL import Image
      import spotipy
      from spotipy.oauth2 import SpotifyOAuth
      import keyboard
      from PyQt5 import QtWidgets, QtGui, QtCore
      
      
      class SpotifyWidget(QtWidgets.QWidget):
          def __init__(self, spotify):
              super().__init__()
              self.spotify = spotify
              self.dragging = False
              self.last_click_time = []
      
              # Fade animation
              self.animation = QtCore.QPropertyAnimation(self, b"windowOpacity")
              self.animation.setDuration(400)
      
              # Auto-hide timer
              self.hide_timer = QtCore.QTimer()
              self.hide_timer.setInterval(1000)
              self.hide_timer.timeout.connect(self.fade_out)
      
              # Window settings
              self.setWindowFlags(
                  QtCore.Qt.FramelessWindowHint |
                  QtCore.Qt.WindowStaysOnTopHint |
                  QtCore.Qt.Tool
              )
              self.setAttribute(QtCore.Qt.WA_TranslucentBackground)
              self.setStyleSheet("background: transparent;")
              self.setMouseTracking(True)
              self.installEventFilter(self)
      
              self.init_ui()
      
              # Set default position (bottom right)
              screen = QtWidgets.QApplication.primaryScreen().availableGeometry()
              self.default_pos = QtCore.QPoint(screen.width() - 420, screen.height() - 140)
              self.move(self.default_pos)
      
              self.is_faded_in = False
              self.recently_shown = False
      
              self.update_timer = QtCore.QTimer()
              self.update_timer.setInterval(1000)
              self.update_timer.timeout.connect(self.update_track)
              self.update_timer.start()
      
              self.user_seeking = False
      
          def mousePressEvent(self, event):
              if event.button() == QtCore.Qt.RightButton:
                  now = time.time()
                  self.last_click_time = [t for t in self.last_click_time if now - t < 0.8]
                  self.last_click_time.append(now)
                  if len(self.last_click_time) >= 3:
                      self.move(self.default_pos)
                      self.last_click_time.clear()
                  else:
                      self.dragging = True
                      self.drag_pos = event.globalPos() - self.frameGeometry().topLeft()
                      event.accept()
      
          def mouseMoveEvent(self, event):
              if self.dragging and event.buttons() & QtCore.Qt.RightButton:
                  self.move(event.globalPos() - self.drag_pos)
                  event.accept()
      
          def mouseReleaseEvent(self, event):
              if event.button() == QtCore.Qt.RightButton:
                  self.dragging = False
      
          def init_ui(self):
              container = QtWidgets.QWidget(self)
              layout = QtWidgets.QHBoxLayout(container)
              layout.setContentsMargins(0, 0, 0, 0)
              layout.setSpacing(0)
      
              # Left section (album cover)
              left = QtWidgets.QWidget()
              left.setFixedWidth(120)
              left.setStyleSheet("background-color: #000000; border-top-left-radius: 20px; border-bottom-left-radius: 20px;")
              left_layout = QtWidgets.QVBoxLayout(left)
              left_layout.setContentsMargins(0, 0, 0, 0)
              left_layout.setAlignment(QtCore.Qt.AlignCenter)
      
              self.album_cover = QtWidgets.QLabel()
              self.album_cover.setFixedSize(80, 80)
              self.album_cover.setPixmap(QtGui.QPixmap(80, 80).filled(QtGui.QColor("gray")))
              left_layout.addWidget(self.album_cover)
      
              # Vertical white separator
              separator = QtWidgets.QFrame()
              separator.setFrameShape(QtWidgets.QFrame.VLine)
              separator.setLineWidth(5)
              separator.setStyleSheet("color: white;")
              separator.setFixedHeight(110)
      
              # Right section (info + progress)
              right = QtWidgets.QWidget()
              right.setStyleSheet("background-color: #000000; border-top-right-radius: 20px; border-bottom-right-radius: 20px;")
              right_layout = QtWidgets.QVBoxLayout(right)
              right_layout.setContentsMargins(20, 12, 20, 12)
              right_layout.setSpacing(2)
      
              # Track name
              self.track_name = QtWidgets.QLabel("Track Name")
              self.track_name.setFont(QtGui.QFont("Segoe UI", 11, QtGui.QFont.Bold))
              self.track_name.setStyleSheet("color: white;")
              right_layout.addWidget(self.track_name)
      
              # Artist name with decoration
              self.artist_label = QtWidgets.QLabel("| ●  Artist")
              self.artist_label.setFont(QtGui.QFont("Segoe UI", 10))
              self.artist_label.setStyleSheet("color: #cccccc; padding-left: 20px;")
              right_layout.addWidget(self.artist_label)
      
              # Progress bar and time display
              progress_layout = QtWidgets.QHBoxLayout()
              progress_layout.setContentsMargins(15, 6, 15, 0)
      
              self.current_time = QtWidgets.QLabel("0:00")
              self.current_time.setStyleSheet("color: white; font-size: 9pt;")
      
              self.total_time = QtWidgets.QLabel("3:45")
              self.total_time.setStyleSheet("color: white; font-size: 9pt;")
      
              self.progress_bar = QtWidgets.QSlider(QtCore.Qt.Horizontal)
              self.progress_bar.setFixedHeight(8)
      
              def set_slider_style(hovered):
                  if hovered:
                      self.progress_bar.setStyleSheet("""
                          QSlider::groove:horizontal {
                              height: 8px;
                              background: #202020;
                              border-radius: 4px;
                          }
                          QSlider::handle:horizontal {
                              background: white;
                              width: 12px;
                              height: 12px;
                              margin: -5px 0;
                              border-radius: 6px;
                          }
                          QSlider::sub-page:horizontal {
                              background: #1DB954;
                              border-radius: 4px;
                          }
                          QSlider::add-page:horizontal {
                              background: #202020;
                              border-radius: 4px;
                          }
                      """)
                  else:
                      self.progress_bar.setStyleSheet("""
                          QSlider::groove:horizontal {
                              height: 8px;
                              background: #202020;
                              border-radius: 4px;
                          }
                          QSlider::handle:horizontal {
                              background: transparent;
                              width: 0px;
                              margin: 0px;
                          }
                          QSlider::sub-page:horizontal {
                              background: white;
                              border-radius: 4px;
                          }
                          QSlider::add-page:horizontal {
                              background: #202020;
                              border-radius: 4px;
                          }
                      """)
      
              set_slider_style(False)
              self.progress_bar.setMouseTracking(True)
              self.progress_bar.enterEvent = lambda e: set_slider_style(True)
              self.progress_bar.leaveEvent = lambda e: set_slider_style(False)
              self.progress_bar.sliderPressed.connect(self.on_seek_start)
              self.progress_bar.sliderReleased.connect(self.on_seek_end)
      
              slider_wrapper = QtWidgets.QWidget()
              slider_layout = QtWidgets.QVBoxLayout(slider_wrapper)
              slider_layout.setContentsMargins(10, 0, 10, 0)
              slider_layout.setSpacing(0)
              slider_layout.addWidget(self.progress_bar)
      
              progress_layout.addWidget(self.current_time)
              progress_layout.addWidget(slider_wrapper)
              progress_layout.addWidget(self.total_time)
              right_layout.addLayout(progress_layout)
      
              # Combine sections
              layout.addWidget(left)
              layout.addWidget(separator)
              layout.addWidget(right)
      
              base_layout = QtWidgets.QVBoxLayout(self)
              base_layout.setContentsMargins(0, 0, 0, 0)
              base_layout.addWidget(container)
      
              self.setFixedSize(400, 110)
              self.setWindowOpacity(0.0)
              self.hide()
          def on_seek_start(self):
              self.user_seeking = True
      
          def on_seek_end(self):
              self.user_seeking = False
              position = self.progress_bar.value()
              self.spotify.seek_track(position * 1000)
      
          def update_track(self):
              if not hasattr(self, 'last_track_id'):
                  self.last_track_id = None
                  self.last_progress = None
              try:
                  if self.user_seeking:
                      return
      
                  playback = self.spotify.current_playback()
                  if playback and playback.get("item"):
                      item = playback["item"]
                      name = item.get("name", "")
                      artists = ", ".join(a["name"] for a in item.get("artists", [])) if item.get("artists") else item.get("show", {}).get("publisher", "")
                      progress = playback["progress_ms"] // 1000
                      duration = item["duration_ms"] // 1000
                      cover_url = item["album"]["images"][0]["url"] if "album" in item else item["images"][0]["url"]
      
                      self.track_name.setText(name)
                      self.artist_label.setText("| ●  " + artists)
      
                      current_track_id = item["id"]
                      if (self.last_track_id is None) or (self.last_track_id != current_track_id) or (progress == 0 and self.last_progress is not None and self.last_progress > 5):
                          self.fade_in()
                      self.last_track_id = current_track_id
                      self.last_progress = progress
                      self.progress_bar.setMaximum(duration)
                      self.progress_bar.setValue(progress)
                      self.current_time.setText(f"{progress // 60}:{progress % 60:02d}")
                      self.total_time.setText(f"{duration // 60}:{duration % 60:02d}")
      
                      response = requests.get(cover_url)
                      img = Image.open(BytesIO(response.content)).resize((80, 80)).convert("RGBA")
                      data = img.tobytes("raw", "RGBA")
                      qimg = QtGui.QImage(data, img.width, img.height, QtGui.QImage.Format_RGBA8888)
                      pixmap = QtGui.QPixmap.fromImage(qimg)
      
                      final_pixmap = QtGui.QPixmap(80, 80)
                      final_pixmap.fill(QtCore.Qt.transparent)
                      painter = QtGui.QPainter(final_pixmap)
                      painter.setRenderHint(QtGui.QPainter.Antialiasing)
                      path = QtGui.QPainterPath()
                      path.addEllipse(1, 1, 78, 78)
                      painter.setClipPath(path)
                      painter.drawPixmap(0, 0, pixmap)
                      pen = QtGui.QPen(QtCore.Qt.white, 2)
                      painter.setPen(pen)
                      painter.setBrush(QtCore.Qt.NoBrush)
                      painter.drawEllipse(1, 1, 78, 78)
                      painter.end()
      
                      self.album_cover.setPixmap(final_pixmap)
              except Exception as e:
                  print("Error retrieving Spotify info:", e)
      
          def fade_in(self):
              self.hide_timer.stop()
              self.animation.stop()
              self.setWindowOpacity(0.0)
              self.show()
              self.raise_()
              self.animation.setStartValue(0.0)
              self.animation.setEndValue(1.0)
              self.animation.start()
              self.recently_shown = True
              QtCore.QTimer.singleShot(1000, self.allow_auto_hide)
              self.hide_timer.start()
              self.is_faded_in = True
      
          def allow_auto_hide(self):
              self.recently_shown = False
      
          def fade_out(self):
              self.hide_timer.stop()
              self.animation.stop()
              self.animation.setStartValue(1.0)
              self.animation.setEndValue(0.0)
              self.animation.start()
              self.animation.finished.connect(self.hide_once)
      
          def hide_once(self):
              self.hide()
              self.animation.finished.disconnect(self.hide_once)
              self.is_faded_in = False
      
          def eventFilter(self, obj, event):
              if event.type() == QtCore.QEvent.Enter:
                  self.hide_timer.stop()
              elif event.type() == QtCore.QEvent.Leave:
                  if not self.recently_shown:
                      self.hide_timer.start()
              return super().eventFilter(obj, event)
      
      
      class ToggleEvent(QtCore.QEvent):
          EVENT_TYPE = QtCore.QEvent.Type(QtCore.QEvent.registerEventType())
          def __init__(self):
              super().__init__(ToggleEvent.EVENT_TYPE)
      
      
      def start_keyboard_listener(widget):
          def on_key_event(e):
              if e.name == "f10" or e.name == "+" or e.name == "equal":
                  QtWidgets.QApplication.postEvent(widget, ToggleEvent())
          keyboard.on_press(on_key_event)
      
      
      def run_app():
          cache_path = ".cache"
          try:
              auth = SpotifyOAuth(
                  client_id="YOUR_CLIENT_ID",
                  client_secret="YOUR_CLIENT_SECRET",
                  redirect_uri="http://127.0.0.1:8000/callback",
                  scope="user-read-playback-state user-read-currently-playing user-modify-playback-state",
                  cache_path=cache_path
              )
              spotify = spotipy.Spotify(auth_manager=auth)
      
          except spotipy.SpotifyOauthError as e:
              print("❌ Authentication error:", e)
              if os.path.exists(cache_path):
                  os.remove(cache_path)
                  print("🗑 Cache cleared. Please run again to reauthorize.")
              else:
                  print("ℹ No cache found to delete.")
              return
      
          app = QtWidgets.QApplication(sys.argv)
          widget = SpotifyWidget(spotify)
      
          def customEvent(event):
              if isinstance(event, ToggleEvent):
                  if widget.is_faded_in:
                      widget.fade_out()
                  else:
                      widget.fade_in()
      
          widget.customEvent = customEvent
          app.installEventFilter(widget)
      
          threading.Thread(target=start_keyboard_listener, args=(widget,), daemon=True).start()
          sys.exit(app.exec_())
      
      
      if __name__ == "__main__":
          run_app()

