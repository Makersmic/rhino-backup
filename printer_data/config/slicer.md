# rhino_portal.py - Advanced Mainsail-Inspired Capability Portal for Rhino System
import os
import re
import shutil
from flask import Flask, render_template_string, request, redirect, url_for, send_from_directory

app = Flask(__name__)
app.secret_key = "rhino_secret_system_key"

# Paths to Klipper configuration tree
CONFIG_DIR = os.path.expanduser("~/printer_data/config")
CONFIG_PATH = os.path.join(CONFIG_DIR, "printer.cfg")
TOOLHEADS_DIR = os.path.join(CONFIG_DIR, "toolheads")
UPLOAD_FOLDER = os.path.join(CONFIG_DIR, "toolhead_images")
ALLOWED_EXTENSIONS = {'png', 'jpg', 'jpeg', 'webp', 'gif'}

app.config['UPLOAD_FOLDER'] = UPLOAD_FOLDER
# Maximum upload size: 16 Megabytes
app.config['MAX_CONTENT_LENGTH'] = 16 * 1024 * 1024 

# Ensure directory structures exist
for folder in [TOOLHEADS_DIR, UPLOAD_FOLDER]:
    if not os.path.exists(folder):
        os.makedirs(folder, exist_ok=True)

def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS

def get_detailed_pin_map():
    """Scans printer.cfg and included files to map out exactly what is using each pin"""
    pin_map = {}
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
                    param = line.split(":")[0].strip()
                    raw_pin = line.split(":")[-1].strip().split("#")[0].strip()
                    clean_pin = raw_pin.lstrip("!").lstrip("^").lstrip("~")
                    if clean_pin:
                        pin_map[clean_pin] = f"{current_section} ({param})"
    return pin_map

def parse_tool_configs():
    """Reads all generated .cfg files in toolheads/ and parses metadata + macros"""
    tools = {}
    if not os.path.exists(TOOLHEADS_DIR):
        return tools
        
    for filename in os.listdir(TOOLHEADS_DIR):
        if filename.endswith(".cfg"):
            tool_id = filename.replace(".cfg", "")
            file_path = os.path.join(TOOLHEADS_DIR, filename)
            
            tool_data = {
                "id": tool_id,
                "name": tool_id.capitalize(),
                "template": "subtractive",
                "pin": "Unassigned",
                "cap_pwm": "0",
                "cap_oscillator": "0",
                "cap_extruder": "0",
                "notes": "",
                "images": [],
                "raw_macro": ""
            }
            
            with open(file_path, "r") as f:
                content = f.read()
                tool_data["raw_macro"] = content
                
                name_match = re.search(r'variable_tool_name:\s*"([^"]+)"', content)
                tmpl_match = re.search(r'variable_base_template:\s*"([^"]+)"', content)
                pin_match = re.search(r'variable_assigned_pin:\s*"([^"]+)"', content)
                pwm_match = re.search(r'variable_capability_pwm:\s*([0-1])', content)
                osc_match = re.search(r'variable_capability_oscillator:\s*([0-1])', content)
                ext_match = re.search(r'variable_capability_extruder:\s*([0-1])', content)
                
                if name_match: tool_data["name"] = name_match.group(1)
                if tmpl_match: tool_data["template"] = tmpl_match.group(1)
                if pin_match: tool_data["pin"] = pin_match.group(1)
                if pwm_match: tool_data["cap_pwm"] = pwm_match.group(1)
                if osc_match: tool_data["cap_oscillator"] = osc_match.group(1)
                if ext_match: tool_data["cap_extruder"] = ext_match.group(1)
                
                notes_block = re.findall(r'# NOTE:\s*(.*)', content)
                if notes_block:
                    tool_data["notes"] = "\n".join(notes_block)
            
            tool_img_dir = os.path.join(UPLOAD_FOLDER, tool_id)
            if os.path.exists(tool_img_dir):
                tool_data["images"] = os.listdir(tool_img_dir)
                
            tools[tool_id] = tool_data
    return tools

