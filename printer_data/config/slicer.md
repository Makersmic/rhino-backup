# templates.py - Mainsail UI Design HTML Template Storage
HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Rhino OS // Capability Center</title>
    <link rel="stylesheet" href="https://cloudflare.com">
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
        .sidebar { position: fixed; top: 50px; bottom: 0; left: -280px; width: var(--sidebar-width); background: var(--card-mainsail); border-right: 1px solid var(--border-mainsail); transition: transform 0.3s cubic-bezier(0.4, 0, 0.2, 1); z-index: 99; padding: 20px 10px; box-sizing: border-box; }
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
        .form-group label { display: block; margin-bottom: 6px; font-size: 0.85rem; color: var(--text-muted); text-transform: uppercase; letter-spacing: 0.5px; }
        input[type="text"], select, textarea { width: 100%; padding: 10px; background: var(--bg-mainsail); border: 1px solid var(--border-mainsail); border-radius: 4px; color: #fff; font-size: 0.9rem; box-sizing: border-box; }
        input[type="text"]:focus, select:focus, textarea:focus { border-color: var(--accent-mainsail); outline: none; }
        .checkbox-group { background: var(--bg-mainsail); padding: 12px; border-radius: 4px; border: 1px solid var(--border-mainsail); }
        .checkbox-row { display: flex; align-items: center; margin-bottom: 8px; font-size: 0.9rem; }
        .checkbox-row input { margin-right: 10px; }
        .btn-mainsail { background: var(--accent-mainsail); color: #fff; border: none; padding: 10px 18px; border-radius: 4px; font-weight: 500; cursor: pointer; transition: 0.2s; display: inline-flex; align-items: center; gap: 8px; font-size: 0.9rem; }
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
                <p>No customized modular tools found in the active toolheads directory cluster tree.</p>
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
                                <img src="/images/{{ tool.id }}/{{ img }}" class="media-thumb" alt="Hardware Image View">
                            {% endfor %}
                        </div>
                    </div>
                    {% endif %}
                    <details style="margin-bottom: 15px;">
                        <summary style="font-size: 0.8rem; color: var(--accent-mainsail); cursor: pointer; user-select: none;">Inspect Generated Klipper G-Code</summary>
