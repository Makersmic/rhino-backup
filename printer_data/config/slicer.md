# rhino_portal.py - Upgraded Capability Provisioning Portal for the Rhino System
from flask import Flask, render_template_string, request, redirect
import os
import re

app = Flask(__name__)

# Paths to your printer configuration tree
CONFIG_DIR = os.path.expanduser("~/printer_data/config")
CONFIG_PATH = os.path.join(CONFIG_DIR, "printer.cfg")
TOOLHEADS_DIR = os.path.join(CONFIG_DIR, "toolheads")

def get_detailed_pin_map():
    """Scans printer.cfg and included files to map out exactly what is using each pin"""
    pin_map = {}
    
    # Simple recursive scanner to capture main config plus loose include files
    files_to_scan = [CONFIG_PATH]
    if os.path.exists(CONFIG_PATH):
        with open(CONFIG_PATH, "r") as f:
            for line in f:
                if line.strip().startswith("[include") and ".cfg" in line:
                    inc_file = line.split("include")[-1].strip().strip("]").strip()
                    full_inc_path = os.path.join(CONFIG_DIR, inc_file)
                    if os.path.exists(full_inc_path):
                        files_to_scan.append(full_inc_path)

    for file_path in files_to_scan:
        if not os.path.exists(file_path):
            continue
        current_section = "root"
        with open(file_path, "r") as f:
            for line in f:
                line = line.strip()
                if not line or line.startswith("#"):
                    continue
                # Detect configuration blocks
                if line.startswith("[") and line.endswith("]"):
                    current_section = line[1:-1]
                elif "pin:" in line:
                    param = line.split(":")[0].strip()
                    raw_pin = line.split(":")[-1].strip().split("#")[0].strip()
                    # Strip Klipper hardware modifiers (! inverted, ^ pullup)
                    clean_pin = raw_pin.lstrip("!").lstrip("^").lstrip("~")
                    if clean_pin:
                        pin_map[clean_pin] = "{} ({})".format(current_section, param)
    return pin_map

@app.route("/")
def index():
    pin_map = get_detailed_pin_map()
    
    # Your 21-pin umbilical's designated hardware channels on your main board
    umbilical_whitelist = ["PE0", "PE1", "PE2", "PE3", "PF4", "PB0", "PB1", "PC0", "PC1"]
    available_pins = [p for p in umbilical_whitelist if p not in pin_map]

    html = """
    <html>
    <head>
        <title>Rhino Toolhead Provisioning Center</title>
        <style>
            body { font-family: sans-serif; margin: 40px; background: #1e1e2e; color: #cdd6f4; line-height: 1.5; }
            .container { max-width: 900px; margin: 0 auto; background: #313244; padding: 30px; border-radius: 8px; box-shadow: 0 4px 10px rgba(0,0,0,0.3); }
            h1, h2, h3 { color: #f5c2e7; }
            .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-top: 20px; }
            .panel { background: #181825; padding: 20px; border-radius: 6px; border: 1px solid #45475a; }
            .btn { padding: 10px 20px; background: #a6e3a1; color: #11111b; border: none; border-radius: 4px; cursor: pointer; font-weight: bold; width: 100%; margin-top: 15px; }
            .btn:hover { background: #94e2d5; }
            input, select { width: 100%; padding: 8px; background: #45475a; color: #cdd6f4; border: 1px solid #585b70; border-radius: 4px; box-sizing: border-box; }
            .badge { background: #f38ba8; color: #11111b; padding: 2px 6px; border-radius: 4px; font-size: 0.85em; font-family: monospace; }
            .avail { background: #a6e3a1; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>🔧 Rhino Toolhead Provisioning Center</h1>
            <p>Unified Capability Wizard & Pin Collision Diagnostic Center</p>
            <hr style="border-color: #45475a;" />

            <div class="grid">
                <!-- LEFT PANEL: THE CAPABILITY GENERATION WIZARD -->
                <div class="panel">
                    <h2>🛠️ Add New Toolhead Wizard</h2>
                    <form action="/create" method="POST">
                        <label><b>1. Toolhead Profile Name:</b></label><br/>
                        <input type="text" name="name" placeholder="e.g., HotWire, NeedleCutter" required><br/><br/>
                        
                        <label><b>2. Select Base Template:</b></label><br/>
                        <select name="template" id="template" onchange="toggleTemplateCapabilities()">
                            <option value="subtractive">⚙️ Subtractive Template (CNC/Cutter Base)</option>
                            <option value="additive">🧵 Additive Template (Extruder/3D Print Base)</option>
                        </select><br/><br/>

                        <label><b>3. Configure Hardware Capabilities:</b></label><br/>
                        <div style="background:#313244; padding:10px; border-radius:4px; margin-top:5px;">
                            <input type="checkbox" name="cap_pwm" value="1" style="width:auto;"> Modulated PWM Power Channel<br/>
                            <input type="checkbox" name="cap_oscillator" value="1" style="width:auto;"> Reciprocating Oscillator Servo Loop<br/>
                            <input type="checkbox" name="cap_extruder" value="1" style="width:auto;"> Synchronized Filament Drive Motor
                        </div><br/>
                        
                        <label><b>4. Umbilical Pin Assignment (Collision Protected):</b></label><br/>
                        <select name="pin">
                            {% for p in available_pins %}
                                <option value="{{p}}">{{p}} (Available)</option>
                            {% endfor %}
                        </select><br/><br/>

                        <input type="submit" class="btn" value="📦 Write Configuration & Reboot">
                    </form>
                </div>

                <!-- RIGHT PANEL: THE DETAILED PIN ASSET DICTIONARY Map -->
                <div class="panel">
                    <h2>🔍 Live Pin Assignment Registry</h2>
                    <p>Scanned active pin asset mappings across your entire system profile tree:</p>
                    <div style="max-height: 400px; overflow-y: auto; background: #11111b; padding: 15px; border-radius: 4px; font-size: 0.9em;">
                        <h4 style="margin-top:0; color:#89b4fa;">🚫 LOCKED / CLAIMED CHANNELS:</h4>
                        {% for pin, component in pin_map.items()|sort %}
                            <div style="margin-bottom: 8px; border-bottom: 1px solid #313244; padding-bottom: 4px;">
                                <span class="badge">{{ pin }}</span> ➡️ <span style="color: #a6adc8;">{{ component }}</span>
                            </div>
                        {% endfor %}
                        
                        <h4 style="margin-top:20px; color:#a6e3a1;">✅ OPEN UMBILICAL CHANNELS:</h4>
                        {% for p in available_pins %}
                            <div style="margin-bottom: 4px;"><span class="badge avail">{{ p }}</span> Ready for custom tool mapping</div>
                        {% endfor %}
                    </div>
                </div>
            </div>
        </div>
    </body>
    </html>
    """
    return render_template_string(html, pin_map=pin_map, available_pins=available_pins)

