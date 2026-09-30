# rhino_portal.py - A lightweight companion service for toolhead generation
from flask import Flask, render_template_string, request, redirect
import os

app = Flask(__name__)

# Paths to your printer configuration tree
CONFIG_PATH = os.path.expanduser("~/printer_data/config/printer.cfg")
TOOLHEADS_DIR = os.path.expanduser("~/printer_data/config/toolheads/")

def get_claimed_pins():
    """Scans printer.cfg to build an inventory of pins already used by the machine"""
    claimed = []
    if os.path.exists(CONFIG_PATH):
        with open(CONFIG_PATH, "r") as f:
            for line in f:
                if "pin:" in line and not line.strip().startswith("#"):
                    parts = line.split(":")[-1].strip().split("#")
                    pin_name = parts[0].strip() if parts else ""
                    if pin_name:
                        claimed.append(pin_name)
    return sorted(list(set(claimed)))

@app.route("/")
def index():
    claimed = get_claimed_pins()
    # A list of available pins on an Octopus board example for your dropdowns
    all_pins = ["PE0", "PE1", "PE2", "PE3", "PF4", "PB0", "PB1", "PC0", "PC1"]
    available_pins = [p for p in all_pins if p not in claimed]

    # HTML Form with drop-down menus rendered right on your network screen
    html = """
    <html>
    <head><title>Rhino Toolhead Provisioning Portal</title></head>
    <body style="font-family: sans-serif; margin: 40px; background: #1e1e2e; color: #cdd6f4;">
        <h2>Rhino Custom Toolhead Generator</h2>
        <p>Pins currently locked by printer.cfg (Protected from conflicts): <b>{{claimed}}</b></p>
        <hr/>
        <form action="/create" method="POST">
            <label>Toolhead Name:</label><br/>
            <input type="text" name="name" placeholder="e.g., NeedleCutter" required style="padding:5px;"><br/><br/>
            
            <label>Select Umbilical Pin (Dropdown):</label><br/>
            <select name="pin" style="padding:5px;">
                {% for p in available_pins %}
                    <option value="{{p}}">{{p}}</option>
                {% endfor %}
            </select><br/><br/>
            
            <input type="submit" value="Write Toolhead Config and Restart" style="padding:10px; background:#a6e3a1; border:none; border-radius:4px; cursor:pointer;">
        </form>
    </body>
    </html>
    """
    return render_template_string(html, claimed=", ".join(claimed), available_pins=available_pins)

@app.route("/create", methods=["POST"])
def create():
    name = request.form.get("name")
    pin = request.form.get("pin")
    
    # Ensure directory exists
    if not os.path.exists(TOOLHEADS_DIR):
        os.makedirs(TOOLHEADS_DIR)
        
    # Write the new unique file to your toolheads directory automatically
    file_path = os.path.join(TOOLHEADS_DIR, f"{name.lower()}.cfg")
    with open(file_path, "w") as f:
        f.write(f"# Automated capability file for {name}\n")
        f.write(f"[gcode_macro CUSTOM_TOOL_{name.upper()}]\n")
        f.write(f"variable_assigned_pin: \"{pin}\"\n")
        
    # Trigger a clean moonraker firmware restart command
    os.system("curl -X POST http://localhost:7125/printer/firmware_restart")
    return "Profile written! Klipper is restarting to register your new dropdown configurations..."

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
