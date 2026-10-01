# app.py - Fixed Main Entry Point for the Rhino Capability Portal
import os
from flask import Flask

app = Flask(__name__)
app.secret_key = "rhino_secret_system_key"

# Define system paths globally
CONFIG_DIR = os.path.expanduser("~/printer_data/config")
UPLOAD_FOLDER = os.path.join(CONFIG_DIR, "toolhead_images")

app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
app.config['MAX_CONTENT_LENGTH'] = 16 * 1024 * 1024  # 16MB Max upload size

# Import and execute the registration directly inside the app namespace
from routes import register_routes
register_routes(app)

if __name__ == "__main__":
    # Runs on port 5000 with debug tracking turned on for troubleshooting
    app.run(host="0.0.0.0", port=5000, debug=True)
