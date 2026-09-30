# rhino_portal.py - A lightweight companion service for toolhead generation
from flask import Flask, render_template_string, request
import os

app = Flask(__name__)

CONFIG_PATH = os.path.expanduser("~/printer_data/config/printer.cfg")
TOOLHEADS_DIR = os.path.expanduser("~/printer_data/config/toolheads/")

def get_claimed_pins():
    """Scans printer.cfg to build an inventory of pins already used by the machine"""
    claimed = []
    if os.path.exists(CONFIG_PATH):
        with open(CONFIG_PATH, "r") as f:
            for line in f:
                if "pin:" in line and not line.strip().startswith("#"):
                    raw_val = line.split(":")[-1].strip()
                    clean_val = raw_val.split("#")[0].strip()
                    if clean_val:
                        claimed.append(clean_val)
    return sorted(list(set(claimed)))

@app.route("/")
def index():
    claimed = get_claimed_pins()
    all_pins = ["PE0", "PE1", "PE2", "PE3", "PF4", "PB0", "PB1", "PC0", "PC1"]
    available_pins = [p for p in all_pins if p not in claimed]

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
    
    if not os.path.exists(TOOLHEADS_DIR):
        os.makedirs(TOOLHEADS_DIR)
        
    fn = "{}.cfg".format(name.lower())
    file_path = os.path.join(TOOLHEADS_DIR, fn)
    with open(file_path, "w") as f:
        f.write("# Automated capability file\n")
        f.write("[gcode_macro CUSTOM_TOOL_{}]\n".format(name.upper()))
        f.write('variable_assigned_pin: "{}"\n'.format(pin))
        
    os.system("curl -X POST http://localhost:7125/printer/firmware_restart")
    return "Profile written! Klipper is restarting to register your new configurations..."

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
