# rhino_portal.py - Complete Unified Slicer, Toolhead & Asset Management Center
from flask import Flask, render_template_string, request, redirect, url_for
import os

app = Flask(__name__)

# Paths to your printer configuration tree
CONFIG_DIR = os.path.expanduser("~/printer_data/config")
CONFIG_PATH = os.path.join(CONFIG_DIR, "printer.cfg")
TOOLHEADS_DIR = os.path.join(CONFIG_DIR, "toolheads")
UPLOAD_FOLDER = os.path.join(CONFIG_DIR, "tool_images")

app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
if not os.path.exists(UPLOAD_FOLDER):
    os.makedirs(UPLOAD_FOLDER)

# Structural Whitelist for 21-pin umbilical available channels
UMBILICAL_WHITELIST = ["PE0", "PE1", "PE2", "PE3", "PF4", "PB0", "PB1", "PC0", "PC1"]

def get_detailed_pin_map():
    """Scans printer.cfg and includes to group pinned assets by functional headers"""
    grouped_map = {
        "📊 STEPPER DRIVES": [],
        "🔥 HEATING & COOLING": [],
        "🔌 SYSTEM SIGNALS & ENDSTOPS": [],
        "🛠️ CUSTOM TOOLING CHANNELS": []
    }
    
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
                if line.startswith("[") and line.endswith("]"):
                    current_section = line[1:-1]
                elif "pin:" in line:
                    raw_parts = line.split(":")
                    param = raw_parts[0].strip()
                    raw_pin = raw_parts[-1].strip().split("#")[0].strip()
                    clean_pin = raw_pin.lstrip("!").lstrip("^").lstrip("~")
                    
                    if clean_pin:
                        entry = {"pin": clean_pin, "desc": "{} ({})".format(current_section, param)}
                        sec_lower = current_section.lower()
                        if "stepper" in sec_lower or "tmc" in sec_lower:
                            grouped_map["📊 STEPPER DRIVES"].append(entry)
                        elif "heater" in sec_lower or "fan" in sec_lower or "temperature" in sec_lower:
                            grouped_map["🔥 HEATING & COOLING"].append(entry)
                        elif "custom_tool" in sec_lower or "spindle" in sec_lower or "laser" in sec_lower:
                            grouped_map["🛠️ CUSTOM TOOLING CHANNELS"].append(entry)
                        else:
                            grouped_map["🔌 SYSTEM SIGNALS & ENDSTOPS"].append(entry)
                            
    return grouped_map

def get_registered_tools():
    """Reads written files inside toolheads/ to populate the visual library card panels"""
    tools = []
    if os.path.exists(TOOLHEADS_DIR):
        for file in os.listdir(TOOLHEADS_DIR):
            if file.endswith(".cfg"):
                name = file[:-4]
                path = os.path.join(TOOLHEADS_DIR, file)
                tool_data = {"name": name, "template": "Unknown", "pin": "None", "pwm": "0", "osc": "0", "ext": "0"}
                with open(path, "r") as f:
                    for line in f:
                        if "variable_base_template:" in line:
                            tool_data["template"] = line.split(":")[-1].strip().strip('"')
                        elif "variable_assigned_pin:" in line:
                            tool_data["pin"] = line.split(":")[-1].strip().strip('"')
                        elif "variable_capability_pwm:" in line:
                            tool_data["pwm"] = line.split(":")[-1].strip()
                        elif "variable_capability_oscillator:" in line:
                            tool_data["osc"] = line.split(":")[-1].strip()
                        elif "variable_capability_extruder:" in line:
                            tool_data["ext"] = line.split(":")[-1].strip()
                tools.append(tool_data)
    return tools

