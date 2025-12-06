# Godot HELICS Integration Guide

## Overview

Connect Godot visualization to the V-ICS.le HELICS federation to receive real-time physics data from `modelica-helics-federate`.

## Strategy

We'll use the existing `godot-modelica-rust-integration` submodule to add HELICS bindings to Godot via GDExtension (Rust).

## Cleaned Up Architecture

```
v-ics.le/
├── modelica-helics-federate/           # Physics federate (AGPL-3.0)
│   ├── Compiles ✅
│   ├── HELICS interface ✅
│   └── Modbus TCP server ✅
│
└── godot-colony-sim/
    ├── godot-modelica-rust-integration/  # Submodule for Godot+Rust
    │   ├── modelica-rust-gdext/        # Full gdext library
    │   └── ADD: helics-bindings/       # NEW: HELICS GDExtension
    │
    └── lunco-sim/
        ├── scripts/core/
        │   └── ADD: helics_manager.gd  # NEW: GDScript HELICS manager
        └── apps/modelica-ui/
            └── scripts/ui/
                └── UPDATE: modelica_ui_controller.gd  # Display physics data
```

## Implementation Plan

### Phase 1: Add HELICS Bindings to godot-modelica-rust-integration

Create a new GDExtension within the submodule:

```
godot-modelica-rust-integration/
├── modelica-rust-gdext/    # Existing gdext library
└── helics-bindings/         # NEW
    ├── Cargo.toml
    ├── build.rs
    ├── src/
    │   └── lib.rs
    └── helics.gdextension
```

### Phase 2: Implement HELICS Wrapper

**File:** `godot-modelica-rust-integration/helics-bindings/src/lib.rs`

```rust
use godot::preload::*;

// HELICS C FFI (reuse from modelica-helics-federate)
mod helics_c;

#[derive(GodotClass)]
#[class(base=Node)]
pub struct HelicsFederate {
    // HELICS federate pointer
    federate: Option<*mut helics_c::HelicsFederate>,

    // Publications and subscriptions
    pubs: HashMap<String, *mut helics_c::HelicsPublication>,
    subs: HashMap<String, *mut helics_c::HelicsInput>,

    #[base]
    base: Base<Node>,
}

#[godot_api]
impl INode for HelicsFederate {
    fn init(base: Base<Node>) -> Self {
        Self {
            federate: None,
            pubs: HashMap::new(),
            subs: HashMap::new(),
            base,
        }
    }
}

#[godot_api]
impl HelicsFederate {
    #[func]
    fn create_federate(&mut self, name: GString, core_type: GString) -> bool {
        // Initialize HELICS federate using C API
        // (Same pattern as modelica-helics-federate/src/helics_c.rs)
        true
    }

    #[func]
    fn register_publication(&mut self, key: GString, data_type: GString) {
        // Register HELICS publication
    }

    #[func]
    fn register_subscription(&mut self, key: GString) {
        // Register HELICS subscription
    }

    #[func]
    fn enter_execution_mode(&mut self) -> bool {
        // Enter HELICS execution mode
        true
    }

    #[func]
    fn request_time(&mut self, requested_time: f64) -> f64 {
        // Request time advancement from HELICS
        0.0
    }

    #[func]
    fn publish_double(&mut self, key: GString, value: f64) {
        // Publish double value to HELICS
    }

    #[func]
    fn get_double(&self, key: GString) -> f64 {
        // Get subscribed double value from HELICS
        0.0
    }

    #[func]
    fn publish_bool(&mut self, key: GString, value: bool) {
        // Publish boolean value to HELICS
    }

    #[func]
    fn get_bool(&self, key: GString) -> bool {
        // Get subscribed boolean value from HELICS
        false
    }
}
```

### Phase 3: Create GDScript Manager

**File:** `lunco-sim/scripts/core/helics_manager.gd`

```gdscript
extends Node

# HELICS federate instance (from GDExtension)
var federate: HelicsFederate

# Signals for UI updates
signal temperature_updated(temp: float)
signal heater_state_updated(state: bool)

# Current state
var current_temperature: float = 0.0
var current_heater_state: bool = false
var federation_active: bool = false

func _ready():
    # Create HELICS federate
    federate = HelicsFederate.new()

    # Initialize federate
    if not federate.create_federate("GodotVisualization", "zmq"):
        push_error("Failed to create HELICS federate")
        return

    # Register subscriptions (receive from physics)
    federate.register_subscription("thermal/temperature")
    federate.register_subscription("thermal/heater_state")

    # Register publications (send to physics)
    federate.register_publication("controls/heater", "boolean")

    # Enter execution mode
    if federate.enter_execution_mode():
        federation_active = true
        print("HELICS federation active")
    else:
        push_error("Failed to enter HELICS execution mode")

func _process(delta):
    if not federation_active:
        return

    # Request next time step
    var current_time = Time.get_ticks_msec() / 1000.0
    var granted_time = federate.request_time(current_time)

    # Read subscribed values
    var new_temp = federate.get_double("thermal/temperature")
    var new_heater = federate.get_bool("thermal/heater_state")

    # Update state and emit signals if changed
    if abs(new_temp - current_temperature) > 0.01:
        current_temperature = new_temp
        emit_signal("temperature_updated", new_temp)

    if new_heater != current_heater_state:
        current_heater_state = new_heater
        emit_signal("heater_state_updated", new_heater)

func set_heater_command(enabled: bool):
    if federation_active:
        federate.publish_bool("controls/heater", enabled)

func get_temperature() -> float:
    return current_temperature

func get_heater_state() -> bool:
    return current_heater_state
```

