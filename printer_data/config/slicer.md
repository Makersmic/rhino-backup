# klipper_parser.py - Fixed Core File Engine
import os
import re

CONFIG_DIR = os.path.expanduser("~/printer_data/config")
CONFIG_PATH = os.path.join(CONFIG_DIR, "printer.cfg")
TOOLHEADS_DIR = os.path.join(CONFIG_DIR, "toolheads")
UPLOAD_FOLDER = os.path.join(CONFIG_DIR, "toolhead_images")

def get_detailed_pin_map():
    """Scans printer configs and tracks exact hardware pin deployments"""
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
                    parts = line.split(":")
                    if len(parts) >= 2:
                        param = parts[0].strip()
                        raw_pin = parts[1].strip().split("#")[0].strip()
                        clean_pin = raw_pin.lstrip("!").lstrip("^").lstrip("~")
                        if clean_pin:
                            pin_map[clean_pin] = f"{current_section} ({param})"
    return pin_map

def parse_tool_configs():
    """Parses existing macro assets back into toolhead library cards"""
    tools = {}
    if not os.path.exists(TOOLHEADS_DIR):
        return tools
        
    for filename in os.listdir(TOOLHEADS_DIR):
        if filename.endswith(".cfg"):
            tool_id = filename.replace(".cfg", "")
            file_path = os.path.join(TOOLHEADS_DIR, filename)
            
            tool_data = {
                "id": tool_id, "name": tool_id.capitalize(), "template": "subtractive",
                "pin": "Unassigned", "cap_pwm": "0", "cap_oscillator": "0", 
                "cap_extruder": "0", "notes": "", "images": [], "raw_macro": ""
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
    """Generates clean Klipper tool configuration macros"""
    tool_id = name.lower().replace(" ", "_")
    if not os.path.exists(TOOLHEADS_DIR):
        os.makedirs(TOOLHEADS_DIR, exist_ok=True)
    
    file_path = os.path.join(TOOLHEADS_DIR, f"{tool_id}.cfg")
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