@app.route("/create", methods=["POST"])
def create():
    name = request.form.get("name")
    template = request.form.get("template")
    pin = request.form.get("pin")
    
    cap_pwm = request.form.get("cap_pwm", "0")
    cap_oscillator = request.form.get("cap_oscillator", "0")
    cap_extruder = request.form.get("cap_extruder", "0")
    
    if not os.path.exists(TOOLHEADS_DIR):
        os.makedirs(TOOLHEADS_DIR)
        
    fn = "{}.cfg".format(name.lower())
    file_path = os.path.join(TOOLHEADS_DIR, fn)
    
    with open(file_path, "w") as f:
        f.write("# ====================================================================\n")
        f.write("# 🛠️ AUTOMATED HARDWARE CAPABILITY PROFILE FOR {}\n".format(name.upper()))
        f.write("# ====================================================================\n\n")
        f.write("[gcode_macro CUSTOM_TOOL_{}]\n".format(name.upper()))
        f.write("variable_tool_name: \"{}\"\n".format(name))
        f.write("variable_base_template: \"{}\"\n".format(template))
        f.write("variable_assigned_pin: \"{}\"\n".format(pin))
        f.write("variable_capability_pwm: {}\n".format(cap_pwm))
        f.write("variable_capability_oscillator: {}\n".format(cap_oscillator))
        f.write("variable_capability_extruder: {}\n\n".format(cap_extruder))
        
        f.write("gcode:\n")
        f.write("  # Automatically launch our unified capability wizard setup loop\n")
        f.write("  GENERIC_CAPABILITY_SETUP NAME=\"{}\" FEED_RATE=600 CAP_PWM={} CAP_OSCILLATOR={}\n".format(name, cap_pwm, cap_oscillator))
        
    # Send an instant core firmware restart command to Moonraker to compile the new file
    os.system("curl -X POST http://localhost:7125/printer/firmware_restart")
    return "Profile written to toolheads/{}.cfg successfully! Klipper is executing a firmware restart...".format(name.lower())

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