def write_tool_config(name, template, pin, pwm, osc, extruder, notes):
    """Generates the Klipper macro configuration file with custom metadata headers"""
    tool_id = name.lower().replace(" ", "_")
    fn = f"{tool_id}.cfg"
    file_path = os.path.join(TOOLHEADS_DIR, fn)
    
    with open(file_path, "w") as f:
        f.write("# ====================================================================\n")
        f.write(f"# 🛠️ AUTOMATED HARDWARE CAPABILITY PROFILE FOR {name.upper()}\n")
        if notes:
            for line in notes.splitlines():
                f.write(f"# NOTE: {line}\n")
        f.write("# ====================================================================\n\n")
        
        f.write(f"[gcode_macro CUSTOM_TOOL_{tool_id.upper()}]\n")
        f.write(f'variable_tool_name: "{name}"\n')
        f.write(f'variable_base_template: "{template}"\n')
        f.write(f'variable_assigned_pin: "{pin}"\n')
        f.write(f"variable_capability_pwm: {pwm}\n")
        f.write(f"variable_capability_oscillator: {osc}\n")
        f.write(f"variable_capability_extruder: {extruder}\n\n")
        
        f.write("gcode:\n")
        f.write(f'  GENERIC_CAPABILITY_SETUP NAME="{name}" FEED_RATE=600 CAP_PWM={pwm} CAP_OSCILLATOR={osc}\n')
    
    # Trigger firmware restart via Moonraker
    os.system("curl -X POST http://localhost:7125/printer/firmware_restart")

@app.route('/images/<tool_id>/<filename>')
def serve_tool_image(tool_id, filename):
    return send_from_directory(os.path.join(app.config['UPLOAD_FOLDER'], tool_id), filename)

@app.route("/")
def index():
    pin_map = get_detailed_pin_map()
    tools = parse_tool_configs()
    
    umbilical_whitelist = ["PE0", "PE1", "PE2", "PE3", "PF4", "PB0", "PB1", "PC0", "PC1"]
    assigned_pins = [t['pin'] for t in tools.values()]
    available_pins = [p for p in umbilical_whitelist if p not in pin_map or p in assigned_pins]

    return render_template_string(HTML_TEMPLATE, pin_map=pin_map, available_pins=available_pins, tools=tools)

@app.route("/create", methods=["POST"])
def create():
    name = request.form.get("name")
    template = request.form.get("template")
    pin = request.form.get("pin")
    pwm = request.form.get("cap_pwm", "0")
    osc = request.form.get("cap_oscillator", "0")
    extruder = request.form.get("cap_extruder", "0")
    notes = request.form.get("notes", "")
    
    tool_id = name.lower().replace(" ", "_")
    write_tool_config(name, template, pin, pwm, osc, extruder, notes)
    
    if 'photos' in request.files:
        files = request.files.getlist('photos')
        tool_img_dir = os.path.join(app.config['UPLOAD_FOLDER'], tool_id)
        os.makedirs(tool_img_dir, exist_ok=True)
        for file in files:
            if file and allowed_file(file.filename):
                from werkzeug.utils import secure_filename
                filename = secure_filename(file.filename)
                file.save(os.path.join(tool_img_dir, filename))
                
    return redirect(url_for('index'))

@app.route("/edit/<tool_id>", methods=["POST"])
def edit(tool_id):
    name = request.form.get("name")
    template = request.form.get("template")
    pin = request.form.get("pin")
    pwm = request.form.get("cap_pwm", "0")
    osc = request.form.get("cap_oscillator", "0")
    extruder = request.form.get("cap_extruder", "0")
    notes = request.form.get("notes", "")
    
    write_tool_config(name, template, pin, pwm, osc, extruder, notes)
    
    if 'photos' in request.files:
        files = request.files.getlist('photos')
        tool_img_dir = os.path.join(app.config['UPLOAD_FOLDER'], tool_id)
        os.makedirs(tool_img_dir, exist_ok=True)
        for file in files:
            if file and allowed_file(file.filename):
                from werkzeug.utils import secure_filename
                filename = secure_filename(file.filename)
                file.save(os.path.join(tool_img_dir, filename))
                
    return redirect(url_for('index'))

@app.route("/delete/<tool_id>")
def delete(tool_id):
    cfg_path = os.path.join(TOOLHEADS_DIR, f"{tool_id}.cfg")
    if os.path.exists(cfg_path):
        os.remove(cfg_path)
        
    tool_img_dir = os.path.join(app.config['UPLOAD_FOLDER'], tool_id)
    if os.path.exists(tool_img_dir):
        shutil.rmtree(tool_img_dir)
        
    return redirect(url_for('index'))

HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Rhino OS // Capability Center</title>
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
    <style>
        :root {
            --bg-mainsail: #111216;
            --card-mainsail: #1a1b23;
            --border-mainsail: #2a2b36;
            --accent-mainsail: #009688;
            --accent-hover: #00bfa5;
            --text-main: #e3e4e6;
            --text-muted: #9a9ca6;
            --sidebar-width: 280px;
        }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; margin: 0; padding: 0; background: var(--bg-mainsail); color: var(--text-main); display: flex; min-height: 100vh; }
        
        .top-bar { position: fixed; top: 0; left: 0; right: 0; height: 50px; background: var(--card-mainsail); display: flex; align-items: center; padding: 0 20px; border-bottom: 1px solid var(--border-mainsail); z-index: 100; }
        .menu-trigger { background: none; border: none; color: var(--text-main); font-size: 1.25rem; cursor: pointer; margin-right: 20px; padding: 5px; border-radius: 4px; }
        .menu-trigger:hover { background: var(--border-mainsail); }
        .top-bar .title { font-weight: 600; font-size: 1.1rem; color: #fff; letter-spacing: 0.5px; }

        .sidebar { position: fixed; top: 50px; bottom: 0; left: -280px; width: var(--sidebar-width); background: var(--card-mainsail); border-right: 1px solid var(--border-mainsail); transition: transform 0.3s ease; z-index: 99; padding: 20px 10px; box-sizing: border-box; }
        .sidebar.active { transform: translateX(280px); }
        .sidebar-btn { display: flex; align-items: center; width: 100%; padding: 12px 15px; background: none; border: none; color: var(--text-main); text-align: left; font-size: 0.95rem; border-radius: 6px; cursor: pointer; margin-bottom: 8px; transition: 0.2s; }
        .sidebar-btn i { margin-right: 12px; width: 20px; text-align: center; color: var(--text-muted); }
        .sidebar-btn:hover { background: var(--border-mainsail); color: #fff; }
        .sidebar-btn:hover i { color: var(--accent-mainsail); }

        .main-content { margin-top: 50px; padding: 30px; width: 100%; transition: margin-left 0.3s ease; box-sizing: border-box; }
        .main-content.shifted { margin-left: var(--sidebar-width); width: calc(100% - var(--sidebar-width)); }

        .card { background: var(--card-mainsail); border: 1px solid var(--border-mainsail); border-radius: 6px; padding: 20px; margin-bottom: 25px; box-shadow: 0 4px 12px rgba(0,0,0,0.15); }
        .card h2 { margin-top: 0; font-size: 1.2rem; font-weight: 500; border-bottom: 1px solid var(--border-mainsail); padding-bottom: 12px; display: flex; align-items: center; justify-content: space-between; }
        
        .grid-3 { display: grid; grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); gap: 20px; }
        
        .form-group { margin-bottom: 15px; }
        .form-group label { display: block; margin-bottom: 6px; font-size: 0.85rem; color: var(--text-muted); text-transform: uppercase; }
        input[type="text"], select, textarea { width: 100%; padding: 10px; background: var(--bg-mainsail); border: 1px solid var(--border-mainsail); border-radius: 4px; color: #fff; font-size: 0.9rem; box-sizing: border-box; }
        
        .checkbox-group { background: var(--bg-mainsail); padding: 12px; border-radius: 4px; border: 1px solid var(--border-mainsail); }
        .checkbox-row { display: flex; align-items: center; margin-bottom: 8px; font-size: 0.9rem; }
        .checkbox-row input { margin-right: 10px; }
        
        .btn-mainsail { background: var(--accent-mainsail); color: #fff; border: none; padding: 10px 18px; border-radius: 4px; font-weight: 500; cursor: pointer; transition: 0.2s; display: inline-flex; align-items: center; gap: 8px; }
        .btn-mainsail:hover { background: var(--accent-hover); }
        .btn-danger { background: #d32f2f; }
        .btn-danger:hover { background: #f44336; }
        
        .badge { background: #3a3b46; color: #fff; padding: 2px 8px; border-radius: 4px; font-family: monospace; font-size: 0.8rem; }
        .badge-success { background: rgba(0, 150, 136, 0.2); color: #4db6ac; border: 1px solid rgba(0, 150, 136, 0.4); }
        
        .media-gallery { display: grid; grid-template-columns: repeat(auto-fill, minmax(70px, 1fr)); gap: 8px; margin-top: 10px; }
        .media-thumb { width: 100%; height: 70px; border-radius: 4px; object-fit: cover; border: 1px solid var(--border-mainsail); }
        
        pre.macro-code { background: #0c0d10; padding: 12px; border-radius: 4px; border: 1px solid var(--border-mainsail); font-family: monospace; font-size: 0.8rem; color: #a9b2c3; overflow-x: auto; max-height: 180px; }
        
        .modal { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.6); z-index: 1000; justify-content: center; align-items: center; }
        .modal.active { display: flex; }
        .modal-content { background: var(--card-mainsail); border: 1px solid var(--border-mainsail); border-radius: 6px; width: 500px; max-width: 90%; padding: 25px; max-height: 85vh; overflow-y: auto; }
    </style>
</head>
<body>
    <header class="top-bar">
        <button class="menu-trigger" onclick="toggleSidebar()"><i class="fas fa-bars"></i></button>
        <div class="title"><i class="fas fa-microchip" style="color:var(--accent-mainsail);"></i> RHINO CORE SYSTEM // TOOL ENGINE</div>
    </header>

    <nav class="sidebar" id="mainSidebar">
        <h3 style="padding: 0 15px; font-size: 0.75rem; color: var(--text-muted); text-transform: uppercase; letter-spacing: 1px;">Tool Actions</h3>
        <button class="sidebar-btn" onclick="openModal('addModal')"><i class="fas fa-plus-circle"></i> Add New Hardware Tool</button>
        
        <h3 style="padding: 15px 15px 5px 15px; font-size: 0.75rem; color: var(--text-muted); text-transform: uppercase; letter-spacing: 1px;">System Health</h3>
        <button class="sidebar-btn" onclick="openSection('library-view')"><i class="fas fa-folder-open"></i> Tool Library Asset Map</button>
        <button class="sidebar-btn" onclick="openSection('diagnostics-view')"><i class="fas fa-bolt"></i> Umbilical Pin Diagnostics</button>
    </nav>

    <main class="main-content" id="workspace">
        <section id="library-view" class="view-section">
            <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px;">
                <h1 style="margin: 0; font-size: 1.5rem; font-weight: 400;">Active Hardware Tool Library</h1>
                <button class="btn-mainsail" onclick="openModal('addModal')"><i class="fas fa-plus"></i> Add Tool</button>
            </div>

            {% if not tools %}
            <div class="card" style="text-align: center; padding: 40px 20px; color: var(--text-muted);">
                <i class="fas fa-tools" style="font-size: 3rem; margin-bottom: 15px; color: var(--border-mainsail);"></i>
                <p>No customized modular tools found in the active `toolheads/` cluster tree.</p>
            </div>
            {% endif %}

            <div class="grid-3">
                {% for id, tool in tools.items() %}
                <div class="card">
                    <h2>
                        <span><i class="fas fa-cube" style="color: var(--accent-mainsail); margin-right: 8px;"></i> {{ tool.name }}</span>
                        <span class="badge badge-success">{{ tool.pin }}</span>
                    </h2>
                    
                    <div style="margin-bottom: 12px; font-size: 0.85rem; color: var(--text-muted);">
                        <strong>Template Archetype:</strong> <span style="color:#fff;">{{ tool.template }}</span>
                    </div>

                    <div style="display: flex; gap: 5px; flex-wrap: wrap; margin-bottom: 15px;">
                        {% if tool.cap_pwm == '1' %}<span class="badge" style="background:#0288d1;">PWM Power</span>{% endif %}
                        {% if tool.cap_oscillator == '1' %}<span class="badge" style="background:#7b1fa2;">Oscillator</span>{% endif %}
                        {% if tool.cap_extruder == '1' %}<span class="badge" style="background:#c2185b;">Extruder Drive</span>{% endif %}
                    </div>

                    {% if tool.notes %}
                    <div style="margin-bottom: 15px;">
                        <label style="font-size: 0.75rem; color: var(--text-muted); text-transform: uppercase;">User Deployment Notes:</label>
                        <div style="background: var(--bg-mainsail); padding: 8px; border-radius: 4px; font-size: 0.85rem; border-left: 3px solid var(--accent-mainsail); white-space: pre-wrap;">{{ tool.notes }}</div>
                    </div>
                    {% endif %}

                    {% if tool.images %}
                    <div style="margin-bottom: 15px;">
                        <label style="font-size: 0.75rem; color: var(--text-muted); text-transform: uppercase;">Tool Media Library ({{ tool.images|length }}):</label>
                        <div class="media-gallery">
                            {% for img in tool.images %}
                                <img src="/images/{{ tool.id }}/{{ img }}" class="media-thumb" alt="Hardware Image">
                            {% endfor %}
                        </div>
                    </div>
                    {% endif %}

                    <details style="margin-bottom: 15px;">
                        <summary style="font-size: 0.8rem; color: var(--accent-mainsail); cursor: pointer; user-select: none;">Inspect Generated Klipper G-Code</summary>
                        <pre class="macro-code">{{ tool.raw_macro }}</pre>
                    </details>

                    <div style="display: flex; gap: 10px; margin-top: 15px; border-top: 1px solid var(--border-mainsail); padding-top: 15px;">
                        <button class="btn-mainsail" style="flex: 1; background: #3a3b46;" onclick="openEditModal('{{ tool.id }}', '{{ tool.name }}', '{{ tool.template }}', '{{ tool.pin }}', '{{ tool.cap_pwm }}', '{{ tool.cap_oscillator }}', '{{ tool.cap_extruder }}', `{{ tool.notes }}`)">
                            <i class="fas fa-edit"></i> Edit Tool
                        </button>
                        <a href="/delete/{{ tool.id }}" class="btn-mainsail btn-danger" style="text-decoration: none;" onclick="return confirm('Are you certain you want to purge this tool setup config?')">
                            <i class="fas fa-trash"></i> Drop
                        </a>
                    </div>
                </div>
                {% endfor %}
            </div>
        </section>

        <section id="diagnostics-view" class="view-section" style="display: none;">
            <h1 style="margin: 0 0 20px 0; font-size: 1.5rem; font-weight: 400;">Umbilical Harness Registry & Collision Map</h1>
            <div class="card">
                <h2><i class="fas fa-shield-halved" style="color:#ffb300; margin-right: 8px;"></i> Pin Resource Lock Registry Matrix</h2>
                <div style="background: var(--bg-mainsail); border-radius: 4px; padding: 10px;">
                    {% for pin, component in pin_map.items()|sort %}
                    <div style="display: flex; align-items: center; justify-content: space-between; padding: 10px; border-bottom: 1px solid var(--border-mainsail);">
                        <span><span class="badge" style="background:#d32f2f; margin-right: 15px;">LOCKED</span> <strong>{{ pin }}</strong></span>
                        <span style="color: var(--text-muted); font-size: 0.9rem;"><i class="fas fa-link"></i> Claimed By: {{ component }}</span>
                    </div>
                    {% endfor %}
                </div>
            </div>
        </section>
    </main>

    <div class="modal" id="addModal">
        <div class="modal-content">
            <h2 style="margin-top:0;">🔧 Register New System Tool</h2>
            <form action="/create" method="POST" enctype="multipart/form-data">
                <div class="form-group">
                    <label>Tool Head Identifier Name</label>
                    <input type="text" name="name" placeholder="e.g. LaserCutter" required>
                </div>
                <div class="form-group">
                    <label>Base Template Engine</label>
                    <select name="template">
                        <option value="subtractive">⚙️ Subtractive Model (CNC Kinematics Base)</option>
                        <option value="additive">🧵 Additive Model (Extruder Core Print Base)</option>
                    </select>
                </div>
                <div class="form-group">
                    <label>Hardware Harness Capabilities</label>
                    <div class="checkbox-group">
                        <div class="checkbox-row"><input type="checkbox" name="cap_pwm" value="1"> PWM Modulated Signal Drive Channel</div>
                        <div class="checkbox-row"><input type="checkbox" name="cap_oscillator" value="1"> Reciprocating Oscillator Loop Channel</div>
                        <div class="checkbox-row"><input type="checkbox" name="cap_extruder" value="1"> Synchronized Extruder Stepper Node</div>
                    </div>
                </div>
                <div class="form-group">
                    <label>Designated Umbilical Channel Connection (Collision Protected)</label>
                    <select name="pin">
                        {% for p in available_pins %}
                            <option value="{{p}}">{{p}}</option>
                        {% endfor %}
                    </select>
                </div>
                <div class="form-group">
                    <label>Operational Metadata Notes Library</label>
                    <textarea name="notes" rows="3" placeholder="Enter calibration offsets..."></textarea>
                </div>
                <div class="form-group">
                    <label>Upload Tool Machine Photos</label>
                    <input type="file" name="photos" multiple accept="image/*">
                </div>
                <div style="display:flex; gap:10px; justify-content: flex-end; margin-top:20px;">
                    <button type="button" class="btn-mainsail" style="background:#3a3b46;" onclick="closeModal('addModal')">Cancel</button>
                    <button type="submit" class="btn-mainsail">Compile Config Asset</button>
                </div>
            </form>
        </div>
    </div>

    <div class="modal" id="editModal">
        <div class="modal-content">
            <h2 style="margin-top:0;">📝 Edit System Tool Configuration</h2>
            <form id="editForm" action="" method="POST" enctype="multipart/form-data">
                <div class="form-group">
                    <label>Tool Head Identifier Name</label>
                    <input type="text" name="name" id="edit_name" required>
                </div>
                <div class="form-group">
                    <label>Base Template Engine</label>
                    <select name="template" id="edit_template">
                        <option value="subtractive">⚙️ Subtractive Model (CNC Kinematics Base)</option>
                        <option value="additive">🧵 Additive Model (Extruder Core Print Base)</option>
                    </select>
                </div>
                <div class="form-group">
                    <label>Hardware Harness Capabilities</label>
                    <div class="checkbox-group">
                        <div class="checkbox-row"><input type="checkbox" name="cap_pwm" id="edit_cap_pwm" value="1"> PWM Modulated Signal Drive Channel</div>
                        <div class="checkbox-row"><input type="checkbox" name="cap_oscillator" id="edit_cap_oscillator" value="1"> Reciprocating Oscillator Loop Channel</div>
                        <div class="checkbox-row"><input type="checkbox" name="cap_extruder" id="edit_cap_extruder" value="1"> Synchronized Extruder Stepper Node</div>
                    </div>
                </div>
                <div class="form-group">
                    <label>Designated Umbilical Channel Connection</label>
                    <select name="pin" id="edit_pin">
                        {% for p in available_pins %}
                            <option value="{{p}}">{{p}}</option>
                        {% endfor %}
                    </select>
                </div>
                <div class="form-group">
                    <label>Operational Metadata Notes Library</label>
                    <textarea name="notes" id="edit_notes" rows="3"></textarea>
                </div>
                <div class="form-group">
                    <label>Append Additional Tool Machine Photos</label>
                    <input type="file" name="photos" multiple accept="image/*">
                </div>
                <div style="display:flex; gap:10px; justify-content: flex-end; margin-top:20px;">
                    <button type="button" class="btn-mainsail" style="background:#3a3b46;" onclick="closeModal('editModal')">Cancel</button>
                    <button type="submit" class="btn-mainsail">Save Updates</button>
                </div>
            </form>
        </div>
    </div>

    <script>
        function toggleSidebar() {
            const sidebar = document.getElementById('mainSidebar');
            const workspace = document.getElementById('workspace');
            sidebar.classList.toggle('active');
            workspace.classList.toggle('shifted');
        }
        function openModal(id) { document.getElementById(id).classList.add('active'); }
        function closeModal(id) { document.getElementById(id).classList.remove('active'); }
        function openSection(sectionId) {
            document.querySelectorAll('.view-section').forEach(s => s.style.display = 'none');
            document.getElementById(sectionId).style.display = 'block';
            if(window.innerWidth < 992) toggleSidebar();
        }
        function openEditModal(id, name, template, pin, pwm, osc, extruder, notes) {
            document.getElementById('editForm').action = '/edit/' + id;
            document.getElementById('edit_name').value = name;
            document.getElementById('edit_template').value = template;
            document.getElementById('edit_pin').value = pin;
            document.getElementById('edit_notes').value = notes;
            document.getElementById('edit_cap_pwm').checked = (pwm === '1');
            document.getElementById('edit_cap_oscillator').checked = (osc === '1');
            document.getElementById('edit_cap_extruder').checked = (extruder === '1');
            openModal('editModal');
        }
        window.addEventListener('DOMContentLoaded', () => {
            if(window.innerWidth >= 1024) toggleSidebar();
        });
    </script>
</body>
</html>
"""

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=True)
