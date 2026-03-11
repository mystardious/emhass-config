# EMHASS Configuration

Energy Management for Home Assistant (EMHASS) configuration for a residential solar + battery system.

## Hardware

| Component | Model | Specs |
|-----------|-------|-------|
| Hybrid Inverter | Foxess KH10 | 10 kW AC output/input |
| Battery | Foxess EQ4800 stack | ~41.9 kWh nominal capacity |
| Solar PV | 16 modules x 3 strings | ~14 kWp, 30° tilt, 205° azimuth |

## Software Stack

- **EMHASS** v0.16.2 running in Docker (`ghcr.io/davidusb-geek/emhass:latest`)
- **Home Assistant** providing sensor data via WebSocket
- **Solcast** for PV power forecasting
- **Amber Electric** for real-time import/export price forecasts

## Key Configuration

The active config is `share/config.json` (mounted into the container at `/share/config.json`).

### Optimization

| Parameter | Value | Notes |
|-----------|-------|-------|
| `costfun` | `profit` | Maximizes profit (minimizes cost, maximizes export revenue) |
| `optimization_time_step` | 5 min | MPC runs every 5 minutes |
| `prediction_horizon` | 288 | 288 x 5 min = 24 hours |
| `lp_solver_timeout` | 45 s | |

### Battery

| Parameter | Value | Notes |
|-----------|-------|-------|
| `set_nocharge_from_grid` | `false` | Allows overnight grid charging at cheap rates |
| `set_nodischarge_to_grid` | `false` | Battery can export to grid |
| `battery_minimum_state_of_charge` | 0.1 | 10% floor |
| `battery_target_state_of_charge` | 0.6 | 60% target at end of optimization |
| `weight_battery_charge` | 0.05 | Small penalty to prefer PV self-consumption over aggressive battery charging |
| `battery_charge_efficiency` | 0.95 | |
| `battery_discharge_efficiency` | 0.95 | |

### Deferrable Loads

| Parameter | Value | Notes |
|-----------|-------|-------|
| Load 0 | Pool pump | 1000 W, 8 hours/day |
| `start_timesteps` | 0 | Full window |
| `end_timesteps` | 192 | First 16 hours of horizon |
| `treat_as_semi_cont` | true | On/off only |
| `single_constant` | true | Single continuous run |

### Grid Constraints

A `maximum_power_from_grid` list of 288 values is passed at runtime, setting grid import to 0 during the demand window (Mon-Fri 16:00-20:00 AEDT) to avoid peak demand charges.

### Sensors

| Config Key | HA Entity |
|------------|-----------|
| `sensor_power_photovoltaics` | `sensor.pv_power_w` |
| `sensor_power_load_no_var_loads` | `sensor.house_load_power_w` |
| `sensor_power_photovoltaics_forecast` | `sensor.p_pv_forecast` |

## File Structure

```
/opt/program-data/emhass/
  share/config.json      <- Active config (mounted at /share/ in container)
  secrets_emhass.yaml    <- HA URL + token (not tracked in git)
  data/                  <- Runtime data: opt results, pickles (not tracked)
  config.json            <- Legacy/orphaned copy (not tracked)
```

## Docker

Defined in `/opt/docker-compose/monitoring/docker-compose.yaml`:

```yaml
emhass:
  image: ghcr.io/davidusb-geek/emhass:latest
  container_name: emhass
  network_mode: host
  volumes:
    - /opt/program-data/emhass/share:/share/
    - /opt/program-data/emhass/secrets_emhass.yaml:/app/secrets_emhass.yaml
    - /opt/program-data/emhass/data:/data
```
