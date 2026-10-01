# app.py - Production Ready Entry Node for Rhino Portal
import os
from flask import Flask

app = Flask(__name__)
app.secret_key = "rhino_secret_system_key"

# Define system paths globally
CONFIG_DIR = os.path.expanduser("~/printer_data/config")
UPLOAD_FOLDER = os.path.join(CONFIG_DIR, "toolhead_images")

app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
app.config['MAX_CONTENT_LENGTH'] = 16 * 1024 * 1024  # 16MB Max upload limit

# Deferred runtime import to completely eliminate cross-file import circular syntax crashes
import routes
routes.register_routes(app)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
