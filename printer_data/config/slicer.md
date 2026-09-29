; --- SwitchFly Multi-Material Path Router ---
{if next_extruder == 0}
  SWITCH_T0 MATERIAL="[filament_type]"
{elsif next_extruder == 1}
  SWITCH_T1 MATERIAL="[filament_type]"
{endif}
