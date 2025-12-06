# Godot HELICS Integration Plan

## Overview

Connect the Godot visualization (lunco-sim) to the V-ICS.le HELICS federation to receive real-time physics data from `modelica-helics-federate`.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      HELICS Broker                              │
└────────┬──────────────────────┬──────────────────────────────────┘
         │                      │
         │                      │
    ┌────▼─────────────┐   ┌────▼──────────────────────┐
    │ Godot Federate   │   │ modelica-helics-federate  │
    │ (Visualization)  │   │ (Physics + Modbus)        │
    │                  │   │                           │
    │  Subscribes:     │   │  Publishes:               │
    │  - temperature   │   │  - thermal/temperature    │
    │  - heater_state  │   │  - thermal/heater_state   │
    │                  │   │                           │
    │  Publishes:      │   │  Subscribes:              │
    │  - heater_cmd    │   │  - controls/heater        │
    └──────────────────┘   └───────────────────────────┘
```

## Integration Options

###Option 1: GDExtension + Rust (Recommended)

**Pros:**
- Direct HELICS C API access
- High performance
- Type-safe integration
- Reuse existing Rust code

**Cons:**
- Requires GDExtension build setup
- More complex initial setup

### Option 2: External Process + IPC

**Pros:**
- Simple to implement
- No GDExtension needed
- Easy to debug

**Cons:**
- Additional latency
- More complex message passing

### Option 3: Network Proxy

**Pros:**
- Language-agnostic
- Easy testing
- Platform independent

**Cons:**
- Additional component
- Network overhead

## Recommended Approach: GDExtension + Rust

We'll create a GDExtension that wraps the HELICS C API and exposes it to GDScript.

## Implementation Steps

### Phase 1: GDExtension Setup

1. **Create GDExtension project structure**
   ```
   godot-colony-sim/
   └── helics-gdextension/
       ├── Cargo.toml
       ├── src/
       │   └── lib.rs
       └── helics_godot.gdextension
   ```

2. **Dependencies:**
   - `gdext` (Godot Rust bindings)
   - HELICS C library
   - modelica-helics-federate (for C FFI examples)

3. **Core classes to expose:**
   - `HelicsFederate` - Main federate wrapper
   - `HelicsPublication` - For publishing data
   - `HelicsSubscription` - For receiving data

### Phase 2: HELICS Wrapper Implementation

Create Rust wrapper that exposes HELICS to Godot:

```rust
// helics-gdextension/src/lib.rs

#[gdextension]
unsafe impl ExtensionLibrary for HelicsGodot {}

#[derive(GodotClass)]
#[class(base=Node)]
struct HelicsFederate {
    federate: *mut helics_sys::HelicsFederate,
    publications: HashMap<String, *mut helics_sys::HelicsPublication>,
    subscriptions: HashMap<String, *mut helics_sys::HelicsInput>,

    #[base]
    base: Base<Node>,
}

#[godot_api]
impl HelicsFederate {
    #[func]
    fn initialize(&mut self, name: String, core_type: String) {
        // Initialize HELICS federate
    }

    #[func]
    fn register_publication(&mut self, key: String, data_type: String) {
        // Register HELICS publication
    }

    #[func]
    fn register_subscription(&mut self, key: String) {
        // Register HELICS subscription
    }

    #[func]
    fn publish_double(&mut self, key: String, value: f64) {
        // Publish double value
    }

    #[func]
    fn get_double(&self, key: String) -> f64 {
        // Get subscribed double value
    }

    #[func]
    fn request_time(&mut self, time: f64) -> f64 {
        // Request time advancement
    }
}
```

### Phase 3: GDScript Integration

Create GDScript manager to handle HELICS federation:

```gdscript
# lunco-sim/scripts/core/helics_manager.gd

extends Node

# HELICS federate instance
var federate: HelicsFederate

# Signals for data updates
signal temperature_updated(value: float)
signal heater_state_updated(state: bool)

# Configuration
var federate_name = "GodotVisualization"
var broker_address = "tcp://127.0.0.1"

func _ready():
    # Create and initialize federate
    federate = HelicsFederate.new()
    federate.initialize(federate_name, "zmq")

    # Subscribe to physics data
    federate.register_subscription("thermal/temperature")
    federate.register_subscription("thermal/heater_state")

    # Publish control commands
    federate.register_publication("controls/heater", "boolean")

    # Enter execution mode
    federate.enter_execution_mode()

