# routes.py - Audited Route Operations Engine
from flask import render_template_string, request, redirect, url_for, send_from_directory
from werkzeug.utils import secure_filename
import os
import shutil
from klipper_parser import get_detailed_pin_map, parse_tool_configs, write_tool_config
from templates import HTML_TEMPLATE

ALLOWED_EXTENSIONS = {'png', 'jpg', 'jpeg', 'webp', 'gif'}

def allowed_file(filename):
    return '.' in filename and filename.rsplit('.', 1)[-1].lower() in ALLOWED_EXTENSIONS

def register_routes(app):
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
                    filename = secure_filename(file.filename)
                    file.save(os.path.join(tool_img_dir, filename))
        return redirect(url_for('index'))

    @app.route("/delete/<tool_id>")
    def delete(tool_id):
        from klipper_parser import TOOLHEADS_DIR
        cfg_path = os.path.join(TOOLHEADS_DIR, f"{tool_id}.cfg")
        if os.path.exists(cfg_path):
            os.remove(cfg_path)
        tool_img_dir = os.path.join(app.config['UPLOAD_FOLDER'], tool_id)
        if os.path.exists(tool_img_dir):
            shutil.rmtree(tool_img_dir)
        return redirect(url_for('index'))

    @app.route('/images/<tool_id>/<filename>')
    def serve_tool_image(tool_id, filename):
        return send_from_directory(os.path.join(app.config['UPLOAD_FOLDER'], tool_id), filename)
