[Unit]
Description=Rhino Custom Toolhead Provisioning Portal
After=network.target moonraker.service

[Service]
Type=simple
User=pi
WorkingDirectory=/home/pi
ExecStart=/usr/bin/python3 /home/pi/rhino_portal.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