### Phase 4: Update Godot UI

**File:** `lunco-sim/apps/modelica-ui/scripts/ui/modelica_ui_controller.gd`

Add HELICS connection:

```gdscript
# Get reference to HELICS manager
@onready var helics_manager = $"/root/HelicsManager"

func _ready():
    # ... existing code ...

    # Connect to HELICS signals
    if helics_manager:
        helics_manager.connect("temperature_updated", _on_helics_temperature_updated)
        helics_manager.connect("heater_state_updated", _on_helics_heater_updated)

func _on_helics_temperature_updated(temp: float):
    # Update temperature display
    var temp_celsius = temp - 273.15
    $TemperatureLabel.text = "Temperature: %.2f°C (%.2f K)" % [temp_celsius, temp]

func _on_helics_heater_updated(state: bool):
    # Update heater indicator
    $HeaterIndicator.modulate = Color.RED if state else Color.GRAY
    $HeaterLabel.text = "Heater: " + ("ON" if state else "OFF")

func _on_heater_toggle_pressed():
    # Toggle heater via HELICS
    var new_state = not helics_manager.get_heater_state()
    helics_manager.set_heater_command(new_state)
```

## Testing Procedure

### Terminal 1: Start HELICS Broker
```bash
helics_broker -f 2 --loglevel=summary
```

### Terminal 2: Start Physics Federate
```bash
cd modelica-helics-federate
export HELICS_DIR=/usr/local
export DYLD_LIBRARY_PATH=/Applications/OpenModelica/build_cmake/install_cmake/lib/omc:$DYLD_LIBRARY_PATH
cargo run --release
```

### Terminal 3: Start Godot
```bash
cd godot-colony-sim/lunco-sim
godot --path . &
```

## Expected Data Flow

```
Physics Federate                 Godot Visualization
================                 ===================

1. Temperature = 250.0 K  ───►  Display: -23.15°C
2. Heater State = OFF     ───►  Indicator: GRAY

User clicks heater button

3.                        ◄───  Send: heater=TRUE
4. Activates heater
5. Temperature = 251.0 K  ───►  Display: -22.15°C
6. Heater State = ON      ───►  Indicator: RED
```

## Build Instructions

### 1. Build HELICS Bindings

```bash
cd godot-colony-sim/godot-modelica-rust-integration/helics-bindings
export HELICS_DIR=/usr/local
cargo build --release
```

### 2. Copy to Godot Project

```bash
# Copy the built library
cp target/release/libhelics_bindings.dylib \
   ../../lunco-sim/bin/libhelics_bindings.macos.framework/libhelics_bindings.macos.dylib

# Copy the .gdextension file
cp helics.gdextension ../../lunco-sim/
```

### 3. Configure Godot

In Godot, the `.gdextension` file should be automatically detected.

## Configuration Files

### helics.gdextension

```ini
[configuration]
entry_symbol = "gdext_rust_init"
compatibility_minimum = 4.1
reloadable = true

[libraries]
macos.debug = "res://bin/libhelics_bindings.macos.framework/libhelics_bindings.macos.dylib"
macos.release = "res://bin/libhelics_bindings.macos.framework/libhelics_bindings.macos.dylib"
windows.debug.x86_64 = "res://bin/helics_bindings.dll"
windows.release.x86_64 = "res://bin/helics_bindings.dll"
linux.debug.x86_64 = "res://bin/libhelics_bindings.so"
linux.release.x86_64 = "res://bin/libhelics_bindings.so"
```

## Alternative: Simple HTTP/WebSocket Proxy

If GDExtension proves too complex, create a simple proxy:

```rust
// Simple HELICS HTTP proxy
use axum::{Json, Router};
use serde_json::Value;

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/helics/temperature", axum::routing::get(get_temperature))
        .route("/helics/heater", axum::routing::post(set_heater));

    axum::Server::bind(&"0.0.0.0:3000".parse().unwrap())
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

Then in Godot:
```gdscript
var http = HTTPRequest.new()

func _process(_delta):
    http.request("http://localhost:3000/helics/temperature")

func _on_request_completed(result, response_code, headers, body):
    var json = JSON.parse_string(body.get_string_from_utf8())
    emit_signal("temperature_updated", json.temperature)
```

## Next Steps

1. Set up helics-bindings in godot-modelica-rust-integration
2. Implement basic HELICS C FFI wrapper
3. Build and test GDExtension
4. Create helics_manager.gd
5. Integrate into Godot UI
6. Test full federation

## License

- helics-bindings: MIT (same as Godot)
- No AGPL contamination (network boundary via HELICS)