# Unified Mainsail Theme Master Stylesheet Overlay
MAINSAIL_THEME = """
<style>
    :root { --bg-dark: #11111b; --bg-panel: #1e1e2e; --bg-surface: #313244; --accent: #89b4fa; --text: #cdd6f4; --text-muted: #a6adc8; --success: #a6e3a1; --error: #f38ba8; --warning: #f9e2af; }
    body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; background-color: var(--bg-dark); color: var(--text); margin: 0; padding: 0; display: flex; height: 100vh; }
    .sidebar { width: 260px; background-color: var(--bg-panel); border-right: 1px solid #45475a; display: flex; flex-direction: column; padding: 20px 0; box-sizing: border-box; }
    .sidebar-brand { padding: 0 24px 20px; font-size: 1.25rem; font-weight: bold; color: var(--accent); border-bottom: 1px solid #313244; display: flex; align-items: center; gap: 10px; }
    .sidebar-menu { list-style: none; padding: 20px 0 0 0; margin: 0; flex-grow: 1; }
    .sidebar-item a { display: flex; align-items: center; gap: 12px; padding: 12px 24px; color: var(--text-muted); text-decoration: none; font-size: 0.95rem; transition: all 0.2s; }
    .sidebar-item.active a, .sidebar-item a:hover { color: var(--text); background-color: var(--bg-surface); border-left: 4px solid var(--accent); padding-left: 20px; }
    .main-content { flex-grow: 1; padding: 30px; overflow-y: auto; box-sizing: border-box; }
    .page-header { font-size: 1.5rem; font-weight: 600; margin-bottom: 24px; color: var(--text); display: flex; justify-content: space-between; align-items: center; }
    .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 24px; }
    .panel { background-color: var(--bg-panel); border: 1px solid #45475a; border-radius: 8px; padding: 24px; box-shadow: 0 4px 12px rgba(0,0,0,0.2); }
    .panel-title { font-size: 1.1rem; font-weight: bold; margin-top: 0; margin-bottom: 16px; color: var(--accent); border-bottom: 1px solid #313244; padding-bottom: 8px; }
    .form-group { margin-bottom: 16px; }
    .form-group label { display: block; font-size: 0.85rem; font-weight: 600; margin-bottom: 8px; color: var(--text-muted); text-transform: uppercase; letter-spacing: 0.5px; }
    input[type="text"], select { width: 100%; padding: 10px; background-color: var(--bg-surface); color: var(--text); border: 1px solid #45475a; border-radius: 4px; box-sizing: border-box; font-size: 0.95rem; }
    input[type="checkbox"] { width: auto; margin-right: 8px; transform: scale(1.1); }
    .checkbox-container { background-color: var(--bg-dark); padding: 12px; border-radius: 4px; border: 1px solid #45475a; margin-top: 6px; }
    .btn { display: inline-block; width: 100%; padding: 12px; background-color: var(--accent); color: var(--bg-dark); border: none; border-radius: 4px; font-weight: bold; font-size: 0.95rem; cursor: pointer; text-align: center; box-sizing: border-box; transition: background 0.2s; }
    .btn:hover { background-color: #a6e3a1; }
    .badge { background-color: var(--bg-surface); border: 1px solid #585b70; color: var(--text); padding: 2px 8px; border-radius: 4px; font-size: 0.8rem; font-family: monospace; font-weight: bold; display: inline-block; }
    .badge.active { background-color: var(--success); color: var(--bg-dark); border: none; }
    .badge.empty { background-color: var(--text-muted); color: var(--bg-dark); border: none; }
    .pin-row { display: flex; justify-content: space-between; align-items: center; padding: 8px 0; border-bottom: 1px solid #313244; font-size: 0.9rem; }
    .pin-row:last-child { border-bottom: none; }
    .header-tag { font-size: 0.8rem; color: var(--accent); font-weight: bold; margin-top: 16px; margin-bottom: 8px; text-transform: uppercase; }
    .tool-card { background-color: var(--bg-panel); border: 1px solid #45475a; border-radius: 8px; padding: 16px; display: flex; flex-direction: column; gap: 12px; }
    .tool-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(260px, 1fr)); gap: 20px; }
    .tool-thumb { width: 100%; height: 160px; background-color: var(--bg-dark); border-radius: 4px; border: 1px solid #313244; display: flex; align-items: center; justify-content: center; font-size: 3rem; overflow: hidden; }
</style>
"""

NAV_PANEL = """
<div class="sidebar">
    <div class="sidebar-brand">🦏 Rhino OS</div>
    <ul class="sidebar-menu">
        <li class="sidebar-item {{'active' if page=='wizard'}}"><a href="/">🛠️ Provisioning Wizard</a></li>
        <li class="sidebar-item {{'active' if page=='library'}}"><a href="/library">📚 Toolhead Library</a></li>
    </ul>
</div>
"""

@app.route("/")
def index():
    grouped_map = get_detailed_pin_map()
    all_claimed = []
    for cat in grouped_map:
        for entry in grouped_map[cat]:
            all_claimed.append(entry["pin"])
            
    available_pins = [p for p in UMBILICAL_WHITELIST if p not in all_claimed]

    html = """
    <html>
    <head><title>Rhino Dashboard</title>""" + MAINSAIL_THEME + """</head>
    <body>
        """ + NAV_PANEL.replace("{{'active' if page=='wizard'}}", "active") + """
        <div class="main-content">
            <div class="page-header"><div>🛠️ Provisioning Wizard</div></div>
            <div class="grid">
                <div class="panel">
                    <div class="panel-title">Add New Capability Profile</div>
                    <form action="/create" method="POST">
                        <div class="form-group">
                            <label>Toolhead Profile Name</label>
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
        f.write("# Automated hardware capability profile for {}\n\n".format(name.upper()))
        f.write("[gcode_macro CUSTOM_TOOL_{}]\n".format(name.upper()))
        f.write('variable_tool_name: "{}"\n'.format(name))
        f.write('variable_base_template: "{}"\n'.format(template))
        f.write('variable_assigned_pin: "{}"\n'.format(pin))
        f.write("variable_capability_pwm: {}\n".format(cap_pwm))
        f.write("variable_capability_oscillator: {}\n".format(cap_oscillator))
        f.write("variable_capability_extruder: {}\n\n".format(cap_extruder))
        f.write("gcode:\n")
        f.write('  GENERIC_CAPABILITY_SETUP NAME="{}" FEED_RATE=600 CAP_PWM={} CAP_OSCILLATOR={}\n'.format(name, cap_pwm, cap_oscillator))
        
    os.system("curl -X POST http://localhost:7125/printer/firmware_restart")
    return redirect(url_for('library'))

@app.route("/upload/<toolname>", methods=["POST"])
def upload_file(toolname):
    if 'file' in request.files:
        file = request.files['file']
        if file.filename != '':
            filename = "{}.png".format(toolname)
            file.save(os.path.join(app.config['UPLOAD_FOLDER'], filename))
    return redirect(url_for('library'))

@app.route("/images/<filename>")
def get_image(filename):
    from flask import send_from_directory
    return send_from_directory(app.config['UPLOAD_FOLDER'], filename)

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