func _process(delta):
    # Request next time step
    var granted_time = federate.request_time(Time.get_ticks_msec() / 1000.0)

    # Read subscribed values
    var temp = federate.get_double("thermal/temperature")
    var heater = federate.get_boolean("thermal/heater_state")

    # Emit signals for UI updates
    emit_signal("temperature_updated", temp)
    emit_signal("heater_state_updated", heater)

func set_heater_command(enabled: bool):
    federate.publish_boolean("controls/heater", enabled)
```

### Phase 4: UI Integration

Update the Modelica UI to display live physics data:

```gdscript
# lunco-sim/apps/modelica-ui/scripts/ui/modelica_ui_controller.gd

var helics_manager: Node

func _ready():
    # Get HELICS manager
    helics_manager = get_node("/root/HelicsManager")

    # Connect to signals
    helics_manager.connect("temperature_updated", _on_temperature_updated)
    helics_manager.connect("heater_state_updated", _on_heater_state_updated)

func _on_temperature_updated(temp: float):
    # Update temperature display
    $TemperatureLabel.text = "Temperature: %.2f K" % temp

func _on_heater_state_updated(state: bool):
    # Update heater indicator
    $HeaterIndicator.modulate = Color.RED if state else Color.GRAY

func _on_heater_button_pressed():
    # Toggle heater
    helics_manager.set_heater_command(!current_heater_state)
```

## Directory Structure

```
v-ics.le/
├── modelica-helics-federate/      # Physics federate (AGPL-3.0)
│   ├── src/
│   ├── configs/
│   └── Cargo.toml
│
├── godot-colony-sim/
│   ├── helics-gdextension/        # NEW: GDExtension for HELICS
│   │   ├── Cargo.toml
│   │   ├── src/
│   │   │   └── lib.rs
│   │   └── helics_godot.gdextension
│   │
│   ├── lunco-sim/
│   │   ├── scripts/
│   │   │   ├── core/
│   │   │   │   ├── helics_manager.gd   # NEW: HELICS manager
│   │   │   │   └── modelica_simulator.gd
│   │   │   └── ui/
│   │   │       └── modelica_ui_controller.gd  # UPDATED
│   │   └── scenes/
│   │
│   └── helics-federates/          # DEPRECATED - to be removed
│       └── ...
```

## Configuration

### HELICS Federation Config

```json
{
  "name": "GodotVisualization",
  "core_type": "zmq",
  "period": 0.1,

  "subscriptions": [
    {"key": "thermal/temperature", "type": "double"},
    {"key": "thermal/heater_state", "type": "boolean"}
  ],

  "publications": [
    {"key": "controls/heater", "type": "boolean"}
  ]
}
```

## Testing Plan

### Test 1: HELICS Connection
```bash
# Terminal 1: Start broker
helics_broker -f 2 --loglevel=summary

# Terminal 2: Start physics federate
cd modelica-helics-federate
./run_federate.sh

# Terminal 3: Start Godot
cd godot-colony-sim/lunco-sim
godot --path . &
```

### Test 2: Data Flow
1. Verify Godot receives temperature updates
2. Verify heater state updates
3. Test heater control from Godot UI
4. Verify physics federate responds to commands

### Test 3: Synchronization
1. Verify time synchronization between federates
2. Test at different time steps
3. Verify data consistency

## Alternative: Quick Prototype with WebSocket

For rapid prototyping, create a simple WebSocket bridge:

```rust
// helics-websocket-bridge/src/main.rs

use tokio_tungstenite::tungstenite::Message;

async fn helics_to_websocket() {
    // Subscribe to HELICS
    // Forward to WebSocket
}

async fn websocket_to_helics() {
    // Receive from WebSocket
    // Publish to HELICS
}
```

Then in Godot:
```gdscript
var ws = WebSocketPeer.new()
ws.connect_to_url("ws://localhost:9001")

func _process(_delta):
    ws.poll()
    if ws.get_available_packet_count() > 0:
        var data = JSON.parse_string(ws.get_packet().get_string_from_utf8())
        # Handle data
```

## License Considerations

- GDExtension code: MIT (same as Godot)
- HELICS: BSD-3-Clause
- No AGPL contamination (network boundary)

## Next Steps

1. Set up GDExtension project
2. Implement basic HELICS wrapper
3. Test with simple pub/sub
4. Integrate into Godot UI
5. Test full federation

## Resources

- [Godot Rust GDExtension](https://godot-rust.github.io/)
- [HELICS C API Reference](https://docs.helics.org/)
- [modelica-helics-federate](../modelica-helics-federate) - Reference implementation
